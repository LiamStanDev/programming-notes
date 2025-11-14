### 簡介
* 阻塞式 I/O: 在網路編程中 `connect`, `accept`, `send`, 與 `receive` 會阻塞，所以該線程會等待該函數的返回值。
* 非阻塞式 I/O: 當調用 I/O 時立即返回，故不會導致阻塞。

#### 為何要非阻塞?
當我們在使用 `while(true)` 事件循環時，我們會透過 epoll 或其他方式來獲取事件通知，所以在**事件循環中所有的操作都不應該阻塞**，而是直接返回讓事件循環進行處理。

#### fcntl
fcntl: 為 file control，於對文件描述符進行控制的系統調用，它可以用來設置或獲取文件描述符的各種屬性，包括文件描述符的狀態標誌、設置文件描述符為非阻塞模式、設置或獲取文件描述符的鎖，等等。
##### 函數簽名
```cpp
#include <fcntl.h>

int fcntl(int fd, int cmd, ... /* arg */ );
```
* cmd 的常標誌作有
	* `F_GETFL`: File-get file flag，獲取文件描述符
	* `F_SETFL`: File-set file flag，設置文件描述符
* 依據不同 cmd，後續提供的參數會有所不同。
* 返回值為: -1 表示錯誤。

##### 設置文件描述符為非阻塞
```cpp
int set_nonblocking(int fd)
{
	// 獲取 fd 的 flags
	int flags = fcntl(sockfd, F_GETFL, 0); 
	if (flags == -1) 
	{ 
		perror("fcntl"); return -1; 
	}
	
	// 額外添加 O_NONBLOCK (O 表示 Options) 選項。 
	return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}
```

### 實現非阻塞
#### 非阻塞 connect
非阻塞的 connection 不會等待三次握手，會立刻返回 -1 並且設置全局錯誤變數 `errno` 為 **`EPROGRESS`**。
##### 實現概念:
1. 設置 `O_NONBLOCK`
2. 檢查是否真的失敗: 使用 `EPROGRESS`
3. 連接最後是否成功: 若 socket 可以寫表示連接已成功，使用 poll 或者 select 檢查。

```cpp
// ...
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
if (sockfd == -1)
{
  std::cout << "socket() fail." << std::endl;
  return -1;
}

// 設置為非阻塞模式 
if (set_nonblocking(sockfd) == -1) 
{ 
	close(sockfd); 
	return 1; 
}

if (::connect(sockfd, reinterpret_cast<struct sockaddr *>(&addr),
              sizeof(addr)) == -1)
{
	if (errno != EINPROGRESS) // 不為 EINPROGRESS 才算失敗
	{ 
		std::cout << "connect() fail." << std::endl;
	  ::close(sockfd);
	  return -1;
	}
}

pollfd fds; // 只要一個就好了，不用使用結構體數組
fds.events = POLLOUT; // 監聽寫事件
poll(&fds, 1, -1);  
if (fds.revents == POLLOUT)
{
	print("Connection OK.");
}
else
{
	print("Connection Fail.");
}

return 0;
```

#### 非阻塞 accept
非阻塞的 accept 不會查看 connection queue 中是否有連接，會立刻返回 -1 並且設置全局錯誤變數 `errno` 為 **`EAGAIN`**。
##### 實現概念:
1. 設置 `O_NONBLOCK`
2. 檢查是否真的失敗: 使用 `EAGAIN`
3. 連接最後是否成功: 若 socket 可以寫表示連接已成功，使用 poll 或者 select 檢查。

```cpp
int listenfd = socket(AF_INET, SOCK_STREAM, 0); 
// bind()..
// listen()..
// 設置為非阻塞模式 
if (set_nonblocking(sockfd) == -1) 
{ 
	close(sockfd); 
	return 1; 
}

int connect_fd = accept(sockfd, reinterpret_cast<struct sockaddr*>(&clientAddr), &clientAddrLen);
if (connect_fd == -1)
{
	if (errno != EAGAIN) // 表示錯誤不為 non-blocking 導致
	{
		return -1;
	}
}
```

> send 與 recv 同 accept 一樣會設置 errno 為 `EAGAIN`。


