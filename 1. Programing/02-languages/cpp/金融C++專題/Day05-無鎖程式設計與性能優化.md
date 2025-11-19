# Day 5: 無鎖程式設計與性能優化

> HFT 核心技能:無鎖數據結構、Memory Ordering、Cache 優化與執行緒綁定

---

## 5.1 無鎖程式設計基礎

### 核心概念

無鎖程式設計 (Lock-Free Programming) 使用原子操作而非互斥鎖實現並發,在高頻交易中至關重要。

### 5.1.1 為什麼需要無鎖

**鎖的問題:**

```cpp
std::mutex mtx;
std::queue<Order> orders;

void add_order(const Order& order) {
    std::lock_guard<std::mutex> lock(mtx);  // 鎖開銷
    orders.push(order);
}
```

**開銷分析:**
- 上下文切換: ~5-10μs
- 優先級反轉: 低優先級執行緒持有鎖
- 死鎖風險
- 不可預測的延遲

**無鎖優勢:**
- 無上下文切換
- 可預測延遲
- 無死鎖
- 更高並發性

### 5.1.2 CAS 深入

**Compare-And-Swap (CAS) 原理:**

```cpp
// 偽代碼
bool CAS(T* ptr, T expected, T desired) {
    if (*ptr == expected) {
        *ptr = desired;
        return true;
    }
    return false;
}
```

**C++ 實作:**

```cpp
#include <atomic>

std::atomic<int> counter{0};

void increment() {
    int expected = counter.load();
    while (!counter.compare_exchange_weak(expected, expected + 1)) {
        // expected 被更新為當前值,重試
    }
}
```

**`compare_exchange_weak` vs `compare_exchange_strong`:**

```cpp
// weak: 可能虛假失敗 (spurious failure),更快
std::atomic<int> value{10};
int expected = 10;
while (!value.compare_exchange_weak(expected, 20)) {
    // 循環中使用,虛假失敗會重試
}

// strong: 保證只有在真正不相等時才失敗,較慢
if (value.compare_exchange_strong(expected, 30)) {
    // 單次嘗試時使用
}
```

### 5.1.3 ABA 問題

**問題描述:**

```cpp
// 初始: ptr -> A
// 執行緒 1 讀取 A,準備 CAS
// 執行緒 2 將 A 改為 B,再改回 A
// 執行緒 1 CAS 成功,但中間狀態已改變
```

**解決方案 1: 版本號 (Versioned Pointer):**

```cpp
template<typename T>
struct VersionedPointer {
    T* ptr;
    uint64_t version;
    
    bool operator==(const VersionedPointer& other) const {
        return ptr == other.ptr && version == other.version;
    }
};

std::atomic<VersionedPointer<Node>> head;

void push(Node* new_node) {
    VersionedPointer<Node> old_head = head.load();
    VersionedPointer<Node> new_head;
    
    do {
        new_node->next = old_head.ptr;
        new_head.ptr = new_node;
        new_head.version = old_head.version + 1;  // 版本遞增
    } while (!head.compare_exchange_weak(old_head, new_head));
}
```

**解決方案 2: Hazard Pointers (延遲釋放):**

```cpp
// 複雜但安全,實際應用中可使用 Folly 庫實作
```

### 5.1.4 無鎖演算法設計原則

**進度保證 (Progress Guarantees):**

1. **Wait-Free (無等待):** 每個執行緒在有限步內完成操作
2. **Lock-Free (無鎖):** 至少有一個執行緒能在有限步內完成
3. **Obstruction-Free (無障礙):** 單獨執行時能在有限步內完成

**設計原則:**
- 使用 CAS 原子操作
- 避免 ABA 問題
- 減少競爭 (contention)
- 選擇合適的 memory order

---

## 5.2 無鎖數據結構

### 5.2.1 無鎖棧 (Lock-Free Stack)

**基於 CAS 的實作:**

```cpp
#include <atomic>
#include <memory>

template<typename T>
class LockFreeStack {
private:
    struct Node {
        T data;
        Node* next;
        Node(const T& value) : data(value), next(nullptr) {}
    };
    
    std::atomic<Node*> head_{nullptr};

public:
    ~LockFreeStack() {
        while (pop()) {}
    }
    
    void push(const T& value) {
        Node* new_node = new Node(value);
        new_node->next = head_.load(std::memory_order_relaxed);
        
        // CAS 循環直到成功
        while (!head_.compare_exchange_weak(
            new_node->next, 
            new_node,
            std::memory_order_release,
            std::memory_order_relaxed)) {
            // new_node->next 被更新為當前 head,重試
        }
    }
    
    std::optional<T> pop() {
        Node* old_head = head_.load(std::memory_order_relaxed);
        
        while (old_head && !head_.compare_exchange_weak(
            old_head, 
            old_head->next,
            std::memory_order_acquire,
            std::memory_order_relaxed)) {
            // old_head 被更新,重試
        }
        
        if (!old_head) {
            return std::nullopt;
        }
        
        T value = old_head->data;
        delete old_head;  // ABA 風險!實際應使用 hazard pointers
        return value;
    }
};
```

### 5.2.2 SPSC 無鎖佇列 (Single Producer Single Consumer)

**環形緩衝區實作 (最高性能):**

```cpp
#include <atomic>
#include <vector>

template<typename T, size_t Size>
class SPSCQueue {
private:
    std::vector<T> buffer_;
    alignas(64) std::atomic<size_t> head_{0};  // 避免 false sharing
    alignas(64) std::atomic<size_t> tail_{0};
    
    static constexpr size_t capacity_ = Size + 1;  // 多一個位置區分滿/空

public:
    SPSCQueue() : buffer_(capacity_) {}
    
    bool push(const T& value) {
        const size_t current_tail = tail_.load(std::memory_order_relaxed);
        const size_t next_tail = (current_tail + 1) % capacity_;
        
        // 檢查是否滿
        if (next_tail == head_.load(std::memory_order_acquire)) {
            return false;  // 佇列滿
        }
        
        buffer_[current_tail] = value;
        tail_.store(next_tail, std::memory_order_release);
        return true;
    }
    
    bool pop(T& value) {
        const size_t current_head = head_.load(std::memory_order_relaxed);
        
        // 檢查是否空
        if (current_head == tail_.load(std::memory_order_acquire)) {
            return false;  // 佇列空
        }
        
        value = buffer_[current_head];
        head_.store((current_head + 1) % capacity_, std::memory_order_release);
        return true;
    }
    
    bool empty() const {
        return head_.load(std::memory_order_acquire) == 
               tail_.load(std::memory_order_acquire);
    }
};
```

**為什麼 SPSC 最快:**
- 無競爭: 單生產者單消費者
- 無 CAS: 只需 load/store
- 最小化 memory ordering: relaxed + acquire/release

### 5.2.3 環形緩衝區進階優化

**避免模運算:**

```cpp
// Size 必須是 2 的冪
template<typename T, size_t Size>
class FastSPSCQueue {
    static_assert((Size & (Size - 1)) == 0, "Size must be power of 2");
    
private:
    static constexpr size_t mask_ = Size - 1;
    
    bool push(const T& value) {
        const size_t current_tail = tail_.load(std::memory_order_relaxed);
        const size_t next_tail = (current_tail + 1) & mask_;  // 位運算代替模
        
        if (next_tail == head_.load(std::memory_order_acquire)) {
            return false;
        }
        
        buffer_[current_tail] = value;
        tail_.store(next_tail, std::memory_order_release);
        return true;
    }
    // ...
};
```

**批次操作:**

```cpp
template<typename T, size_t Size>
class BatchSPSCQueue : public FastSPSCQueue<T, Size> {
public:
    size_t push_batch(const T* values, size_t count) {
        size_t pushed = 0;
        for (size_t i = 0; i < count; ++i) {
            if (!push(values[i])) break;
            ++pushed;
        }
        return pushed;
    }
    
    size_t pop_batch(T* values, size_t max_count) {
        size_t popped = 0;
        for (size_t i = 0; i < max_count; ++i) {
            if (!pop(values[i])) break;
            ++popped;
        }
        return popped;
    }
};
```

---

## 5.3 記憶體順序實戰

### 核心概念

Memory Ordering 控制原子操作間的可見性與順序。

### 5.3.1 六種 Memory Order

```cpp
namespace std {
    enum memory_order {
        memory_order_relaxed,   // 無同步,僅保證原子性
        memory_order_consume,   // 數據依賴排序 (少用)
        memory_order_acquire,   // 讀獲取
        memory_order_release,   // 寫釋放
        memory_order_acq_rel,   // 讀寫兼有
        memory_order_seq_cst    // 順序一致 (預設,最慢)
    };
}
```

### 5.3.2 Relaxed Ordering

**適用場景:** 計數器、統計 (不需要同步其他記憶體)

```cpp
class MetricsCollector {
private:
    alignas(64) std::atomic<uint64_t> total_orders_{0};
    alignas(64) std::atomic<uint64_t> total_fills_{0};

public:
    void on_order() {
        total_orders_.fetch_add(1, std::memory_order_relaxed);
    }
    
    void on_fill() {
        total_fills_.fetch_add(1, std::memory_order_relaxed);
    }
    
    uint64_t get_total_orders() const {
        return total_orders_.load(std::memory_order_relaxed);
    }
};
```

### 5.3.3 Acquire-Release Ordering

**適用場景:** 生產者-消費者,SPSC 佇列

**語義:**
- **Release**: 寫操作前的所有記憶體操作對其他執行緒可見
- **Acquire**: 讀操作後能看到 release 前的所有記憶體操作

```cpp
// 生產者-消費者範例
class ProducerConsumer {
private:
    std::atomic<bool> ready_{false};
    int data_ = 0;

public:
    // 生產者
    void produce(int value) {
        data_ = value;                              // 1. 寫數據
        ready_.store(true, std::memory_order_release);  // 2. Release:保證 1 對消費者可見
    }
    
    // 消費者
    std::optional<int> consume() {
        if (ready_.load(std::memory_order_acquire)) {  // Acquire:看到 produce 的所有操作
            return data_;                              // 保證能讀到正確的 data_
        }
        return std::nullopt;
    }
};
```

**SPSC 佇列中的應用:**

```cpp
// push 使用 release
tail_.store(next_tail, std::memory_order_release);
// 保證 buffer_[current_tail] = value 對消費者可見

// pop 使用 acquire
if (current_head == tail_.load(std::memory_order_acquire)) {
    // 保證能看到生產者寫入的數據
}
```

### 5.3.4 Sequential Consistency

**特性:**
- 全局一致順序
- 性能最差 (需要記憶體柵欄)
- 預設值

```cpp
std::atomic<int> x{0}, y{0};

// 執行緒 1
void thread1() {
    x.store(1, std::memory_order_seq_cst);
    int r1 = y.load(std::memory_order_seq_cst);
}

// 執行緒 2
void thread2() {
    y.store(1, std::memory_order_seq_cst);
    int r2 = x.load(std::memory_order_seq_cst);
}

// seq_cst 保證不會出現 r1 == 0 && r2 == 0
// acquire-release 可能出現
```

### 5.3.5 性能對比

**基準測試 (x86-64):**

| Memory Order | 延遲 (ns) | 相對性能 |
|--------------|-----------|----------|
| relaxed      | ~1        | 1.0x     |
| acquire/release | ~1-2   | 1.5x     |
| seq_cst      | ~10-20    | 10x      |

**HFT 建議:**
- 優先使用 relaxed + acquire/release
- 避免 seq_cst (除非必須)
- SPSC 佇列: relaxed (內部) + acquire/release (邊界)

---

## 5.4 Cache 與並發性能

### 核心概念

Cache 是影響並發性能的關鍵因素。

### 5.4.1 Cache 架構

**典型多核 Cache 層次:**

```
Core 0          Core 1          Core 2          Core 3
L1 I$ L1 D$    L1 I$ L1 D$    L1 I$ L1 D$    L1 I$ L1 D$
  32KB  32KB     32KB  32KB     32KB  32KB     32KB  32KB
       |               |               |               |
       +---------L2 Cache (256KB)------+               |
                      |                                |
                      +---------L3 Cache (8MB)---------+
                                    |
                            Main Memory (DDR4)
```

**特性:**
- **L1**: ~4 cycles (~1ns)
- **L2**: ~10 cycles (~3ns)
- **L3**: ~40 cycles (~12ns)
- **RAM**: ~200 cycles (~60ns)

**Cache Line:**
- 大小: 64 bytes (x86-64)
- 原子單位: CPU 以 cache line 為單位加載/同步

### 5.4.2 False Sharing (偽共享)

**問題:**

```cpp
struct Counters {
    std::atomic<uint64_t> counter1;  // 偏移 0
    std::atomic<uint64_t> counter2;  // 偏移 8
    // 同一個 cache line!
};

Counters c;

// 執行緒 1
void thread1() {
    for (int i = 0; i < 1000000; ++i) {
        c.counter1.fetch_add(1, std::memory_order_relaxed);
    }
}

// 執行緒 2
void thread2() {
    for (int i = 0; i < 1000000; ++i) {
        c.counter2.fetch_add(1, std::memory_order_relaxed);
    }
}
// 性能災難!每次修改都導致 cache line 在 CPU 間同步
```

**檢測:**

```bash
# Linux perf c2c (cache-to-cache)
perf c2c record ./program
perf c2c report
```

**解決: Cache Line 對齊與填充**

```cpp
struct alignas(64) AlignedCounters {
    std::atomic<uint64_t> counter1;
    char padding1[64 - sizeof(std::atomic<uint64_t>)];
    
    std::atomic<uint64_t> counter2;
    char padding2[64 - sizeof(std::atomic<uint64_t>)];
};

// C++17 簡化版
struct Counters {
    alignas(64) std::atomic<uint64_t> counter1;
    alignas(64) std::atomic<uint64_t> counter2;
};

// 現在 counter1 和 counter2 在不同 cache line
```

**性能改善:** 10-100 倍

### 5.4.3 Cache-Friendly 程式設計

**資料局部性 (Data Locality):**

```cpp
// ✗ 差:跳躍訪問
struct Order {
    int id;
    double price;
    // ... 100 bytes ...
};

std::vector<Order> orders;
for (const auto& order : orders) {
    if (order.price > 100.0) {  // 每次加載整個 Order
        // ...
    }
}

// ✓ 好:連續訪問熱數據
std::vector<double> prices;  // 緊密排列
for (double price : prices) {
    if (price > 100.0) {
        // 高 cache 命中率
    }
}
```

**熱數據與冷數據分離:**

```cpp
struct Order {
    // 熱數據 (頻繁訪問)
    int id;
    double price;
    uint64_t timestamp;
    
    // 冷數據 (很少訪問)
    std::string client_info;  // 使用指標或索引
};

// 更好:分離
struct HotOrderData {
    int id;
    double price;
    uint64_t timestamp;
};

struct ColdOrderData {
    std::string client_info;
};

std::vector<HotOrderData> hot_data;     // Cache-friendly
std::vector<ColdOrderData> cold_data;   // 按需訪問
```

### 5.4.4 Prefetching

**手動預取:**

```cpp
void process_orders(const std::vector<Order>& orders) {
    for (size_t i = 0; i < orders.size(); ++i) {
        // 預取下一個 Order
        if (i + 1 < orders.size()) {
            __builtin_prefetch(&orders[i + 1], 0, 3);
            // 參數: 地址, 0=讀/1=寫, 3=高時間局部性
        }
        
        process(orders[i]);
    }
}
```

**效果:** 減少 cache miss,提升 5-20%

---

## 5.5 執行緒池設計

### 基本實作

```cpp
#include <vector>
#include <thread>
#include <queue>
#include <functional>
#include <condition_variable>
#include <future>

class ThreadPool {
private:
    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    
    std::mutex queue_mutex_;
    std::condition_variable condition_;
    bool stop_ = false;

public:
    explicit ThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    
                    {
                        std::unique_lock<std::mutex> lock(queue_mutex_);
                        condition_.wait(lock, [this] {
                            return stop_ || !tasks_.empty();
                        });
                        
                        if (stop_ && tasks_.empty()) return;
                        
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    
                    task();
                }
            });
        }
    }
    
    template<typename F, typename... Args>
    auto enqueue(F&& f, Args&&... args) 
        -> std::future<typename std::result_of<F(Args...)>::type> {
        
        using return_type = typename std::result_of<F(Args...)>::type;
        
        auto task = std::make_shared<std::packaged_task<return_type()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
        
        std::future<return_type> res = task->get_future();
        
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            if (stop_) throw std::runtime_error("enqueue on stopped ThreadPool");
            tasks_.emplace([task]() { (*task)(); });
        }
        
        condition_.notify_one();
        return res;
    }
    
    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            stop_ = true;
        }
        condition_.notify_all();
        for (std::thread& worker : workers_) {
            worker.join();
        }
    }
};

// 使用
ThreadPool pool(4);
auto result = pool.enqueue([](int x) { return x * x; }, 10);
std::cout << result.get();  // 100
```

### 執行緒數量選擇

```cpp
// CPU 密集型
size_t num_threads = std::thread::hardware_concurrency();

// I/O 密集型
size_t num_threads = std::thread::hardware_concurrency() * 2;

// HFT: 專用執行緒,避免執行緒池
// 原因:減少上下文切換,綁定 CPU
```

---

## 5.6 CPU 親和性 (CPU Affinity)

### 核心概念

將執行緒綁定到特定 CPU 核心,減少 cache miss 和上下文切換。

### 5.6.1 為什麼需要執行緒綁定

**問題:**
- 執行緒在不同核心間遷移
- L1/L2 cache 失效
- NUMA 架構下跨節點記憶體訪問

**HFT 應用:**
- 市場數據接收執行緒 → Core 0
- 策略執行緒 → Core 1
- 訂單發送執行緒 → Core 2
- 減少干擾,可預測延遲

### 5.6.2 Linux API

**pthread 版本:**

```cpp
#include <pthread.h>
#include <sched.h>

void bind_thread_to_cpu(int cpu_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    
    pthread_t current_thread = pthread_self();
    int result = pthread_setaffinity_np(current_thread, sizeof(cpu_set_t), &cpuset);
    
    if (result != 0) {
        std::cerr << "Failed to set CPU affinity\n";
    }
}

// 使用
std::thread t([]{
    bind_thread_to_cpu(0);  // 綁定到 Core 0
    // 執行關鍵任務
});
```

**std::thread 版本:**

```cpp
#include <thread>

void pin_thread_to_core(std::thread& thread, int core_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(core_id, &cpuset);
    
    int result = pthread_setaffinity_np(thread.native_handle(), 
                                        sizeof(cpu_set_t), &cpuset);
    if (result != 0) {
        throw std::runtime_error("Failed to pin thread to core");
    }
}

// 使用
std::thread market_data_thread(market_data_handler);
pin_thread_to_core(market_data_thread, 0);
```

### 5.6.3 配合 `isolcpus` 內核參數

**隔離 CPU 核心:**

```bash
# /etc/default/grub
GRUB_CMDLINE_LINUX="isolcpus=0,1,2,3 nohz_full=0,1,2,3 rcu_nocbs=0,1,2,3"

# 更新 grub
sudo update-grub
sudo reboot
```

**效果:**
- Core 0-3 不被排程器使用
- 減少中斷和定時器
- 專用於 HFT 關鍵執行緒

**完整範例:**

```cpp
class HFTSystem {
private:
    std::thread market_data_thread_;
    std::thread strategy_thread_;
    std::thread order_thread_;

public:
    void start() {
        // 市場數據接收
        market_data_thread_ = std::thread([this] {
            bind_thread_to_cpu(0);
            market_data_loop();
        });
        
        // 策略計算
        strategy_thread_ = std::thread([this] {
            bind_thread_to_cpu(1);
            strategy_loop();
        });
        
        // 訂單發送
        order_thread_ = std::thread([this] {
            bind_thread_to_cpu(2);
            order_loop();
        });
    }
    
    // ...
};
```

---

## 課後練習

### 練習 1: 實作無鎖棧

**任務:** 實作線程安全的無鎖棧。


```cpp
template<typename T>
class LockFreeStack {
private:
    struct Node {
        T data;
        std::atomic<Node*> next;
        Node(const T& value) : data(value), next(nullptr) {}
    };
    
    std::atomic<Node*> head_{nullptr};
    std::atomic<size_t> size_{0};

public:
    ~LockFreeStack() {
        while (pop()) {}
    }
    
    void push(const T& value) {
        Node* new_node = new Node(value);
        new_node->next = head_.load(std::memory_order_relaxed);
        
        while (!head_.compare_exchange_weak(
            new_node->next, 
            new_node,
            std::memory_order_release,
            std::memory_order_relaxed)) {
        }
        
        size_.fetch_add(1, std::memory_order_relaxed);
    }
    
    std::optional<T> pop() {
        Node* old_head = head_.load(std::memory_order_acquire);
        
        while (old_head && !head_.compare_exchange_weak(
            old_head, 
            old_head->next.load(std::memory_order_relaxed),
            std::memory_order_acquire,
            std::memory_order_relaxed)) {
        }
        
        if (!old_head) {
            return std::nullopt;
        }
        
        T value = old_head->data;
        delete old_head;
        size_.fetch_sub(1, std::memory_order_relaxed);
        return value;
    }
    
    size_t size() const {
        return size_.load(std::memory_order_relaxed);
    }
};
```

### 練習 2: SPSC 無鎖佇列性能測試

**任務:** 對比 SPSC 無鎖佇列與 mutex 保護佇列的性能。

```cpp
#include <chrono>
#include <iostream>
#include <mutex>
#include <queue>

// Mutex 版本
template<typename T>
class MutexQueue {
private:
    std::queue<T> queue_;
    std::mutex mutex_;

public:
    bool push(const T& value) {
        std::lock_guard<std::mutex> lock(mutex_);
        queue_.push(value);
        return true;
    }
    
    bool pop(T& value) {
        std::lock_guard<std::mutex> lock(mutex_);
        if (queue_.empty()) return false;
        value = queue_.front();
        queue_.pop();
        return true;
    }
};

// 性能測試
template<typename QueueType>
void benchmark(const std::string& name) {
    QueueType queue;
    constexpr size_t N = 1000000;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::thread producer([&] {
        for (size_t i = 0; i < N; ++i) {
            while (!queue.push(static_cast<int>(i))) {}
        }
    });
    
    std::thread consumer([&] {
        int value;
        for (size_t i = 0; i < N; ++i) {
            while (!queue.pop(value)) {}
        }
    });
    
    producer.join();
    consumer.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << name << ": " << duration.count() << " ms, "
              << (N * 1000.0 / duration.count()) << " ops/sec\n";
}

int main() {
    benchmark<SPSCQueue<int, 1024>>("SPSC Lock-Free");
    benchmark<MutexQueue<int>>("Mutex Queue");
}

// 預期結果:
// SPSC Lock-Free: ~50ms, 20M ops/sec
// Mutex Queue: ~500ms, 2M ops/sec
// 10x 性能差異
```

### 練習 3: False Sharing 重現與修正

**任務:** 重現 false sharing 問題並修正。

```cpp
#include <atomic>
#include <thread>
#include <chrono>
#include <iostream>

// 問題版本
struct BadCounters {
    std::atomic<uint64_t> counter1{0};
    std::atomic<uint64_t> counter2{0};
};

// 修正版本
struct alignas(128) GoodCounters {
    alignas(64) std::atomic<uint64_t> counter1{0};
    alignas(64) std::atomic<uint64_t> counter2{0};
};

template<typename T>
void benchmark_counters(const std::string& name) {
    T counters;
    constexpr size_t N = 10000000;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::thread t1([&] {
        for (size_t i = 0; i < N; ++i) {
            counters.counter1.fetch_add(1, std::memory_order_relaxed);
        }
    });
    
    std::thread t2([&] {
        for (size_t i = 0; i < N; ++i) {
            counters.counter2.fetch_add(1, std::memory_order_relaxed);
        }
    });
    
    t1.join();
    t2.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << name << ": " << duration.count() << " ms\n";
}

int main() {
    benchmark_counters<BadCounters>("False Sharing");
    benchmark_counters<GoodCounters>("Cache Aligned");
}

// 預期結果:
// False Sharing: ~200ms
// Cache Aligned: ~50ms
// 4x 性能改善
```

### 練習 4: 執行緒綁定性能測試

**任務:** 測量執行緒綁定對 cache 命中率的影響。

```cpp
#include <thread>
#include <vector>
#include <chrono>
#include <iostream>

void compute_intensive_task(std::vector<int>& data) {
    for (size_t iter = 0; iter < 1000; ++iter) {
        for (size_t i = 0; i < data.size(); ++i) {
            data[i] = data[i] * 2 + 1;
        }
    }
}

void test_without_affinity() {
    std::vector<int> data(1000000, 1);
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::thread t([&] {
        compute_intensive_task(data);
    });
    t.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "Without affinity: " << duration.count() << " ms\n";
}

void test_with_affinity() {
    std::vector<int> data(1000000, 1);
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::thread t([&] {
        // 綁定到 Core 0
        cpu_set_t cpuset;
        CPU_ZERO(&cpuset);
        CPU_SET(0, &cpuset);
        pthread_setaffinity_np(pthread_self(), sizeof(cpu_set_t), &cpuset);
        
        compute_intensive_task(data);
    });
    t.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "With affinity: " << duration.count() << " ms\n";
}

int main() {
    test_without_affinity();
    test_with_affinity();
    
    // 使用 perf 測量 cache miss
    // perf stat -e cache-misses,cache-references ./program
}
```


---

## 總結

### 核心要點

1. **無鎖程式設計**
   - CAS 是基礎操作
   - 注意 ABA 問題
   - SPSC 最快 (無競爭)

2. **Memory Ordering**
   - relaxed: 計數器
   - acquire/release: 生產者-消費者
   - 避免 seq_cst

3. **Cache 優化**
   - False sharing 是性能殺手
   - 使用 `alignas(64)` 對齊
   - 熱冷數據分離

4. **CPU 親和性**
   - 減少 cache miss
   - 配合 `isolcpus` 使用
   - HFT 關鍵優化

### HFT 最佳實踐

```cpp
// 1. SPSC 無鎖佇列通訊
alignas(64) SPSCQueue<Order, 1024> order_queue;

// 2. Cache line 對齊
struct alignas(64) OrderBookLevel {
    double price;
    uint64_t quantity;
};

// 3. 執行緒綁定
bind_thread_to_cpu(0);  // 關鍵執行緒

// 4. Memory ordering
ready_.store(true, std::memory_order_release);
if (ready_.load(std::memory_order_acquire)) { ... }
```

---

## 參考資料 (References)

1. [C++ Reference - Atomic Operations](https://en.cppreference.com/w/cpp/atomic)
2. [C++ Reference - Memory Order](https://en.cppreference.com/w/cpp/atomic/memory_order)
3. Anthony Williams, "C++ Concurrency in Action, 2nd Edition" (2019)
4. Preshing on Programming, "Memory Ordering at Compile Time" (Blog)
5. [Linux man pages - pthread_setaffinity_np](https://man7.org/linux/man-pages/man3/pthread_setaffinity_np.3.html)
6. Brendan Gregg, "Systems Performance, 2nd Edition" (2020)
7. Intel, "Intel 64 and IA-32 Architectures Optimization Reference Manual"
