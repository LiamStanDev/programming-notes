# 常用指令與 Shell 技巧

## 目錄
- [主題簡介](#主題簡介)
- [Shell 原理與架構](#shell-原理與架構)
- [常用指令範例與解析](#常用指令範例與解析)
- [常見錯誤與排查方式](#常見錯誤與排查方式)
- [最佳實踐與安全性建議](#最佳實踐與安全性建議)
- [實戰案例](#實戰案例)

---

## 主題簡介

Shell 是 Linux/UNIX 系統最核心的互動介面，負責接收使用者輸入、解析指令、調用系統資源。熟練 Shell 指令與技巧，能顯著提升日常維運、開發自動化、故障排查效率，也是面試必考重點。

---
- [現代搜尋工具：fd 與 ripgrep](#現代搜尋工具fd-與-ripgrep)

## Shell 原理與架構

### Shell 類型與差異

| 類型   | 說明                       | 特點                  |
| ------ | -------------------------- | --------------------- |
| sh     | 最基本，兼容性高           | 輕量、跨平台          |
| bash   | 最常用，功能強大           | 指令補全、歷史、陣列  |
| zsh    | 進階補全、主題豐富         | 開發者常用            |
| ksh/csh| 早期商用系統常見           | 特殊語法              |

### Shell 運作流程

1. 讀取輸入（互動或腳本）
2. 解析指令（語法分析、變數展開、通配符展開）
3. 執行指令（fork/exec 調用）
4. 回傳輸出與狀態碼

### 指令結構與語法

```sh
指令 [選項] [參數]
```
範例：`ls -l /etc`

### 管線與重導

- 管線（Pipe）：`|`，將前一指令輸出傳給下一指令
- 輸出重導：`>` 覆蓋、`>>` 追加
- 輸入重導：`<`
- 錯誤重導：`2>`, `2>&1`

### 環境變數與通配符

- 查看：`echo $PATH`
- 設定：`export VAR=value`
- 通配符：`*` 任意字元、`?` 單一字元、`[abc]` 任一字元

---

## 常用指令範例與解析

### 1. grep — 文字搜尋

```sh
grep "pattern" 檔案
```
**範例**
```sh
$ grep "root" /etc/passwd
root:x:0:0:root:/root:/bin/bash
```
**說明**：搜尋檔案內含有 "root" 的行。

---

### 2. awk — 欄位處理

```sh
awk -F: '{print $1,$3}' /etc/passwd
```
**輸出**
```
root 0
daemon 1
```
**說明**：以冒號分隔，輸出第 1 與第 3 欄。

---

### 3. sed — 文字取代/刪除

```sh
sed 's/foo/bar/g' file.txt
```
**範例**
```sh
$ echo "foo123" | sed 's/foo/bar/'
bar123
```
**說明**：將 foo 取代為 bar。

---

### 4. sort/uniq — 排序與去重

```sh
sort 檔案 | uniq
```
**範例**
```sh
$ cat list.txt
b
a
c
a
$ sort list.txt | uniq
a
b
c
```
**說明**：先排序再去除重複行。

---

### 5. cut — 欄位擷取

```sh
cut -d: -f1 /etc/passwd
```
**輸出**
```
root
daemon
```
**說明**：以冒號分隔，取第一欄。

---

### 6. xargs — 輸入轉參數

```sh
ls *.log | xargs rm
```
**說明**：將 ls 輸出檔案逐一傳給 rm。

---

### 7. tee — 輸出到螢幕與檔案

```sh
ls | tee files.txt
```
**說明**：同時顯示並寫入檔案。

---

### 8. history — 查詢歷史指令

```sh
history | grep ssh
```
**說明**：查詢過去執行過的 ssh 相關指令。

---

### 9. 進階技巧

#### 參數展開

```sh
echo ${VAR:-default}   # VAR 未設則輸出 default
```

#### 陣列（bash）

```bash
arr=(a b c)
echo ${arr[1]}   # 輸出 b
```

#### 條件判斷

```sh
if [ -f file.txt ]; then
  echo "存在"
fi
```

#### 迴圈

```sh
for i in $(seq 1 3); do
  echo $i
done
```

#### trap — 捕捉訊號

```sh
trap 'echo "中斷"; exit' INT
```

#### 命令替換

```sh
files=$(ls /tmp)
echo "$files"
```

#### 函式

```sh
myfunc() {
  echo "Hello $1"
}
myfunc world
```

---

## 常見錯誤與排查方式

| 問題類型     | 排查步驟與建議                                   |
| ------------ | ----------------------------------------------- |
| 語法錯誤     | 檢查分號、括號、引號是否缺漏，善用 `shellcheck`  |
| 路徑問題     | 用 `pwd`、`ls` 確認檔案/目錄存在與否            |
| 權限問題     | `ls -l` 檢查權限，必要時 `chmod`、`sudo`         |
| 環境變數未設 | `echo $VAR` 檢查，必要時 `export`                |
| 指令不存在   | 檢查 `$PATH`，用 `which` 查詢                    |
| 管線/重導失敗| 檢查檔案權限、磁碟空間、指令拼寫                |
| 參數錯誤     | 查閱 `man`、`--help`，確認參數格式               |

---

## 最佳實踐與安全性建議

- **腳本可攜性**：shebang 建議用 `/bin/sh`，避免專屬語法。
- **變數命名**：全大寫，避免與系統變數衝突。
- **引號使用**：字串含空白或特殊字元時，務必加引號。
- **路徑處理**：優先用絕對路徑，減少相對路徑錯誤。
- **錯誤處理**：腳本開頭加 `set -euo pipefail`，遇錯即停。
- **權限最小化**：避免腳本預設 root 執行，必要時才用 sudo。
- **敏感操作前備份**：如批次刪檔、覆蓋檔案，先備份原始資料。
- **審查外部輸入**：xargs、eval 等指令需特別小心注入風險。
- **審核腳本來源**：勿直接執行不明來源腳本。

---

## 實戰案例

### 案例一：批次刪除超過 30 天的 log 檔

```sh
find /var/log -type f -name "*.log" -mtime +30 | xargs rm -v
```
**說明**：尋找 30 天前的 log 檔並刪除，`-v` 顯示刪除過程。

---

### 案例二：分析日誌中出現最多的錯誤類型

```sh
grep "ERROR" app.log | awk -F' ' '{print $3}' | sort | uniq -c | sort -nr | head
```
**說明**：統計錯誤類型出現次數，找出最常見問題。

---

### 案例三：檢查目錄下檔案權限異常

```sh
find /home/user -type f ! -perm 600 -ls
```
**說明**：列出權限非 600 的檔案，利於安全稽核。

---

### 案例四：批次轉檔並備份

```sh
for f in *.txt; do
  cp "$f" "$f.bak"
  iconv -f big5 -t utf8 "$f" -o "$f.utf8"
done
```
**說明**：所有 txt 先備份再轉碼，避免資料遺失。

---

### 案例五：自動化監控腳本（CPU 過高警示）

```sh
#!/bin/sh
THRESHOLD=80
USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d. -f1)
if [ "$USAGE" -gt "$THRESHOLD" ]; then
  echo "CPU 使用率過高：$USAGE%" | mail -s "警告" admin@example.com
fi
```
**說明**：CPU 超過門檻自動發警告信。

---

## 現代搜尋工具：fd 與 ripgrep

### 主題簡介

`fd` 與 `rg`（ripgrep）是現代化的檔案與內容搜尋工具，分別對應傳統的 `find` 與 `grep`，主打速度快、語法簡潔、預設行為更貼近實務需求。適合日常開發、維運、專案大規模檔案管理。

### 原理與與傳統 find/grep 差異

| 工具   | 對應傳統 | 主要用途     | 特色與差異                                       |
| ------ | -------- | ------------ | ------------------------------------------------ |
| fd     | find     | 檔案搜尋     | 預設忽略 .gitignore，語法簡單，支援正則、彩色輸出 |
| rg     | grep     | 內容搜尋     | 基於 Rust，極速搜尋，預設遞迴、支援語法高亮       |

- **fd**：自動忽略 .gitignore、隱藏檔，語法直覺，支援多平台。
- **rg**：比 grep/ack/ag 更快，支援語法高亮、正則、檔案類型過濾。

### 多組命令範例

#### fd 範例

```sh
$ fd config
config.yaml
src/config/defaults.py
```

```sh
$ fd '\.md$' docs/
docs/README.md
docs/guide.md
```

```sh
$ fd -e jpg
images/photo1.jpg
images/photo2.jpg
```

#### rg (ripgrep) 範例

```sh
$ rg TODO
main.py:12:# TODO: refactor this function
README.md:5:- [ ] TODO: add usage
```

```sh
$ rg 'def\s+\w+' src/
src/utils.py:3:def add(a, b):
src/main.py:10:def main():
```

```sh
$ rg --type py 'import os'
src/main.py:1:import os
```

### 常見錯誤與排查方式

| 問題類型         | 排查建議                                         |
| ---------------- | ----------------------------------------------- |
| 指令找不到       | 確認已安裝 fd/rg，或檢查 $PATH                   |
| 無法搜尋隱藏檔   | fd/rg 預設忽略隱藏檔，加上 `-H` 或 `--hidden`    |
| 結果過少         | 檢查 .gitignore 是否排除了目標檔案               |
| 正則無效         | 檢查語法，fd/rg 預設使用 Rust regex              |
| 權限不足         | 以 sudo 執行或調整目錄權限                      |

### 最佳實踐與安全性建議

- 善用 `.gitignore` 避免搜尋雜訊檔案。
- 需搜尋隱藏檔時加 `--hidden`，避免遺漏。
- 大型專案建議加 `--type` 或 `-e` 過濾副檔名，加速搜尋。
- 輸出結果建議搭配 `less`、`head` 等分頁工具。
- 避免直接用 `xargs rm` 等危險操作，先確認結果。
- 正則表達式建議加引號，避免 shell 展開誤判。

### 實戰案例

#### 案例一：快速找出所有 Python 檔案

```sh
fd -e py
```
**說明**：列出所有副檔名為 py 的檔案，預設遞迴。

#### 案例二：搜尋程式碼中所有 TODO 標記

```sh
rg TODO
```
**說明**：遞迴搜尋專案內含 TODO 的行，支援語法高亮。

#### 案例三：搜尋被 .gitignore 排除的檔案

```sh
fd --no-ignore
```
**說明**：包含被 .gitignore 排除的檔案進行搜尋。

#### 案例四：批次刪除搜尋結果（安全做法）

```sh
fd 'bak$' | tee bak.list
# 確認 bak.list 無誤後再執行
xargs -a bak.list rm
```
**說明**：先將搜尋結果存檔，確認後再批次刪除，避免誤刪。

---
