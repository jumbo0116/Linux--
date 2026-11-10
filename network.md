# 網路

## 基本概念

### IP 位址

**網路上的門牌號碼**，4 個 0–255 的數字（IPv4）。

```
192.168.151 . 200
└────┬────┘   └┬┘
  網路部分    主機部分
  （路名）   （門牌號）
```

- **網路部分一樣 = 同一個網段**，可以直接溝通
- 網路部分不一樣，要透過**閘道**轉送

### 網段大小（/24）

IP 後面的 `/數字` 代表**前面多少位元是網路部分**：

| 寫法 | 子網路遮罩 | 網路部分 | 可用的主機數 |
|------|------|------|------|
| `/8` | `255.0.0.0` | 第 1 個數字 | 約 1600 萬 |
| `/16` | `255.255.0.0` | 前 2 個數字 | 約 6.5 萬 |
| **`/24`** | `255.255.255.0` | **前 3 個數字** | **254**（最常見） |
| `/28` | `255.255.255.240` | — | 14（iPhone 熱點） |

**`/24` 的保留位址**

| 位址 | 用途 |
|------|------|
| `192.168.151.0` | 代表整個網段 |
| `192.168.151.255` | **廣播位址**（`ip addr` 裡的 `brd`），送給網段裡所有人 |

### 閘道、DNS

| 名詞 | 意思 | 規則 |
|------|------|------|
| **閘道（gateway）** | 連到**其他網段**的出口，通常是路由器 | **一定要跟自己在同一個網段** |
| **DNS** | 把名稱（`google.com`）翻譯成 IP | 不一定要同網段 |

- DNS 壞掉時：`ping 8.8.8.8` 通，`ping google.com` 不通

### 特殊位址

| 範圍 | 用途 |
|------|------|
| `10.0.0.0` – `10.255.255.255` | 私人 IP（公司內部） |
| `172.16.0.0` – `172.31.255.255` | 私人 IP（iPhone 熱點是 `172.20.10.x`） |
| `192.168.0.0` – `192.168.255.255` | 私人 IP（家用路由器、VMware） |
| `127.0.0.1` | **自己**（`lo` 網卡） |

- 私人 IP 只在內部網路使用，不同地方可以重複

### DHCP 和固定 IP

| 方式 | 說明 | nmtui / nmcli |
|------|------|------|
| **DHCP（自動）** | 路由器自動分配，`ip addr` 會顯示 `dynamic` | `<Automatic>` / `ipv4.method auto` |
| **固定 IP（手動）** | 自己指定 IP、閘道、DNS，伺服器常用 | `<Manual>` / `ipv4.method manual` |

### 路由表

系統依照路由表決定封包往哪裡送：

```
目的地              網卡       經過的路由器
192.168.5.0/24     enp3s0     （同網段，直接送）
0.0.0.0/0（預設）   enp3s0     192.168.5.254   ← 其他所有地方都交給閘道
```

- `0.0.0.0/0` = **預設路線**，`ip route` 裡顯示成 `default via 閘道`
- **預設路線只能有一條**，有兩條會互相打架，網路時好時壞

## 檢查網路狀態

```bash
ip link                    # 列出所有網卡（link/ether 後面是 MAC 位址）
ip addr                    # 查看所有 IP
ip -4 addr show ens160     # 只看某張網卡的 IPv4
ip -s link show ens160     # 收送了多少封包、有沒有錯誤
ip route                   # 路由表，看 default via 後面的閘道
ip route get 目標IP        # 連到某個 IP 時，會走哪張網卡、用哪個來源 IP
ping -c3 8.8.8.8           # 測試連線（-c3：只送 3 次）
tracepath 網址             # 封包經過了哪些路由器
ss -tlnp                   # 有哪些服務在等待連線、用哪個 port
```

**`ip addr` 怎麼看**

```
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> ... state UP
    link/ether 00:0c:29:f7:25:64 brd ff:ff:ff:ff:ff:ff     ← MAC 位址
    inet 192.168.151.200/24 brd 192.168.151.255 scope global ens160   ← IPv4
    inet6 fe80::20c:29ff:fef7:2564/64 scope link            ← IPv6
```

- `UP`：網卡已啟用
- `dynamic`：DHCP 拿到的；`valid_lft 3473sec` 是剩幾秒到期（會自動續約）

## nmcli（重要）

**NetworkManager** 是管理網路的服務；`nmcli`（指令）和 `nmtui`（選單介面）都是用來操作它的工具。

| 名詞 | 意思 | 例子 |
|------|------|------|
| **裝置（device）** | 實體的網卡 | `ens160`、`ens256` |
| **連線（connection）** | 一組設定：IP、閘道、DNS…… | `nat-ssh` |

- 一張網卡可以有好幾組連線設定，但**同一時間只能啟用一組**

### 查看

```bash
nmcli dev status              # 每張網卡的狀態、用的是哪個連線
nmcli con show                # 列出所有連線設定
nmcli con show --active       # 只列出啟用中的
nmcli con show 連線名稱        # 某個連線的詳細設定
```

### 新增

```bash
# DHCP
nmcli con add con-name 連線名稱 type ethernet ifname 網卡名稱 ipv4.method auto

# 固定 IP
nmcli con add con-name 連線名稱 type ethernet ifname 網卡名稱 \
  ipv4.method manual ipv4.addresses 192.168.151.100/24 \
  ipv4.gateway 192.168.151.2 ipv4.dns 192.168.151.2
```

### 修改

```bash
nmcli con mod 連線名稱 ipv4.method manual
nmcli con mod 連線名稱 ipv4.addresses 192.168.151.200/24
nmcli con mod 連線名稱 ipv4.gateway 192.168.151.2
nmcli con mod 連線名稱 ipv4.dns 8.8.8.8          # 取代原本的 DNS
nmcli con mod 連線名稱 +ipv4.dns 1.1.1.1         # + 是新增，- 是刪除
nmcli con mod 連線名稱 connection.interface-name ens256   # 改綁定的網卡
nmcli con mod 連線名稱 ipv4.never-default yes    # 不要當預設路線
```

> **改成固定 IP 時，`ipv4.method` 一定要改成 `manual`**。
> 維持 `auto` 又加上固定 IP，會同時用 DHCP 和固定 IP，變成**一張網卡兩個 IP**、兩條預設路線。

### 啟用、停用、刪除

```bash
nmcli con up 連線名稱          # 啟用（修改後要執行才會生效）
nmcli con down 連線名稱        # 停用
nmcli dev dis 網卡名稱         # 中斷某張網卡
nmcli con del 連線名稱         # 刪除連線
nmcli con reload              # 直接改了設定檔後，重新讀取
```

### 指令速查

| 指令 | 用途 |
|------|------|
| `nmcli dev status` | 所有網卡的狀態 |
| `nmcli con show` | 所有連線 |
| `nmcli con add con-name 名稱 ...` | 新增連線 |
| `nmcli con mod 名稱 ...` | 修改連線 |
| `nmcli con up 名稱` | 啟用連線 |
| `nmcli dev dis 網卡` | 中斷網卡 |
| `nmcli con del 名稱` | 刪除連線 |

- `nmtui` 是選單版本：**Edit a connection** 修改、**Activate a connection** 啟用 / 停用
- `nmtui` 的 Device 欄位可以隨便輸入，**網卡不存在也能存檔**，所以要先用 `nmcli dev status` 確認網卡名稱

## 網路設定檔

`nmcli` 修改的設定會存成檔案：

| 版本 | 位置 |
|------|------|
| RHEL 8 | `/etc/sysconfig/network-scripts/ifcfg-連線名稱` |
| RHEL 9 | `/etc/NetworkManager/system-connections/連線名稱.nmconnection` |

- 建議用 `nmcli` / `nmtui` 修改，不要直接改檔案
- 直接改了檔案，要 `nmcli con reload`，再 `nmcli con up 連線名稱`

## 主機名稱和名稱解析

### 主機名稱

```bash
hostname                                       # 查看
hostnamectl set-hostname servera.example.com   # 永久修改（寫進 /etc/hostname）
hostnamectl status                             # 主機名稱 + 作業系統資訊
```

### 名稱解析

把名稱翻譯成 IP 時，依序查詢：

```
1. /etc/hosts        ← 先查手動寫的對照表
        ↓ 找不到
2. DNS 伺服器         ← 再問 /etc/resolv.conf 裡的 nameserver
```

**`/etc/hosts` 格式**

```
192.168.151.100   servera   servera.example.com
IP                名稱      （可以寫多個名稱）
```

**`/etc/resolv.conf`**

```
search example.com          ← 只打短名稱時，自動補上的網域
nameserver 172.25.254.254   ← DNS 伺服器（最多 3 個）
```

- `resolv.conf` 由 NetworkManager 自動產生，**不要直接改**，用 `nmcli con mod 連線名稱 ipv4.dns IP`

```bash
getent hosts servera       # 測試名稱能不能解析成 IP
```

- 名稱查不到時：`Could not resolve hostname`
- Windows 的 hosts 檔：`C:\Windows\System32\drivers\etc\hosts`（要用系統管理員身分編輯）

## VMware 網路

| 模式 | 說明 | 適合 |
|------|------|------|
| **NAT**（VMnet8） | VM 透過 Windows 上網，IP 不受外部網路影響 | 練習用，最穩定 |
| **Bridged**（VMnet0） | VM 直接接到實體網路，跟 Windows 同網段 | 要讓其他電腦連到 VM |
| **Host-only**（VMnet1） | 只有 Windows 和 VM 之間互通，不能上網 | 只需要 Windows 連 VM |

**VMware NAT 的預設位址**（以 `192.168.151.0/24` 為例）

| 位址 | 用途 |
|------|------|
| `.1` | Windows 本機的虛擬網卡 |
| **`.2`** | **NAT 閘道**（設定閘道要填這個） |
| `.254` | DHCP 伺服器（不是閘道） |

- 實際網段到 VMware 的 **Edit → Virtual Network Editor** 查看
- **網卡名稱依照插槽編號**：`ens160` 在 160 號插槽，新增的網卡可能是 `ens224`、`ens256`

**兩張網卡的做法**：一張上網、一張固定 IP 給 SSH

| 網卡 | 模式 | IP 設定 |
|------|------|------|
| `ens160` | Bridged / NAT | 自動，負責上網 |
| 新網卡 | NAT / Host-only | 固定 IP，**不設閘道**，勾選 `never-default` |

## 網路不通時怎麼排查

```bash
ip -4 addr          # 1. 有沒有 IP？IP 是不是對的網段？有沒有兩個 IP？
ip route            # 2. default via 有沒有？是不是只有一條？閘道跟 IP 同網段嗎？
ping -c3 閘道IP     # 3. 連得到閘道嗎？
ping -c3 8.8.8.8    # 4. 連得到外面嗎？
ping -c3 google.com # 5. DNS 正常嗎？
```

| 結果 | 問題在 |
|------|------|
| 1 沒有 IP | 連線沒啟用（`nmcli con up`）、網卡沒接上 |
| 2 有兩條 default | 有兩個連線都設了閘道 |
| 3 不通 | IP 或閘道設錯網段、VMware 網卡模式不對 |
| 4 通、5 不通 | DNS 設定錯誤 |
