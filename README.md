# Informix 14.10 測試靶機 — 連線 / 操作速查筆記

> 容器名稱：`ifx`　映像檔：`icr.io/informix/informix-developer-database:14.10.FC9W1DE`
> 目前 kali 使用者不在 docker 群組，所有 `docker` 指令前面都要加 `sudo`
> （一勞永逸：`sudo usermod -aG docker $USER` 後重新登入，即可省略 sudo）

---

## 1. 容器生命週期

| 動作 | 指令 |
|------|------|
| 啟動（背景） | `sudo docker-compose up -d` |
| 看即時日誌 | `sudo docker-compose logs -f` |
| 停止（保留資料） | `sudo docker-compose down` |
| 重建（清除資料 volume） | `sudo docker-compose down -v` |
| 查看容器狀態 | `sudo docker container ls` |
| 查看健康狀態 | `sudo docker inspect --format '{{.State.Health.Status}}' ifx` |

> 狀態要是 `Up ... (healthy)` 才代表 Informix server 已經初始化完成，可以連線。

---

## 2. 連線資訊一覽

| 項目 | 值 |
|------|------|
| 主機 | `127.0.0.1` |
| SQLI 主連線埠（onsoctcp） | `9088` |
| DRDA 埠 | `9089` |
| Wire Listener（MongoDB 相容） | `27017` / `27018` |
| Wire Listener（REST） | `27883` |
| Server 名稱（INFORMIXSERVER） | `informix`（進容器用 `echo $INFORMIXSERVER` 確認） |
| 使用者 | `informix` |
| 密碼 | compose 的 `INFORMIX_PASSWORD` 值 |
| 練習資料庫 | `testdb`（**developer 版需手動建立**，見第 3 節） |

> ⚠️ 目前 compose 裡的 `INFORMIX_PASSWORD` / `HQADMIN_PASSWORD` 還是佔位字串 `<strong-password>`，
> 連線前請改成實際密碼並 `down -v` 後重建容器，否則初始化的密碼就是那串佔位字。
>
> ⚠️ **映像檔差異**：`DB_INIT` 只有 **innovator-c** 映像檔會自動建庫；目前使用的
> **developer-database** 映像檔不吃這個變數，啟動後 `testdb` 不存在，需手動建立（第 3 節）。
> compose 裡的 `DB_INIT: testdb` 留著無害，但對 developer 版沒有作用。

---

## 3. 建立 testdb + 進容器用 dbaccess（內建互動介面）

### 3-1. 第一次啟動：手動建立 testdb（developer 版必做）

developer 映像檔不會自動建庫，首次 `up -d` 後先建一次（之後只要不 `down -v`，testdb 都在）：

```bash
sudo docker exec -it ifx bash

# 建立練習資料庫（WITH LOG 啟用交易紀錄，支援 BEGIN WORK / COMMIT）
echo "CREATE DATABASE testdb WITH LOG;" | dbaccess -
```

> 不需要交易就用 `CREATE DATABASE testdb;`（省去交易紀錄負擔）。
> 一步到位版（不先開 bash）：
> ```bash
> echo "CREATE DATABASE testdb WITH LOG;" | sudo docker exec -i ifx dbaccess -
> ```

確認建好了：

```bash
echo "SELECT name FROM sysdatabases;" | dbaccess sysmaster
```

> 若遇到 `329: Database not found`，就是 testdb 還沒建，回到上一步建立即可。

### 3-2. 進入 dbaccess 互動介面

```bash
# 進容器 shell（會自動載入 INFORMIXDIR / PATH 等環境變數）
sudo docker exec -it ifx bash

# 開啟練習資料庫，進入互動選單（結尾的 - 代表互動模式）
dbaccess testdb -
```

一步到位（不先開 bash）：

```bash
sudo docker exec -it ifx dbaccess testdb -
```

dbaccess 選單操作：
- 方向鍵移動、`Enter` 選取
- **Query-language** → 直接打 SQL
- `New` 新增 SQL、`Run` 執行、`Save` 存檔
- 離開：選 **Exit** 或按 `q`

快速跑一句 SQL（不進選單）：

```bash
echo "SELECT FIRST 5 * FROM systables;" | sudo docker exec -i ifx dbaccess testdb
```

---

## 4. 從宿主機用外部工具連線（DBeaver 等）

DBeaver 新增連線 → 選 **Informix**，填入：

| 欄位 | 值 |
|------|------|
| Host | `127.0.0.1` |
| Port | `9088` |
| Database | `testdb` |
| Server | `informix` |
| User | `informix` |
| Password | 你的 `INFORMIX_PASSWORD` |

JDBC URL 參考格式：

```
jdbc:informix-sqli://127.0.0.1:9088/testdb:INFORMIXSERVER=informix
```

---

## 5. 常用 server 檢查指令（在容器內）

| 指令 | 用途 |
|------|------|
| `onstat -` | 查 server 是否 On-Line |
| `onstat -g dis` | 列出執行個體資訊 |
| `onstat -g ses` | 查看目前 session |
| `onstat -d` | 查看 dbspace / chunk |
| `echo $INFORMIXSERVER` | 確認 server 名稱 |
| `onstat -g sql` | 查看正在執行的 SQL |

若以 root 進容器導致環境變數沒載入，先切換使用者：

```bash
su - informix
```

---

## 6. 常用 SQL 片段（dbaccess / DBeaver 皆可）

```sql
-- 列出所有使用者資料表（排除系統表）
SELECT tabname FROM systables WHERE tabid > 99;

-- 查目前連到哪個資料庫、哪個使用者
SELECT DBINFO('dbname') AS db, USER AS usr FROM systables WHERE tabid = 1;

-- 列出所有資料庫
SELECT name FROM sysdatabases;

-- 建一張測試表
CREATE TABLE t_demo (id SERIAL PRIMARY KEY, name VARCHAR(50));
INSERT INTO t_demo (name) VALUES ('hello');
SELECT * FROM t_demo;
```

---

## 7. Wire Listener（MongoDB 相容面）快速驗證

compose 有開 `27017`，是 `CVE-2026-13476` 的攻擊面。測試連通性：

```bash
# 從宿主機確認埠有在聽
sudo ss -ltnp | grep -E '9088|27017'

# 若本機有 mongosh，可試連（視容器內 Wire Listener 設定而定）
mongosh "mongodb://127.0.0.1:27017"
```

---

## 備註
- 埠全部綁在 `127.0.0.1`，只有本機連得到，不會對外網曝露（適合靶機測試）。
- `DB_LOCALE: zh_tw.utf8` 已開啟中文語系，中文欄位值不會亂碼。
- 資料存在 named volume `ifx_data`，`down`（不加 `-v`）重建不會遺失資料。

