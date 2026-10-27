# 服務管理（systemd）

## 基本概念

| 名詞 | 意思 |
|------|------|
| 守護進程（daemon） | 在**背景一直執行**、提供某種服務的程式，名稱習慣以 `d` 結尾 |
| `systemd` | 管理所有服務的總管，開機第一個啟動（PID 1） |
| `systemctl` | 用來對 systemd 下指令的工具（**system** + **c**on**t**ro**l**） |

```
你 ──下指令──→ systemctl ──傳話──→ systemd ──管理──→ 各種服務
```

**常見的服務**

| 服務 | 用途 |
|------|------|
| `sshd` | 讓別人用 SSH 連進來 |
| `firewalld` | 防火牆 |
| `crond` | 定時執行工作 |
| `chronyd` | 校正時間 |
| `httpd` | 網頁伺服器（要先安裝） |

**單元（unit）類型**：systemd 管理的東西叫「單元」，用副檔名區分

| 副檔名 | 是什麼 |
|------|------|
| `.service` | 系統服務（最常用） |
| `.socket` | 有人連線時才啟動對應的服務 |
| `.path` | 某個檔案或資料夾有變化時才啟動服務 |

- `.service` 可以省略：`systemctl status sshd` 等於 `systemctl status sshd.service`

## 查看服務

```bash
systemctl status sshd
```

```
● sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; ...)
                                                          ↑ 開機會自動啟動
   Active: active (running) since ...                    ← 正在執行
 Main PID: 1073 (sshd)                                   ← 進程編號
```

**狀態關鍵字**

| 關鍵字 | 意思 |
|------|------|
| `active (running)` | 正在執行 |
| `active (exited)` | 已完成一次性的工作 |
| `active (waiting)` | 執行中，正在等待事件 |
| `inactive` | 沒有在執行 |
| `failed` | 啟動失敗 |
| `enabled` | **開機會自動啟動** |
| `disabled` | 開機**不會**自動啟動 |
| `static` | 不能自己設定開機啟動，只能被其他服務帶起來 |
| `masked` | 被屏蔽，完全不能啟動 |

**快速確認**

```bash
systemctl is-active sshd        # active / inactive
systemctl is-enabled sshd       # enabled / disabled
systemctl is-failed sshd        # 有沒有啟動失敗
```

**列出服務**

```bash
systemctl list-units --type=service          # 正在執行的服務
systemctl list-units --type=service --all    # 包含沒在執行的
systemctl list-unit-files --type=service     # 所有已安裝的服務 + 開機會不會啟動
systemctl --failed --type=service            # 啟動失敗的服務
```

## 控制服務

需要 root 權限，要加 `sudo`。

```bash
sudo systemctl start sshd             # 啟動
sudo systemctl stop sshd              # 停止
sudo systemctl restart sshd           # 重新啟動（先停再啟動，PID 會變）
sudo systemctl reload sshd            # 重新載入設定檔（不中斷服務，PID 不變）
sudo systemctl reload-or-restart sshd # 支援 reload 就 reload，不支援就 restart
```

| | `restart` | `reload` |
|------|------|------|
| 做什麼 | 先停止再啟動 | 只重新讀取設定檔 |
| 服務會中斷嗎 | 會，短暫中斷 | 不會 |
| 所有服務都支援嗎 | 是 | 不一定 |

- **改完設定檔要 `restart` 或 `reload`**，新設定才會生效（例如改 `/etc/ssh/sshd_config` 後要 `restart sshd`）

## start 和 enable 的差別（重要）

| 指令 | 影響現在 | 影響開機後 |
|------|------|------|
| `start` | 開啟 | — |
| `stop` | 關閉 | — |
| `enable` | — | 開機自動啟動 |
| `disable` | — | 開機不啟動 |

- `start` / `stop`：管**現在**
- `enable` / `disable`：管**開機**
- 加上 `--now`：**兩個一起管**

```bash
sudo systemctl enable --now sshd      # = start + enable
sudo systemctl disable --now sshd     # = stop + disable
```

**例子**：防火牆現在是開啟的

```bash
sudo systemctl disable firewalld
systemctl is-active firewalld         # active   ← 現在還開著
systemctl is-enabled firewalld        # disabled ← 下次開機才不會啟動
```

> **考試重點**：RHCSA 考完會**重新開機再評分**。
> 只用 `start` 沒有 `enable`，重開機後服務就沒了，這題就沒分。
> 習慣一律用 `enable --now`。

**原理**：`enable` 會在 `/etc/systemd/system/` 底下建立一個「開機要啟動」的連結檔，`disable` 就是刪掉這個連結

```
Created symlink /etc/systemd/system/multi-user.target.wants/sshd.service → /usr/lib/systemd/system/sshd.service.
```

## 屏蔽服務：mask

最徹底的停用，**連手動啟動都不行**。

```bash
sudo systemctl mask sendmail          # 屏蔽
sudo systemctl start sendmail         # Failed: Unit sendmail.service is masked.
sudo systemctl unmask sendmail        # 解除屏蔽
```

| | `disable` | `mask` |
|------|------|------|
| 開機自動啟動 | 不會 | 不會 |
| 手動 `start` | 可以 | **不行** |
| 被其他服務帶起來 | 可以 | **不行** |

- 用途：兩個功能衝突的服務（例如兩種郵件伺服器），mask 其中一個避免被意外啟動
- 原理：建立一個指向 `/dev/null` 的連結

## 依賴關係

有些服務要**先啟動其他服務**才能運作。

```bash
systemctl list-dependencies sshd              # sshd 需要哪些單元
systemctl list-dependencies sshd --reverse    # 哪些單元需要 sshd
```

- 停止一個服務時，如果它還能被其他單元（例如 `.socket`）帶起來，會出現警告：

```
Warning: Stopping cups, but it can still be activated by:
  cups.path
  cups.socket
```

## 常用指令速查

```bash
systemctl status 服務                 # 查看狀態
sudo systemctl start 服務             # 啟動
sudo systemctl stop 服務              # 停止
sudo systemctl restart 服務           # 重新啟動
sudo systemctl enable --now 服務      # 開機自動啟動 + 現在啟動
sudo systemctl disable --now 服務     # 開機不啟動 + 現在停止
```

> 不要隨便關閉防火牆（`firewalld`）。練習時可以暫時關掉來排除問題，
> 找到原因後應該打開防火牆，只開放需要的 port（用 `firewall-cmd`）。
