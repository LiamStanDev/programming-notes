# Reactor 與 Proactor 模式 (Reactor and Proactor Patterns)

## 概述

Reactor 與 Proactor 是兩種經典的異步 I/O 架構模式,廣泛應用於高效能網路伺服器。理解兩者的差異與實作細節,是設計 HFT 系統網路層的關鍵。

## Reactor 模式

### 核心概念

```
Reactor 模式 (同步非阻塞 I/O):

1. Reactor: I/O 多路復用器 (epoll, select, poll)
2. Event Handler: 事件處理器
3. Demultiplexer: 事件分發器

流程:
Application -> register handlers -> Reactor (epoll_wait)
  -> events ready -> dispatch -> Event Handler -> process
```

### 單執行緒 Reactor

```cpp
#include <sys/epoll.h>
#include <functional>
#include <unordered_map>

class Reactor {
public:
    using EventCallback = std::function<void(uint32_t events)>;
    
private:
    int epfd_;
    bool running_ = false;
    std::unordered_map<int, EventCallback> handlers_;
    
public:
    Reactor() {
        epfd_ = epoll_create1(EPOLL_CLOEXEC);
        if (epfd_ < 0) {
            throw std::runtime_error("epoll_create1() failed");
        }
    }
    
    ~Reactor() {
        close(epfd_);
    }
    
    // 註冊事件處理器
    void register_handler(int fd, uint32_t events, EventCallback callback) {
        epoll_event ev{};
        ev.events = events | EPOLLET;  // Edge-Triggered
        ev.data.fd = fd;
        
        if (epoll_ctl(epfd_, EPOLL_CTL_ADD, fd, &ev) < 0) {
            throw std::runtime_error("epoll_ctl(ADD) failed");
        }
        
        handlers_[fd] = std::move(callback);
    }
    
    void unregister_handler(int fd) {
        epoll_ctl(epfd_, EPOLL_CTL_DEL, fd, nullptr);
        handlers_.erase(fd);
    }
    
    // 事件循環
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
            
            // 分發事件
            for (int i = 0; i < n; ++i) {
                int fd = events[i].data.fd;
                uint32_t ev = events[i].events;
                
                auto it = handlers_.find(fd);
                if (it != handlers_.end()) {
                    it->second(ev);  // 同步呼叫處理器
                }
            }
        }
    }
    
    void stop() {
        running_ = false;
    }
};
```

### Reactor 完整範例: Echo Server

```cpp
#include <string>
#include <memory>

class Connection {
    int sockfd_;
    Reactor& reactor_;
    std::string read_buffer_;
    std::string write_buffer_;
    
public:
    Connection(int sockfd, Reactor& reactor) 
        : sockfd_(sockfd), reactor_(reactor) {
        
        set_nonblocking(sockfd_);
        
        // 註冊讀事件
        reactor_.register_handler(sockfd_, EPOLLIN, 
            [this](uint32_t events) {
                if (events & EPOLLIN) {
                    handle_read();
                }
                if (events & EPOLLOUT) {
                    handle_write();
                }
                if (events & (EPOLLERR | EPOLLHUP)) {
                    handle_close();
                }
            });
    }
    
    ~Connection() {
        reactor_.unregister_handler(sockfd_);
        close(sockfd_);
    }
    
private:
    void handle_read() {
        char buffer[4096];
        
        while (true) {
            ssize_t n = recv(sockfd_, buffer, sizeof(buffer), 0);
            
            if (n > 0) {
                read_buffer_.append(buffer, n);
            } else if (n == 0) {
                // 連線關閉
                handle_close();
                return;
            } else {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    break;  // 讀取完畢
                }
                handle_close();
                return;
            }
        }
        
        // Echo: 讀取的資料寫回
        write_buffer_ = read_buffer_;
        read_buffer_.clear();
        
        // 啟用寫事件
        modify_events(EPOLLIN | EPOLLOUT);
    }
    
    void handle_write() {
        while (!write_buffer_.empty()) {
            ssize_t n = send(sockfd_, write_buffer_.data(), 
                           write_buffer_.size(), MSG_DONTWAIT);
            
            if (n > 0) {
                write_buffer_.erase(0, n);
            } else {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    break;  // 寫緩衝區滿
                }
                handle_close();
                return;
            }
        }
        
        if (write_buffer_.empty()) {
            // 寫完畢,停用寫事件
            modify_events(EPOLLIN);
        }
    }
    
    void handle_close() {
        delete this;  // 簡化: 實際應使用智慧指標
    }
    
    void modify_events(uint32_t events) {
        epoll_event ev{};
        ev.events = events | EPOLLET;
        ev.data.fd = sockfd_;
        
        // 使用 reactor_.epfd_ 需要友元或 public 介面
        // 此處簡化
    }
};

// Echo Server
class EchoServer {
    Reactor reactor_;
    int server_fd_;
    
public:
    EchoServer(uint16_t port) {
        server_fd_ = create_tcp_server(port);
        set_nonblocking(server_fd_);
        
        // 註冊接受連線處理器
        reactor_.register_handler(server_fd_, EPOLLIN, 
            [this](uint32_t events) {
                if (events & EPOLLIN) {
                    handle_accept();
                }
            });
    }
    
    void run() {
        reactor_.run();
    }
    
private:
    void handle_accept() {
        while (true) {
            int client_fd = accept(server_fd_, nullptr, nullptr);
            
            if (client_fd < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    break;
                }
                throw std::runtime_error("accept() failed");
            }
            
            // 建立連線物件 (自動註冊到 reactor)
            new Connection(client_fd, reactor_);
        }
    }
};

// 使用
int main() {
    EchoServer server(8080);
    server.run();
}
```

### 多執行緒 Reactor (Main Reactor + Sub Reactors)

```cpp
// Netty, Nginx 使用的模式
class MultiThreadReactor {
    Reactor main_reactor_;              // 接受連線
    std::vector<Reactor> sub_reactors_; // 處理 I/O
    std::vector<std::thread> threads_;
    std::atomic<size_t> next_reactor_{0};
    
public:
    explicit MultiThreadReactor(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            sub_reactors_.emplace_back();
            
            threads_.emplace_back([this, i]() {
                sub_reactors_[i].run();
            });
        }
    }
    
    ~MultiThreadReactor() {
        for (auto& reactor : sub_reactors_) {
            reactor.stop();
        }
        for (auto& thread : threads_) {
            thread.join();
        }
    }
    
    void accept_connection(int server_fd) {
        int client_fd = accept(server_fd, nullptr, nullptr);
        set_nonblocking(client_fd);
        
        // Round-Robin 分發到 Sub Reactor
        size_t idx = next_reactor_.fetch_add(1) % sub_reactors_.size();
        
        sub_reactors_[idx].register_handler(client_fd, EPOLLIN,
            [client_fd](uint32_t events) {
                // 處理客戶端 I/O
                handle_client(client_fd, events);
            });
    }
    
    void run() {
        // Main Reactor 僅處理接受連線
        main_reactor_.run();
    }
};
```

## Proactor 模式

### 核心概念

```
Proactor 模式 (真正異步 I/O):

1. Proactor: 異步操作發起器
2. Completion Handler: 完成處理器
3. Asynchronous Operation Processor: 異步操作處理器 (內核)

流程:
Application -> async_read() -> Kernel (處理 I/O)
  -> I/O complete -> Proactor -> Completion Handler

特點:
- 真正異步 (非同步)
- 無需應用層輪詢
- I/O 由內核完成
```

### Proactor 實作: 使用 io_uring

```cpp
#include <liburing.h>
#include <functional>
#include <memory>

class Proactor {
public:
    using CompletionHandler = std::function<void(int result)>;
    
private:
    io_uring ring_;
    bool running_ = false;
    
    struct Operation {
        CompletionHandler handler;
        std::unique_ptr<char[]> buffer;  // 緩衝區生命週期管理
        size_t buffer_size;
    };
    
public:
    Proactor(unsigned entries = 256) {
        if (io_uring_queue_init(entries, &ring_, 0) < 0) {
            throw std::runtime_error("io_uring_queue_init() failed");
        }
    }
    
    ~Proactor() {
        io_uring_queue_exit(&ring_);
    }
    
    // 異步讀取
    void async_read(int fd, size_t size, CompletionHandler handler) {
        auto* op = new Operation{
            .handler = std::move(handler),
            .buffer = std::make_unique<char[]>(size),
            .buffer_size = size
        };
        
        io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
        io_uring_prep_recv(sqe, fd, op->buffer.get(), size, 0);
        io_uring_sqe_set_data(sqe, op);
        
        io_uring_submit(&ring_);
    }
    
    // 異步寫入
    void async_write(int fd, const void* data, size_t size, CompletionHandler handler) {
        auto* op = new Operation{
            .handler = std::move(handler),
            .buffer = std::make_unique<char[]>(size),
            .buffer_size = size
        };
        
        std::memcpy(op->buffer.get(), data, size);
        
        io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
        io_uring_prep_send(sqe, fd, op->buffer.get(), size, 0);
        io_uring_sqe_set_data(sqe, op);
        
        io_uring_submit(&ring_);
    }
    
    // 事件循環
    void run() {
        running_ = true;
        
        while (running_) {
            io_uring_cqe* cqe;
            int ret = io_uring_wait_cqe(&ring_, &cqe);
            
            if (ret < 0) {
                throw std::runtime_error("io_uring_wait_cqe() failed");
            }
            
            // 取得操作上下文
            auto* op = static_cast<Operation*>(io_uring_cqe_get_data(cqe));
            int result = cqe->res;
            
            io_uring_cqe_seen(&ring_, cqe);
            
            // 呼叫完成處理器
            if (op) {
                op->handler(result);
                delete op;
            }
        }
    }
    
    void stop() {
        running_ = false;
    }
};
```

### Proactor 完整範例: Echo Server

```cpp
class ProactorConnection : public std::enable_shared_from_this<ProactorConnection> {
    int sockfd_;
    Proactor& proactor_;
    
public:
    ProactorConnection(int sockfd, Proactor& proactor)
        : sockfd_(sockfd), proactor_(proactor) {
        start_read();
    }
    
    ~ProactorConnection() {
        close(sockfd_);
    }
    
private:
    void start_read() {
        auto self = shared_from_this();
        
        proactor_.async_read(sockfd_, 4096, [this, self](int result) {
            if (result > 0) {
                // 讀取成功,Echo 回去
                handle_read(result);
            } else {
                // 連線關閉或錯誤
                // Connection 將被銷毀 (shared_ptr 計數歸零)
            }
        });
    }
    
    void handle_read(int bytes_read) {
        // 此處可存取讀取的資料 (在 Proactor 的 Operation 中)
        // 簡化: 假設已拷貝到某處
        
        auto self = shared_from_this();
        
        proactor_.async_write(sockfd_, "Echo data", bytes_read, 
            [this, self](int result) {
                if (result > 0) {
                    // 寫入成功,繼續讀取
                    start_read();
                }
                // 寫入失敗,連線將被關閉
            });
    }
};

class ProactorEchoServer {
    Proactor proactor_;
    int server_fd_;
    
public:
    ProactorEchoServer(uint16_t port) {
        server_fd_ = create_tcp_server(port);
        start_accept();
    }
    
    void run() {
        proactor_.run();
    }
    
private:
    void start_accept() {
        // 異步接受連線
        proactor_.async_accept(server_fd_, [this](int client_fd) {
            if (client_fd >= 0) {
                // 建立連線物件
                auto conn = std::make_shared<ProactorConnection>(client_fd, proactor_);
                
                // 繼續接受下一個連線
                start_accept();
            }
        });
    }
};
```

## Reactor vs Proactor 比較

### 實現差異

```cpp
// Reactor (同步非阻塞)
void reactor_read(int fd) {
    // 1. 等待事件 (epoll_wait)
    epoll_wait(...);
    
    // 2. 應用層主動讀取
    ssize_t n = recv(fd, buffer, size, 0);
    
    // 3. 處理資料
    process_data(buffer, n);
}

// Proactor (真正異步)
void proactor_read(int fd) {
    // 1. 發起異步讀取 (內核執行)
    io_uring_prep_recv(...);
    io_uring_submit(...);
    
    // 2. 等待完成通知
    io_uring_wait_cqe(...);
    
    // 3. 資料已由內核讀入 buffer,直接處理
    process_data(buffer, cqe->res);
}
```

### 適用場景

```
Reactor:
✓ 成熟穩定 (epoll 自 Linux 2.5.44)
✓ 適合高並發連線 (C10K)
✓ 應用層控制力強
✗ 需要應用層主動 I/O
✗ 大量小 I/O 有開銷

Proactor:
✓ 真正異步 (內核處理 I/O)
✓ 適合大量小 I/O
✓ CPU 效率高
✗ 相對較新 (io_uring 自 Linux 5.1)
✗ 緩衝區管理複雜
✗ 除錯困難

HFT 推薦:
- 網路 I/O 密集: Proactor (io_uring)
- 計算密集: Reactor (更可控)
- 混合: Reactor + 任務池
```

## HFT 實戰: 混合模式

### Reactor 處理網路 + 任務池處理計算

```cpp
#include <tbb/concurrent_queue.h>
#include <thread>

struct MarketDataTask {
    uint32_t symbol_id;
    double price;
    int64_t volume;
};

class HybridMarketDataProcessor {
    Reactor reactor_;
    tbb::concurrent_queue<MarketDataTask> task_queue_;
    std::vector<std::thread> worker_threads_;
    std::atomic<bool> running_{true};
    
public:
    HybridMarketDataProcessor(size_t num_workers) {
        // 啟動工作執行緒
        for (size_t i = 0; i < num_workers; ++i) {
            worker_threads_.emplace_back([this] {
                worker_loop();
            });
        }
    }
    
    ~HybridMarketDataProcessor() {
        running_ = false;
        for (auto& t : worker_threads_) {
            t.join();
        }
    }
    
    void run(int sockfd) {
        set_nonblocking(sockfd);
        
        // Reactor 處理網路接收
        reactor_.register_handler(sockfd, EPOLLIN, [this, sockfd](uint32_t events) {
            if (events & EPOLLIN) {
                handle_network_read(sockfd);
            }
        });
        
        reactor_.run();
    }
    
private:
    void handle_network_read(int sockfd) {
        char buffer[4096];
        
        while (true) {
            ssize_t n = recv(sockfd, buffer, sizeof(buffer), 0);
            
            if (n > 0) {
                // 解析市場資料
                auto task = parse_market_data(buffer, n);
                
                // 投遞到任務佇列 (Lock-Free)
                task_queue_.push(task);
            } else {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    break;
                }
                return;
            }
        }
    }
    
    void worker_loop() {
        while (running_) {
            MarketDataTask task;
            
            if (task_queue_.try_pop(task)) {
                // 處理市場資料 (計算密集)
                process_market_data(task);
            } else {
                // 無任務,短暫休眠
                std::this_thread::sleep_for(std::chrono::microseconds(10));
            }
        }
    }
    
    void process_market_data(const MarketDataTask& task) {
        // 更新訂單簿、計算指標、執行策略等
        update_order_book(task.symbol_id, task.price, task.volume);
        execute_strategy(task.symbol_id);
    }
};
```

## 檢查清單

- [ ] Reactor 模式使用 epoll Edge-Triggered
- [ ] Reactor 中所有 fd 設為非阻塞
- [ ] Reactor 處理器中讀取所有資料 (直到 EAGAIN)
- [ ] 多執行緒 Reactor 使用 One Loop Per Thread
- [ ] Proactor 正確管理緩衝區生命週期
- [ ] Proactor 使用 io_uring Fixed Files/Buffers
- [ ] HFT 系統考慮混合模式 (Reactor + Worker Pool)
- [ ] 測量 Reactor vs Proactor 的延遲差異
- [ ] 避免在 Reactor/Proactor 中執行阻塞操作
- [ ] 使用 shared_ptr 管理連線物件生命週期

---

## 參考資料 (References)

1. Schmidt, Douglas C. *Reactor: An Object Behavioral Pattern for Demultiplexing and Dispatching Handles for Synchronous Events* (1995)
2. Proactor Design Pattern - POSA2 (Pattern-Oriented Software Architecture)
3. [libuv Design Overview](http://docs.libuv.org/en/v1.x/design.html) - Proactor on Unix
4. [Boost.Asio Documentation](https://www.boost.org/doc/libs/release/doc/html/boost_asio.html)
5. [muduo Network Library](https://github.com/chenshuo/muduo) - Reactor 實作
6. Stevens, W. Richard. *Unix Network Programming, Volume 1* (2003)
