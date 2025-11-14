# Async Rust In Detail

## 操作系統層知識

### Firmware
韌體就像是嵌入在硬體設備內部的「迷你」軟體 ，它直接控制著硬體的基本操作和功能。不同於 OS 是安裝在硬碟上，通常是固化在唯獨記憶體中 (ROM)。

#### 韌體在非同步的作用與工作方式
1. 直接控制硬體
    * 網卡韌體知道如何接收與發送網路封包
    * 硬碟韌體知道如何讀取與寫入硬碟
2. 事件偵查與通知
    * 外設事件 (網卡接收到封包)，為控制器上的韌體檢測到狀態變化
    * 韌體進行通知系統
3. 觸發中斷 (Interrupts)
    * 韌體監測事件之後，指示硬體向 CPU 發送中斷訊號 (透過中斷請求線 IRQ 發送)
    * 中斷信號打斷 CPU 執行任務，強制 CPU 處裡中斷，透過 OS 的中斷處裡程序 (Interrupt Handler，通常為裝置驅動程式的一部分) 處理。
4. DMA (可選)
    * 當資料需要傳輸時（例如從網路卡接收資料到記憶體），韌體會設定好 DMA 控制器，讓資料直接在裝置和主記憶體之間傳輸， 繞過 CPU ，CPU 無需介入每個位元組的傳輸。
    * 傳輸完成後，韌體（或 DMA 控制器）才會觸發中斷，通知 CPU「資料已經準備好了」。
5. 非同步關鍵 - 『通知』而非『輪詢』
    * OS 收到韌體發起的外設中斷，就知道 I/O 操作可以往下推進。然後 OS 尋找哪個應用程序等待該事件，並喚起等待的任務 (Rust 中就是呼叫 Waker)，讓非同步執行運行時 (Excuter) 可以重新開始執行。

### 事件隊列 (Event Queue)
事件隊列是 OS 用於管理異步事件的數據結構，功能為：
* 事件存儲
* 事件調度
* 解耦生產與消費：事件產生與處裡分離

#### 場景
* GUI 事件：用戶點擊, 鍵盤輸入等事件通過 Event Queue 傳遞給應用程序
* 異步 I/O：網卡收到數據後觸發中斷，OS 將事件加入 Event Queue 中
* 定時任務：定時器到期事件通過 Event Queue 進行調度


### epoll/kqueue/IOCP
| **機制**     | **操作系统**  | **核心原理**                            | **與事件隊列的關聯**                                                                                                                                         |
| ---------- | --------- | ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **epoll**  | Linux     | 基于**红黑树+就绪链表**，仅返回活跃的文件描述符（FD）。     | 内核维护一个**I/O事件队列**，epoll通过`epoll_wait`从队列中获取就绪事件[4](https://www.cnblogs.com/fqlGlog/articles/7536449.html)[7](https://gpp.tkchu.me/event-queue.html)。 |
| **kqueue** | BSD/macOS | 支持多种事件类型（文件、信号、定时器等），通过**事件过滤器**管理。 | 内核将事件分类存储到**多个队列**，kqueue通过`kevent`统一监听和提取事件[4](https://www.cnblogs.com/fqlGlog/articles/7536449.html)，[7](https://gpp.tkchu.me/event-queue.html)。   |
| **IOCP**   | Windows   | 基于**完成端口模型**，与线程池结合实现异步I/O。         | 内核将I/O操作结果直接写入**完成队列**，应用程序通过`GetQueuedCompletionStatus`轮询队列[7](https://gpp.tkchu.me/event-queue.html)。                                              |

以網路服務接收連接為例，epoll 工作流程如下：
1. **註冊事件**：應用程序通過 `epoll_ctl` 將 Socket FD 註冊到 epoll 實例，內核將該事件添加到**監聽隊列**
2. **事件觸發**：網卡收到數據後發起中斷，內核將該 FD 標記為就緒，事件移入**就緒隊列**
3. **事件獲取**：`epoll_wait` 從就緒隊列中取出事件，返回給應用程序
4. **事件處裡**：應用程序根據事件類型(可讀, 可寫)執行回調或者線程任務

> 註1：註冊事件就是註冊回調始得可以進行事件觸發
> 註2：`epoll_wait` 若沒有取得事件會阻塞當前線程

## 建立異步庫

### 理解 epoll

##### 使用 epoll 實現 TCP 監聽
```c
// 建立 epoll 實例
int epfd = epoll_create1(0) 

// 監聽 socket 並初始化
int listen_sock = socket(...);
bind(...);
listen(...);

// 註冊 socket FD 到 epoll
// 1. 建立 epoll 事件
struct epoll_event ev; 
ev.events = EPOLLIN | EPOLLET; // 設定監聽可讀事件並設置為邊緣觸發
ev.data.fd = listen_sock; // 添加 socket FD
// 2. 添加
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_sock, &ev);

// 事件循環
struct epoll_event events[MAX_EVENTS];
while(1) {
	// 等待通知
    int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
}
```

##### mio
Rust 異步生態中最出名的 tokio，底層依賴的是 `mio`，這類似於 python 中 uvloop 背後的 `libuv`，本質上都是對 OS 提供的事件隊列接口進行封裝。

##### ET vs LT 

| **特性**   | **LV (Level Trigger) 默認**  | **ET (Edge Trigger) - `EPOLLET`** |
| -------- | -------------------------- | --------------------------------- |
| **觸發條件** | 只要文件描述符**處於就緒**            | 文件描述符**從未就緒變為就緒**那一刻              |
| **通知次數** | 只要條件滿足，每次 epoll_wait 都可能通知 | 狀態變化時**僅通知一次**                    |
| **數據處裡** | 可以不一次完成讀/寫處裡，下次還會通知        | 必須一次性完成讀/寫                        |
| **複雜程度** | 相對簡單，不容易丟失事件               | 較高，必須正確處裡循環讀/寫，否則可能丟失數據           |
| **效率**   | 可能有冗餘通知，效率稍低               | 通知次數少，效率通常更高                      |
| **配合模式** | 可與阻塞與非阻塞 I/O 配合，但非阻塞更常用    | 必須與非阻塞 I/O 配合                     |

### 


## Reference

* [bilibili Rust 異步編程](https://www.bilibili.com/video/BV1E8d8YcEdw?spm_id_from=333.788.player.switch&vd_source=81381cbff2baf8dfd1b22050d029f496&p=2)
