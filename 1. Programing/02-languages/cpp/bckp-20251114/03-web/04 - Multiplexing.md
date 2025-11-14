### 介紹
#### 甚麼是 Multiplexing ?
允許單個進程可以處理多個輸入或輸出通道，而無須為每一個通道使用獨立的線程，常用於網路程式中，用來處理多個客戶端連接。當使用多路復用機制，OS 會監聽指定的文件描述符，當文件描述符準備好進行 I/O，OS 再通知應用程式，使應用程式無須再沒準備好時，等待文件描述符準備，使其不會阻塞等待。

##### Linux 下的多路復用機制有以下三種:
1. select: 最早的多路復用機制之一，用於監視一組文件描述符號。
```cpp
#include <sys/select.h>

int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```
* nfds: 表示文件描述符 bitmap 要開多大，為當前最大文件描述符 + 1。
* readfds: 表示監視**讀事件**的文件描述符集。
* writefds: 表示監視**寫事件**的文件描述符集。
* exceptfds: 表示監視**異常事件**的文件描述符集。
* timeout: 超時時間。

2. poll: 對 select 進行改進，使其能監聽更多的文件描述符，且更容易使用。
```cpp
#include <sys/poll.h> 
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

5. epoll: 被認為是最靈活高效的多路復用機制，採用事件驅動模型，允許監視大量文件描述符。
```cpp
#include <sys/epoll.h>

int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```
* epoll_create: 建立 epoll 實例。
* epoll_ctl: 註冊或刪除文件描述符。
* epoll_wait: 等待文件描述符上的事件。

#### 網路通訊中的事件
##### 讀事件
1. 有新連接: 已連接佇列中已經有準備好的 socket。
2. 對端發送報文已到達: 接收緩存中有數據可讀。
3. tcp 連接以斷開: 對端調用 close()。 
##### 寫事件
1. 可以發送報文: 發送緩衝區沒有滿

### Select 模型
默認使用只能同時監聽 1024 個文件描述符，使用 `int[32] = 1024 bits` 大小的bitmap，當某個 bit 為 1 表示事件發生，0 表示沒發生。
##### 幫助函數
```cpp
void FD_CLR(int fd, fd_set *set); // 從 bitmap 中，去掉指定文件描述符
int  FD_ISSET(int fd, fd_set *set); // 確認指定文件描述符在 bitmap 中
void FD_SET(int fd, fd_set *set); // 添加指定文件描述符到 bitmap 中
void FD_ZERO(fd_set *set); // 清空 bitmap
```

##### 使用 select 建立服務端
```cpp
#include "server.hpp"
#include <bits/types/struct_timeval.h>
#include <cstring>
#include <netinet/in.h>
#include <sys/select.h>
#include <sys/socket.h>
#include <unistd.h>

int main()
{
  const unsigned short PORT = 5003;
  int listenfd = initserver(PORT);

  std::cout << "listen on port " << PORT << "." << std::endl;

  fd_set readfds{}; // 建立 bitmap 大小為 1024

  FD_ZERO(&readfds); // 清空 bitmap
  FD_SET(listenfd, &readfds); // 添加監聽 socket
  int maxfd = listenfd; // 目前最大為監聽 socket

  char buffer[1024]; // 接收信息的 buffer
  memset(buffer, 0, sizeof(buffer));

  while (true) // event loop
  {
	  // 設定超時時間
    timeval timeout{}; 
    timeout.tv_sec = 5; // 秒
    timeout.tv_usec = 10000; // 毫秒
		// 因為 select 會修改傳入的 bitmap，所以要拷貝一份。
    fd_set tempfds = readfds;

		// select 會返回觸發事件的文件描述符數量
    int numfd = select(maxfd + 1, &tempfds, nullptr, nullptr, &timeout);
	
    if (numfd < 0)
    {
      std::cerr << "select() fail." << std::endl;
      return -1;
    }
    else if (numfd == 0)
    {
      std::cout << "select() timeout." << std::endl;
      continue;
    }
		// 發生事件後，遍歷所有文件描述符
    for (int eventfd = 0; eventfd <= maxfd; eventfd++)
    {
	    // 文件描述符事件沒有發生: bitmap 中他的值為 0
      if (FD_ISSET(eventfd, &tempfds) == 0)
      {
        continue;
      }

      // 對端有連接: 因為 listenfd 的讀事件只有這個會發生。
      if (eventfd == listenfd)
      {
        int connect_sock = accept(listenfd, nullptr, nullptr);
        if (connect_sock == -1)
        {
          std::cerr << "accept() fail." << std::endl;
        }
        std::cout << "accept connected_sock: " << connect_sock << " OK!"
                  << std::endl;
				// 添加到 bitmap 中 (注意不是 tempfds)
        FD_SET(connect_sock, &readfds);

        maxfd = connect_sock > maxfd ? connect_sock : maxfd;
      }
      // 不是 listen fd 的話有兩種可能的讀事件
      // 1. 對端關閉 tcp 連接
      // 2. 有訊息可以接收
      else
      {
        // 對端關閉 tcp 連接
        if (::recv(eventfd, buffer, sizeof(buffer), 0) <= 0) // 0 表示關閉連接
        {
          std::cout << "client " << eventfd << " disconnected." << std::endl;
          ::close(eventfd);
          FD_CLR(eventfd, &readfds);

          if (eventfd == maxfd)
          {
            for (int i = maxfd; i >= 0; i--) // 由後往前檢查，最大 fd 
            {
              if (FD_ISSET(i, &readfds))
              {
                maxfd = i;
              }
            }
          }
        }
        // 有訊息可以接收
        else
        {
          std::cout << "Received from " << eventfd << ": " << buffer
                    << std::endl;
          if (::send(eventfd, buffer, sizeof(buffer), 0) <= 0)
          {
            std::cerr << "send() fail." << std::endl;
            close(eventfd);
            return -1;
          }
          std::cout << "Send: " << buffer << std::endl;
        }
      }
    }
  }

  return 0;
}
```
##### 細節
1. 為甚麼要使用 event loop + timeout 每次都重新 select?
		event loop，用來刷新新一輪的事件。
		timeout: 若長時間沒有事件發生，可以做其他週期性事件，後再重新進入 select
2. 水平觸發: 當上一輪 select 已經通知的事件，但該輪沒有處理，在下一輪還是會再次通知。
##### 性能問題
1. 採用輪詢的方式來掃描 bitmap，會隨者監聽的文件描述符增加而性能下降。
2. 每次調用 select 會進行兩次拷貝: 
	1. 因為 select 會修改 bitmap 所以要自行拷貝一份 e.g. `tempfds`
	2. select 會將 bitmap 從用戶態拷貝到內核態
3. FD_SETSIZE 可以設定 bitmap 大小 (莫認為 1024)，但數量增多性能會下降。


### poll 模型
poll 為 select 的改進，擴大監聽的數量、減少拷貝次數並使其更容易使用。
##### pollfd 結構體
```cpp
struct pollfd {
	 int   fd;         /* file descriptor */
	 short events;     /* requested events */
	 short revents;    /* returned events */
};
```
> 有事件發生 poll 只會修改 revents，不會修改 events，比 select 更方便。

* 網路中常用的 event 種類
	* `POLLIN`: 讀事件
	* `POLLOUT`: 寫事件
	* `POLLIN & POLLOUT`: 讀寫事件
##### 使用 poll 模型建立服務端
除了 `fds` 使用方式不同，幾乎與 poll 一樣。

```cpp
int main()
{
  const unsigned short PORT = 5003;
  int listenfd = initserver(PORT);

  std::cout << "listen on port " << PORT << "." << std::endl;

  // 可以自訂義結構體數組大小。
  const size_t POLL_FD_SIZE = 2048;
  pollfd fds[POLL_FD_SIZE];

  // 初始化 fdz
  for (size_t i = 0; i < POLL_FD_SIZE; i++)
  {
    fds[i].fd = -1; // 設定 fd 為 -1 poll 會忽略他。
  }

  fds[listenfd].fd = listenfd;
  fds[listenfd].events = POLLIN; // 設定讀事件

  int maxfd = listenfd;

  char buffer[1024];
  memset(buffer, 0, sizeof(buffer));

  while (true) // event loop
  {
    int numfd = poll(fds, maxfd + 1, 10000); // 這邊 timeout 單位為毫秒, -1 表示不啟用超時

    if (numfd < 0)
    {
      std::cerr << "poll() fail." << std::endl;
      return -1;
    }
    else if (numfd == 0)
    {
      std::cout << "poll() timeout." << std::endl;
      continue;
    }

    for (int eventfd = 0; eventfd <= maxfd; eventfd++)
    {
      if (fds[eventfd].fd < 0) // 忽略
      {
        continue;
      }

      if ((fds[eventfd].revents & POLLIN) == 0) // 沒有讀事件
      {
        continue;
      }

      // 用戶端連接
      if (eventfd == listenfd)
      {
        int connect_sock = accept(listenfd, nullptr, nullptr);
        if (connect_sock == -1)
        {
          std::cerr << "accept() fail." << std::endl;
        }
        std::cout << "accept connected_sock: " << connect_sock << " OK!"
                  << std::endl;

        fds[connect_sock].fd = connect_sock;
        fds[connect_sock].events = POLLIN;

        maxfd = connect_sock > maxfd ? connect_sock : maxfd;
      }
      // 連接中斷或者有訊息接收
      else
      {
				// 連接中斷
        if (::recv(eventfd, buffer, sizeof(buffer), 0) <= 0)
        {
          std::cout << "client " << eventfd << " disconnected." << std::endl;
          ::close(eventfd);
          fds[eventfd].fd = -1; // 設定其為忽略

          if (eventfd == maxfd)
          {
            for (int i = maxfd; i >= 0; i--)
            {
              if (fds[i].fd != -1) // 由後往前檢查，最大 fd
              {
                maxfd = i;
              }
            }
          }
        }
        // 有訊息可接收
        else
        {
          std::cout << "Received from " << eventfd << ": " << buffer
                    << std::endl;
          if (::send(eventfd, buffer, sizeof(buffer), 0) <= 0)
          {
            std::cerr << "send() fail." << std::endl;
            close(eventfd);
            return -1;
          }
          std::cout << "Send: " << buffer << std::endl;
        }
      }
    }
  }

  return 0;
}

```
##### 細節
1. 水平觸發: 當上一輪 select 已經通知的事件，但該輪沒有處理，在下一輪還是會再次通知。
2. poll 傳入結構體數組，傳入內核後轉換成鏈表。
##### 性能問題
1. 每次調用 poll 會進行一次拷貝: 
	1. poll 會將結構體數組從用戶態拷貝到內核態
2. poll 沒有 1024 監視限制，但是同 select 一樣使用遍歷，故 socket 越多效率越低。

### epoll 模型
##### epoll_create
```cpp
// 自從 Linux 2.6.8，裡面參數已被忽略，可以填非零的整數。 
int epollfd = epoll_create(1);
```
##### epoll event
用來綁定文件描述符與事件。
```cpp
// 用於 epoll 監聽事件後返回的內容，可以指定另外別的東西，不僅限於自己。
typedef union epoll_data {
	void *ptr; // 任意數據的數組
	int fd; // 文件描述符號
	uint_32_t u32;
	uint64_t u64;
} epoll_data_t;

struct epoll_event 
{
	uint32_t events; // epoll 事件
	epoll_data_t data;
};
```

##### 使用 epoll 實現服務端
```cpp
#include "server.hpp"
#include <bits/types/struct_timeval.h>
#include <cstring>
#include <netinet/in.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

int main()
{
  const unsigned short PORT = 5003;
  int listenfd = initserver(PORT);

  std::cout << "listen on port " << PORT << "." << std::endl;

  // 建立 epollfd 句柄 (文件描述符)
  int epollfd = epoll_create(1);

  epoll_event event; // 建立事件
  event.data.fd = listenfd; // 設定事件觸發後，返回的值，常用於指定處理對象。
  event.events = EPOLLIN; // 讀取事件，與 poll 類似。

  epoll_ctl(epollfd, EPOLL_CTL_ADD, listenfd, &event); // 綁定文件描述符與事件。

  const size_t MAX_EVENTS = 10;
  epoll_event events[MAX_EVENTS]; // 返回事件收集數組，每次調用 epoll 都會從頭開始放。

  char buffer[1024];
  memset(buffer, 0, sizeof(buffer));

  while (true) // event loop
  {

    int numevent = epoll_wait(epollfd, events, MAX_EVENTS, -1); // -1 表示不使用超時

    if (numevent < 0)
    {
      std::cerr << "epoll() fail." << std::endl;
      return -1;
    }
    else if (numevent == 0)
    {
      std::cout << "epoll() timeout." << std::endl;
      continue;
    }

    for (int i = 0; i < numevent; i++)
    {
      // 客戶端連接
      if (events[i].data.fd == listenfd)
      {
        int connect_sock = accept(listenfd, nullptr, nullptr);
        if (connect_sock == -1)
        {
          std::cerr << "accept() fail." << std::endl;
        }
        std::cout << "accept connected_sock: " << connect_sock << " OK!"
                  << std::endl;

        event.data.fd = connect_sock;
        event.events = EPOLLIN;
        epoll_ctl(epollfd, EPOLL_CTL_ADD, connect_sock, &event);
      }
      else
      {

        // 客戶端斷開連接
        if (::recv(events[i].data.fd, buffer, sizeof(buffer), 0) <= 0)
        {
          std::cout << "client " << events[i].data.fd << " disconnected."
                    << std::endl;
          ::close(events[i].data.fd);
          // 以下可以不用寫，因為調用 close 後文件描述符會自動從 epoll 中刪除。
          // epoll_ctl(epollfd, EPOLL_CTL_DEL, events[i].data.fd, nullptr);
        }
        // 客戶端發送信息
        else
        {
          std::cout << "Received from " << events[i].data.fd << ": " << buffer
                    << std::endl;
          if (::send(events[i].data.fd, buffer, sizeof(buffer), 0) <= 0)
          {
            std::cerr << "send() fail." << std::endl;
            close(events[i].data.fd);
            return -1;
          }
          std::cout << "Send: " << buffer << std::endl;
        }
      }
    }
  }

  return 0;
}
```
##### 細節
epoll 提供兩種觸發方式: 水平觸發與邊緣觸發。

###### 水平觸發(默認)
* 讀事件: 在調用 `epoll_wait()` 後，觸發了讀取事件後並沒有讀完，再次調用 `epoll_wait()` 會再次觸發讀事件。
* 寫事件: 若此次寫入緩衝區沒有滿，再次調用 `epoll_wait()` 會再次觸發寫事件。
###### 邊緣觸發
* 讀事件: 在調用 `epoll_wait()` 後，觸發了讀取事件後並沒有讀完，再次調用 `epoll_wait()` **不會再次觸發讀事件**，直到有**新的數據到達才會再次觸發**。
* 寫事件: 若此次寫入緩衝區沒有滿，再次調用 `epoll_wait()` 不會再次觸發寫事件，**直到緩衝區從滿變成不滿時才會再次觸發**。
```cpp
epoll_event event; // 建立事件
event.data.fd = listenfd; // 設定事件觸發後，返回的值，常用於指定處理對象。
event.events = EPOLLIN | EPOLLET; // 啟用邊緣觸發模式(ET)。
```

> 邊緣觸發在上的問題:
> 	1. 當有多個連接同時進入，我們一次只 accept 一個用戶，在邊緣觸發下非有新的連接，不然 connection queue 的永遠都不會 accept。
> 	2. 當讀取時 buffer 滿了，進入下一次事件循環，會導致不觸發讀取事件。
>
> 解決方式:
> 	1. 採用非阻塞的 accept 循環的 accept 直到 connection queue 為空 `if ( (connect_fd == -1) && (errno == EAGAIN) )`。
> 	2. 採用非阻塞的 recv 循環的 recv 直到讀取為空 `if ( (readn == -1) && (errno == EAGAIN) )`。

#### epoll 原理
