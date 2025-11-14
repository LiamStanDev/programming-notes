# C++20 Coroutine 與異步 IO (C++20 Coroutines and Asynchronous IO)

## 概述

C++20 引入協程 (Coroutines),提供語言層級的異步程式設計支援。協程可在執行中暫停 (suspend) 與恢復 (resume),非常適合處理 I/O 密集型任務,如網路通訊、檔案讀寫等。

## 協程基礎

### 三個關鍵字

```cpp
// co_await: 暫停協程,等待異步操作完成
co_await async_operation();

// co_yield: 產生值並暫停 (用於生成器)
co_yield value;

// co_return: 返回值並結束協程
co_return result;
```

### 協程要求

```cpp
#include <coroutine>

// 協程必須返回特定型別,包含以下方法:
struct Task {
    struct promise_type {
        Task get_return_object() { /* ... */ }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() {}
    };
    
    std::coroutine_handle<promise_type> handle_;
};

// 使用
Task my_coroutine() {
    co_await std::suspend_always{};  // 使用 co_await 即為協程
    co_return;
}
```

## 簡單協程範例: Generator

### 惰性序列生成器

```cpp
#include <coroutine>
#include <iostream>

template<typename T>
struct Generator {
    struct promise_type {
        T current_value;
        
        Generator get_return_object() {
            return Generator{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        
        std::suspend_always yield_value(T value) {
            current_value = value;
            return {};
        }
        
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
    
    std::coroutine_handle<promise_type> handle_;
    
    explicit Generator(std::coroutine_handle<promise_type> h) : handle_(h) {}
    
    ~Generator() {
        if (handle_) handle_.destroy();
    }
    
    // 移動語意
    Generator(Generator&& other) noexcept : handle_(other.handle_) {
        other.handle_ = nullptr;
    }
    
    bool move_next() {
        handle_.resume();
        return !handle_.done();
    }
    
    T current_value() const {
        return handle_.promise().current_value;
    }
};

// 使用 Generator
Generator<int> fibonacci() {
    int a = 0, b = 1;
    
    while (true) {
        co_yield a;
        auto next = a + b;
        a = b;
        b = next;
    }
}

int main() {
    auto gen = fibonacci();
    
    for (int i = 0; i < 10; ++i) {
        gen.move_next();
        std::cout << gen.current_value() << " ";
    }
    // 輸出: 0 1 1 2 3 5 8 13 21 34
}
```

## 異步任務: Task

### 基礎 Task 實作

```cpp
#include <coroutine>
#include <exception>
#include <optional>

template<typename T>
struct Task {
    struct promise_type {
        std::optional<T> result_;
        std::exception_ptr exception_;
        
        Task get_return_object() {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        
        void return_value(T value) {
            result_ = std::move(value);
        }
        
        void unhandled_exception() {
            exception_ = std::current_exception();
        }
    };
    
    std::coroutine_handle<promise_type> handle_;
    
    explicit Task(std::coroutine_handle<promise_type> h) : handle_(h) {}
    
    ~Task() {
        if (handle_) handle_.destroy();
    }
    
    Task(Task&& other) noexcept : handle_(other.handle_) {
        other.handle_ = nullptr;
    }
    
    T get() {
        if (!handle_.done()) {
            handle_.resume();
        }
        
        if (handle_.promise().exception_) {
            std::rethrow_exception(handle_.promise().exception_);
        }
        
        return *handle_.promise().result_;
    }
    
    // Awaiter 介面
    bool await_ready() const noexcept {
        return handle_.done();
    }
    
    void await_suspend(std::coroutine_handle<> awaiting) {
        // 簡化: 實際需更複雜的調度邏輯
        handle_.resume();
    }
    
    T await_resume() {
        return get();
    }
};

// 使用範例
Task<int> async_add(int a, int b) {
    // 模擬異步計算
    co_return a + b;
}

Task<int> compute() {
    int result = co_await async_add(10, 20);
    co_return result * 2;
}

int main() {
    auto task = compute();
    std::cout << "Result: " << task.get() << "\n";  // 60
}
```

## 異步 I/O: io_uring 整合

### Linux io_uring Awaiter

```cpp
#include <liburing.h>
#include <coroutine>

class IoUringAwaiter {
    io_uring* ring_;
    io_uring_sqe* sqe_;
    io_uring_cqe* cqe_ = nullptr;
    
public:
    IoUringAwaiter(io_uring* ring, io_uring_sqe* sqe) 
        : ring_(ring), sqe_(sqe) {}
    
    bool await_ready() const noexcept { return false; }
    
    void await_suspend(std::coroutine_handle<> handle) {
        // 設定用戶資料為協程 handle
        io_uring_sqe_set_data(sqe_, handle.address());
        
        // 提交請求
        io_uring_submit(ring_);
    }
    
    int await_resume() {
        // 返回 I/O 結果
        io_uring_wait_cqe(ring_, &cqe_);
        int res = cqe_->res;
        io_uring_cqe_seen(ring_, cqe_);
        return res;
    }
};

// 異步讀取檔案
Task<std::vector<char>> async_read_file(io_uring* ring, int fd, size_t size) {
    std::vector<char> buffer(size);
    
    io_uring_sqe* sqe = io_uring_get_sqe(ring);
    io_uring_prep_read(sqe, fd, buffer.data(), size, 0);
    
    int bytes_read = co_await IoUringAwaiter{ring, sqe};
    
    if (bytes_read < 0) {
        throw std::runtime_error("Read failed");
    }
    
    buffer.resize(bytes_read);
    co_return buffer;
}
```

### HFT 應用: 異步市場資料接收

```cpp
#include <sys/socket.h>
#include <netinet/in.h>

struct MarketData {
    uint32_t symbol_id;
    double price;
    int64_t volume;
    uint64_t timestamp;
};

Task<MarketData> async_receive_market_data(io_uring* ring, int sockfd) {
    char buffer[1024];
    
    io_uring_sqe* sqe = io_uring_get_sqe(ring);
    io_uring_prep_recv(sqe, sockfd, buffer, sizeof(buffer), 0);
    
    int bytes_received = co_await IoUringAwaiter{ring, sqe};
    
    if (bytes_received <= 0) {
        throw std::runtime_error("Connection closed");
    }
    
    // 解碼市場資料 (簡化)
    MarketData md;
    std::memcpy(&md, buffer, sizeof(MarketData));
    
    co_return md;
}

// 協程式市場資料處理循環
Task<void> market_data_loop(io_uring* ring, int sockfd) {
    while (true) {
        try {
            MarketData md = co_await async_receive_market_data(ring, sockfd);
            
            // 處理市場資料
            update_order_book(md);
            
            // 執行交易策略
            co_await execute_strategy_async(md);
            
        } catch (const std::exception& e) {
            std::cerr << "Error: " << e.what() << "\n";
            break;
        }
    }
    
    co_return;
}
```

## 協程調度器

### 簡單執行緒池調度器

```cpp
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>

class ThreadPoolScheduler {
    std::vector<std::thread> workers_;
    std::queue<std::coroutine_handle<>> tasks_;
    std::mutex mtx_;
    std::condition_variable cv_;
    bool stop_ = false;
    
public:
    explicit ThreadPoolScheduler(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this] { worker_loop(); });
        }
    }
    
    ~ThreadPoolScheduler() {
        {
            std::lock_guard<std::mutex> lock(mtx_);
            stop_ = true;
        }
        cv_.notify_all();
        
        for (auto& w : workers_) {
            w.join();
        }
    }
    
    void schedule(std::coroutine_handle<> handle) {
        {
            std::lock_guard<std::mutex> lock(mtx_);
            tasks_.push(handle);
        }
        cv_.notify_one();
    }
    
private:
    void worker_loop() {
        while (true) {
            std::coroutine_handle<> task;
            
            {
                std::unique_lock<std::mutex> lock(mtx_);
                cv_.wait(lock, [this] { return stop_ || !tasks_.empty(); });
                
                if (stop_ && tasks_.empty()) {
                    return;
                }
                
                task = tasks_.front();
                tasks_.pop();
            }
            
            task.resume();
        }
    }
};

// 可調度的 Task
template<typename T>
struct ScheduledTask {
    struct promise_type {
        ThreadPoolScheduler* scheduler_ = nullptr;
        std::optional<T> result_;
        
        ScheduledTask get_return_object() {
            return ScheduledTask{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        
        std::suspend_always initial_suspend() { return {}; }
        
        struct FinalAwaiter {
            bool await_ready() noexcept { return false; }
            void await_suspend(std::coroutine_handle<promise_type> h) noexcept {
                // 最終暫停時不調度
            }
            void await_resume() noexcept {}
        };
        
        FinalAwaiter final_suspend() noexcept { return {}; }
        
        void return_value(T value) {
            result_ = std::move(value);
        }
        
        void unhandled_exception() { std::terminate(); }
    };
    
    std::coroutine_handle<promise_type> handle_;
    
    void schedule_on(ThreadPoolScheduler& scheduler) {
        handle_.promise().scheduler_ = &scheduler;
        scheduler.schedule(handle_);
    }
    
    T get() {
        while (!handle_.done()) {
            std::this_thread::yield();
        }
        return *handle_.promise().result_;
    }
};
```

## cppcoro 函式庫

### 使用 cppcoro 簡化協程開發

```cpp
// cppcoro 提供生產級協程基礎設施
#include <cppcoro/task.hpp>
#include <cppcoro/sync_wait.hpp>
#include <cppcoro/when_all.hpp>

// 簡單異步任務
cppcoro::task<int> compute_async(int x) {
    co_return x * 2;
}

// 組合多個異步任務
cppcoro::task<int> parallel_compute() {
    auto [a, b, c] = co_await cppcoro::when_all(
        compute_async(10),
        compute_async(20),
        compute_async(30)
    );
    
    co_return a + b + c;  // 60 + 40 + 120 = 120
}

int main() {
    int result = cppcoro::sync_wait(parallel_compute());
    std::cout << "Result: " << result << "\n";
}
```

### 異步生成器

```cpp
#include <cppcoro/generator.hpp>

cppcoro::generator<int> range(int start, int end) {
    for (int i = start; i < end; ++i) {
        co_yield i;
    }
}

// 使用
for (int i : range(0, 10)) {
    std::cout << i << " ";
}
```

## HFT 實戰: 協程式訂單管理

### 異步訂單生命週期

```cpp
#include <cppcoro/task.hpp>
#include <cppcoro/sync_wait.hpp>
#include <chrono>

enum class OrderStatus {
    Created, Submitted, Acknowledged, Filled, Rejected
};

struct Order {
    uint64_t order_id;
    uint32_t symbol_id;
    double price;
    int64_t quantity;
    OrderStatus status;
};

// 模擬異步等待回應
cppcoro::task<OrderStatus> wait_for_ack(uint64_t order_id) {
    // 實際會等待網路回應
    std::this_thread::sleep_for(std::chrono::microseconds(100));
    co_return OrderStatus::Acknowledged;
}

cppcoro::task<OrderStatus> wait_for_fill(uint64_t order_id) {
    std::this_thread::sleep_for(std::chrono::microseconds(500));
    co_return OrderStatus::Filled;
}

// 協程式訂單提交流程
cppcoro::task<Order> submit_order_async(Order order) {
    // 1. 提交訂單
    order.status = OrderStatus::Submitted;
    send_to_exchange(order);
    
    // 2. 等待確認 (異步)
    order.status = co_await wait_for_ack(order.order_id);
    log_order_status(order);
    
    // 3. 等待成交 (異步)
    order.status = co_await wait_for_fill(order.order_id);
    log_order_status(order);
    
    co_return order;
}

// 批次提交訂單
cppcoro::task<std::vector<Order>> submit_batch_async(std::vector<Order> orders) {
    std::vector<cppcoro::task<Order>> tasks;
    
    for (auto& order : orders) {
        tasks.push_back(submit_order_async(std::move(order)));
    }
    
    // 並行等待所有訂單完成
    auto results = co_await cppcoro::when_all(std::move(tasks));
    
    co_return results;
}
```

## 協程 vs 執行緒

### 效能比較

```cpp
#include <chrono>
#include <iostream>
#include <thread>

// 傳統執行緒
void thread_benchmark() {
    constexpr int N = 10000;
    auto start = std::chrono::high_resolution_clock::now();
    
    std::vector<std::thread> threads;
    for (int i = 0; i < N; ++i) {
        threads.emplace_back([] { /* work */ });
    }
    
    for (auto& t : threads) t.join();
    
    auto duration = std::chrono::high_resolution_clock::now() - start;
    std::cout << "Threads: " << duration.count() << " ns\n";
}

// 協程
cppcoro::task<void> coroutine_task() {
    co_return;
}

void coroutine_benchmark() {
    constexpr int N = 10000;
    auto start = std::chrono::high_resolution_clock::now();
    
    std::vector<cppcoro::task<void>> tasks;
    for (int i = 0; i < N; ++i) {
        tasks.push_back(coroutine_task());
    }
    
    cppcoro::sync_wait(cppcoro::when_all(std::move(tasks)));
    
    auto duration = std::chrono::high_resolution_clock::now() - start;
    std::cout << "Coroutines: " << duration.count() << " ns\n";
}

// 典型結果:
// Threads:    ~1000 ms (context switch 開銷大)
// Coroutines: ~10 ms   (用戶態切換,開銷小)
```

### 記憶體使用

```
執行緒:      ~2 MB stack per thread (Linux 預設)
協程:        ~幾百 bytes per coroutine (取決於局部變數)

10,000 個執行緒:    ~20 GB 記憶體
10,000 個協程:      ~幾 MB 記憶體
```

## 最佳實踐

### 1. 避免協程中的阻塞操作

```cpp
// 錯誤: 阻塞整個執行緒
cppcoro::task<void> bad_coroutine() {
    std::this_thread::sleep_for(std::chrono::seconds(1));  // 阻塞!
    co_return;
}

// 正確: 使用異步等待
cppcoro::task<void> good_coroutine(cppcoro::io_service& io) {
    co_await io.schedule_after(std::chrono::seconds(1));  // 異步等待
    co_return;
}
```

### 2. 正確管理協程生命週期

```cpp
// 錯誤: 協程可能在返回前被銷毀
void dangerous() {
    auto task = async_operation();  // 協程立即銷毀!
}

// 正確: 確保協程完成
void safe() {
    auto task = async_operation();
    cppcoro::sync_wait(task);  // 等待完成
}
```

### 3. 例外處理

```cpp
cppcoro::task<int> may_throw() {
    if (error_condition) {
        throw std::runtime_error("Error");
    }
    co_return 42;
}

cppcoro::task<void> handle_errors() {
    try {
        int result = co_await may_throw();
        std::cout << "Result: " << result << "\n";
    } catch (const std::exception& e) {
        std::cerr << "Caught: " << e.what() << "\n";
    }
}
```

## 檢查清單

- [ ] 理解協程的三個關鍵字: `co_await`, `co_yield`, `co_return`
- [ ] 實作或使用現有的 `promise_type`
- [ ] 使用 `cppcoro` 或類似函式庫簡化開發
- [ ] 協程中避免阻塞操作 (使用異步 I/O)
- [ ] 正確管理協程生命週期 (避免過早銷毀)
- [ ] 使用協程處理 I/O 密集型任務 (網路、檔案)
- [ ] HFT 系統中使用協程管理訂單生命週期
- [ ] 測量協程 vs 執行緒的效能差異
- [ ] 注意協程的記憶體使用 (避免大型局部變數)

---

## 參考資料 (References)

1. [C++20 Coroutines - cppreference](https://en.cppreference.com/w/cpp/language/coroutines)
2. [cppcoro Library](https://github.com/lewissbaker/cppcoro)
3. Baker, Lewis. *C++ Coroutines: Understanding operator co_await* (2017)
4. [Asymmetric Transfer Blog](https://lewissbaker.github.io/)
5. [io_uring Documentation](https://kernel.dk/io_uring.pdf)
6. [C++20 Coroutines in Practice](https://www.youtube.com/watch?v=8C8NnE1Dg4A) - CppCon
