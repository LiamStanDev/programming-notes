# epoll 與 io_uring 詳解 (epoll and io_uring Deep Dive)

## 概述

epoll 是 Linux 高效能 I/O 多路復用機制,是 HFT 系統的基石。io_uring (Kernel 5.1+) 是新一代異步 I/O 介面,提供更低延遲與更高吞吐量。

## epoll 深入剖析

### epoll 三大系統調用

```cpp
#include <sys/epoll.h>

// 1. epoll_create1: 建立 epoll 實例
int epfd = epoll_create1(EPOLL_CLOEXEC);
// EPOLL_CLOEXEC: exec 時自動關閉

// 2. epoll_ctl: 控制 epoll 實例
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// op: EPOLL_CTL_ADD, EPOLL_CTL_MOD, EPOLL_CTL_DEL

// 3. epoll_wait: 等待事件
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

### Level-Triggered vs Edge-Triggered

```cpp
// Level-Triggered (預設, LT)
epoll_event ev{};
ev.events = EPOLLIN;  // 只要可讀就觸發
ev.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

// Edge-Triggered (高效能, ET)
ev.events = EPOLLIN | EPOLLET;  // 僅在狀態變化時觸發
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

// ET 模式要求:
// 1. fd 必須是非阻塞的
// 2. 必須讀取所有資料直到 EAGAIN
while (true) {
    ssize_t n = recv(sockfd, buffer, sizeof(buffer), 0);
    if (n < 0) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            break;  // 無更多資料
        }
        handle_error();
    }
    process_data(buffer, n);
}
```

**效能差異**:
- **LT**: 每次 `epoll_wait` 都可能返回相同事件 (未處理完的 fd)
- **ET**: 僅在狀態變化時觸發,減少系統調用次數

### EPOLLONESHOT - 避免競態條件

```cpp
// 多執行緒環境下,避免同一 fd 被多個執行緒同時處理
epoll_event ev{};
ev.events = EPOLLIN | EPOLLET | EPOLLONESHOT;
ev.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

// 處理完成後需重新註冊
void handle_request(int sockfd) {
    // 處理資料...
    
    // 重新啟用監控
    epoll_event ev{};
    ev.events = EPOLLIN | EPOLLET | EPOLLONESHOT;
    ev.data.fd = sockfd;
    epoll_ctl(epfd, EPOLL_CTL_MOD, sockfd, &ev);
}
```

### epoll_event.data 聯合體

```cpp
// epoll_event 結構
typedef union epoll_data {
    void    *ptr;
    int      fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;  // EPOLLIN, EPOLLOUT, EPOLLERR, ...
    epoll_data_t data;
};

// 使用 ptr 儲存自訂資料
struct Connection {
    int sockfd;
    std::string buffer;
    // ... 其他狀態
};

epoll_event ev{};
ev.events = EPOLLIN | EPOLLET;
ev.data.ptr = new Connection{sockfd};  // 儲存連線物件指標
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

// epoll_wait 取回
epoll_event events[MAX_EVENTS];
int n = epoll_wait(epfd, events, MAX_EVENTS, -1);

for (int i = 0; i < n; ++i) {
    auto* conn = static_cast<Connection*>(events[i].data.ptr);
    handle_connection(conn);
}
```

## epoll 實戰: Reactor 模式

### 單執行緒 Reactor

```cpp
#include <sys/epoll.h>
#include <unordered_map>
#include <functional>

class EpollReactor {
public:
    using EventHandler = std::function<void(uint32_t events)>;
    
private:
    int epfd_;
    bool running_ = false;
    std::unordered_map<int, EventHandler> handlers_;
    
public:
    EpollReactor() {
        epfd_ = epoll_create1(EPOLL_CLOEXEC);
        if (epfd_ < 0) {
            throw std::runtime_error("epoll_create1() failed");
        }
    }
    
    ~EpollReactor() {
        close(epfd_);
    }
    
    void register_handler(int fd, uint32_t events, EventHandler handler) {
        epoll_event ev{};
        ev.events = events | EPOLLET;
        ev.data.fd = fd;
        
        if (epoll_ctl(epfd_, EPOLL_CTL_ADD, fd, &ev) < 0) {
            throw std::runtime_error("epoll_ctl(ADD) failed");
        }
        
        handlers_[fd] = std::move(handler);
    }
    
    void unregister_handler(int fd) {
        epoll_ctl(epfd_, EPOLL_CTL_DEL, fd, nullptr);
        handlers_.erase(fd);
    }
    
    void run() {
        constexpr int MAX_EVENTS = 1024;
        epoll_event events[MAX_EVENTS];
        running_ = true;
        
        while (running_) {
            int n = epoll_wait(epfd_, events, MAX_EVENTS, -1);
            
            if (n < 0) {
                if (errno == EINTR) continue;
                throw std::runtime_error("epoll_wait() failed");
            }
            
            for (int i = 0; i < n; ++i) {
                int fd = events[i].data.fd;
                uint32_t ev = events[i].events;
                
                auto it = handlers_.find(fd);
                if (it != handlers_.end()) {
                    it->second(ev);  // 呼叫處理函數
                }
            }
        }
    }
    
    void stop() {
        running_ = false;
    }
};

// 使用範例
void example_usage() {
    EpollReactor reactor;
    
    int server_fd = create_tcp_server(8080);
    set_nonblocking(server_fd);
    
    // 註冊接受連線處理器
    reactor.register_handler(server_fd, EPOLLIN, [&](uint32_t events) {
        if (events & EPOLLIN) {
            int client_fd = accept(server_fd, nullptr, nullptr);
            set_nonblocking(client_fd);
            
            // 註冊客戶端讀取處理器
            reactor.register_handler(client_fd, EPOLLIN, [&, client_fd](uint32_t ev) {
                if (ev & EPOLLIN) {
                    char buffer[4096];
                    ssize_t n = recv(client_fd, buffer, sizeof(buffer), 0);
                    
                    if (n <= 0) {
                        reactor.unregister_handler(client_fd);
                        close(client_fd);
                    } else {
                        process_data(buffer, n);
                    }
                }
            });
        }
    });
    
    reactor.run();
}
```

### 多執行緒 Reactor (One Loop Per Thread)

```cpp
class MultiThreadReactor {
    std::vector<std::unique_ptr<EpollReactor>> reactors_;
    std::vector<std::thread> threads_;
    std::atomic<size_t> next_reactor_{0};
    
public:
    explicit MultiThreadReactor(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            auto reactor = std::make_unique<EpollReactor>();
            auto* r = reactor.get();
            
            reactors_.push_back(std::move(reactor));
            threads_.emplace_back([r] { r->run(); });
        }
    }
    
    ~MultiThreadReactor() {
        for (auto& reactor : reactors_) {
            reactor->stop();
        }
        for (auto& thread : threads_) {
            thread.join();
        }
    }
    
    EpollReactor* get_next_reactor() {
        size_t idx = next_reactor_.fetch_add(1, std::memory_order_relaxed) % reactors_.size();
        return reactors_[idx].get();
    }
};
```

## io_uring 簡介

### io_uring 架構

```
User Space                 Kernel Space
┌──────────────┐          ┌──────────────┐
│ Submission   │ ────────>│ Processing   │
│ Queue (SQ)   │          │              │
└──────────────┘          └──────────────┘
                                 │
┌──────────────┐                 │
│ Completion   │ <───────────────┘
│ Queue (CQ)   │
└──────────────┘
```

**優勢**:
1. **零拷貝**: SQ/CQ 與內核共享記憶體
2. **批次提交**: 一次系統調用提交多個請求
3. **真正異步**: 無需阻塞等待
4. **統一介面**: 支援網路、檔案、eventfd 等所有 I/O

### io_uring 基礎使用

```cpp
#include <liburing.h>

class IoUringContext {
    io_uring ring_;
    
public:
    explicit IoUringContext(unsigned entries = 256) {
        if (io_uring_queue_init(entries, &ring_, 0) < 0) {
            throw std::runtime_error("io_uring_queue_init() failed");
        }
    }
    
    ~IoUringContext() {
        io_uring_queue_exit(&ring_);
    }
    
    // 提交接收請求
    void submit_recv(int sockfd, void* buffer, size_t len, void* user_data) {
        io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
        if (!sqe) {
            throw std::runtime_error("io_uring_get_sqe() failed");
        }
        
        io_uring_prep_recv(sqe, sockfd, buffer, len, 0);
        io_uring_sqe_set_data(sqe, user_data);
    }
    
    // 提交發送請求
    void submit_send(int sockfd, const void* buffer, size_t len, void* user_data) {
        io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
        io_uring_prep_send(sqe, sockfd, buffer, len, 0);
        io_uring_sqe_set_data(sqe, user_data);
    }
    
    // 批次提交所有請求
    int submit() {
        return io_uring_submit(&ring_);
    }
    
    // 等待完成事件
    int wait_cqe(io_uring_cqe** cqe) {
        return io_uring_wait_cqe(&ring_, cqe);
    }
    
    void cqe_seen(io_uring_cqe* cqe) {
        io_uring_cqe_seen(&ring_, cqe);
    }
};
```

### io_uring 非阻塞輪詢

```cpp
// 使用 IORING_SETUP_SQPOLL: 內核執行緒處理 SQ
IoUringContext ring(256);

// 提交請求
ring.submit_recv(sockfd, buffer, sizeof(buffer), &context);
ring.submit();

// 輪詢完成事件 (非阻塞)
io_uring_cqe* cqe;
while (true) {
    int ret = io_uring_peek_cqe(&ring.ring_, &cqe);
    
    if (ret == 0) {
        // 有完成事件
        auto* ctx = static_cast<Context*>(io_uring_cqe_get_data(cqe));
        handle_completion(ctx, cqe->res);
        
        ring.cqe_seen(cqe);
    } else if (ret == -EAGAIN) {
        // 無事件,繼續其他工作
        do_other_work();
    }
}
```

### HFT 應用: io_uring 市場資料接收

```cpp
#include <liburing.h>

struct RecvContext {
    int sockfd;
    char buffer[4096];
    size_t buffer_size = sizeof(buffer);
};

class IoUringMarketDataReceiver {
    io_uring ring_;
    std::vector<std::unique_ptr<RecvContext>> contexts_;
    
public:
    IoUringMarketDataReceiver(const std::vector<int>& sockfds) {
        io_uring_queue_init(256, &ring_, 0);
        
        // 為每個 socket 準備接收請求
        for (int fd : sockfds) {
            auto ctx = std::make_unique<RecvContext>();
            ctx->sockfd = fd;
            
            io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
            io_uring_prep_recv(sqe, fd, ctx->buffer, ctx->buffer_size, 0);
            io_uring_sqe_set_data(sqe, ctx.get());
            
            contexts_.push_back(std::move(ctx));
        }
        
        io_uring_submit(&ring_);
    }
    
    ~IoUringMarketDataReceiver() {
        io_uring_queue_exit(&ring_);
    }
    
    void run() {
        while (true) {
            io_uring_cqe* cqe;
            int ret = io_uring_wait_cqe(&ring_, &cqe);
            
            if (ret < 0) {
                throw std::runtime_error("io_uring_wait_cqe() failed");
            }
            
            auto* ctx = static_cast<RecvContext*>(io_uring_cqe_get_data(cqe));
            int bytes_received = cqe->res;
            
            if (bytes_received > 0) {
                // 處理市場資料
                process_market_data(ctx->buffer, bytes_received);
                
                // 重新提交接收請求
                io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
                io_uring_prep_recv(sqe, ctx->sockfd, ctx->buffer, ctx->buffer_size, 0);
                io_uring_sqe_set_data(sqe, ctx);
                io_uring_submit(&ring_);
            } else {
                // 連線關閉或錯誤
                handle_error(ctx->sockfd);
            }
            
            io_uring_cqe_seen(&ring_, cqe);
        }
    }
};
```

## io_uring 進階功能

### Fixed Files (固定檔案描述符)

```cpp
// 避免每次 I/O 查找 fd,降低開銷
int fds[] = {sockfd1, sockfd2, sockfd3};

io_uring_register_files(&ring_, fds, 3);

// 使用固定 fd 索引
io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
io_uring_prep_recv(sqe, 0, buffer, len, 0);  // 0 為索引,非實際 fd
sqe->flags |= IOSQE_FIXED_FILE;
```

### Fixed Buffers (固定緩衝區)

```cpp
// 註冊緩衝區,避免內核重複 pin memory
iovec iovecs[2] = {
    {.iov_base = buffer1, .iov_len = 4096},
    {.iov_base = buffer2, .iov_len = 4096}
};

io_uring_register_buffers(&ring_, iovecs, 2);

// 使用固定緩衝區
io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
io_uring_prep_read_fixed(sqe, fd, buffer1, 4096, 0, 0);  // 最後的 0 為緩衝區索引
```

### Linked Requests (鏈式請求)

```cpp
// 請求 A 成功後自動執行請求 B
io_uring_sqe* sqe1 = io_uring_get_sqe(&ring_);
io_uring_prep_recv(sqe1, sockfd, buffer, len, 0);
sqe1->flags |= IOSQE_IO_LINK;  // 鏈接下一個請求

io_uring_sqe* sqe2 = io_uring_get_sqe(&ring_);
io_uring_prep_send(sqe2, sockfd, response, resp_len, 0);

// sqe1 成功後自動執行 sqe2,失敗則取消 sqe2
io_uring_submit(&ring_);
```

## 效能比較

### Benchmark: epoll vs io_uring

```cpp
#include <chrono>

// epoll 版本
void benchmark_epoll(int num_requests) {
    int epfd = epoll_create1(0);
    auto start = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < num_requests; ++i) {
        epoll_event events[1];
        epoll_wait(epfd, events, 1, 0);
    }
    
    auto duration = std::chrono::high_resolution_clock::now() - start;
    auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(duration).count();
    
    std::cout << "epoll: " << ns / num_requests << " ns/op\n";
}

// io_uring 版本
void benchmark_io_uring(int num_requests) {
    io_uring ring;
    io_uring_queue_init(256, &ring, 0);
    
    auto start = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < num_requests; ++i) {
        io_uring_cqe* cqe;
        io_uring_peek_cqe(&ring, &cqe);
        if (cqe) io_uring_cqe_seen(&ring, cqe);
    }
    
    auto duration = std::chrono::high_resolution_clock::now() - start;
    auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(duration).count();
    
    std::cout << "io_uring: " << ns / num_requests << " ns/op\n";
    
    io_uring_queue_exit(&ring);
}

// 典型結果 (無實際 I/O):
// epoll:    ~500 ns/op
// io_uring: ~100 ns/op (5x faster)
```

## 何時使用何種機制

```cpp
// epoll: 成熟穩定,適用於
// - Kernel < 5.1 的系統
// - 純網路 I/O (socket only)
// - 低並發場景 (< 10,000 connections)

// io_uring: 高效能,適用於
// - Kernel >= 5.1 的系統
// - 混合 I/O (網路 + 檔案)
// - 高並發場景 (> 10,000 connections)
// - 低延遲要求 (HFT)
```

## 調校與監控

### 檢查 io_uring 支援

```bash
# 檢查內核版本
uname -r  # >= 5.1

# 檢查 io_uring 系統調用
strace -e io_uring_setup,io_uring_enter ./program 2>&1 | grep io_uring
```

### 調整 io_uring 參數

```cpp
// 增大 SQ/CQ 大小
io_uring_params params{};
params.sq_entries = 2048;  // 預設 128
params.cq_entries = 4096;  // 預設 2 * sq_entries

io_uring_queue_init_params(2048, &ring, &params);

// 啟用 SQPOLL (內核執行緒處理 SQ)
params.flags = IORING_SETUP_SQPOLL;
params.sq_thread_idle = 2000;  // 2 秒無操作後休眠

// CPU 綁定
params.flags |= IORING_SETUP_SQ_AFF;
params.sq_thread_cpu = 0;  // 綁定到 CPU 0
```

### 監控 epoll 使用

```bash
# 檢視程式的 epoll fd
ls -l /proc/<pid>/fd | grep eventpoll

# 統計 epoll_wait 調用
strace -c -e epoll_wait ./program
```

## 檢查清單

- [ ] epoll 使用 Edge-Triggered 模式 (EPOLLET)
- [ ] ET 模式下讀取所有資料直到 EAGAIN
- [ ] 使用 `epoll_event.data.ptr` 儲存連線上下文
- [ ] 多執行緒環境使用 EPOLLONESHOT
- [ ] io_uring 使用 Fixed Files/Buffers 減少開銷
- [ ] io_uring 批次提交請求 (batch submit)
- [ ] 檢查內核版本是否支援 io_uring (>= 5.1)
- [ ] HFT 系統考慮 io_uring 的延遲優勢
- [ ] 使用 strace 監控系統調用次數
- [ ] Benchmark 驗證 epoll vs io_uring 效能差異

---

## 參考資料 (References)

1. [epoll(7) - Linux Manual Page](https://man7.org/linux/man-pages/man7/epoll.7.html)
2. [io_uring - Efficient I/O with io_uring](https://kernel.dk/io_uring.pdf)
3. [liburing GitHub Repository](https://github.com/axboe/liburing)
4. Axboe, Jens. *Efficient IO with io_uring* (2019)
5. [Lord of the io_uring](https://unixism.net/loti/) - Tutorial Series
6. [io_uring by Example](https://github.com/shuveb/io_uring-by-example)
