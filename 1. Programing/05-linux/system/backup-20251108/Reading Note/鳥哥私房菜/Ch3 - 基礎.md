# 常用按鍵
---
* `TAB`: 命令補全 or 文件補全
* `CTRL + c`: 中斷當前程序，本質向程序發送 signal，詳情請見 [[06 - 進程#信號 (Signal)]]
* `CTRL + d`: 與輸入 `exit` 一樣
* `CTRL + ALT + F1/.../F6`: 切換 tty
* `SHIFT + PageUp/PageDown`: 在 shell 翻頁

# 幫助文檔
---
### --help
Linux 下大部分指令都可以使用 --help 來查看命令的大致使用方式。
```shell
date --help
```

### man page
```shell
man date
```
顯示如下，
```text
DATE(1)                                                                                         User Commands                                                                                         DATE(1)

NAME
       date - print or set the system date and time

SYNOPSIS
       date [OPTION]... [+FORMAT]
       date [-u|--utc|--universal] [MMDDhhmm[[CC]YY][.ss]]

DESCRIPTION
       Display the current time in the given FORMAT, or set the system date.

       Mandatory arguments to long options are mandatory for short options too.

       -d, --date=STRING
              display time described by STRING, not 'now'

       --debug
              annotate the parsed date, and warn about questionable usage to stderr

       -f, --file=DATEFILE
              like --date; once for each line of DATEFILE

       -I[FMT], --iso-8601[=FMT]
              output  date/time  in  ISO  8601  format.   FMT='date'  for  date  only  (the  default),  'hours',  'minutes',  'seconds',  or  'ns'  for  date  and time to the indicated precision.  Example:
              2006-08-14T02:34:56-06:00

       -R, --rfc-email
              output date and time in RFC 5322 format.  Example: Mon, 14 Aug 2006 02:34:56 -0600

       --rfc-3339=FMT
              output date/time in RFC 3339 format.  FMT='date', 'seconds', or 'ns' for date and time to the indicated precision.  Example: 2006-08-14 02:34:56-06:00

       -r, --reference=FILE
              display the last modification time of FILE
...
```

##### 指令代號
我們在上面看到 `DATE(1)` 其中括號中的數字就是指令代號，共有以下幾種代號

| 代號  | 含意                |
| --- | ----------------- |
| 1   | 使用者可執行指令，為指令或可執行檔 |
| 2   | 系統調用              |
| 3   | 常用函數庫，大部分為 libc   |
| 4   | /dev 下檔案說明        |
| 5   | 設定檔格式             |
| 6   | 遊戲                |
| 7   | 協議說明              |
| 8   | 系統管理員指令           |
| 9   | kernal 文件         |
##### man page 內容
以下為絕大部分都會包含的內容。

| 代號          | 內容                     |
| ----------- | ---------------------- |
| NAME        | 簡短指令與簡介                |
| SYNOPSIS    | 簡短指令的語法 (最簡單的 example) |
| DESCRIPTION | 完整說明                   |
| OPTIONS     | 可用選項                   |
| COMMANDS    | 在軟體運行中能下的指令            |
| FILES       | 參考檔案                   |
| SEE ALSO    | 查看其他相關指令的說明            |
| EXAMPLE     | 參考範例                   |

##### man page 操作
* `SPACE`: 下翻一頁
* `PageDown`: 同上
* `PageUp`: 上翻一頁
* `/`: 搜尋
* `n` or `N`: 同 vim 的下/上一個匹配
* `q`: 離開

#### 其他操作
* 查詢所有說明文件
```shell
man -f man

// 結果如下
man (1)              - an interface to the system reference manuals
man (7)              - macros to format man pages
```
> alias: `whatis`
* 模糊匹配查詢所有說明文件
```shell
man -k ma
```
> alias: `aprops`

### 其他幫助文件
* `/usr/share/doc/*`: 裡面都是各種程式的幫助文檔

# 正確關機
---
正確順序如下，
1. 觀察系統狀態:
	1. `who`: 查看有哪些用戶正在使用
	2. `ss -tnupl`: 查看網路連接
	3. `ps -aux`: 查看進程
2. 通知線上使用者關機資訊
3. 執行關機指令

### 關機指令
關機都要在 root 用戶下才能使用。
##### sync
```shell
sync # 資料同步寫入磁碟，使用 root 用戶才會全部寫入，使用 user 的話只會寫入 user 資料
```
##### shutdown (功能最多)
* -h: 關機
* -r: 重啟
* -c: 取消進行的關機
```shell
su 
shutdown -r 10 "I will reboot after 10 mins"
shutdown -h now # 現在關機
```
##### reboot
立刻重啟，記得先 `sync`

#### 本質
以上命令本質是調用 `systemctl`
```shell
systemctl reboot # 重啟
systemctl poweroff # 關機
systemctl suspend # 休眠
systemctl halt # 掛起
```