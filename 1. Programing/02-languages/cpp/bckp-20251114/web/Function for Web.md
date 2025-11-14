# 字節序轉換
---
字節序表示當一個數據大小超過一個byte時，該如何存放或者進行網路傳輸。
### 主機字節序
以`0x12345678`作為例子，以下地址由低到高
#### 大端序（big-endian）
* 表示大地址存放結尾
* 結果：12 23 56 78
* 使用於：Power PC, MIPS UNIX
#### 小端序（little-endian）
* 表示小地址存放結尾
* 結果：78 56 34 12
* 使用於：intel x86, ARM

### 網路字節序
在TCP/IP規定使用big-endian作為網路字節序。
#### 轉換函數
* 頭文件`#include <arpa/inet.h>`
1. `htons()` : 2 bytes, host to net short
2. `htonl()` : 4 bytes, host to net long
3. `ntohs()`
4. `ntohl()`

# 地址轉換函數
---
頭文件 `#include <arpa/inet.h>`
### inet_addr
用於將字符串轉為ipv4網路字節序表達。
```cpp
sockaddr_in addr;
addr.sin_addr.s_addr = inet_addr("192.168.0.1");
```

### inet_pton
用於將字符串依照指定ip協議寫入到sin_addr或者sin6_addr結構體中。（更通用）
```cpp
sockaddr_in6 addr;
inet_pton(AF_INET6, "::1", &addr.sin6_addr);
```

# Socket函數
---
有關面向socket編程的函數。
頭文件：`#include <sys/socket.h>`

#### 查看文檔方法
```shell
man 2 socket
```
* 1 : 用戶命令
* 2 : 系統調用
* 3 : C語言庫
* 5 : 文件格式

## 通用函數
### socket()
用來創建一個新的socket對象，也就是系統申請socket資源。
#### 函數聲明
```cpp
int socket(int domain, int type, int protocol);
```
#### 參數含義
* domain: ip 協議族
	* AF_INET(or PF_INET), AF_INET6(or PF_INET6)
* type: 表示socket類型
	* SOCK_STREAM : 面向連接、可靠傳輸與雙向的通訊，讀取與寫入是連續的，不必關注數據的邊界。TCP就是基於此。
	* SOCK_DATAGRAM：無連接、不可靠的通訊，應用程式需要自行處理數據邊界。UDP基於此。
	* SOCK_RAW : 用於自己建構底層網路的協以使用。
* protocol : 設定是什麼協議有以下選項。
	* 0 : 表示依據type字型推斷，如SOCK_STREAM就是IPPROTO_TCP
	* IPPROTO_TCP
	* IPPROTO_UDP
	* IPPROTO_ICMP
#### 返回值
返回文件描述符，為大於等於3的整數。
> 文件描述符，當使用open(), socket()等系統調用時，會返回一個文件描述符，用來唯一識別該文件，描述符作為其他系統函數來操作該文件。
##### 為何大於等於3?
因為1, 2, 3分別為標準輸入、標準輸出與標準錯誤。
##### 如何看一個進程能打開多少文件描述符
```shell
ulimit -a
```

```shell
結果為： 
-t: cpu time (seconds) unlimited 
-f: file size (blocks) unlimited 
-d: data seg size (kbytes) unlimited 
-s: stack size (kbytes) 8192 
-c: core file size (blocks) unlimited 
-m: resident set size (kbytes) unlimited 
-u: processes 62787 
-n: file descriptors 1024 <- 表示一個進程可以使用的最大文件描述符
-l: locked-in-memory size (kbytes) 8192 
-v: address space (kbytes) unlimited 
-x: file locks unlimited 
-i: pending signals 62787 
-q: bytes in POSIX msg queues 819200 
-e: max nice 0 
-r: max rt priority 0 
-N 15: rt cpu time (microseconds) unlimited
```
##### 修改最大限制
* 編輯`/etc/security/limits.conf`
```shell
添加 
* hard nofile 10000 
* soft nofile 10000
```


### bind()
將創建好的socket綁定到指定ip與端口上。
#### 函數聲明
```cpp
int bind(int sockfd, const struct sockaddr* addr, socklen_t addrlen);
```
#### 參數含義
* addr：類型為sockaddr指針，要自行轉換。
* socklen_t : 要傳入地址長度，因為sockaddr_in6長度比sockaddr還長。詳細請看[[Structure for Web#為什麼可以比socketaddr大，那這樣參數傳遞是否有問題？|ipv6長度]]
#### 返回值
* -1 表示失敗, 0 表示成功。
> 為什麼unix使用-1表示失敗？因為成功只有一種可能，失敗有很多種，雖然0表示flase
#### 失敗原因
* 權限不足：使用端口小於等於1024 (系統保留端口)。
* 超過端口最大值：unsigned short 最大值。
* 端口已經被佔用。

## TCP函數
### listen()
用於TCP服務端，表示開始接收客戶端連接請求，在這時候客戶端使用connection()方法會與客戶端進行TCP三次握手，然後將建立好的連接請求放入connection queue等待accept取出。
#### 函數聲明
```cpp
int listen(int sockfd, int backlog);
```
#### 參數含義
* backlog : 表示要建立多大的connection queue
> connection queue 用來作為緩衝區，因為服務端是逐一accept，所以需要一個緩衝區。若緩衝區佔滿則新的連接請求則會被中斷。
> linux中listen會建立兩個對列分別為已連接的connection queue, 與半連接的queue，半連接表示客戶端只收到對方的SYN包但是還沒收到SYN+ACK包。而backlog是指定已連接對列的長度。
##### 半連接對列長度如何設定？
ans : 寫入/proc/sys/net/ipv4/tcp_max_syn_backlog文件
#### 返回值
* -1 表示失敗, 0 表示成功。

### connect()
為TCP客戶端向服務端發起連接的函數，若客戶端queue未滿就會進行三次握手。
#### 函數聲明
```cpp
int connect(int sockfd,  const struct sockaddr* addr, socklen_t addrlen);
```
#### 參數含義
* sockfd : 客戶端向系統申請的socket文件描述符
* addr : 服務端地址與端口
* addrlen : 地址長度
#### 返回值
* -1 表示失敗, 0 表示成功。
### accept()
用於TCP服務端，從connection queue獲取連接，若對列為空則阻塞。
#### 函數聲明
```cpp
int acceopt(int sockfd, struct sockaddr* _Nullable restrict addr,
		   socklen_t *_Nullable restrict addrlen);
```
#### 參數含義
* sockfd : 監聽的資源使用的文件描述符
* addr : 可空，會將傳入sockaddr對象寫入客戶端地址
* addrlen : 可空，會將傳入參數寫入地址長度
#### 返回值
* 文件描述符，用來表示與某個客戶端溝通管道的資源，是系統抽象給用戶的句柄，可以用來寫入與讀取與客戶端進行溝通。注：與監聽使用的文件描述符不同。

### send()
用於TCP，在已建立連接通道中發送數據。若緩衝區沒有空間則會等待。
#### 函數聲明
```cpp
ssize_t recv(int sockfd, const void* buf, size_t len, int flags);
```
#### 參數含義
* sockfd : 已經建立連接通道的文件描述符。
* buf : 要發送數據的地址。可以是任何東西。(要考慮字節序)
	* 字符串不用考慮字節序：因為每一個字符是一個字節。(ASCII)
* len : 發送數據的長度, 單位為byte
* flags : 都填0就好，其他的也沒什麼意義
#### 返回值
* 返回成功發送的字節數，發送失敗會返回-1, 並標記錯誤信息errno
* 連接斷開send不會報錯，要等幾秒才會報錯
* <= 0 表示此通信已經不可用

### recv()
用於TCP，在已建立連接通道中獲取數據。若緩衝區沒有據則會等待。
#### 函數聲明
```cpp
ssize_t recv(int sockfd, void* buf, size_t len, int flags);
```
#### 參數含義
* sockfd : 已經建立連接通道的文件描述符。
* buf : 緩衝區指針，用來存放數據的地方要與接收數據匹配。(要考慮字節序)
* len : 緩衝區大小，單位為byte
* flags : 都填0就好，其他的也沒什麼意義
#### 返回值
* 返回成功接收的字節數，發送失敗會返回-1
* 0 表示連接斷開(連接斷開recv函數會立刻返回)
* <=0 表示此通信已經不能使用。

## UDP函數(以下未完成)
### sendto()
從connection queue獲取請求，若對列為空則阻塞。
#### 函數聲明
```cpp
int acceopt(int sockfd, struct sockaddr* _Nullable restrict addr,
		   socklen_t *_Nullable restrict addrlen);
```
#### 參數含義
* sockfd : 監聽的資源使用的文件描述符
* addr : 可空，會將傳入sockaddr對象寫入客戶端地址
* addrlen : 可空，會將傳入參數寫入地址長度
#### 返回值
* 文件描述符，用來表示與某個客戶端連接的資源，是系統抽象給用戶的句柄，可以用來寫入與讀取與客戶端進行溝通。注：與監聽使用的文件描述符不同。
### recvfrom()
從connection queue獲取請求，若對列為空則阻塞。
#### 函數聲明
```cpp
int acceopt(int sockfd, struct sockaddr* _Nullable restrict addr,
		   socklen_t *_Nullable restrict addrlen);
```
#### 參數含義
* sockfd : 監聽的資源使用的文件描述符
* addr : 可空，會將傳入sockaddr對象寫入客戶端地址
* addrlen : 可空，會將傳入參數寫入地址長度
#### 返回值
* 文件描述符，用來表示與某個客戶端連接的資源，是系統抽象給用戶的句柄，可以用來寫入與讀取與客戶端進行溝通。注：與監聽使用的文件描述符不同。
