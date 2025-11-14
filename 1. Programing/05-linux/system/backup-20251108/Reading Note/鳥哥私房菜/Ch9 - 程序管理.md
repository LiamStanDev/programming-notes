# 程序管理
---
### Bash job 管理
Bash 進程中也能管理多個任務。但要注意若 bash 也退出那他管理的所有 job 也會退出。
#### 背景運行
使用 `&` 就可以在 Bash 中背景運行程序，並會返回該程序 PID 
```shell
tar -zpcf /tmp/etc.tar.gz /etc &

# 啟動返回 PID
[1] 14432  <== [job number] PID
# 結束返回 Done
[1]+  Done                    tar -zpcf test/etc.tar.gz /etc

# 此時螢幕還是會顯示資訊，故我們可以使用重導向
tar -zpfvc /tmp/etc.tar.gz /etc > /tmp/log.txt 2>&1 &
```

#### 背景暫停
這用於當我們使用 vim 修改文件時，需要查找別的資料，我們可以先將 vim 背景暫停，查找完之後在切回前景。
```shell
vim /etc/passwd
# 按下 ctrl + z 切換前景
[1] Stopped
# 做一些事情
# 切回前景
fg %1 # 後面的數字爲 job number
```

#### 觀察背景工作狀態
```shell
jobs -l
```

#### 將背景暫停轉為背景運行
```shell
find / -perm /7000 > /tmp/text.txt
# 按下 ctlr + z

# 背景運行並觀察
jobs; bg %1; jobs
```

#### 管理背景工作
```shell
# 使用方式
kill -signal [jobnumber]

# -1  發送重新讀取設定檔 reload
kill -1 %2; jobs
# -2 代表鍵盤輸入 ctrl + c
kill -2 %2; jobs
# -9 立即刪除工作
kill -9 %2; jobs
# -15 正常退出工作 (爲 kill 預設值)
kill -15 %2; jobs
```
> 這邊注意一下 kill 也能向 process 發送信號，不單只用於 bash jobs，故 kill 後面接的數字默認爲 PID，使用 `%` 作前綴才是 bash job


### 程序管理
#### ps 程序運行的快照
```shell
ps -aux # -a 表示全部、u 表示 effective user 不使用表示自己、x 詳細列出
```
STAT 表示運行狀態有以下幾種：
* R: Running
* S: Sleep
* D: 監聽外設，無法被喚醒，可能在等待 IO
* T: Stop 工作暫停可能爲背景暫停或者 Debug
* Z: Zombie 程序以終止但是還佔用記憶體

#### top: 動態觀察程序
```shell
top 
```
排序可以在進入 top 中後按下：
* `P`: 以 CPU 使用率排序
* `M` 以 Mem 使用率排序

#### pstree: 程序的依賴樹
```shell
pstree
```

#### kill/killall: 發送信號
```shell
kill -9 [PID]
killall -9 [command name]
```
> 只要記得 1, 9, 15 的信號就可以了


### 系統資源
#### free: 觀察記憶體使用情況
```shell
free -h # human redable
free -m # MBytes 單位
```
> 這邊可以注意是否有使用到 Swap，盡量不要使用到 Swap 會很慢

#### uptime: 系統啟動時間
```shell
uptime
```

#### ss/netstat: 追蹤網路
```shell
ss -tnupl # t: tcp, u: udp, p: show PID, l: listening, n: show port
```
* Recv-Q： 對方收到的 bytes 數
* Send-Q： 對發發送的 byptes 數
* Local Address: 本地 IP 端口
* Peer Address: 對方 IP 端口

#### vmstat: 偵測系統資源變化
```shell
vmstat 1 3 # 每秒一次紀錄三次
```
結果如下：
```text
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 0  0      0 23127184   6768 9821612    0    0    45   169 1092    2  1  1 99  0  0  0
 0  0      0 23117136   6768 9821548    0    0     0   225 3282 4263  1  1 98  0  0  0
 0  0      0 23124676   6768 9821612    0    0     0   222 2823 3707  1  1 98  0  0  0
```
* procs
	* r：表示等待的程序數量
	* b：表示掛起等待的程序數量
* swap
	* si: swap in，從 swap 取出的量
	* so: swap out，因 memory 不足寫到 swap 的量
* io
	* bi: block in，由磁碟讀入的區塊數量
	* bo: block out，寫入到磁碟去的區塊數量
* system
	* in：每秒被中斷的程序次數
	* cs：每秒鐘進行的事件切換次數
* cpu
    us：非核心層的 CPU 使用狀態
    sy：核心層所使用的 CPU 狀態
    id：閒置的狀態
    wa：等待 I/O 所耗費的 CPU 狀態
    st：被虛擬機器 (virtual machine) 所盜用的 CPU 使用狀態。
    
### /proc 目錄
進程存在於記憶體中，而記憶體中的資料會被映射到 `/proc/*` 之下，主機運行的每個程序的 PID 都是以目錄的形式存在於 `/proc` 中。其中 `/proc` 目錄下有幾個重要的文件：

| 檔名                | 檔案內容                                                                                                                                                                                            |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| /proc/cmdline     | 載入 kernel 時所下達的相關指令與參數！查閱此檔案，可瞭解指令是如何啟動的！                                                                                                                                                       |
| /proc/cpuinfo     | 本機的 CPU 的相關資訊，包含時脈、類型與運算功能等                                                                                                                                                                     |
| /proc/devices     | 這個檔案記錄了系統各個主要裝置的主要裝置代號，與 [mknod](https://linux.vbird.org/linux_basic/0230filesystem.php#mknod) 有關呢！                                                                                             |
| /proc/filesystems | 目前系統已經載入的檔案系統囉！                                                                                                                                                                                 |
| /proc/interrupts  | 目前系統上面的 IRQ 分配狀態。                                                                                                                                                                               |
| /proc/ioports     | 目前系統上面各個裝置所配置的 I/O 位址。                                                                                                                                                                          |
| /proc/kcore       | 這個就是記憶體的大小啦！好大對吧！但是不要讀他啦！                                                                                                                                                                       |
| /proc/loadavg     | 還記得 [top](https://linux.vbird.org/linux_basic/centos7/0440processcontrol.php#topm) 以及 [uptime](https://linux.vbird.org/linux_basic/centos7/0440processcontrol.php#uptime) 吧？沒錯！上頭的三個平均數值就是記錄在此！ |
| /proc/meminfo     | 使用 [free](https://linux.vbird.org/linux_basic/centos7/0440processcontrol.php#free) 列出的記憶體資訊，嘿嘿！在這裡也能夠查閱到！                                                                                       |
| /proc/modules     | 目前我們的 Linux 已經載入的模組列表，也可以想成是驅動程式啦！                                                                                                                                                              |
| /proc/mounts      | 系統已經掛載的資料，就是用 mount 這個指令呼叫出來的資料啦！                                                                                                                                                               |
| /proc/swaps       | 到底系統掛載入的記憶體在哪裡？呵呵！使用掉的 partition 就記錄在此啦！                                                                                                                                                        |
| /proc/partitions  | 使用 fdisk -l 會出現目前所有的 partition 吧？在這個檔案當中也有紀錄喔！                                                                                                                                                  |
| /proc/uptime      | 就是用 uptime 的時候，會出現的資訊啦！                                                                                                                                                                         |
| /proc/version     | 核心的版本，就是用 uname -a 顯示的內容啦！                                                                                                                                                                      |
| /proc/bus/*       | 一些匯流排的裝置，還有 USB 的裝置也記錄在此喔！                                                                                                                                                                      |
#### fuser: 找到使用該檔案的程序
* file user
```shell
sudo fuser -uv /dir/file # -u: 顯示 user, v: 詳細訊息
```

#### lsof: 列出被程序開啟的檔案
* list open file
```shell
lsof -u liam | grep bash
```

# SELinux
---