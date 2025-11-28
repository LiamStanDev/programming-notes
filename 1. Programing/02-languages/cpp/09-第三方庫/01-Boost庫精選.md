# Boost庫精選

> **優先級**: ⭐⭐ 建議
> **適用場景**: 通用開發、跨平台系統
> **前置知識**: 模板、智能指針、STL容器

## 目錄

- [核心概念](#核心概念)
- [Boost.Lockfree - 無鎖數據結構](#boostlockfree---無鎖數據結構)
- [Boost.Pool - 內存池](#boostpool---內存池)
- [Boost.Circular_Buffer - 環形緩衝區](#boostcircular_buffer---環形緩衝區)
- [Boost.Intrusive - 侵入式容器](#boostintrusive---侵入式容器)
- [Boost.Spirit - 解析器框架](#boostspirit---解析器框架)
- [Boost.Multiprecision - 高精度計算](#boostmultiprecision---高精度計算)
- [Boost.Interprocess - 進程間通信](#boostinterprocess---進程間通信)
- [HFT應用場景](#hft應用場景)
- [性能對比](#性能對比)
- [最佳實踐](#最佳實踐)
- [參考資料](#參考資料)

---

## 核心概念

### Boost庫簡介

**Boost** 是C++標準庫的試驗場,許多C++11/14/17/20的特性都源自Boost。本章聚焦於高性能開發相關的Boost組件,**排除Boost.Asio**(已在異步編程章節詳述)。

**為什麼使用Boost:**

- **久經考驗**: 20多年的生產環境驗證
- **高質量**: 嚴格的代碼審查和測試
- **跨平台**: Linux/Windows/macOS全面支持
- **性能優異**: 針對高性能場景優化

**安裝:**

```bash
# Ubuntu/Debian
sudo apt-get install libboost-all-dev

# Fedora/RHEL
sudo dnf install boost-devel

# macOS
brew install boost

# 驗證安裝
dpkg -L libboost-dev | grep "\.hpp$" | head -5
```

**CMake集成:**

```cmake
find_package(Boost 1.75 REQUIRED COMPONENTS 
    system thread filesystem lockfree)

target_link_libraries(myapp 
    Boost::system
    Boost::thread
    Boost::lockfree)
```

---

## Boost.Lockfree - 無鎖數據結構

### 核心原理

Boost.Lockfree提供無鎖(Lock-Free)數據結構,使用CAS(Compare-And-Swap)原子操作實現線程安全,避免互斥鎖的開銷。

**優勢:**

- **無阻塞**: 無鎖競爭,適合高並發
- **低延遲**: 避免上下文切換
- **無死鎖**: 不使用互斥鎖
- **進度保證**: 系統級進度保證(System-wide Progress)

### Lock-Free Queue

```cpp
#include <boost/lockfree/queue.hpp>
#include <thread>
#include <iostream>
#include <atomic>

// 固定容量的無鎖隊列
boost::lockfree::queue<int, boost::lockfree::capacity<1024>> fixed_queue;

// 動態容量的無鎖隊列 (使用malloc)
boost::lockfree::queue<int> dynamic_queue(1024);

// 生產者
void producer(int id, std::atomic<bool>& running) {
    int count = 0;
    while (running) {
        if (fixed_queue.push(count++)) {
            // 成功推入
        } else {
            // 隊列滿,稍後重試
            std::this_thread::yield();
        }
    }
    std::cout << "Producer " << id << " pushed " << count << " items\n";
}

// 消費者
void consumer(int id, std::atomic<bool>& running) {
    int value;
    int count = 0;
    while (running) {
        if (fixed_queue.pop(value)) {
            // 成功彈出
            count++;
        } else {
            // 隊列空,稍後重試
            std::this_thread::yield();
        }
    }
    std::cout << "Consumer " << id << " popped " << count << " items\n";
}

void lockfree_queue_demo() {
    std::atomic<bool> running{true};
    
    // 啟動多個生產者和消費者
    std::thread prod1(producer, 1, std::ref(running));
    std::thread prod2(producer, 2, std::ref(running));
    std::thread cons1(consumer, 1, std::ref(running));
    std::thread cons2(consumer, 2, std::ref(running));
    
    std::this_thread::sleep_for(std::chrono::seconds(1));
    running = false;
    
    prod1.join();
    prod2.join();
    cons1.join();
    cons2.join();
}
```

### Lock-Free Stack

```cpp
#include <boost/lockfree/stack.hpp>

// 固定容量無鎖棧
boost::lockfree::stack<int, boost::lockfree::capacity<1024>> fixed_stack;

// 動態容量無鎖棧
boost::lockfree::stack<int> dynamic_stack(1024);

void lockfree_stack_demo() {
    // 推入元素
    for (int i = 0; i < 100; ++i) {
        fixed_stack.push(i);
    }
    
    // 彈出元素
    int value;
    while (fixed_stack.pop(value)) {
        std::cout << value << " ";
    }
}
```

### Lock-Free SPSC Queue (單生產者單消費者)

```cpp
#include <boost/lockfree/spsc_queue.hpp>

// 單生產者單消費者隊列 (更高效)
boost::lockfree::spsc_queue<int, boost::lockfree::capacity<1024>> spsc_queue;

void spsc_demo() {
    // 生產者
    std::thread producer([&]() {
        for (int i = 0; i < 10000; ++i) {
            while (!spsc_queue.push(i)) {
                // Busy-wait
            }
        }
    });
    
    // 消費者
    std::thread consumer([&]() {
        int value;
        int count = 0;
        while (count < 10000) {
            if (spsc_queue.pop(value)) {
                count++;
            }
        }
        std::cout << "Consumed " << count << " items\n";
    });
    
    producer.join();
    consumer.join();
}
```

### HFT市場數據應用

```cpp
#include <boost/lockfree/spsc_queue.hpp>
#include <thread>
#include <chrono>

struct MarketTick {
    uint32_t symbol_id;
    double bid_price;
    double ask_price;
    uint32_t bid_size;
    uint32_t ask_size;
    uint64_t timestamp;
};

class MarketDataPipeline {
public:
    MarketDataPipeline() 
        : running_(true) {
        // 啟動處理線程
        processor_thread_ = std::thread(&MarketDataPipeline::process_loop, this);
    }
    
    ~MarketDataPipeline() {
        running_ = false;
        if (processor_thread_.joinable()) {
            processor_thread_.join();
        }
    }
    
    // 網絡線程調用 (生產者)
    void on_tick_received(const MarketTick& tick) {
        while (!tick_queue_.push(tick)) {
            // 隊列滿,Busy-wait (HFT中可接受)
        }
    }
    
private:
    // 處理線程 (消費者)
    void process_loop() {
        MarketTick tick;
        while (running_) {
            if (tick_queue_.pop(tick)) {
                // 更新訂單簿
                update_order_book(tick);
                
                // 觸發策略
                strategy_callback(tick);
            }
        }
    }
    
    void update_order_book(const MarketTick& tick) {
        // 實現訂單簿更新邏輯
    }
    
    void strategy_callback(const MarketTick& tick) {
        // 觸發交易策略
    }
    
    boost::lockfree::spsc_queue<MarketTick, 
        boost::lockfree::capacity<8192>> tick_queue_;
    std::thread processor_thread_;
    std::atomic<bool> running_;
};
```

---

## Boost.Pool - 內存池

### 核心概念

Boost.Pool提供高效的內存分配器,通過預分配大塊內存並分割為固定大小的塊,避免頻繁的malloc/free調用。

**適用場景:**

- 頻繁分配/釋放小對象
- 對象大小固定
- 需要確定性延遲

### Simple Pool

```cpp
#include <boost/pool/pool.hpp>
#include <iostream>

void simple_pool_demo() {
    // 創建內存池 (每塊32字節)
    boost::pool<> memory_pool(32);
    
    // 分配內存
    void* ptr1 = memory_pool.malloc();
    void* ptr2 = memory_pool.malloc();
    
    std::cout << "Allocated: " << ptr1 << ", " << ptr2 << "\n";
    
    // 釋放內存
    memory_pool.free(ptr1);
    memory_pool.free(ptr2);
    
    // 析構時自動釋放所有內存
}
```

### Object Pool

```cpp
#include <boost/pool/object_pool.hpp>
#include <string>

struct Order {
    int order_id;
    std::string symbol;
    double price;
    int quantity;
    
    Order(int id, const std::string& sym, double p, int q)
        : order_id(id), symbol(sym), price(p), quantity(q) {
        std::cout << "Order constructed: " << order_id << "\n";
    }
    
    ~Order() {
        std::cout << "Order destructed: " << order_id << "\n";
    }
};

void object_pool_demo() {
    // 創建對象池
    boost::object_pool<Order> order_pool;
    
    // 分配並構造對象
    Order* order1 = order_pool.construct(1, "AAPL", 150.50, 100);
    Order* order2 = order_pool.construct(2, "GOOGL", 2800.00, 50);
    
    std::cout << "Order 1: " << order1->symbol << " @ " << order1->price << "\n";
    
    // 釋放對象 (調用析構函數)
    order_pool.destroy(order1);
    order_pool.destroy(order2);
    
    // 析構池時釋放所有內存
}
```

### Pool Allocator (STL容器)

```cpp
#include <boost/pool/pool_alloc.hpp>
#include <vector>
#include <list>

void pool_allocator_demo() {
    // 使用Pool分配器的vector
    std::vector<int, boost::pool_allocator<int>> vec;
    
    for (int i = 0; i < 10000; ++i) {
        vec.push_back(i);  // 使用內存池分配
    }
    
    // 使用Pool分配器的list
    std::list<int, boost::pool_allocator<int>> lst;
    
    for (int i = 0; i < 10000; ++i) {
        lst.push_back(i);
    }
}
```

### HFT訂單池

```cpp
#include <boost/pool/object_pool.hpp>
#include <array>
#include <mutex>

class OrderPool {
public:
    struct Order {
        uint64_t order_id;
        uint32_t symbol_id;
        double price;
        uint32_t quantity;
        char side;  // 'B' or 'S'
        uint64_t timestamp;
    };
    
    OrderPool() = default;
    
    // 分配訂單
    Order* allocate_order(uint32_t symbol_id, double price, 
                         uint32_t quantity, char side) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        Order* order = pool_.construct();
        order->order_id = next_order_id_++;
        order->symbol_id = symbol_id;
        order->price = price;
        order->quantity = quantity;
        order->side = side;
        order->timestamp = get_timestamp();
        
        return order;
    }
    
    // 釋放訂單
    void free_order(Order* order) {
        std::lock_guard<std::mutex> lock(mutex_);
        pool_.destroy(order);
    }
    
private:
    uint64_t get_timestamp() const {
        return std::chrono::duration_cast<std::chrono::nanoseconds>(
            std::chrono::high_resolution_clock::now().time_since_epoch()).count();
    }
    
    boost::object_pool<Order> pool_;
    std::mutex mutex_;
    uint64_t next_order_id_ = 1;
};

void hft_order_pool_demo() {
    OrderPool pool;
    
    // 分配訂單
    auto* buy_order = pool.allocate_order(1, 150.50, 100, 'B');
    auto* sell_order = pool.allocate_order(1, 150.75, 100, 'S');
    
    std::cout << "Buy Order: " << buy_order->order_id << "\n";
    std::cout << "Sell Order: " << sell_order->order_id << "\n";
    
    // 釋放訂單
    pool.free_order(buy_order);
    pool.free_order(sell_order);
}
```

---

## Boost.Circular_Buffer - 環形緩衝區

### 核心概念

環形緩衝區(Circular Buffer)是固定大小的緩衝區,寫滿後會覆蓋最舊的數據。適合流式數據處理和時間窗口統計。

```cpp
#include <boost/circular_buffer.hpp>
#include <iostream>
#include <numeric>

void circular_buffer_basic() {
    // 創建大小為5的環形緩衝區
    boost::circular_buffer<int> cb(5);
    
    // 推入元素
    for (int i = 1; i <= 7; ++i) {
        cb.push_back(i);
        std::cout << "After push " << i << ": ";
        for (int val : cb) std::cout << val << " ";
        std::cout << "\n";
    }
    
    // 輸出:
    // After push 1: 1
    // After push 2: 1 2
    // After push 3: 1 2 3
    // After push 4: 1 2 3 4
    // After push 5: 1 2 3 4 5
    // After push 6: 2 3 4 5 6  (覆蓋1)
    // After push 7: 3 4 5 6 7  (覆蓋2)
}
```

### 滑動窗口統計

```cpp
#include <boost/circular_buffer.hpp>
#include <numeric>
#include <iostream>

class SlidingWindowStats {
public:
    explicit SlidingWindowStats(size_t window_size) 
        : buffer_(window_size) {}
    
    void add_price(double price) {
        buffer_.push_back(price);
    }
    
    double get_average() const {
        if (buffer_.empty()) return 0.0;
        return std::accumulate(buffer_.begin(), buffer_.end(), 0.0) / buffer_.size();
    }
    
    double get_max() const {
        if (buffer_.empty()) return 0.0;
        return *std::max_element(buffer_.begin(), buffer_.end());
    }
    
    double get_min() const {
        if (buffer_.empty()) return 0.0;
        return *std::min_element(buffer_.begin(), buffer_.end());
    }
    
    size_t size() const { return buffer_.size(); }
    
private:
    boost::circular_buffer<double> buffer_;
};

void sliding_window_demo() {
    SlidingWindowStats stats(5);  // 5個價格的滑動窗口
    
    std::vector<double> prices = {150.0, 151.0, 149.5, 152.0, 150.5, 148.0, 153.0};
    
    for (double price : prices) {
        stats.add_price(price);
        std::cout << "Price: " << price 
                  << ", Avg: " << stats.get_average()
                  << ", Max: " << stats.get_max()
                  << ", Min: " << stats.get_min() << "\n";
    }
}
```

### HFT時間序列分析

```cpp
#include <boost/circular_buffer.hpp>
#include <chrono>
#include <cmath>

struct PriceTick {
    double price;
    uint64_t timestamp;
};

class VolatilityCalculator {
public:
    explicit VolatilityCalculator(size_t window_size) 
        : ticks_(window_size) {}
    
    void add_tick(double price, uint64_t timestamp) {
        ticks_.push_back({price, timestamp});
    }
    
    // 計算波動率 (標準差)
    double calculate_volatility() const {
        if (ticks_.size() < 2) return 0.0;
        
        // 計算平均價格
        double sum = 0.0;
        for (const auto& tick : ticks_) {
            sum += tick.price;
        }
        double mean = sum / ticks_.size();
        
        // 計算標準差
        double variance = 0.0;
        for (const auto& tick : ticks_) {
            variance += (tick.price - mean) * (tick.price - mean);
        }
        variance /= ticks_.size();
        
        return std::sqrt(variance);
    }
    
    // 計算價格變化率
    double calculate_return() const {
        if (ticks_.size() < 2) return 0.0;
        return (ticks_.back().price - ticks_.front().price) / ticks_.front().price;
    }
    
private:
    boost::circular_buffer<PriceTick> ticks_;
};

void hft_volatility_demo() {
    VolatilityCalculator calc(100);  // 100個tick的滑動窗口
    
    // 模擬市場數據
    double base_price = 150.0;
    for (int i = 0; i < 200; ++i) {
        double price = base_price + (rand() % 100 - 50) / 100.0;
        uint64_t timestamp = i * 1000000;  // 微秒
        
        calc.add_tick(price, timestamp);
        
        if (i % 50 == 0) {
            std::cout << "Volatility: " << calc.calculate_volatility() 
                      << ", Return: " << calc.calculate_return() * 100 << "%\n";
        }
    }
}
```

---

## Boost.Intrusive - 侵入式容器

### 核心概念

侵入式容器(Intrusive Containers)將鏈表指針直接嵌入對象內部,避免額外的內存分配,提供更好的緩存局部性和可預測的性能。

**優勢:**

- **零額外分配**: 不分配節點內存
- **緩存友好**: 對象和鏈表指針連續
- **確定性**: 無動態分配,適合實時系統

```cpp
#include <boost/intrusive/list.hpp>
#include <iostream>

namespace bi = boost::intrusive;

// 侵入式鏈表節點
struct Order : public bi::list_base_hook<> {
    int order_id;
    std::string symbol;
    double price;
    
    Order(int id, const std::string& sym, double p)
        : order_id(id), symbol(sym), price(p) {}
};

void intrusive_list_demo() {
    // 創建侵入式鏈表
    bi::list<Order> order_list;
    
    // 創建對象 (可以在棧上或使用對象池)
    Order o1(1, "AAPL", 150.0);
    Order o2(2, "GOOGL", 2800.0);
    Order o3(3, "MSFT", 300.0);
    
    // 添加到鏈表 (無額外分配!)
    order_list.push_back(o1);
    order_list.push_back(o2);
    order_list.push_back(o3);
    
    // 遍歷
    for (const auto& order : order_list) {
        std::cout << "Order: " << order.order_id << ", " 
                  << order.symbol << " @ " << order.price << "\n";
    }
    
    // 從鏈表中移除
    order_list.erase(order_list.iterator_to(o2));
    
    // 對象生命週期獨立於鏈表
}
```

### 侵入式集合

```cpp
#include <boost/intrusive/set.hpp>

struct MarketData : public bi::set_base_hook<> {
    uint32_t symbol_id;
    double last_price;
    
    MarketData(uint32_t id, double price)
        : symbol_id(id), last_price(price) {}
    
    // 排序依據
    friend bool operator<(const MarketData& a, const MarketData& b) {
        return a.symbol_id < b.symbol_id;
    }
};

void intrusive_set_demo() {
    bi::set<MarketData> market_data_set;
    
    MarketData md1(1, 150.0);
    MarketData md2(2, 2800.0);
    MarketData md3(3, 300.0);
    
    market_data_set.insert(md1);
    market_data_set.insert(md2);
    market_data_set.insert(md3);
    
    // O(log n) 查找
    auto it = market_data_set.find(MarketData(2, 0));
    if (it != market_data_set.end()) {
        std::cout << "Found: Symbol " << it->symbol_id 
                  << ", Price: " << it->last_price << "\n";
    }
}
```

### HFT訂單簿實現

```cpp
#include <boost/intrusive/list.hpp>
#include <boost/intrusive/unordered_set.hpp>
#include <array>

struct OrderEntry : public bi::list_base_hook<>,
                   public bi::unordered_set_base_hook<> {
    uint64_t order_id;
    uint32_t price_level;  // 價格等級 (分)
    uint32_t quantity;
    uint64_t timestamp;
    
    // 哈希函數
    friend std::size_t hash_value(const OrderEntry& entry) {
        return std::hash<uint64_t>{}(entry.order_id);
    }
    
    friend bool operator==(const OrderEntry& a, const OrderEntry& b) {
        return a.order_id == b.order_id;
    }
};

class IntrusiveOrderBook {
public:
    IntrusiveOrderBook() : order_index_(1024) {}
    
    // 添加訂單
    void add_order(OrderEntry& order) {
        // 添加到價格等級鏈表
        price_levels_[order.price_level].push_back(order);
        
        // 添加到哈希表索引
        order_index_.insert(order);
    }
    
    // 查找訂單
    OrderEntry* find_order(uint64_t order_id) {
        OrderEntry dummy;
        dummy.order_id = order_id;
        
        auto it = order_index_.find(dummy);
        return it != order_index_.end() ? &(*it) : nullptr;
    }
    
    // 移除訂單
    void remove_order(OrderEntry& order) {
        price_levels_[order.price_level].erase(
            price_levels_[order.price_level].iterator_to(order));
        order_index_.erase(order_index_.iterator_to(order));
    }
    
    // 獲取最佳買價的訂單
    bi::list<OrderEntry>& get_best_bid() {
        for (int i = price_levels_.size() - 1; i >= 0; --i) {
            if (!price_levels_[i].empty()) {
                return price_levels_[i];
            }
        }
        static bi::list<OrderEntry> empty_list;
        return empty_list;
    }
    
private:
    std::array<bi::list<OrderEntry>, 10000> price_levels_;  // 價格等級
    bi::unordered_set<OrderEntry> order_index_;  // 訂單索引
};
```

---

## Boost.Spirit - 解析器框架

### 核心概念

Boost.Spirit是基於表達式模板的解析器框架,允許直接用C++編寫解析規則,無需外部工具。

**適用場景:**

- FIX協議解析
- 配置文件解析
- 自定義協議解析

```cpp
#include <boost/spirit/include/qi.hpp>
#include <string>
#include <iostream>

namespace qi = boost::spirit::qi;

// 解析逗號分隔的整數
void parse_csv() {
    std::string input = "150,200,300,400";
    std::vector<int> numbers;
    
    auto it = input.begin();
    bool success = qi::phrase_parse(it, input.end(),
        qi::int_ % ',',  // 規則: 整數,逗號分隔
        qi::space,       // 跳過空白
        numbers);
    
    if (success && it == input.end()) {
        std::cout << "Parsed numbers: ";
        for (int n : numbers) std::cout << n << " ";
        std::cout << "\n";
    }
}
```

### FIX協議解析示例

```cpp
#include <boost/spirit/include/qi.hpp>
#include <boost/fusion/include/adapt_struct.hpp>
#include <map>

struct FIXMessage {
    std::map<int, std::string> fields;
};

// 簡化的FIX消息解析
void parse_fix_message() {
    // FIX消息格式: tag=value|tag=value|...
    std::string fix_msg = "8=FIX.4.4|35=D|49=SENDER|56=TARGET|55=AAPL|54=1|38=100|40=2|44=150.50|";
    
    std::map<int, std::string> fields;
    
    auto it = fix_msg.begin();
    bool success = qi::phrase_parse(it, fix_msg.end(),
        (qi::int_ >> '=' >> +(qi::char_ - '|') >> '|') % qi::eps,
        qi::space,
        fields);
    
    if (success) {
        std::cout << "FIX Message:\n";
        for (const auto& [tag, value] : fields) {
            std::cout << "  Tag " << tag << " = " << value << "\n";
        }
    }
}
```

---

## Boost.Multiprecision - 高精度計算

### 核心概念

Boost.Multiprecision提供任意精度的整數和浮點數運算,適合金融計算和科學計算。

```cpp
#include <boost/multiprecision/cpp_int.hpp>
#include <boost/multiprecision/cpp_dec_float.hpp>
#include <iostream>

namespace mp = boost::multiprecision;

void multiprecision_demo() {
    // 128位整數
    mp::int128_t large_int = 1;
    for (int i = 0; i < 30; ++i) {
        large_int *= 10;
    }
    std::cout << "10^30 = " << large_int << "\n";
    
    // 50位精度浮點數
    mp::cpp_dec_float_50 precise_float("0.123456789012345678901234567890");
    mp::cpp_dec_float_50 result = precise_float * precise_float;
    std::cout << "Precise result: " << result << "\n";
}
```

### 金融計算應用

```cpp
#include <boost/multiprecision/cpp_dec_float.hpp>

using Decimal = boost::multiprecision::cpp_dec_float_50;

class PreciseOrderBook {
public:
    void add_order(const Decimal& price, const Decimal& quantity) {
        total_volume_ += price * quantity;
        count_++;
    }
    
    Decimal get_vwap() const {  // 成交量加權平均價格
        if (count_ == 0) return Decimal(0);
        return total_volume_ / count_;
    }
    
private:
    Decimal total_volume_{0};
    int count_ = 0;
};
```

---

## Boost.Interprocess - 進程間通信

### 共享內存

```cpp
#include <boost/interprocess/shared_memory_object.hpp>
#include <boost/interprocess/mapped_region.hpp>
#include <cstring>
#include <iostream>

namespace bip = boost::interprocess;

void shared_memory_writer() {
    // 創建共享內存
    bip::shared_memory_object shm(bip::create_only, "MySharedMemory", bip::read_write);
    
    // 設置大小
    shm.truncate(1024);
    
    // 映射到進程地址空間
    bip::mapped_region region(shm, bip::read_write);
    
    // 寫入數據
    std::strcpy(static_cast<char*>(region.get_address()), "Hello from Producer!");
    
    std::cout << "Data written to shared memory\n";
}

void shared_memory_reader() {
    // 打開共享內存
    bip::shared_memory_object shm(bip::open_only, "MySharedMemory", bip::read_only);
    
    // 映射到進程地址空間
    bip::mapped_region region(shm, bip::read_only);
    
    // 讀取數據
    const char* data = static_cast<const char*>(region.get_address());
    std::cout << "Data read from shared memory: " << data << "\n";
}
```

### 進程間消息隊列

```cpp
#include <boost/interprocess/ipc/message_queue.hpp>

void message_queue_sender() {
    namespace bip = boost::interprocess;
    
    // 創建消息隊列
    bip::message_queue mq(bip::create_only, "MyMessageQueue", 10, sizeof(int));
    
    // 發送消息
    for (int i = 0; i < 5; ++i) {
        mq.send(&i, sizeof(i), 0);
        std::cout << "Sent: " << i << "\n";
    }
}

void message_queue_receiver() {
    namespace bip = boost::interprocess;
    
    // 打開消息隊列
    bip::message_queue mq(bip::open_only, "MyMessageQueue");
    
    // 接收消息
    int data;
    size_t received_size;
    unsigned int priority;
    
    for (int i = 0; i < 5; ++i) {
        mq.receive(&data, sizeof(data), received_size, priority);
        std::cout << "Received: " << data << "\n";
    }
}
```

---

## HFT應用場景

### 場景1: 多級緩存系統

```cpp
#include <boost/circular_buffer.hpp>
#include <boost/lockfree/spsc_queue.hpp>
#include <boost/pool/object_pool.hpp>

class HFTCacheSystem {
public:
    struct MarketSnapshot {
        uint32_t symbol_id;
        double bid;
        double ask;
        uint64_t timestamp;
    };
    
    HFTCacheSystem()
        : recent_snapshots_(1000) {}
    
    void update_market(const MarketSnapshot& snapshot) {
        // 使用無鎖隊列傳遞給處理線程
        while (!update_queue_.push(snapshot)) {
            // Busy-wait
        }
    }
    
    void process_updates() {
        MarketSnapshot snapshot;
        while (update_queue_.pop(snapshot)) {
            // 更新環形緩衝區 (最近1000個快照)
            recent_snapshots_.push_back(snapshot);
            
            // 更新統計
            update_statistics(snapshot);
        }
    }
    
    // 獲取最近N個快照的平均價差
    double get_average_spread(size_t n) const {
        if (recent_snapshots_.empty()) return 0.0;
        
        size_t count = std::min(n, recent_snapshots_.size());
        double total_spread = 0.0;
        
        auto it = recent_snapshots_.end() - count;
        for (; it != recent_snapshots_.end(); ++it) {
            total_spread += (it->ask - it->bid);
        }
        
        return total_spread / count;
    }
    
private:
    void update_statistics(const MarketSnapshot& snapshot) {
        // 更新統計信息
    }
    
    boost::lockfree::spsc_queue<MarketSnapshot, 
        boost::lockfree::capacity<4096>> update_queue_;
    boost::circular_buffer<MarketSnapshot> recent_snapshots_;
};
```

---

## 性能對比

### Lock-Free Queue vs Mutex Queue

| 特性       | boost::lockfree::queue | std::queue + mutex |
| ---------- | ---------------------- | ------------------ |
| 延遲       | ~50-100 ns             | ~500-1000 ns       |
| 吞吐量     | 10M+ ops/s             | 1M ops/s           |
| 可擴展性   | 優秀                   | 較差               |
| 死鎖風險   | 無                     | 有                 |
| **HFT推薦** | ⭐⭐⭐⭐⭐              | ⭐⭐               |

### Object Pool vs malloc

| 特性       | boost::object_pool | malloc/new |
| ---------- | ------------------ | ---------- |
| 分配延遲   | ~10-20 ns          | ~200-500 ns |
| 確定性     | 高                 | 低         |
| 內存碎片   | 無                 | 有         |
| **HFT推薦** | ⭐⭐⭐⭐⭐          | ⭐⭐       |

---

## 最佳實踐

### 1. 選擇合適的容器

```cpp
// ✅ HFT場景推薦
boost::lockfree::spsc_queue<Data, capacity<4096>> queue;  // 單生產者單消費者
boost::object_pool<Order> order_pool;                     // 訂單對象池
boost::circular_buffer<Tick> tick_buffer(1000);           // 滑動窗口

// ❌ 避免使用
std::queue<Data> queue_with_lock;  // 需要外部鎖,延遲高
std::vector<Order*> order_list;    // 使用malloc,不確定
```

### 2. 固定容量 vs 動態容量

```cpp
// ✅ HFT推薦: 固定容量,避免動態分配
boost::lockfree::queue<int, boost::lockfree::capacity<1024>> fixed_queue;

// ❌ 避免: 動態容量,可能觸發malloc
boost::lockfree::queue<int> dynamic_queue(1024);
```

### 3. 對象生命週期管理

```cpp
// ✅ 正確: 確保對象在容器使用期間存活
void correct_usage() {
    boost::intrusive::list<Order> order_list;
    
    Order order1(1, "AAPL", 150.0);
    order_list.push_back(order1);
    
    // order1在作用域結束前保持有效
    for (auto& order : order_list) {
        process(order);
    }
}  // order1析構,自動從鏈表移除

// ❌ 錯誤: 對象過早析構
void incorrect_usage() {
    boost::intrusive::list<Order> order_list;
    
    {
        Order order1(1, "AAPL", 150.0);
        order_list.push_back(order1);
    }  // order1析構,但鏈表仍持有懸空引用!
    
    // 未定義行為!
    for (auto& order : order_list) {
        process(order);
    }
}
```

### 4. 線程安全考量

```cpp
// ✅ SPSC隊列: 單生產者單消費者,無需額外同步
boost::lockfree::spsc_queue<Data, capacity<1024>> spsc;

// ✅ MPMC隊列: 多生產者多消費者,內建同步
boost::lockfree::queue<Data, capacity<1024>> mpmc;

// ❌ 對象池: 需要外部同步
boost::object_pool<Order> pool;  // 多線程訪問需要mutex
```

---

## 參考資料

1. **Boost Documentation**
   - [Boost.Lockfree](https://www.boost.org/doc/libs/release/doc/html/lockfree.html)
   - [Boost.Pool](https://www.boost.org/doc/libs/release/libs/pool/doc/html/index.html)
   - [Boost.Circular_Buffer](https://www.boost.org/doc/libs/release/doc/html/circular_buffer.html)
   - [Boost.Intrusive](https://www.boost.org/doc/libs/release/doc/html/intrusive.html)
   - [Boost.Spirit](https://www.boost.org/doc/libs/release/libs/spirit/doc/html/index.html)
   - [Boost.Multiprecision](https://www.boost.org/doc/libs/release/libs/multiprecision/doc/html/index.html)
   - [Boost.Interprocess](https://www.boost.org/doc/libs/release/doc/html/interprocess.html)

2. **Performance Analysis**
   - 《The Art of Multiprocessor Programming》 - Maurice Herlihy
   - 《Lock-Free Programming》- Herb Sutter (Dr. Dobb's)

3. **Intrusive Containers**
   - [Boost.Intrusive Design Rationale](https://www.boost.org/doc/libs/release/doc/html/intrusive/intrusive_vs_nontrusive.html)

4. **Memory Pools**
   - 《Game Programming Patterns》- Robert Nystrom (Object Pool章節)

5. **Financial Applications**
   - 《Trading and Exchanges: Market Microstructure for Practitioners》- Larry Harris
