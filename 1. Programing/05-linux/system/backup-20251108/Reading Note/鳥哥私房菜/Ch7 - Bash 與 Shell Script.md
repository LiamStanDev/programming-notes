### 變數
---
##### 使用變量
```shell
echo $PATH
echo ${PATH}
echo ${a[1]} # 讀取 array
```
##### 設定變量
```shell
# 無空格
myname=VBird # 注意不能有任何空格

# 有空格
var="lang is $LANG" # "" 中表示保持原有特性
var='lang is $LANG' # '' 表示純文本
var=lang\ is\ english # 可使用 `\ `表示空格
var=$(uname -r) # $() 表示計算表達式後替換
a[1]=10
a[2]="$PATH"

unset var # 取消變數

export PATH # 變成環境變數

env # 顯示所有環境變量
export # 功能同 env
set # 顯示所有環境變量 + bash 變量, e.g. $PS1, $?
```
* 變量默認是字符串，若想要爲整數要使用 `declare
```shell
declare -i sum
sum=100+300+50
echo ${sum}
```

##### Array
```shell
var[1]="small min"
var[2]="big min"
echo "${var[1]}, ${var[2]}"
```

### 命令關聯執行
---
* 指令無關聯性，連續執行
```shell
sync; sync; shutdown -h now
```
* 指令有關連性
```shell
cmd1 && cmd2 # 若 cmd1 指令返回 $?=0 才會執行 cmd2
cmd1 || cmd2 # cmd1 不正確才會執行 cmd2
```


### 管道命令 (pipe)
---
* 管道命令後面接的一定是命令
* 命令必須要能接受 stdin
* 管線命令只能傳遞 stdout，stderr 會忽略
```shell
dmesg | grep -in "eth"
```

##### 參數代換 `xargs`
以空白或換行字符作為段行，將 stdin 分割成 arguments。
```shell
find . -type f -name "*.jpg" -print | xargs -I {} tar -czvf images.tar.gz {}
```
* `-I` 表示 replace string 用來表示每個 argument。

##### `-` 的用途
用來替代 stdin 與 stdout
```shell
tar -cvf - /home | tar -xvf - -C /tmp/homeback
```


### 系統限制
---
```shell
# 查看該用戶所有限制
ulimit -a  # 他還會列出 options 的名稱，很貼心

# 設定用戶當前限制
ulimit -f 1024 # 設定能取得的文件句柄數量
```
* 永久修改最大限制的方式
![[01 - Socket#修改最大限制]]

### Regular Expression (grep, sed, awk)
---
linux 並不是所有工具都支持正規表達式的，如 ls, cp 等只能使用 `*` wildcard 來匹配全部，但無法使用正規表達式的語法。
> 因為要輸入給工具，所以批配都是以 `''` 而不是以 `""` 的形式傳入。
#### grep
```shell
dmesg | grep -n -A 3 -B 2 'eth'
```
* `-n` 顯示行號
* `-A`, `-B`: 表示 after, before 多顯示幾行

##### 搜索相關
```shell
grep -in 'the' regular_express.txt # 大小寫不敏感
grep -vn 'the' regular_express.txt # -v 表示不包含
```

##### 基礎
```shell
grep -in 't[ae]st' regular_expores.txt # 批配 t?st ? 可以爲 a or e
grep -in '[^g]oo' regular_expores.txt # 批配 oo 前面不要有 g
grep -n '[^a-z]oo' regular_expores.txt # 批配 oo 前面不要有 a-z 任意字符
grep -n '^the' regular_expores.txt # 批配開頭爲 the
grep -n '\.$' regular_expores.txt #  批配結尾爲 . 
grep -n 'g..d' regular_expores.txt # 批配 g??d
grep -n 'oo*' regular_expores.txt # 批配 o 後面接空或者多個 o，* 用於前一個字符
grep -n 'go\{2,5\}' regular_expores.txt # 指定g 後面接 2~5 個 o，注意 \{ \} 才能生效
```


#### sed
根據 row 進行文本處理的工具，可以擷取、添加、刪除等操作。
###### 參數說明：
* `-i`: 表示修改讀取內容
* `-n`: 只顯示被處理的行
* `-e`: 若後面有多個動作順序執行時
> 默認都會顯示在螢幕上，不會修改內容也不會生成文件
###### 功能

| 動作  | 說明                    |
| --- | --------------------- |
| a   | append                |
| c   | change                |
| d   | delete                |
| i   | insert                |
| p   | print 打印出來            |
| s   | subsitue，使用方式與 vim 一樣 |
> 可以指定行數也能在 `/ /` 中寫正則
 
##### 案例
```shell
# nl file_name 爲 number line of file，列出文件的行數

# 行爲整體單位的操作
nl /etc/passwd | sed '2,5d' # 刪除 2~5 行
nl /etc/passwd | sed '2,$d' # 刪除 2~最後一行
nl /etc/passwd | sed '2a drink tea' # 在第二行插入 drink tea
nl /etc/passwd | sed '2,5c No 2-5 number' # 刪除 2~5 行後添加 No 2-5 number，所以會少掉 3 行

# 對每行的部分進行操作
# 例子一： 列出所有 ipv4 地址
ip a | grep 'inet ' # 列出所有 ipv4
ip a | grep 'inet' | sed 's/^.*inet //g' # 去掉所有 '   inet' 替換成空
ip a | grep 'inet ' | sed 's/^.*inet //g' | sed 's/ .*$//g' # 去掉後面所有，只留下 ipv4 地址
ip a | grep 'inet ' | sed 's/^.*inet //g' | sed 's/ .*$//g' > ipv4.txt # 存成文件
# 例子二：刪除 man_db.conf 中註解的行
cat /etc/man_db.conf| grep 'MAN' | sed '/^#.*$/d' 

# 多個操作
cat /etc/passwd | sed -e '4d' -e '6c no six line' > passwd.new
```

#### awk
以欄位來處理的文本工具，他默認使用空格來作欄位的分隔
```shell
awk -F : '{print $1 "\t" $3}' /etc/passwd
```
* `-F` 指定分隔符號

### 資料重導向 (redirect)
---
![[Pasted image 20240507132255.png]]
資料流向有三個方向
1. 標準輸入 (stdin): 代碼為 0, 使用 `<` 或者 `<<`
2. 標準輸出 (stdout): 代碼為 1，使用 `>` 或者 `>>`
	-> 可以使用 `1>` 或 `1>>`
3. 標準錯誤 (stderr): 代碼為 2，使用 `2>` 或者 `2>>`

> `>` 表示覆蓋，`>>` 表示追加。

##### `/dev/null` 黑洞裝置
我們可以將錯誤資訊丟到黑洞裝置
```shell
find /root -name .bashrc 2> null/dev
```
這樣就只會顯示標準輸出，而不會顯示表準錯誤輸出。

##### 將標準輸出與標準錯誤一併輸出
```shell
find /rrot -name .bashrc > /tmp/temp 2>&1 # Method 1
find /rrot -name .bashrc &> /tmp/temp # Method 2
```

