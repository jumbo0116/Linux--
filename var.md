# Linux 變數

## 設定與讀取

```bash
COUNT=40              # 設定變數
echo $COUNT           # 讀取變數要加 $ → 40
echo COUNT            # 沒加 $ 只是普通文字 → COUNT
```

**規則**

- `=` 兩邊**不能有空格**：`COUNT = 40` 會被當成執行 `COUNT` 這個指令
- 名稱只能用英文字母、數字、底線 `_`，而且不能用數字開頭
- 名稱有分大小寫：`count` 和 `COUNT` 是兩個不同的變數
- 讀取不存在的變數**不會報錯**，只會得到空字串

**範例：用變數代替很長的檔名**

```bash
file1=/tmp/tmp.z9pXW0HqcC
ls -l $file1
rm $file1
```

## 大括號 `${}`

變數名稱後面緊接著其他文字時，要用 `{}` 把名稱包起來：

```bash
COUNT=40
echo $COUNTx          # 會去找名叫 COUNTx 的變數，結果是空的
echo ${COUNT}x        # 40x
```

- 隨時都可以加 `{}`，不確定時就加

## 引號

值裡面有空格時，一定要加引號：

```bash
name="Jumbo Chen"     # 正確
name=Jumbo Chen       # 錯誤：會被當成設定 name=Jumbo 後執行 Chen 這個指令
```

**單引號和雙引號的差別**

| 寫法 | 結果 | 說明 |
|------|------|------|
| `echo "Hi $name"` | `Hi Jumbo Chen` | 雙引號：裡面的變數**會**被替換 |
| `echo 'Hi $name'` | `Hi $name` | 單引號：內容原封不動 |

## 把指令結果存進變數 `$()`

```bash
today=$(date +%F)     # 執行 date，把輸出存進變數
echo $today           # 2026-10-17
mkdir backup_$today   # 建立 backup_2026-10-17 目錄
```

## Shell 變數 vs 環境變數

| 種類 | 誰可以用 | 怎麼建立 |
|------|------|------|
| Shell 變數 | 只有當前這個 shell | `NAME=value` |
| 環境變數 | 當前 shell 和**從它執行的程式** | `export NAME=value` |

```bash
EDITOR=vim
export EDITOR         # 把已經存在的變數匯出成環境變數

export EDITOR=vim     # 設定和匯出一次完成
```

- 每個終端機視窗、每個 SSH 連線都是獨立的 shell，變數**互不影響**
- 只有 shell 自己用的變數（例如 `PS1`、`HISTFILE`）不需要 `export`
- 系統的變數名稱都是全大寫，自己取的變數建議用小寫，避免撞名

## 查看變數

變數存在**目前這個 shell 的記憶體**裡，沒有一個檔案專門記錄，要用指令查看：

| 指令 | 列出什麼 |
|------|------|
| `env` / `printenv` | 所有環境變數 |
| `printenv PATH` | 單一環境變數（不用加 `$`） |
| `echo $PATH` | 單一變數（shell 變數也可以） |
| `compgen -v` | 所有變數的**名稱** |
| `declare -p` | 所有變數，並標出類型 |
| `set` | 所有變數，但會混著一大堆函式 |

**常用組合**

```bash
env | less                       # 分頁查看所有環境變數
env | grep PATH                  # 找名稱或內容有 PATH 的
compgen -v | less                # 看目前有哪些變數名稱
declare -p | grep 'declare -x'   # 只看有 export 的
```

**`declare -p` 怎麼看**

```
declare -- COUNT="40"              ← --  一般 shell 變數
declare -x HOME="/home/jumbo"      ← -x  環境變數（有 export）
declare -i NUM="5"                 ← -i  整數
```

### 變數從哪裡來

開啟 shell 時，會從這些地方載入變數：

| 來源 | 例子 |
|------|------|
| 系統登入程式自動設定 | `HOME`、`USER`、`SHELL` |
| `/etc/profile`、`/etc/profile.d/*.sh` | 所有使用者共用的 `PATH`、`LANG` 等 |
| `~/.bash_profile`、`~/.bashrc` | 自己加的設定 |
| bash 自己設定 | `PWD`、`PS1`、`HISTFILE` |

**找出某個變數是在哪個檔案設定的**

```bash
grep -r EDITOR ~/.bashrc ~/.bash_profile /etc/profile /etc/profile.d/ 2>/dev/null
```

- `2>/dev/null` 會把沒有權限讀取的錯誤訊息丟掉

## 刪除變數

```bash
unset file1           # 刪除變數
export -n EDITOR      # 取消匯出，變回 shell 變數（變數本身還在）
```

## 常見的系統變數

| 變數 | 用途 | 範例值 |
|------|------|------|
| `HOME` | 家目錄 | `/home/jumbo` |
| `USER` | 目前的使用者名稱 | `jumbo` |
| `PWD` | 目前所在路徑 | `/home/jumbo/temp` |
| `PATH` | 去哪些目錄找指令 | `/usr/local/bin:/usr/bin:...` |
| `PS1` | 提示字元的外觀 | `[\u@\h \W]\$ ` |
| `LANG` | 語系（語言、日期格式、排序） | `en_US.UTF-8` |
| `EDITOR` | 預設文字編輯器 | `vim` |
| `HISTFILE` | 指令歷史紀錄存在哪 | `~/.bash_history` |
| `HISTFILESIZE` | 歷史紀錄最多存幾筆 | `1000` |

### PATH

打指令（例如 `ls`）時，shell 會依照 `PATH` 裡的目錄**從左到右**找，用找到的第一個。

```bash
echo $PATH
# /home/jumbo/.local/bin:/home/jumbo/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin

export PATH=${PATH}:/home/jumbo/sbin    # 在最後面加上一個目錄
```

- 目錄之間用 `:` 分隔
- 加的時候一定要保留原本的 `${PATH}`，寫成 `export PATH=/home/jumbo/sbin` 會把原本的全部蓋掉，`ls` 等指令都會找不到
- 當前目錄不在 `PATH` 裡，所以執行當前目錄的程式要寫 `./script.sh`

### PS1

```bash
PS1="bash\$ "             # 提示字元變成 bash$
PS1="[\u@\h \W]\$ "       # 改回 [jumbo@servera temp]$
```

| 符號 | 意思 |
|------|------|
| `\u` | 使用者名稱 |
| `\h` | 主機名稱 |
| `\W` | 目前目錄名稱（只顯示最後一層） |
| `\$` | 一般使用者顯示 `$`，root 顯示 `#` |

- 結尾留一個空格，打指令時才不會黏在一起

### LANG

```bash
date                      # Sat Oct 17 14:05:00 CST 2026
export LANG=fr_FR.UTF-8
date                      # sam. oct. 17 14:05:00 CST 2026（變成法文）
```

## 永久設定

直接在終端機設定的變數，**關掉終端機或登出就消失**。要每次開啟都自動設定，就寫進設定檔：

| 檔案 | 影響範圍 |
|------|------|
| `~/.bashrc` | 只有自己，每次開 bash 都會執行 |
| `/etc/profile.d/xxx.sh` | 所有使用者（要 root 權限，檔名要用 `.sh` 結尾） |

**範例：在 `~/.bashrc` 最後面加上**

```bash
export EDITOR=vim
export PATH=${PATH}:/home/jumbo/sbin
```

改完不會馬上生效，要重新開一個終端機，或執行：

```bash
source ~/.bashrc
```
