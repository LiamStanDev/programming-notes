# Lock-Free 資料結構 (Lock-Free Data Structures)

## 概述

Lock-Free 資料結構使用原子操作 (Atomic Operations) 替代鎖 (Locks),在高並發環境下提供更好的可擴展性與延遲特性。對 HFT 系統而言,避免鎖爭用 (Lock Contention) 是降低延遲的關鍵。

## Lock-Free 的定義與特性

### 進度保證 (Progress Guarantees)

從弱到強的進度保證:

```cpp
// 1. Obstruction-Free: 單執行緒執行時保證完成
// 2. Lock-Free: 至少一個執行緒保證完成 (系統整體前進)
// 3. Wait-Free: 所有執行緒保證在有限步驟內完成 (最強)
```

**Lock-Free 特性**:
- 無死鎖 (Deadlock-Free)
- 無優先權反轉 (Priority Inversion)
- 更可預測的延遲 (適合即時系統)
- 但可能有活鎖 (Livelock) 或飢餓 (Starvation)

### Lock-Free vs Lock-Based

```cpp
// Lock-Based (傳統)
class CounterWithLock {
    std::mutex mtx_;
    int value_ = 0;
public:
    void increment() {
        std::lock_guard<std::mutex> lock(mtx_);  // 阻塞
        ++value_;
    }
};

// Lock-Free
class LockFreeCounter {
    std::atomic<int> value_{0};
public:
    void increment() {
        value_.fetch_add(1, std::memory_order_relaxed);  // 非阻塞
    }
};
```

## Lock-Free Stack

### 基礎實作

```cpp
#include <atomic>
#include <memory>

template<typename T>
class LockFreeStack {
private:
    struct Node {
        T data;
        Node* next;
        
        Node(const T& d) : data(d), next(nullptr) {}
    };
    
    std::atomic<Node*> head_{nullptr};
    
public:
    void push(const T& data) {
        Node* new_node = new Node(data);
        
        // 設定 next 指向當前 head
        new_node->next = head_.load(std::memory_order_relaxed);
        
        // CAS 迴圈: 如果 head 未改變,則更新為 new_node
        while (!head_.compare_exchange_weak(
            new_node->next,           // expected: 更新為實際 head
            new_node,                 // desired: 新節點
            std::memory_order_release,  // 成功時的記憶體順序
            std::memory_order_relaxed   // 失敗時的記憶體順序
        ));
    }
    
    std::shared_ptr<T> pop() {
        Node* old_head = head_.load(std::memory_order_acquire);
        
        // CAS 迴圈直到成功或 stack 為空
        while (old_head && !head_.compare_exchange_weak(
            old_head,                 // expected: 更新為實際 head
            old_head->next,           // desired: head->next
            std::memory_order_release,
            std::memory_order_acquire
        ));
        
        if (!old_head) {
            return std::shared_ptr<T>();  // Stack 空
        }
        
        // 安全地讀取資料後刪除節點
        std::shared_ptr<T> result = std::make_shared<T>(old_head->data);
        delete old_head;  // 警告: 簡化版,需處理記憶體回收
        return result;
    }
};
```

### 記憶體回收問題

```cpp
// 問題: pop() 中的 delete 可能導致 use-after-free
// Thread 1: auto p = head.load();  // p 指向 Node A
// Thread 2: pop() 成功,delete Node A
// Thread 1: 訪問 p->data  // Crash! A 已被刪除

// 解決方案 1: Hazard Pointer (危險指標)
// 解決方案 2: Reference Counting (引用計數)
// 解決方案 3: Epoch-Based Reclamation (世代回收)
```

### 使用 Hazard Pointer 的 Stack

```cpp
template<typename T>
class LockFreeStackWithHP {
private:
    struct Node {
        std::shared_ptr<T> data;  // 使用 shared_ptr
        std::atomic<Node*> next;
        std::atomic<int> ref_count{0};  // 引用計數
        
        Node(const T& d) : data(std::make_shared<T>(d)) {}
    };
    
    std::atomic<Node*> head_{nullptr};
    std::atomic<Node*> to_delete_{nullptr};  // 待刪除列表
    
    void try_reclaim(Node* old_head) {
        if (old_head->ref_count.load() == 1) {
            // 安全刪除
            delete old_head;
        } else {
            // 加入待刪除列表
            chain_pending_node(old_head);
        }
    }
    
    void chain_pending_node(Node* node) {
        node->next = to_delete_.load();
        while (!to_delete_.compare_exchange_weak(node->next, node));
    }
    
public:
    void push(const T& data) {
        Node* new_node = new Node(data);
        new_node->next = head_.load();
        
        while (!head_.compare_exchange_weak(new_node->next, new_node));
    }
    
    std::shared_ptr<T> pop() {
        Node* old_head = head_.load();
        
        while (old_head) {
            // 增加引用計數 (保護節點)
            old_head->ref_count.fetch_add(1);
            
            if (head_.compare_exchange_strong(old_head, old_head->next.load())) {
                std::shared_ptr<T> result = old_head->data;
                
                // 減少引用計數並嘗試回收
                if (old_head->ref_count.fetch_sub(1) == 1) {
                    delete old_head;
                } else {
                    try_reclaim(old_head);
                }
                
                return result;
            } else {
                // CAS 失敗,減少引用計數
                old_head->ref_count.fetch_sub(1);
            }
        }
        
        return std::shared_ptr<T>();
    }
};
```

## Lock-Free Queue (SPSC)

### 單生產者單消費者佇列

```cpp
template<typename T, size_t Size>
class SPSCQueue {
    static_assert((Size & (Size - 1)) == 0, "Size must be power of 2");
    
private:
    std::array<T, Size> buffer_;
    
    // Cache-line 對齊,避免 false sharing
    alignas(64) std::atomic<size_t> write_idx_{0};
    alignas(64) std::atomic<size_t> read_idx_{0};
    
    size_t increment(size_t idx) const {
        return (idx + 1) & (Size - 1);  // 快速取模 (Size 為 2 的冪次)
    }
    
public:
    bool try_push(const T& item) {
        size_t current_write = write_idx_.load(std::memory_order_relaxed);
        size_t next_write = increment(current_write);
        
        // 檢查佇列是否滿
        if (next_write == read_idx_.load(std::memory_order_acquire)) {
            return false;
        }
        
        buffer_[current_write] = item;
        
        // Release: 確保資料寫入對消費者可見
        write_idx_.store(next_write, std::memory_order_release);
        return true;
    }
    
    bool try_pop(T& item) {
        size_t current_read = read_idx_.load(std::memory_order_relaxed);
        
        // 檢查佇列是否空
        if (current_read == write_idx_.load(std::memory_order_acquire)) {
            return false;
        }
        
        item = buffer_[current_read];
        
        // Release: 確保讀取完成對生產者可見
        read_idx_.store(increment(current_read), std::memory_order_release);
        return true;
    }
    
    size_t size() const {
        size_t w = write_idx_.load(std::memory_order_acquire);
        size_t r = read_idx_.load(std::memory_order_acquire);
        return (w >= r) ? (w - r) : (Size - r + w);
    }
    
    bool empty() const {
        return read_idx_.load(std::memory_order_acquire) == 
               write_idx_.load(std::memory_order_acquire);
    }
};
```

### HFT 應用: 市場資料佇列

```cpp
struct MarketData {
    uint64_t timestamp;
    uint32_t symbol_id;
    double price;
    int64_t quantity;
    char side;  // 'B' or 'A'
};

// 高頻交易系統中的市場資料佇列
using MarketDataQueue = SPSCQueue<MarketData, 65536>;  // 64K entries

class MarketDataProcessor {
    MarketDataQueue queue_;
    
public:
    // 網路執行緒 (生產者)
    void on_market_data(const MarketData& md) {
        if (!queue_.try_push(md)) {
            // 佇列滿: 記錄警告或丟棄舊資料
            handle_queue_full();
        }
    }
    
    // 策略執行緒 (消費者)
    void process() {
        MarketData md;
        while (queue_.try_pop(md)) {
            // 處理市場資料
            execute_strategy(md);
        }
    }
};
```

## Lock-Free Queue (MPMC)

### 多生產者多消費者佇列 (簡化版)

```cpp
template<typename T>
class MPMCQueue {
private:
    struct Node {
        std::shared_ptr<T> data;
        std::atomic<Node*> next{nullptr};
    };
    
    alignas(64) std::atomic<Node*> head_;
    alignas(64) std::atomic<Node*> tail_;
    
public:
    MPMCQueue() {
        Node* dummy = new Node();
        head_.store(dummy);
        tail_.store(dummy);
    }
    
    void enqueue(const T& item) {
        auto new_node = new Node();
        new_node->data = std::make_shared<T>(item);
        
        while (true) {
            Node* last = tail_.load(std::memory_order_acquire);
            Node* next = last->next.load(std::memory_order_acquire);
            
            if (last == tail_.load(std::memory_order_acquire)) {
                if (next == nullptr) {
                    // 嘗試將新節點加入尾部
                    if (last->next.compare_exchange_weak(
                        next, new_node,
                        std::memory_order_release,
                        std::memory_order_relaxed)) {
                        // 成功,更新 tail
                        tail_.compare_exchange_weak(
                            last, new_node,
                            std::memory_order_release,
                            std::memory_order_relaxed);
                        return;
                    }
                } else {
                    // 協助其他執行緒完成 enqueue
                    tail_.compare_exchange_weak(
                        last, next,
                        std::memory_order_release,
                        std::memory_order_relaxed);
                }
            }
        }
    }
    
    std::shared_ptr<T> dequeue() {
        while (true) {
            Node* first = head_.load(std::memory_order_acquire);
            Node* last = tail_.load(std::memory_order_acquire);
            Node* next = first->next.load(std::memory_order_acquire);
            
            if (first == head_.load(std::memory_order_acquire)) {
                if (first == last) {
                    if (next == nullptr) {
                        return std::shared_ptr<T>();  // 佇列空
                    }
                    // 協助完成 enqueue
                    tail_.compare_exchange_weak(
                        last, next,
                        std::memory_order_release,
                        std::memory_order_relaxed);
                } else {
                    std::shared_ptr<T> result = next->data;
                    
                    // 嘗試移動 head
                    if (head_.compare_exchange_weak(
                        first, next,
                        std::memory_order_release,
                        std::memory_order_relaxed)) {
                        delete first;  // 回收 dummy 節點
                        return result;
                    }
                }
            }
        }
    }
};
```

## Lock-Free Hash Table

### 基於 Hopscotch Hashing 的實作 (概念)

```cpp
template<typename Key, typename Value, size_t Size = 1024>
class LockFreeHashMap {
    static constexpr size_t HOP_RANGE = 32;  // Hopscotch 範圍
    
    struct Entry {
        std::atomic<Key> key;
        std::atomic<Value> value;
        std::atomic<uint32_t> hop_bitmap{0};  // 32-bit bitmap
        
        Entry() : key(Key{}), value(Value{}) {}
    };
    
    alignas(64) std::array<Entry, Size> table_;
    
    size_t hash(const Key& k) const {
        return std::hash<Key>{}(k) % Size;
    }
    
public:
    bool insert(const Key& key, const Value& value) {
        size_t idx = hash(key);
        
        // 在 hop_range 內尋找空位
        for (size_t i = 0; i < HOP_RANGE; ++i) {
            size_t pos = (idx + i) % Size;
            Key expected = Key{};
            
            if (table_[pos].key.compare_exchange_strong(
                expected, key,
                std::memory_order_release,
                std::memory_order_relaxed)) {
                
                table_[pos].value.store(value, std::memory_order_release);
                
                // 更新 bitmap
                uint32_t bit = 1u << i;
                table_[idx].hop_bitmap.fetch_or(bit, std::memory_order_release);
                return true;
            }
        }
        
        return false;  // 插入失敗
    }
    
    std::optional<Value> find(const Key& key) {
        size_t idx = hash(key);
        uint32_t bitmap = table_[idx].hop_bitmap.load(std::memory_order_acquire);
        
        // 檢查 bitmap 中標記的位置
        for (size_t i = 0; i < HOP_RANGE; ++i) {
            if (bitmap & (1u << i)) {
                size_t pos = (idx + i) % Size;
                
                if (table_[pos].key.load(std::memory_order_acquire) == key) {
                    return table_[pos].value.load(std::memory_order_acquire);
                }
            }
        }
        
        return std::nullopt;
    }
};
```

## 實戰: 訂單簿 (Order Book) Lock-Free 設計

```cpp
struct Order {
    uint64_t order_id;
    double price;
    int64_t quantity;
    uint64_t timestamp;
};

// 價格層級 (Price Level)
class PriceLevel {
    std::atomic<int64_t> total_quantity_{0};
    SPSCQueue<Order, 1024> orders_;  // 該價格的訂單
    
public:
    void add_order(const Order& order) {
        orders_.try_push(order);
        total_quantity_.fetch_add(order.quantity, std::memory_order_relaxed);
    }
    
    int64_t get_quantity() const {
        return total_quantity_.load(std::memory_order_relaxed);
    }
    
    bool remove_order(uint64_t order_id, int64_t quantity) {
        // 簡化: 實際需更複雜的搜尋機制
        total_quantity_.fetch_sub(quantity, std::memory_order_relaxed);
        return true;
    }
};

// 簡化的訂單簿 (單邊)
class OrderBookSide {
    static constexpr size_t MAX_LEVELS = 10000;
    
    // 使用 flat_map 或自訂 lock-free map
    std::array<PriceLevel, MAX_LEVELS> levels_;
    std::atomic<size_t> level_count_{0};
    
public:
    void add_order(const Order& order) {
        size_t level_idx = price_to_index(order.price);
        levels_[level_idx].add_order(order);
    }
    
    int64_t get_quantity_at_price(double price) const {
        size_t level_idx = price_to_index(price);
        return levels_[level_idx].get_quantity();
    }
    
private:
    size_t price_to_index(double price) const {
        // 簡化: 實際需更精細的價格映射
        return static_cast<size_t>(price * 100) % MAX_LEVELS;
    }
};
```

## 效能考量

### False Sharing 避免

```cpp
// 錯誤: 兩個原子變數在同一 cache line
struct BadCounter {
    std::atomic<int> counter1{0};  // 假設在 offset 0
    std::atomic<int> counter2{0};  // 假設在 offset 4
    // 同一 cache line (64 bytes)
};

// 修正: Cache-line 對齊
struct GoodCounter {
    alignas(64) std::atomic<int> counter1{0};
    alignas(64) std::atomic<int> counter2{0};
};
```

### 減少 CAS 失敗

```cpp
// 使用 Backoff 策略
class Backoff {
    int delay_ = 1;
    static constexpr int MAX_DELAY = 1024;
    
public:
    void backoff() {
        for (int i = 0; i < delay_; ++i) {
            __builtin_ia32_pause();  // CPU pause 指令
        }
        delay_ = std::min(delay_ * 2, MAX_DELAY);
    }
    
    void reset() {
        delay_ = 1;
    }
};

// 在 CAS 迴圈中使用
void push_with_backoff(Node* new_node) {
    Backoff backoff;
    new_node->next = head_.load(std::memory_order_relaxed);
    
    while (!head_.compare_exchange_weak(new_node->next, new_node,
                                       std::memory_order_release,
                                       std::memory_order_relaxed)) {
        backoff.backoff();
    }
}
```

## 測試與驗證

```cpp
#include <thread>
#include <vector>
#include <cassert>

// 並發測試
void test_lock_free_stack() {
    LockFreeStack<int> stack;
    constexpr int NUM_THREADS = 8;
    constexpr int OPS_PER_THREAD = 10000;
    
    // 生產者
    std::vector<std::thread> producers;
    for (int t = 0; t < NUM_THREADS; ++t) {
        producers.emplace_back([&stack, t]() {
            for (int i = 0; i < OPS_PER_THREAD; ++i) {
                stack.push(t * OPS_PER_THREAD + i);
            }
        });
    }
    
    // 消費者
    std::vector<std::thread> consumers;
    std::atomic<int> pop_count{0};
    
    for (int t = 0; t < NUM_THREADS; ++t) {
        consumers.emplace_back([&stack, &pop_count]() {
            for (int i = 0; i < OPS_PER_THREAD; ++i) {
                if (stack.pop()) {
                    pop_count.fetch_add(1);
                }
            }
        });
    }
    
    for (auto& t : producers) t.join();
    for (auto& t : consumers) t.join();
    
    assert(pop_count.load() == NUM_THREADS * OPS_PER_THREAD);
}
```

## 檢查清單

- [ ] 確認所有原子操作真正 lock-free (`is_lock_free()`)
- [ ] 使用適當的記憶體順序 (避免過度使用 `seq_cst`)
- [ ] 處理 ABA 問題 (使用版本計數或 Hazard Pointer)
- [ ] Cache-line 對齊,避免 false sharing
- [ ] CAS 迴圈使用 backoff 策略
- [ ] 記憶體回收機制 (Hazard Pointer, Epoch-Based, RCU)
- [ ] 充分的並發測試 (ThreadSanitizer, stress test)
- [ ] 效能測試 (延遲分布、吞吐量、擴展性)

---

## 參考資料 (References)

1. Herlihy, Maurice & Shavit, Nir. *The Art of Multiprocessor Programming* (2012)
2. Michael, Maged M. "Hazard pointers: Safe memory reclamation for lock-free objects" (2004)
3. [Folly: Facebook's Lock-Free Structures](https://github.com/facebook/folly)
4. [Boost.Lockfree](https://www.boost.org/doc/libs/release/doc/html/lockfree.html)
5. [1024cores - Lock-Free Algorithms](http://www.1024cores.net/home/lock-free-algorithms)
6. Williams, Anthony. *C++ Concurrency in Action*, 2nd Edition (2019)
