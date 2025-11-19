# Day 4: 多線程編程與同步

## 學習目標

本課程深入探討 C++ 多線程編程的核心技術,涵蓋線程創建、同步原語、原子操作與記憶體順序模型。這些技術是構建高性能並發系統(尤其是高頻交易系統)的基礎。

**核心主題:**
- `std::thread` 線程管理
- 互斥鎖、條件變量、讀寫鎖
- 原子操作與 CAS
- 記憶體順序模型
- 線程安全設計模式

---

## 4.1 多線程基礎

### 4.1.1 std::thread 基本操作

```cpp
#include <iostream>
#include <thread>
#include <chrono>

void worker(int id, int iterations) {
    for (int i = 0; i < iterations; ++i) {
        std::cout << "Thread " << id << " iteration " << i << "\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

int main() {
    // 創建線程
    std::thread t1(worker, 1, 5);
    std::thread t2(worker, 2, 3);
    
    // 等待線程完成
    t1.join();
    t2.join();
    
    std::cout << "All threads completed\n";
    return 0;
}
```

### 4.1.2 join() vs detach()

```cpp
#include <thread>
#include <iostream>

void task() {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Task completed\n";
}

int main() {
    // join: 主線程等待子線程完成
    std::thread t1(task);
    t1.join();  // 阻塞直到 t1 完成
    
    // detach: 線程獨立運行
    std::thread t2(task);
    t2.detach();  // t2 在後台運行,主線程不等待
    
    // 注意: detach 後無法再 join,且程序退出時可能終止未完成的 detached 線程
    std::this_thread::sleep_for(std::chrono::seconds(2));  // 確保 t2 有時間完成
    return 0;
}
```

**關鍵差異:**
- `join()`: 主線程阻塞等待,確保線程完成後才繼續
- `detach()`: 線程獨立運行,主線程不等待,但需確保程序退出時線程已完成或可安全終止

### 4.1.3 參數傳遞與引用捕獲

```cpp
#include <thread>
#include <iostream>
#include <vector>

void modify_by_value(int x) {
    x += 10;  // 不影響原始變量
}

void modify_by_ref(int& x) {
    x += 10;  // 修改原始變量
}

int main() {
    int value = 5;
    
    // 按值傳遞(默認)
    std::thread t1(modify_by_value, value);
    t1.join();
    std::cout << "After t1: " << value << "\n";  // 輸出: 5
    
    // 按引用傳遞: 必須使用 std::ref
    std::thread t2(modify_by_ref, std::ref(value));
    t2.join();
    std::cout << "After t2: " << value << "\n";  // 輸出: 15
    
    return 0;
}
```

**注意事項:**
- 默認按值拷貝參數
- 傳遞引用需使用 `std::ref` 或 `std::cref`(常量引用)
- 傳遞指針或引用時需確保生命週期安全

### 4.1.4 thread_local 存儲

```cpp
#include <thread>
#include <iostream>

thread_local int counter = 0;  // 每個線程有獨立的副本

void increment() {
    for (int i = 0; i < 5; ++i) {
        ++counter;
    }
    std::cout << "Thread " << std::this_thread::get_id() 
              << " counter: " << counter << "\n";
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    
    t1.join();
    t2.join();
    
    std::cout << "Main thread counter: " << counter << "\n";  // 輸出: 0
    return 0;
}
```

### 4.1.5 std::jthread (C++20)

```cpp
#include <thread>
#include <iostream>
#include <chrono>

void worker(std::stop_token stoken) {
    while (!stoken.stop_requested()) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        std::cout << "Working...\n";
    }
    std::cout << "Stop requested\n";
}

int main() {
    std::jthread t(worker);  // 自動 join,支持停止信號
    
    std::this_thread::sleep_for(std::chrono::seconds(1));
    t.request_stop();  // 請求停止
    
    // t 析構時自動 join
    return 0;
}
```

**jthread 優勢:**
- 析構時自動 join
- 內建停止機制 (`stop_token`)
- 更安全,避免忘記 join/detach

---

## 4.2 互斥與同步

### 4.2.1 std::mutex 基本用法

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <vector>

std::mutex mtx;
int shared_counter = 0;

void increment(int iterations) {
    for (int i = 0; i < iterations; ++i) {
        mtx.lock();
        ++shared_counter;
        mtx.unlock();
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back(increment, 1000);
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    std::cout << "Final counter: " << shared_counter << "\n";  // 輸出: 10000
    return 0;
}
```

### 4.2.2 RAII 鎖管理

```cpp
#include <mutex>
#include <iostream>

std::mutex mtx;
int shared_data = 0;

void safe_increment() {
    std::lock_guard<std::mutex> lock(mtx);  // 構造時加鎖,析構時解鎖
    ++shared_data;
    // 函數退出時自動解鎖,即使拋出異常
}

void conditional_operation(bool condition) {
    std::unique_lock<std::mutex> lock(mtx);  // 更靈活的鎖管理
    
    if (condition) {
        ++shared_data;
        lock.unlock();  // 可手動解鎖
        // 執行不需要鎖的操作
    }
    // lock 析構時如果仍持有鎖會自動解鎖
}
```

**鎖類型比較:**

| 鎖類型 | 手動解鎖 | 延遲加鎖 | 轉移所有權 | 使用場景 |
|--------|----------|----------|------------|----------|
| `lock_guard` | ❌ | ❌ | ❌ | 簡單臨界區保護 |
| `unique_lock` | ✅ | ✅ | ✅ | 需要靈活控制的場景 |
| `scoped_lock` (C++17) | ❌ | ❌ | ❌ | 同時鎖定多個互斥鎖 |

### 4.2.3 避免死鎖

```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx1, mtx2;

// 錯誤示範: 可能死鎖
void deadlock_example() {
    std::thread t1([]() {
        std::lock_guard<std::mutex> lock1(mtx1);
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        std::lock_guard<std::mutex> lock2(mtx2);  // 等待 mtx2
    });
    
    std::thread t2([]() {
        std::lock_guard<std::mutex> lock2(mtx2);
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        std::lock_guard<std::mutex> lock1(mtx1);  // 等待 mtx1 -> 死鎖!
    });
    
    t1.join();
    t2.join();
}

// 解決方案 1: 統一鎖定順序
void solution1() {
    std::thread t1([]() {
        std::lock_guard<std::mutex> lock1(mtx1);
        std::lock_guard<std::mutex> lock2(mtx2);
    });
    
    std::thread t2([]() {
        std::lock_guard<std::mutex> lock1(mtx1);  // 相同順序
        std::lock_guard<std::mutex> lock2(mtx2);
    });
    
    t1.join();
    t2.join();
}

// 解決方案 2: 使用 scoped_lock 同時鎖定
void solution2() {
    std::thread t1([]() {
        std::scoped_lock lock(mtx1, mtx2);  // 原子性鎖定
    });
    
    std::thread t2([]() {
        std::scoped_lock lock(mtx2, mtx1);  // 順序無關,不會死鎖
    });
    
    t1.join();
    t2.join();
}
```

### 4.2.4 std::shared_mutex (讀寫鎖)

```cpp
#include <shared_mutex>
#include <thread>
#include <vector>
#include <iostream>

class ThreadSafeCounter {
private:
    mutable std::shared_mutex mtx;
    int value = 0;
    
public:
    // 寫操作: 獨占鎖
    void increment() {
        std::unique_lock lock(mtx);
        ++value;
    }
    
    // 讀操作: 共享鎖(多個讀者可同時進入)
    int get() const {
        std::shared_lock lock(mtx);
        return value;
    }
};

int main() {
    ThreadSafeCounter counter;
    
    // 多個寫線程
    std::vector<std::thread> writers;
    for (int i = 0; i < 5; ++i) {
        writers.emplace_back([&]() {
            for (int j = 0; j < 100; ++j) {
                counter.increment();
            }
        });
    }
    
    // 多個讀線程
    std::vector<std::thread> readers;
    for (int i = 0; i < 10; ++i) {
        readers.emplace_back([&]() {
            for (int j = 0; j < 100; ++j) {
                std::cout << "Counter: " << counter.get() << "\n";
            }
        });
    }
    
    for (auto& t : writers) t.join();
    for (auto& t : readers) t.join();
    
    return 0;
}
```

### 4.2.5 條件變量

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>

template<typename T>
class ThreadSafeQueue {
private:
    std::queue<T> queue_;
    mutable std::mutex mtx_;
    std::condition_variable cv_;
    
public:
    void push(T value) {
        {
            std::lock_guard<std::mutex> lock(mtx_);
            queue_.push(std::move(value));
        }
        cv_.notify_one();  // 通知一個等待線程
    }
    
    bool try_pop(T& value) {
        std::lock_guard<std::mutex> lock(mtx_);
        if (queue_.empty()) {
            return false;
        }
        value = std::move(queue_.front());
        queue_.pop();
        return true;
    }
    
    void wait_and_pop(T& value) {
        std::unique_lock<std::mutex> lock(mtx_);
        // 使用 predicate 避免虛假喚醒
        cv_.wait(lock, [this] { return !queue_.empty(); });
        value = std::move(queue_.front());
        queue_.pop();
    }
};

// 生產者-消費者示例
int main() {
    ThreadSafeQueue<int> queue;
    
    // 生產者
    std::thread producer([&]() {
        for (int i = 0; i < 10; ++i) {
            queue.push(i);
            std::cout << "Produced: " << i << "\n";
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    });
    
    // 消費者
    std::thread consumer([&]() {
        for (int i = 0; i < 10; ++i) {
            int value;
            queue.wait_and_pop(value);
            std::cout << "Consumed: " << value << "\n";
        }
    });
    
    producer.join();
    consumer.join();
    
    return 0;
}
```

**條件變量關鍵點:**
- 必須與 `unique_lock` 配合使用
- `wait()` 會原子性地釋放鎖並等待,被喚醒時重新獲取鎖
- 使用 predicate 避免虛假喚醒 (spurious wakeup)
- `notify_one()` 喚醒一個等待線程,`notify_all()` 喚醒所有

---

## 4.3 原子操作

### 4.3.1 std::atomic 基礎

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <vector>

std::atomic<int> atomic_counter{0};
int regular_counter = 0;

void increment_atomic(int iterations) {
    for (int i = 0; i < iterations; ++i) {
        atomic_counter.fetch_add(1);  // 原子遞增
    }
}

void increment_regular(int iterations) {
    for (int i = 0; i < iterations; ++i) {
        ++regular_counter;  // 非線程安全!
    }
}

int main() {
    const int num_threads = 10;
    const int iterations = 10000;
    
    // 測試原子操作
    std::vector<std::thread> threads;
    for (int i = 0; i < num_threads; ++i) {
        threads.emplace_back(increment_atomic, iterations);
    }
    for (auto& t : threads) t.join();
    std::cout << "Atomic counter: " << atomic_counter << "\n";  // 100000
    
    // 測試非原子操作
    threads.clear();
    for (int i = 0; i < num_threads; ++i) {
        threads.emplace_back(increment_regular, iterations);
    }
    for (auto& t : threads) t.join();
    std::cout << "Regular counter: " << regular_counter << "\n";  // 不確定,可能 < 100000
    
    return 0;
}
```

### 4.3.2 原子操作類型

```cpp
#include <atomic>
#include <iostream>

int main() {
    std::atomic<int> counter{0};
    
    // 檢查是否無鎖實現
    std::cout << std::boolalpha;
    std::cout << "int is lock-free: " << counter.is_lock_free() << "\n";
    
    // 基本操作
    counter.store(10);                      // 存儲
    int value = counter.load();             // 加載
    int old = counter.exchange(20);         // 交換並返回舊值
    
    // 算術操作
    counter.fetch_add(5);                   // counter += 5, 返回舊值
    counter.fetch_sub(3);                   // counter -= 3, 返回舊值
    counter++;                              // 等價於 fetch_add(1) + 1
    
    // CAS (Compare-And-Swap)
    int expected = 22;
    bool success = counter.compare_exchange_strong(expected, 100);
    // 如果 counter == expected, 設置 counter = 100 並返回 true
    // 否則設置 expected = counter 並返回 false
    
    std::cout << "CAS success: " << success << ", value: " << counter << "\n";
    
    return 0;
}
```

### 4.3.3 CAS 深入: compare_exchange_weak vs strong

```cpp
#include <atomic>
#include <iostream>

std::atomic<int> value{0};

void demonstrate_cas() {
    int expected = 0;
    int desired = 42;
    
    // compare_exchange_strong: 只在實際不相等時失敗
    if (value.compare_exchange_strong(expected, desired)) {
        std::cout << "Strong CAS succeeded\n";
    } else {
        std::cout << "Strong CAS failed, current value: " << expected << "\n";
    }
    
    // compare_exchange_weak: 可能虛假失敗 (spurious failure)
    // 適合在循環中使用,性能更好
    expected = 42;
    desired = 100;
    while (!value.compare_exchange_weak(expected, desired)) {
        // 失敗後 expected 已更新為當前值
        // 可在此處調整 desired
        std::cout << "Weak CAS failed, retrying...\n";
    }
    std::cout << "Weak CAS succeeded\n";
}

int main() {
    demonstrate_cas();
    return 0;
}
```

**使用建議:**
- `compare_exchange_strong`: 單次 CAS,必須成功或明確失敗
- `compare_exchange_weak`: 循環中使用,允許虛假失敗,性能更好

### 4.3.4 無鎖計數器實現

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>

class LockFreeCounter {
private:
    std::atomic<uint64_t> value_{0};
    
public:
    void increment() {
        value_.fetch_add(1, std::memory_order_relaxed);
    }
    
    uint64_t get() const {
        return value_.load(std::memory_order_relaxed);
    }
};

int main() {
    LockFreeCounter counter;
    
    std::vector<std::thread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back([&]() {
            for (int j = 0; j < 100000; ++j) {
                counter.increment();
            }
        });
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    std::cout << "Final count: " << counter.get() << "\n";
    return 0;
}
```

### 4.3.5 自旋鎖實現

```cpp
#include <atomic>
#include <thread>

class SpinLock {
private:
    std::atomic_flag flag_ = ATOMIC_FLAG_INIT;
    
public:
    void lock() {
        // 自旋等待直到獲取鎖
        while (flag_.test_and_set(std::memory_order_acquire)) {
            // 可選: CPU 暫停指令減少功耗
            #if defined(__x86_64__) || defined(_M_X64)
            __builtin_ia32_pause();
            #endif
        }
    }
    
    void unlock() {
        flag_.clear(std::memory_order_release);
    }
};

// RAII 包裝
class SpinLockGuard {
private:
    SpinLock& lock_;
    
public:
    explicit SpinLockGuard(SpinLock& lock) : lock_(lock) {
        lock_.lock();
    }
    
    ~SpinLockGuard() {
        lock_.unlock();
    }
};

// 使用示例
SpinLock spin_lock;
int shared_data = 0;

void worker() {
    for (int i = 0; i < 10000; ++i) {
        SpinLockGuard guard(spin_lock);
        ++shared_data;
    }
}
```

**自旋鎖 vs 互斥鎖:**

| 特性 | 自旋鎖 | 互斥鎖 |
|------|--------|--------|
| 等待方式 | 忙等待 (busy-wait) | 睡眠等待 |
| CPU 使用 | 高 | 低 |
| 上下文切換 | 無 | 有 |
| 適用場景 | 極短臨界區 (<100ns) | 較長臨界區 |
| HFT 應用 | 市場數據更新 | 訂單持久化 |

---

## 4.4 記憶體順序模型

### 4.4.1 記憶體順序類型

```cpp
enum memory_order {
    memory_order_relaxed,    // 無同步,僅保證原子性
    memory_order_consume,    // 數據依賴排序 (rarely used)
    memory_order_acquire,    // 讀操作: 後續讀寫不能重排到之前
    memory_order_release,    // 寫操作: 之前讀寫不能重排到之後
    memory_order_acq_rel,    // 讀-修改-寫: acquire + release
    memory_order_seq_cst     // 順序一致 (默認,最強)
};
```

### 4.4.2 Relaxed 順序

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> counter{0};

void increment_relaxed() {
    for (int i = 0; i < 100000; ++i) {
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    std::thread t1(increment_relaxed);
    std::thread t2(increment_relaxed);
    
    t1.join();
    t2.join();
    
    std::cout << "Counter: " << counter << "\n";  // 200000
    return 0;
}
```

**使用場景:** 僅需原子性,無需同步 (如計數器、統計信息)

### 4.4.3 Acquire-Release 順序

```cpp
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<bool> ready{false};
int data = 0;

void producer() {
    data = 42;                                      // 1. 寫數據
    ready.store(true, std::memory_order_release);  // 2. 釋放柵欄
    // 保證 1 在 2 之前完成
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {  // 3. 獲取柵欄
        // 自旋等待
    }
    // 保證 3 在 4 之前完成
    assert(data == 42);  // 4. 讀數據,保證看到 42
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    
    t1.join();
    t2.join();
    
    return 0;
}
```

**關鍵概念:**
- **Release**: 之前的所有寫操作對其他線程可見
- **Acquire**: 能看到其他線程 release 之前的所有寫操作
- 建立 **happens-before** 關係

### 4.4.4 順序一致 (Sequential Consistency)

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<bool> x{false};
std::atomic<bool> y{false};
std::atomic<int> z{0};

void write_x() {
    x.store(true, std::memory_order_seq_cst);
}

void write_y() {
    y.store(true, std::memory_order_seq_cst);
}

void read_x_then_y() {
    while (!x.load(std::memory_order_seq_cst));
    if (y.load(std::memory_order_seq_cst)) {
        ++z;
    }
}

void read_y_then_x() {
    while (!y.load(std::memory_order_seq_cst));
    if (x.load(std::memory_order_seq_cst)) {
        ++z;
    }
}

int main() {
    std::thread t1(write_x);
    std::thread t2(write_y);
    std::thread t3(read_x_then_y);
    std::thread t4(read_y_then_x);
    
    t1.join(); t2.join(); t3.join(); t4.join();
    
    std::cout << "z: " << z << "\n";  // 保證至少為 1
    return 0;
}
```

**特點:**
- 全局一致的操作順序
- 性能開銷最大
- 默認選擇,最安全

### 4.4.5 記憶體順序性能對比

```cpp
#include <atomic>
#include <chrono>
#include <iostream>
#include <thread>

constexpr int ITERATIONS = 10000000;

template<std::memory_order Order>
void benchmark() {
    std::atomic<int> counter{0};
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::thread t1([&]() {
        for (int i = 0; i < ITERATIONS; ++i) {
            counter.fetch_add(1, Order);
        }
    });
    
    std::thread t2([&]() {
        for (int i = 0; i < ITERATIONS; ++i) {
            counter.fetch_add(1, Order);
        }
    });
    
    t1.join();
    t2.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "Order: " << static_cast<int>(Order) 
              << ", Time: " << duration.count() << "ms\n";
}

int main() {
    std::cout << "Relaxed:\n";
    benchmark<std::memory_order_relaxed>();
    
    std::cout << "Acquire-Release:\n";
    benchmark<std::memory_order_acq_rel>();
    
    std::cout << "Seq_cst:\n";
    benchmark<std::memory_order_seq_cst>();
    
    return 0;
}
```

**預期結果 (x86):**
- Relaxed: 最快
- Acquire-Release: 中等
- Seq_cst: 最慢 (但 x86 有強內存模型,差距不大)

**選擇建議:**

| 場景 | 推薦順序 | 理由 |
|------|----------|------|
| 計數器 | Relaxed | 只需原子性 |
| SPSC 隊列 | Acquire-Release | 需要同步但無全局順序要求 |
| 複雜同步 | Seq_cst | 簡單安全,性能可接受 |

---

## 實作練習

### 練習 1: 線程安全的單例模式

**需求:** 實現線程安全的單例,使用雙重檢查鎖定。

```cpp
#include <mutex>
#include <atomic>
#include <iostream>

class Singleton {
private:
    static std::atomic<Singleton*> instance_;
    static std::mutex mtx_;
    
    int data_;
    
    Singleton(int data) : data_(data) {
        std::cout << "Singleton created with data: " << data << "\n";
    }
    
public:
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
    
    static Singleton* getInstance(int data) {
        Singleton* tmp = instance_.load(std::memory_order_acquire);
        if (tmp == nullptr) {
            std::lock_guard<std::mutex> lock(mtx_);
            tmp = instance_.load(std::memory_order_relaxed);
            if (tmp == nullptr) {
                tmp = new Singleton(data);
                instance_.store(tmp, std::memory_order_release);
            }
        }
        return tmp;
    }
    
    int getData() const { return data_; }
};

std::atomic<Singleton*> Singleton::instance_{nullptr};
std::mutex Singleton::mtx_;

// 測試
int main() {
    std::vector<std::thread> threads;
    
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back([i]() {
            Singleton* s = Singleton::getInstance(i);
            std::cout << "Thread " << i << " got data: " << s->getData() << "\n";
        });
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    return 0;
}
```

**輸出:** 只創建一次實例,所有線程獲取相同數據。

### 練習 2: 生產者-消費者隊列 (條件變量)

**需求:** 實現有界阻塞隊列,使用條件變量處理滿/空情況。

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>

template<typename T>
class BoundedBlockingQueue {
private:
    std::queue<T> queue_;
    size_t capacity_;
    mutable std::mutex mtx_;
    std::condition_variable cv_not_full_;
    std::condition_variable cv_not_empty_;
    
public:
    explicit BoundedBlockingQueue(size_t capacity) : capacity_(capacity) {}
    
    void push(T value) {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_not_full_.wait(lock, [this] { return queue_.size() < capacity_; });
        queue_.push(std::move(value));
        cv_not_empty_.notify_one();
    }
    
    T pop() {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_not_empty_.wait(lock, [this] { return !queue_.empty(); });
        T value = std::move(queue_.front());
        queue_.pop();
        cv_not_full_.notify_one();
        return value;
    }
    
    size_t size() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return queue_.size();
    }
};

// 測試
int main() {
    BoundedBlockingQueue<int> queue(5);
    
    std::thread producer([&]() {
        for (int i = 0; i < 20; ++i) {
            queue.push(i);
            std::cout << "Produced: " << i << ", Queue size: " << queue.size() << "\n";
            std::this_thread::sleep_for(std::chrono::milliseconds(50));
        }
    });
    
    std::thread consumer([&]() {
        for (int i = 0; i < 20; ++i) {
            int value = queue.pop();
            std::cout << "Consumed: " << value << ", Queue size: " << queue.size() << "\n";
            std::this_thread::sleep_for(std::chrono::milliseconds(150));
        }
    });
    
    producer.join();
    consumer.join();
    
    return 0;
}
```

### 練習 3: 自旋鎖性能對比

**需求:** 對比自旋鎖和 `std::mutex` 在不同臨界區大小下的性能。

```cpp
#include <atomic>
#include <mutex>
#include <chrono>
#include <thread>
#include <iostream>
#include <vector>

class SpinLock {
private:
    std::atomic_flag flag_ = ATOMIC_FLAG_INIT;
public:
    void lock() {
        while (flag_.test_and_set(std::memory_order_acquire));
    }
    void unlock() {
        flag_.clear(std::memory_order_release);
    }
};

template<typename Lock>
void benchmark(const char* name, int work_size) {
    Lock lock;
    volatile int counter = 0;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back([&]() {
            for (int j = 0; j < 100000; ++j) {
                lock.lock();
                for (int k = 0; k < work_size; ++k) {
                    ++counter;  // 模擬工作
                }
                lock.unlock();
            }
        });
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << name << " (work size " << work_size << "): " 
              << duration.count() << "ms\n";
}

int main() {
    std::cout << "Short critical section (1 operation):\n";
    benchmark<SpinLock>("SpinLock", 1);
    benchmark<std::mutex>("Mutex", 1);
    
    std::cout << "\nMedium critical section (10 operations):\n";
    benchmark<SpinLock>("SpinLock", 10);
    benchmark<std::mutex>("Mutex", 10);
    
    std::cout << "\nLong critical section (100 operations):\n";
    benchmark<SpinLock>("SpinLock", 100);
    benchmark<std::mutex>("Mutex", 100);
    
    return 0;
}
```

**預期結果:**
- 極短臨界區: SpinLock 更快
- 中等臨界區: 性能相近
- 長臨界區: Mutex 更快 (避免 CPU 浪費)

### 練習 4: 無鎖計數器

**需求:** 實現支持並發遞增/遞減的無鎖計數器,並與有鎖版本對比性能。

```cpp
#include <atomic>
#include <mutex>
#include <thread>
#include <chrono>
#include <iostream>
#include <vector>

// 無鎖版本
class LockFreeCounter {
private:
    std::atomic<int64_t> value_{0};
public:
    void increment() {
        value_.fetch_add(1, std::memory_order_relaxed);
    }
    void decrement() {
        value_.fetch_sub(1, std::memory_order_relaxed);
    }
    int64_t get() const {
        return value_.load(std::memory_order_relaxed);
    }
};

// 有鎖版本
class MutexCounter {
private:
    int64_t value_ = 0;
    mutable std::mutex mtx_;
public:
    void increment() {
        std::lock_guard<std::mutex> lock(mtx_);
        ++value_;
    }
    void decrement() {
        std::lock_guard<std::mutex> lock(mtx_);
        --value_;
    }
    int64_t get() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return value_;
    }
};

template<typename Counter>
void benchmark(const char* name) {
    Counter counter;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::vector<std::thread> threads;
    for (int i = 0; i < 8; ++i) {
        threads.emplace_back([&]() {
            for (int j = 0; j < 1000000; ++j) {
                counter.increment();
                counter.decrement();
            }
        });
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << name << ": " << duration.count() << "ms, final value: " 
              << counter.get() << "\n";
}

int main() {
    benchmark<LockFreeCounter>("LockFree");
    benchmark<MutexCounter>("Mutex");
    
    return 0;
}
```

**預期結果:** LockFree 版本顯著更快 (2-5倍)。

---

## 關鍵要點總結

### 線程管理
- 使用 `std::jthread` (C++20) 簡化線程管理
- 傳遞引用必須使用 `std::ref`
- `thread_local` 提供線程局部存儲

### 同步原語選擇

| 場景 | 推薦工具 | 理由 |
|------|----------|------|
| 簡單互斥 | `std::mutex` + `lock_guard` | 簡單安全 |
| 需手動控制 | `unique_lock` | 靈活 |
| 多鎖同時鎖定 | `scoped_lock` | 避免死鎖 |
| 讀多寫少 | `shared_mutex` | 讀並發 |
| 極短臨界區 | `SpinLock` | 避免上下文切換 |
| 生產者-消費者 | 條件變量 | 高效等待 |

### 原子操作與記憶體順序

| 操作類型 | 推薦順序 | 典型場景 |
|----------|----------|----------|
| 計數器 | Relaxed | 統計信息 |
| 生產者-消費者 | Acquire-Release | SPSC 隊列 |
| 複雜同步 | Seq_cst | 多變量同步 |

### HFT 應用建議
- **市場數據接收**: 無鎖隊列 + acquire-release
- **訂單簿更新**: 細粒度鎖或無鎖結構
- **風控檢查**: 原子變量 + relaxed 順序
- **日誌**: 無鎖環形緩衝區
- **臨界區優化**: 盡量縮短鎖持有時間

---

## 參考資料 (References)

1. Anthony Williams, *C++ Concurrency in Action*, 2nd Edition (2019)
2. Herb Sutter, ["atomic<> Weapons: The C++ Memory Model and Modern Hardware"](https://herbsutter.com/2013/02/11/atomic-weapons-the-c-memory-model-and-modern-hardware/) (2013)
3. cppreference.com, [Thread support library](https://en.cppreference.com/w/cpp/thread)
4. cppreference.com, [std::memory_order](https://en.cppreference.com/w/cpp/atomic/memory_order)
5. Preshing on Programming, [Memory Ordering at Compile Time](https://preshing.com/20120625/memory-ordering-at-compile-time/)
6. Jeff Preshing, [An Introduction to Lock-Free Programming](https://preshing.com/20120612/an-introduction-to-lock-free-programming/)
7. Linux man pages, [pthread(7)](https://man7.org/linux/man-pages/man7/pthreads.7.html)
