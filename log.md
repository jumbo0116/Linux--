# 日誌（Log）

日誌是系統的「日記」，記錄誰登入了、哪個服務啟動失敗、有什麼錯誤。出問題時，第一件事通常就是看日誌。

## 日誌的架構

RHEL 有兩個負責記錄日誌的服務：

```
各種程式、系統核心、服務……
          ↓ 產生訊息
   systemd-journald          → 存成 journal（二進位檔，用 journalctl 查看）
          ↓ 轉交 syslog 訊息
       rsyslog               → 分類後寫成文字檔，放在 /var/log/
```

| 服務 | 存在哪 | 怎麼看 |
|------|------|------|
| `systemd-journald` | `/run/log/journal/` 或 `/var/log/journal/` | `journalctl` |
| `rsyslog` | `/var/log/` 底下的文字檔 | `cat`、`less`、`tail` |

**`/var/log/` 常見的日誌檔**

| 檔案 | 記錄什麼 |
|------|------|
| `/var/log/messages` | **大部分的系統訊息**（最常看） |
| `/var/log/secure` | **登入、`sudo`、SSH** 等安全相關訊息 |
| `/var/log/maillog` | 郵件 |
| `/var/log/cron` | 排程工作 |
| `/var/log/boot.log` | 開機過程 |

- 這些檔案大多要 root 才能讀，用 `sudo`
- 有些程式不用 syslog，而是自己在 `/var/log/` 底下建子目錄存放（例如 Apache 網頁伺服器）

## syslog 日誌檔

### 優先等級

每則訊息都有嚴重程度，**數字越小越嚴重**：

| 代碼 | 名稱 | 意思 |
|------|------|------|
| 0 | `emerg` | 系統不能用了 |
| 1 | `alert` | 必須立刻處理 |
| 2 | `crit` | 嚴重 |
| 3 | `err` | **錯誤** |
| 4 | `warning` | **警告** |
| 5 | `notice` | 正常但重要 |
| 6 | `info` | 一般資訊 |
| 7 | `debug` | 除錯用 |

### 一行日誌怎麼看

```
Feb 11 20:11:48  localhost  sshd[1433]:  Failed password for student from 172.25.0.10 ...
└─── 時間 ───┘   └ 主機 ┘   └ 程式[PID] ┘  └──────────────── 訊息內容 ────────────────┘
```

- 舊的訊息在檔案**開頭**，新的在**結尾**

### 常用操作

```bash
sudo less /var/log/secure         # 分頁查看（/關鍵字 搜尋，G 跳到最後）
sudo tail -n 20 /var/log/messages # 看最後 20 行
sudo tail -f /var/log/secure      # 即時監看，有新訊息就會跑出來（Ctrl+C 離開）
logger "測試訊息"                  # 自己寫一筆日誌（測試用）
```

**例子：監看登入紀錄**

```bash
# 視窗 1
sudo tail -f /var/log/secure

# 視窗 2 用 SSH 登入，視窗 1 會即時出現：
# sshd[2712]: Accepted password for jumbo from 192.168.151.1 port 56801 ssh2
```

### rsyslog 的設定

設定檔：`/etc/rsyslog.conf`、`/etc/rsyslog.d/*.conf`

```
authpriv.*                          /var/log/secure
│        │                          │
類型     優先等級（* = 全部）         要寫到哪個檔案
```

### logrotate（日誌輪替）

日誌會一直變大，所以系統會**定期把舊的日誌改名封存**，避免塞滿硬碟：

```
/var/log/messages            ← 新的
/var/log/messages-20261102   ← 舊的，用日期命名
```

- 通常每週輪替一次，大約四週後刪除最舊的

## journalctl（重要）

```bash
sudo journalctl                       # 全部（用 less 顯示，按 q 離開）
sudo journalctl -n 5                  # 最後 5 筆
sudo journalctl -f                    # 即時監看（Ctrl+C 離開）
sudo journalctl -p err                # 只看 err 以上（錯誤、嚴重……）
sudo journalctl -u sshd               # 只看某個服務
sudo journalctl --since today         # 今天的
sudo journalctl --since "-1 hour"     # 最近一小時的
sudo journalctl --since "2026-11-01 09:00:00" --until "2026-11-01 12:00:00"
sudo journalctl -b                    # 這次開機以來的
sudo journalctl -b -1                 # 上一次開機的（要先設定永久保存）
```

| 選項 | 意思 | 記法 |
|------|------|------|
| `-n 數字` | 最後幾筆 | **n**umber |
| `-f` | 即時監看 | **f**ollow |
| `-p 等級` | 這個等級以上 | **p**riority |
| `-u 服務` | 某個服務 | **u**nit |
| `--since` / `--until` | 時間範圍 | |
| `-b` | 某次開機 | **b**oot |

- 選項可以組合：`journalctl -u sshd -p err --since today`
- 錯誤會顯示成**紅色**、警告顯示成**粗體**
- 一般使用者只能看到部分訊息，**用 `sudo` 才能看到全部**
- `systemctl status 服務` 最下面那幾行，就是從 journal 抓出來的

**時間格式**：`"YYYY-MM-DD hh:mm:ss"`，要用雙引號包起來；也可以用 `today`、`yesterday`、`"-1 hour"`

### 詳細欄位

```bash
sudo journalctl -o verbose                          # 顯示每筆日誌的所有欄位
sudo journalctl _SYSTEMD_UNIT=sshd.service _PID=1182  # 用欄位篩選
```

| 欄位 | 意思 |
|------|------|
| `_COMM` | 指令名稱 |
| `_EXE` | 程式的路徑 |
| `_PID` | 進程編號 |
| `_UID` | 執行者的 UID |
| `_SYSTEMD_UNIT` | 是哪個 systemd 單元 |

## 日誌永久保存

**預設的 journal 存在 `/run/log/journal/`，重開機就會清空**（`/run` 是暫時的）。

**設定永久保存**：編輯 `/etc/systemd/journald.conf`

```
[Journal]
Storage=persistent
```

```bash
sudo systemctl restart systemd-journald
ls /var/log/journal/                    # 確認目錄有建立
```

| `Storage=` | 存在哪 | 重開機後 |
|------|------|------|
| `persistent` | `/var/log/journal/` | 還在 |
| `volatile` | `/run/log/journal/` | 消失 |
| `auto`（預設） | `/var/log/journal/` 存在就用它，不存在就用 `/run` | 看情況 |

- journal 有大小上限，不會超過所在檔案系統的 10%
- 永久保存後，才能用 `journalctl -b -1` 查看上次開機的日誌（排查當機原因很有用）

## 時間同步

日誌的時間要準確，才能判斷事情發生的先後順序。

### timedatectl

```bash
timedatectl                                   # 查看時間、時區、有沒有自動同步
timedatectl list-timezones                    # 列出所有時區
sudo timedatectl set-timezone Asia/Taipei     # 設定時區
sudo timedatectl set-time "2026-11-09 09:00:00"  # 手動設定時間
sudo timedatectl set-ntp true                 # 開啟自動校時（會啟動 chronyd）
```

```
               Local time: Mon 2026-11-09 16:10:29 CST     ← 本地時間
           Universal time: Mon 2026-11-09 08:10:29 UTC     ← 世界標準時間
                Time zone: Asia/Taipei (CST, +0800)        ← 時區
System clock synchronized: yes                             ← 有沒有同步成功
              NTP service: active                          ← 自動校時有沒有開
```

- `tzselect`：用問答的方式找出正確的時區名稱（只顯示，不會修改設定）

### chronyd

負責**自動校時**的服務，會連到 **NTP 伺服器**（網路上的標準時間來源）校正時間。

**設定檔**：`/etc/chrony.conf`

```
server classroom.example.com iburst
```

- `server`：NTP 伺服器的位址，可以寫好幾行
- `iburst`：剛啟動時快速同步，建議加上

```bash
sudo systemctl restart chronyd     # 改完設定要重新啟動
chronyc sources -v                 # 確認同步狀態
```

```
MS Name/IP address         Stratum Poll Reach LastRx Last sample
^* classroom.example.com         8    6    17    23   -497ns[-7000ns] +/- 956us
↑↑
│└ * = 目前正在同步的來源
└─ ^ = server
```

- 看到 **`^*`** 就代表正在跟這台同步

## 常用情境速查

| 想做的事 | 指令 |
|------|------|
| 查服務為什麼啟動失敗 | `sudo journalctl -u 服務名稱` |
| 查最近的錯誤 | `sudo journalctl -p err` |
| 即時監看 | `sudo journalctl -f` 或 `sudo tail -f /var/log/messages` |
| 查登入和 sudo 紀錄 | `sudo less /var/log/secure` |
| 查上次開機發生什麼事 | `sudo journalctl -b -1` |
| 日誌永久保存 | `journald.conf` 設定 `Storage=persistent` |
| 設定時區 | `sudo timedatectl set-timezone Asia/Taipei` |
| 確認有沒有自動校時 | `timedatectl`、`chronyc sources` |

> **考試重點**：日誌永久保存、設定 NTP 伺服器（`chrony.conf`）常出現。改完設定記得 `restart` 服務。
