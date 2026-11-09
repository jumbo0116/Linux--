# SSH

## 基本連線

從 Windows 的 CMD / PowerShell 連到 Linux：

```bash
ssh 使用者名稱@IP
ssh 使用者名稱@IP -p 2222   # SSH 不是預設 22 port 時
```

- 第一次連線會問是否信任主機，輸入 `yes`
- 輸入密碼時畫面不會顯示字元（正常）
- `exit` 離開連線

**其他寫法**

```bash
ssh remotehost                    # 沒寫使用者，就用跟本機相同的帳號名稱
ssh user02@remotehost hostname    # 不登入，只在遠端執行一個指令，結果顯示在本機
```

**查看誰登入了**

```bash
w
# USER    TTY    FROM            LOGIN@  IDLE ...
# user02  pts/0  172.25.250.10   16:13   7:30
```

- `FROM`：從哪個 IP 連進來

## ssh 和 sshd

| | `ssh` | `sshd` |
|------|------|------|
| 角色 | 用戶端（client） | 伺服器端（server） |
| 做什麼 | **連出去** | 在背景**等別人連進來** |
| 在哪台執行 | 你操作的那台（例如 Windows） | 被連的那台（例如 servera） |
| 設定檔 | `/etc/ssh/ssh_config` | `/etc/ssh/sshd_config` |

- 兩邊都要有才能連線：對方的 `sshd` 要在執行中
- `sshd` 是服務，用 `systemctl` 管理，見 [service.md](service.md)
- 停止 `sshd` 只會停掉「接受新連線」的主程式，**已經連上的連線不會斷**；但一斷線就連不回來

## 主機金鑰（known_hosts）

SSH 第一次連到某台主機時，會把對方的**身分指紋**記下來，之後每次連線都會比對。

| 檔案 | 用途 |
|------|------|
| `~/.ssh/known_hosts` | 記錄自己連過的主機指紋（Windows 在 `C:\Users\使用者\.ssh\known_hosts`） |
| `/etc/ssh/ssh_known_hosts` | 系統層級，由管理員預先設定 |
| 伺服器的 `/etc/ssh/ssh_host_*_key.pub` | 伺服器自己的公鑰（身分證） |

**出現這個警告時**

```
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
...
Offending ECDSA key in C:\\Users\\user01/.ssh/known_hosts:6
Host key verification failed.
```

- 意思：同一個 IP，指紋跟上次記的不一樣
- 常見原因：**同一個 IP 換了一台主機**、重灌系統、改了網路設定
- 確定是自己的環境造成的，就刪掉舊紀錄：

```bash
ssh-keygen -R 192.168.151.100      # 刪除這個 IP 的舊紀錄（會自動備份成 known_hosts.old）
```

> 沒有重灌、沒換主機、沒改 IP，卻突然出現這個警告，就要小心可能有人冒充這台主機。

## 金鑰登入（不用打密碼）

用一對鑰匙取代密碼：

| | 私鑰 `id_rsa` | 公鑰 `id_rsa.pub` |
|------|------|------|
| 比喻 | **鑰匙** | **鎖** |
| 放在哪 | **自己電腦**，絕對不能給別人 | **複製到要登入的伺服器上** |
| 權限 | `600` | `644` |

```
你的電腦（拿著鑰匙）               伺服器（裝上了你的鎖）
  ~/.ssh/id_rsa       ──登入──→    ~/.ssh/authorized_keys
                                    （裡面放著你的公鑰）
```

**設定步驟**

```bash
ssh-keygen                         # 1. 產生金鑰，問題都可以直接按 Enter
ssh-copy-id user@remotehost        # 2. 把公鑰複製到伺服器（這次還要輸入密碼）
ssh user@remotehost                # 3. 之後登入不用密碼
```

| 指令 | 說明 |
|------|------|
| `ssh-keygen` | 產生金鑰，預設存在 `~/.ssh/id_rsa` 和 `~/.ssh/id_rsa.pub` |
| `ssh-keygen -f ~/.ssh/mykey` | 指定檔名 |
| `ssh-copy-id -i ~/.ssh/mykey.pub user@host` | 複製指定的公鑰 |
| `ssh -i ~/.ssh/mykey user@host` | 用指定的私鑰登入 |

- 再執行一次 `ssh-keygen` 時，會問要不要覆蓋原本的金鑰；覆蓋的話，**所有伺服器上的舊公鑰都要換掉**
- Windows 有 `ssh-keygen`，但**沒有 `ssh-copy-id`**

### passphrase 和 ssh-agent

**passphrase** 是私鑰的密碼：就算私鑰被偷了，沒有 passphrase 也不能用。

但設了 passphrase，每次登入都要輸入，所以用 **`ssh-agent`** 暫存起來：

```bash
eval $(ssh-agent)      # 啟動 ssh-agent
ssh-add                # 輸入一次 passphrase，這次登入期間都不用再輸入
ssh-add ~/.ssh/mykey   # 加入其他私鑰
```

- 登出時 `ssh-agent` 會結束，暫存的 passphrase 也會清除
- 圖形桌面（GNOME）通常會自動啟動 `ssh-agent`

## 加強安全性（sshd_config）

伺服器端的設定檔：`/etc/ssh/sshd_config`

| 設定 | 意思 |
|------|------|
| `PermitRootLogin no` | **禁止 root 直接用 SSH 登入**（建議） |
| `PermitRootLogin without-password` | root 只能用金鑰登入 |
| `PasswordAuthentication no` | 所有人都只能用金鑰登入 |
| `Port 22` | SSH 使用的 port |

**為什麼要禁止 root 登入**
- 每台 Linux 都有 `root`，攻擊者只要猜密碼就好
- root 權限最大，被入侵的損害最大
- 大家都用 root 登入，**看不出是誰做了什麼**；各自用自己的帳號登入再 `sudo`，紀錄才清楚

**為什麼要禁止密碼登入**：攻擊者沒辦法靠一直猜密碼入侵

> **關閉密碼登入前，一定要先確認金鑰登入可以用**，不然會把自己鎖在門外。

**設定檔裡的 `#`**
- 開頭有 `#`：註解，不生效，通常只是在說明預設值（例如 `#Port 22` 代表預設就是 22）
- 要改成不同的值，才需要拿掉 `#` 並修改

**`sshd_config.d` 會覆蓋主設定檔**

```
Include /etc/ssh/sshd_config.d/*.conf      ← 在主設定檔最上面
```

- SSH 的規則是**同一個設定以最先讀到的為準**，所以 `sshd_config.d/` 裡的設定優先
- 確認實際生效的值：

```bash
sudo sshd -T | grep -iE '^(port|permitrootlogin|passwordauthentication)'
```

**改完要重新載入**

```bash
sudo systemctl reload sshd     # 重新讀取設定，不會中斷現有連線
```

- 課本建議用 `reload`；`restart` 可能會中斷現有的 SSH 連線

## 連不上時怎麼排查

從錯誤訊息判斷問題在哪：

| 錯誤訊息 | 原因 | 檢查什麼 |
|------|------|------|
| `Connection timed out` | **網路不通** | IP 對不對、同一網段嗎、`ping` 得到嗎 |
| `Connection refused` | 網路通，但 sshd 沒在跑或 port 不對 | `systemctl status sshd`、`Port` 設定 |
| `Permission denied (publickey)` | 不允許密碼登入 | `PasswordAuthentication`、有沒有放好公鑰 |
| `Permission denied, please try again` | 密碼錯，或不允許 root 登入 | 密碼、`PermitRootLogin` |
| `REMOTE HOST IDENTIFICATION HAS CHANGED!` | 主機指紋變了 | 見上面〈主機金鑰〉 |

**排查順序**

```bash
ping IP                         # 1. 網路通不通（在 Windows 上）
systemctl status sshd           # 2. sshd 有沒有在跑（在 Linux 上）
ip -4 addr                      # 3. Linux 的 IP 是不是你連的那個
```

- VMware 環境的網路設定問題，常見原因是 IP 和閘道不在同一網段、或網卡沒接到正確的虛擬網路（NAT / Bridged / Host-only）
