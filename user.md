# 使用者與群組

## 基本概念

**使用者分三種**

| 類型 | UID | 說明 |
|------|------|------|
| 超級使用者 | `0` | root，擁有全部權限 |
| 系統使用者 | `1–999` | 給服務程式用（nginx、mysql），不能登入 |
| 一般使用者 | `1000` 以上 | 一般人用的帳號 |

- 系統內部用 **UID** 辨認使用者、用 **GID** 辨認群組，名稱只是方便人看
- 每個使用者**一定有 1 個主要群組**，可以另外加入多個**補充群組**
- 建立使用者時，預設會建立一個**同名群組**當作主要群組
- 自己建立的新檔案，群組會是自己的**主要群組**

**查看使用者資訊**

```bash
id                 # 查看自己
id jumbo           # 查看別人
# uid=1000(jumbo) gid=1000(jumbo) groups=1000(jumbo),10(wheel)
#     UID            主要群組          所有群組
```

相關檔案格式見 [filesystem.md](filesystem.md)：`/etc/passwd`、`/etc/group`、`/etc/shadow`

## 管理使用者

**建立使用者**

```bash
sudo useradd jumbo         # 建立使用者（同時建立家目錄和同名群組）
sudo passwd jumbo          # 設定密碼，設定後才能登入
passwd                     # 修改自己的密碼
```

- `useradd` 的選項大多和 `usermod` 一樣，例如 `-c`、`-g`、`-G`、`-d`、`-s`
- 預設值（UID 範圍、密碼期限）寫在 `/etc/login.defs`，修改後只影響**之後建立**的使用者
- 一般使用者設密碼至少 8 個字元，且不能是字典單字；root 設定時可以無視限制，只會顯示警告

**刪除使用者**

```bash
sudo userdel jumbo         # 刪除使用者，保留家目錄
sudo userdel -r jumbo      # 連同家目錄一起刪除（建議用這個）
```

> **為什麼建議加 `-r`？**
> 沒加 `-r` 時，舊檔案還會留著，擁有者會顯示成一個數字（原本的 UID）。
> 之後新建的使用者可能會拿到**同一個 UID**，就會直接擁有舊使用者的所有檔案。
> 找出沒有擁有者的檔案：`sudo find / -nouser -o -nogroup`

## 修改使用者 usermod

改的大多是 `/etc/passwd` 裡的欄位：

```
jumbo:x:1000:1000:Jumbo Chen:/home/jumbo:/bin/bash
              ↑     ↑         ↑           ↑
             -g     -c        -d          -s
```

| 選項 | 改什麼 | 記法 |
|------|------|------|
| `-c` | 說明欄（真實姓名、備註） | **c**omment |
| `-g` | 主要群組（只有一個） | 小寫 g，只有一個 |
| `-G` | 補充群組（可多個，逗號分隔不能有空格） | 大寫 G，可以很多個 |
| `-a` | 搭配 `-G`，附加而不是取代 | **a**ppend |
| `-d` | 家目錄路徑 | **d**irectory |
| `-m` | 搬移家目錄內容（搭配 `-d`） | **m**ove |
| `-s` | 登入 shell | **s**hell |
| `-L` / `-U` | 鎖定 / 解鎖帳號 | **L**ock / **U**nlock |
| `-e` | 帳號到期日（`YYYY-MM-DD`） | **e**xpire |

**範例**

```bash
sudo usermod -c "Jumbo Chen" jumbo          # 改說明欄
sudo usermod -g admins jumbo                # 改主要群組（群組要先存在）
sudo usermod -aG admins,docker jumbo        # 加入補充群組
sudo usermod -d /data/jumbo -m jumbo        # 改家目錄並搬移檔案
sudo usermod -s /sbin/nologin jumbo         # 禁止登入
sudo usermod -L jumbo                       # 鎖定帳號
sudo usermod -U jumbo                       # 解鎖帳號
sudo usermod -L -e 2026-10-17 jumbo         # 鎖定並設定帳號到期（離職常用）
```

**`-G` 有沒有加 `-a` 的差別**

| 原本補充群組 | 指令 | 結果 |
|------|------|------|
| `wheel, docker` | `usermod -G admins jumbo` | `admins`（原本的被清掉） |
| `wheel, docker` | `usermod -aG admins jumbo` | `wheel, docker, admins` |

**注意**

- `-d` 只改路徑設定，**檔案不會跟著搬**，要搭配 `-m`
- `-s /sbin/nologin` 常用在服務帳號（nginx、mysql），只跑程式不讓人登入；可用的 shell 列在 `/etc/shells`
- `-L` 是在 `/etc/shadow` 密碼前加 `!`，只擋**密碼登入**，SSH 金鑰還是可能登得進來；要完全擋住可搭配 `-s /sbin/nologin`
- 適合離職或暫時停用：帳號和檔案都還在，之後可以用 `-U` 恢復；如果有設 `-e` 到期日，也要一起改掉

## 群組

**建立群組**

```bash
groupadd xxx
groupadd -g 10000 xxx      # 指定 GID
groupadd -r xxx            # 建立系統群組（GID 在 1000 以下）
```

**修改群組**

```bash
groupmod -n 新名稱 舊名稱     # 改名稱
groupmod -g 20000 xxx        # 改 GID
```

**刪除群組**

```bash
groupdel xxx
```

- 如果是某個使用者的**主要群組**，就刪不掉

**將使用者加入群組**

```bash
sudo usermod -aG 群組名稱 使用者名稱
```

```bash
sudo gpasswd -a 使用者名稱 群組名稱
```

- 加入群組後要**重新登入**才會生效

**查看群組**

```bash
getent group 群組名稱    #查看群組內有哪些使用者
groups 使用者名稱        #查看使用者在哪些群組裡
```

## 密碼期限 chage

```bash
sudo chage -l jumbo                        # 查看密碼期限設定
sudo chage -m 0 -M 90 -W 7 -I 14 jumbo     # 設定密碼期限
sudo chage -d 0 jumbo                      # 強制下次登入時修改密碼
sudo chage -E 2026-12-31 jumbo             # 設定帳號到期日
```

```
上次修改 ──[-m 最短]────────[-W 警告]──→ 密碼到期 ──[-I 寬限]──→ 帳號停用
          └──────────── -M 最長 ────────────┘
```

| 選項 | 意思 | 記法 |
|------|------|------|
| `-m` | 修改密碼後，至少要隔幾天才能再改 | **m**in（小寫） |
| `-M` | 密碼最多可以用幾天 | **M**ax（大寫） |
| `-W` | 到期前幾天開始警告 | **W**arn |
| `-I` | 過期後還有幾天寬限期可以登入（要求改密碼） | **I**nactive |
| `-E` | 帳號到期日（`YYYY-MM-DD`） | **E**xpire |
| `-d` | 上次修改密碼的日期，設 `0` 等於讓密碼立刻過期 | **d**ate |
| `-l` | 列出目前設定 | **l**ist |

- 新使用者的預設密碼期限在 `/etc/login.defs`：`PASS_MAX_DAYS`、`PASS_MIN_DAYS`、`PASS_WARN_AGE`
- 計算日期可以用 `date -d "+45 days"`

## 切換使用者

```bash
su - 使用者名稱    # 切換到其他使用者
su -              # 切換到 root
exit              # 切回原本的使用者
```

- 有加 `-`：連同環境變數、家目錄一起切換（建議用這個）
- 沒加 `-`：只換身分，還留在原本的目錄和環境
- `su` 要輸入的是**目標使用者的密碼**；root 切換到其他人則不用密碼

## 超級執行 sudo

**以root權限執行**

```bash
sudo vim /etc/hosts
```

- `sudo` 輸入的是**自己的密碼**，不需要知道 root 密碼
- 建議平常用一般帳號登入，需要時才用 `sudo`，不要直接用 root 登入
- 用 `sudo` 執行的指令都會記錄在 `/var/log/secure`；沒有權限的人使用 `sudo` 也會被記錄

**取得 root shell**

```bash
sudo su -          # 透過 sudo 切換成 root（不用 root 密碼）
sudo -i            # 效果類似，但 PATH 等環境變數有些細節不同
```

| 指令 | 輸入誰的密碼 |
|------|------|
| `su -` | root 的密碼 |
| `sudo su -` / `sudo -i` | 自己的密碼 |

### sudo權限

在 RHEL 7 / 8 中，**`wheel` 群組**的成員預設就能使用 `sudo`：

```bash
sudo usermod -aG wheel jumbo
```

其他使用者或群組要自己設定：在路徑 `/etc/sudoers.d` 新增檔案（建議用 `sudo visudo -f /etc/sudoers.d/檔名` 編輯，會檢查語法）

**使用者**

```bash
使用者名稱 ALL=(ALL) ALL
```

**群組**

```bash
%群組名稱 ALL=(ALL) ALL  # 若無此群組需先建立群組
```

**使用sudo無需打密碼**
```bash
XXX ALL=(ALL) NOPASSWD:ALL
```

**格式說明**

```
%wheel   ALL=(ALL)   ALL
  │       │    │      │
 誰     哪台主機 以誰的身分 可以執行哪些指令
```

- 開頭有 `%` 代表群組，沒有就是使用者
