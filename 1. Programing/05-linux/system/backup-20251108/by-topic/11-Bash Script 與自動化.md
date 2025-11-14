# Bash Script 與自動化

---

## 目錄
- [主題簡介](#主題簡介)
- [原理與架構](#原理與架構)
- [語法與命令範例](#語法與命令範例)
- [常見錯誤與排查](#常見錯誤與排查)
- [最佳實踐與安全性](#最佳實踐與安全性)
- [實戰案例](#實戰案例)

---

## 主題簡介

Bash Script 是 Linux/UNIX 系統最常用的自動化工具，能批次執行命令、流程控制、檔案處理與系統管理。常見應用如自動部署、備份、批次處理、監控、CI/CD 等。熟練 Bash Script 可大幅提升工程效率與穩定性，是面試與實務必備技能。

---

## 原理與架構

- **執行流程**：由上而下逐行執行，遇到流程控制（if/for/while）依條件分支。
- **Shebang**：第一行指定執行環境（如 `#!/bin/bash`），建議用 `#!/usr/bin/env bash` 增加可攜性。
- **組成要素**：
  - 命令列指令
  - 變數與參數
  - 條件判斷、迴圈、函式
  - 輸入輸出、錯誤處理
- **自動化架構設計**：
  - 主流程（入口）
  - 參數驗證
  - 日誌與錯誤處理
  - 模組化（函式/外部設定檔）
  - 結合 crontab、SSH、rsync、CI 工具

---

## 語法與命令範例

### 1. 變數與參數

```bash
name="world"
echo "Hello, $name"
# 輸出：Hello, world
```

### 2. 條件判斷 if/else

```bash
num=5
if [ "$num" -gt 3 ]; then
  echo "大於3"
else
  echo "小於等於3"
fi
# 輸出：大於3
```

### 3. for 迴圈

```bash
for f in *.log; do
  echo "處理 $f"
done
# 輸出：
# 處理 access.log
# 處理 error.log
```

### 4. while 迴圈

```bash
i=1
while [ $i -le 3 ]; do
  echo "第 $i 次"
  i=$((i+1))
done
# 輸出：
# 第 1 次
# 第 2 次
# 第 3 次
```

### 5. case 條件分支

```bash
case "$1" in
  start) echo "啟動";;
  stop)  echo "停止";;
  *)     echo "用法：$0 {start|stop}";;
esac
```

### 6. 讀取輸入

```bash
read -p "請輸入名稱：" name
echo "Hi, $name"
# 輸入：Alice
# 輸出：Hi, Alice
```

### 7. 陣列

```bash
arr=(a b c)
for x in "${arr[@]}"; do
  echo $x
done
# 輸出：
# a
# b
# c
```

### 8. 字串處理

```bash
str="hello world"
echo "${str:0:5}"        # hello
echo "${str/world/bash}" # hello bash
```

### 9. 捕捉訊號 trap

```bash
trap 'echo "收到中斷"; exit' INT
while true; do sleep 1; done
# Ctrl+C 時輸出：收到中斷
```

### 10. exit 狀態碼

```bash
if [ ! -f "$1" ]; then
  echo "檔案不存在"
  exit 1
fi
```

### 11. set -euxo pipefail

```bash
set -euxo pipefail
# -e: 遇錯即停
# -u: 未定義變數即錯
# -x: 顯示執行過程
# -o pipefail: 管線任一命令失敗即失敗
```

---

## 常見錯誤與排查

| 錯誤現象           | 可能原因                | 排查方式/解法                        |
|--------------------|------------------------|--------------------------------------|
| command not found  | 指令拼錯、PATH 未設     | 檢查指令拼寫，echo $PATH             |
| Permission denied  | 無執行權限             | chmod +x script.sh                   |
| 語法錯誤           | 括號/引號/空格錯誤      | 檢查對應行，善用 shellcheck          |
| 變數未定義         | 拼錯、未宣告           | set -u，或加預設值 ${VAR:-default}   |
| 路徑錯誤           | 相對/絕對路徑混用       | pwd、ls 檢查實際路徑                 |
| 參數錯誤           | 未檢查 $1/$#           | 增加參數驗證與提示                   |
| 管線失敗未偵測     | 未設 pipefail           | set -o pipefail                      |

### 排查步驟

1. 加入 `set -euxo pipefail` 觀察錯誤行
2. 使用 `bash -x script.sh` 逐步追蹤
3. 檢查權限與路徑
4. 善用 `echo` 或 `logger` 輸出除錯資訊
5. 用 shellcheck 靜態檢查語法

---

## 最佳實踐與安全性

- **嚴格模式**：開頭加 `set -euxo pipefail`，減少隱性錯誤
- **參數驗證**：檢查 `$#` 與 `$1`，避免未預期輸入
- **權限控管**：腳本權限最小化，避免 sudo 執行非必要命令
- **日誌紀錄**：重要操作加 log，方便追蹤與審計
- **錯誤處理**：每步檢查 `$?` 或用 `|| exit 1`
- **避免硬編碼**：參數化、外部設定檔
- **安全執行外部輸入**：避免 eval，對變數加引號
- **資安建議**：
  - 不要在腳本內寫明密碼
  - 避免直接下載並執行網路腳本
  - 定期檢查腳本內容與權限

---

## 實戰案例

### 1. 自動備份目錄

```bash
#!/usr/bin/env bash
set -euo pipefail

SRC="/etc"
DST="/backup/etc-$(date +%F).tar.gz"

tar czf "$DST" "$SRC"
echo "備份完成：$DST"
```
- 用 crontab 定時執行，實現每日自動備份。

---

### 2. 批次部署多台主機

```bash
#!/usr/bin/env bash
set -euo pipefail

HOSTS=(host1 host2 host3)
for h in "${HOSTS[@]}"; do
  scp ./deploy.sh "$h":/tmp/
  ssh "$h" 'bash /tmp/deploy.sh'
done
```
- 結合 scp/ssh，批次部署腳本到多台主機。

---

### 3. 日誌輪替與壓縮

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_DIR="/var/log/myapp"
find "$LOG_DIR" -name "*.log" -mtime +7 -exec gzip {} \;
find "$LOG_DIR" -name "*.log.gz" -mtime +30 -delete
```
- 7 天前的 log 壓縮，30 天前的 log 刪除。

---

### 4. 自動化互動命令（expect）

```bash
#!/usr/bin/expect -f
set timeout 10
spawn ssh user@host
expect "password:"
send "mypassword\r"
interact
```
- 用 expect 自動輸入密碼（建議用 SSH Key 取代）。

---

### 5. 定時任務 crontab

```bash
# 每天凌晨 2 點自動備份
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

---

## 參考

- [Bash 官方手冊](https://www.gnu.org/software/bash/manual/)
- [ShellCheck 靜態分析](https://www.shellcheck.net/)
- [Advanced Bash-Scripting Guide](https://tldp.org/LDP/abs/html/)
- [my/ 筆記風格參考](01.系統參數.md)
