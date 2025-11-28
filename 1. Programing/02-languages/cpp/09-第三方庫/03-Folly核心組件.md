# Folly 核心組件

## 概述

Folly (Facebook Open-source LibrarY) 是 Meta 開發的高性能 C++ 基礎庫，專為大規模分散式系統設計。包含記憶體管理、並發原語、資料結構、字串處理等高效能組件。

### 核心特點

1. **生產驗證**: Meta 內部海量服務實戰
2. **效能導向**: 針對現代 CPU 架構優化
3. **並發友善**: 豐富的無鎖資料結構與同步原語
4. **現代 C++**: 充分利用 C++17/20 特性

### 安裝與整合

```bash
# Ubuntu/Debian
sudo apt-get install libfolly-dev

# 從源碼編譯
git clone https://github.com/facebook/folly.git
cd folly
mkdir _build && cd _build
cmake ..
make -j$(nproc)
sudo make install
```

CMake 整合:

```cmake
find_package(folly REQUIRED)
target_link_libraries(your_target folly)
```

---

## FBVector - 高效能動態陣列

### 設計原理

`FBVector` 是 `std::vector` 的替代品，針對以下場景優化:

1. **記憶體局部性**: 更激進的增長策略減少重新配置
2. **Small Size Optimization**: 小型向量避免堆分配
3. **移動語意**: 徹底優化的移動操作

### 基本使用

```cpp
#include <folly/FBVector.h>
#include <iostream>

int main() {
    folly::fbvector<int> vec;
    
    // 預留容量避免重新配置
    vec.reserve(1000);
    
    for (int i = 0; i < 1000; ++i) {
        vec.push_back(i);
    }
    
    // 容量增長策略更激進
    std::cout << "Size: " << vec.size() 
              << ", Capacity: " << vec.capacity() << "\n";
    
    return 0;
}
```

### 效能比較

```cpp
#include <folly/FBVector.h>
#include <folly/Benchmark.h>
#include <vector>

BENCHMARK(std_vector_push_back, n) {
    std::vector<int> vec;
    for (size_t i = 0; i < n; ++i) {
        vec.push_back(i);
    }
    folly::doNotOptimizeAway(vec);
}

BENCHMARK_RELATIVE(fbvector_push_back, n) {
    folly::fbvector<int> vec;
    for (size_t i = 0; i < n; ++i) {
        vec.push_back(i);
    }
    folly::doNotOptimizeAway(vec);
}

// 典型結果:
// std_vector_push_back    100000    1.25us
// fbvector_push_back      100000    1.05us  (1.19x faster)
```

### HFT 應用場景

```cpp
#include <folly/FBVector.h>

class MarketDataBuffer {
    struct Tick {
        uint64_t timestamp;
        double price;
        uint64_t volume;
    };
    
    folly::fbvector<Tick> buffer_;
    
public:
    MarketDataBuffer() {
        // 預留一個交易時段的典型容量
        buffer_.reserve(23400);  // 6.5小時 * 3600秒
    }
    
    void add_tick(uint64_t ts, double price, uint64_t vol) {
        buffer_.emplace_back(Tick{ts, price, vol});
    }
    
    // 快速清空但保留容量
    void reset_session() {
        buffer_.clear();
        // capacity 保持不變，避免下一交易時段重新配置
    }
};
```

---

## AtomicHashMap - 無鎖雜湊表

### 設計原理

`AtomicHashMap` 提供完全無鎖的讀寫操作:

1. **無鎖查找**: 讀取操作零鎖競爭
2. **原子插入**: CAS 操作保證線程安全
3. **固定容量**: 避免擴容時的全局鎖

### 基本操作

```cpp
#include <folly/AtomicHashMap.h>
#include <iostream>
#include <thread>

using AtomicMap = folly::AtomicHashMap<int, std::string>;

void writer_thread(AtomicMap& map, int id) {
    for (int i = 0; i < 1000; ++i) {
        int key = id * 1000 + i;
        auto result = map.insert(key, std::to_string(key));
        // result.first: iterator, result.second: 是否插入成功
    }
}

void reader_thread(const AtomicMap& map, int id) {
    for (int i = 0; i < 1000; ++i) {
        int key = (id - 1) * 1000 + i;
        auto it = map.find(key);
        if (it != map.end()) {
            // 無鎖讀取
            std::string value = it->second;
        }
    }
}

int main() {
    // 最大容量 100000 個元素
    AtomicMap map(100000);
    
    std::vector<std::thread> threads;
    
    // 4 個寫入線程
    for (int i = 1; i <= 4; ++i) {
        threads.emplace_back(writer_thread, std::ref(map), i);
    }
    
    // 4 個讀取線程
    for (int i = 1; i <= 4; ++i) {
        threads.emplace_back(reader_thread, std::ref(map), i);
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    std::cout << "Map size: " << map.size() << "\n";
    return 0;
}
```

### 高並發場景

```cpp
#include <folly/AtomicHashMap.h>
#include <atomic>

class SymbolCache {
    using SymbolMap = folly::AtomicHashMap<uint32_t, std::string>;
    
    SymbolMap symbol_to_name_;
    std::atomic<uint32_t> next_id_{1};
    
public:
    SymbolCache() : symbol_to_name_(10000) {}
    
    // 多線程安全的符號註冊
    uint32_t register_symbol(const std::string& name) {
        uint32_t id = next_id_.fetch_add(1, std::memory_order_relaxed);
        symbol_to_name_.insert(id, name);
        return id;
    }
    
    // 無鎖查找
    std::string get_symbol_name(uint32_t id) const {
        auto it = symbol_to_name_.find(id);
        if (it != symbol_to_name_.end()) {
            return it->second;
        }
        return "";
    }
};
```

### 效能特性

```cpp
#include <folly/AtomicHashMap.h>
#include <folly/Benchmark.h>
#include <unordered_map>
#include <shared_mutex>

// 傳統方案: 讀寫鎖 + unordered_map
class LockedMap {
    std::unordered_map<int, int> map_;
    mutable std::shared_mutex mutex_;
    
public:
    LockedMap(size_t capacity) { map_.reserve(capacity); }
    
    void insert(int key, int value) {
        std::unique_lock lock(mutex_);
        map_[key] = value;
    }
    
    bool find(int key, int& value) const {
        std::shared_lock lock(mutex_);
        auto it = map_.find(key);
        if (it != map_.end()) {
            value = it->second;
            return true;
        }
        return false;
    }
};

// Folly 方案
class AtomicMapWrapper {
    folly::AtomicHashMap<int, int> map_;
    
public:
    AtomicMapWrapper(size_t capacity) : map_(capacity) {}
    
    void insert(int key, int value) {
        map_.insert(key, value);
    }
    
    bool find(int key, int& value) const {
        auto it = map_.find(key);
        if (it != map_.end()) {
            value = it->second;
            return true;
        }
        return false;
    }
};

// 基準測試
BENCHMARK(locked_map_mixed_ops, n) {
    static LockedMap map(10000);
    for (size_t i = 0; i < n; ++i) {
        map.insert(i, i);
        int value;
        map.find(i - 100, value);
    }
}

BENCHMARK_RELATIVE(atomic_map_mixed_ops, n) {
    static AtomicMapWrapper map(10000);
    for (size_t i = 0; i < n; ++i) {
        map.insert(i, i);
        int value;
        map.find(i - 100, value);
    }
}

// 典型結果 (8 線程):
// locked_map_mixed_ops     10000    850ns
// atomic_map_mixed_ops     10000    320ns  (2.66x faster)
```

---

## ProducerConsumerQueue - 無鎖佇列

### 單生產者單消費者佇列

```cpp
#include <folly/ProducerConsumerQueue.h>
#include <thread>
#include <iostream>

struct Order {
    uint64_t order_id;
    uint32_t symbol_id;
    double price;
    uint64_t quantity;
};

void producer(folly::ProducerConsumerQueue<Order>& queue) {
    for (uint64_t i = 1; i <= 10000; ++i) {
        Order order{i, 1001, 150.5, 100};
        
        // 無鎖寫入
        while (!queue.write(order)) {
            // 佇列滿，自旋等待
            std::this_thread::yield();
        }
    }
}

void consumer(folly::ProducerConsumerQueue<Order>& queue) {
    uint64_t processed = 0;
    Order order;
    
    while (processed < 10000) {
        // 無鎖讀取
        if (queue.read(order)) {
            // 處理訂單
            ++processed;
        } else {
            std::this_thread::yield();
        }
    }
    
    std::cout << "Processed " << processed << " orders\n";
}

int main() {
    // 容量必須是 2 的冪次
    folly::ProducerConsumerQueue<Order> queue(1024);
    
    std::thread prod(producer, std::ref(queue));
    std::thread cons(consumer, std::ref(queue));
    
    prod.join();
    cons.join();
    
    return 0;
}
```

### 延遲測量

```cpp
#include <folly/ProducerConsumerQueue.h>
#include <chrono>
#include <thread>

struct TimestampedMessage {
    uint64_t data;
    std::chrono::nanoseconds send_time;
};

void latency_test() {
    folly::ProducerConsumerQueue<TimestampedMessage> queue(1024);
    std::atomic<bool> running{true};
    std::vector<uint64_t> latencies;
    latencies.reserve(100000);
    
    // 生產者
    std::thread producer([&]() {
        for (uint64_t i = 0; i < 100000; ++i) {
            auto now = std::chrono::steady_clock::now().time_since_epoch();
            TimestampedMessage msg{i, std::chrono::duration_cast<std::chrono::nanoseconds>(now)};
            
            while (!queue.write(msg)) {
                std::this_thread::yield();
            }
        }
    });
    
    // 消費者
    std::thread consumer([&]() {
        TimestampedMessage msg;
        uint64_t count = 0;
        
        while (count < 100000) {
            if (queue.read(msg)) {
                auto now = std::chrono::steady_clock::now().time_since_epoch();
                auto recv_time = std::chrono::duration_cast<std::chrono::nanoseconds>(now);
                
                uint64_t latency_ns = (recv_time - msg.send_time).count();
                latencies.push_back(latency_ns);
                ++count;
            }
        }
    });
    
    producer.join();
    consumer.join();
    
    // 計算統計數據
    std::sort(latencies.begin(), latencies.end());
    
    std::cout << "Latency P50: " << latencies[latencies.size() * 50 / 100] << " ns\n";
    std::cout << "Latency P99: " << latencies[latencies.size() * 99 / 100] << " ns\n";
    std::cout << "Latency P99.9: " << latencies[latencies.size() * 999 / 1000] << " ns\n";
}

// 典型結果:
// Latency P50: 45 ns
// Latency P99: 120 ns
// Latency P99.9: 280 ns
```

### 市場數據處理管線

```cpp
#include <folly/ProducerConsumerQueue.h>

struct MarketTick {
    uint64_t timestamp;
    uint32_t symbol_id;
    double bid;
    double ask;
    uint64_t bid_size;
    uint64_t ask_size;
};

class MarketDataPipeline {
    // 原始數據佇列
    folly::ProducerConsumerQueue<MarketTick> raw_queue_;
    
    // 處理後的數據佇列
    folly::ProducerConsumerQueue<MarketTick> processed_queue_;
    
    std::atomic<bool> running_{true};
    std::thread processor_thread_;
    
    void processor_loop() {
        MarketTick tick;
        
        while (running_.load(std::memory_order_relaxed)) {
            if (raw_queue_.read(tick)) {
                // 數據驗證與標準化
                if (tick.bid > 0 && tick.ask > tick.bid) {
                    // 寫入處理後佇列
                    while (!processed_queue_.write(tick) && 
                           running_.load(std::memory_order_relaxed)) {
                        std::this_thread::yield();
                    }
                }
            } else {
                std::this_thread::yield();
            }
        }
    }
    
public:
    MarketDataPipeline() 
        : raw_queue_(4096), processed_queue_(4096) {
        processor_thread_ = std::thread(&MarketDataPipeline::processor_loop, this);
    }
    
    ~MarketDataPipeline() {
        running_.store(false, std::memory_order_relaxed);
        if (processor_thread_.joinable()) {
            processor_thread_.join();
        }
    }
    
    // 網路線程寫入
    bool enqueue_raw(const MarketTick& tick) {
        return raw_queue_.write(tick);
    }
    
    // 策略線程讀取
    bool dequeue_processed(MarketTick& tick) {
        return processed_queue_.read(tick);
    }
};
```

---

## Synchronized - 同步原語

### 基本概念

`Synchronized<T>` 將數據與保護它的鎖綁定在一起，避免忘記加鎖的錯誤。

```cpp
#include <folly/Synchronized.h>
#include <map>
#include <string>

class OrderBook {
    struct Level {
        double price;
        uint64_t quantity;
    };
    
    // 自動管理鎖的訂單簿
    folly::Synchronized<std::map<double, uint64_t>> bids_;
    folly::Synchronized<std::map<double, uint64_t>> asks_;
    
public:
    void add_bid(double price, uint64_t quantity) {
        // RAII 風格的鎖管理
        auto locked = bids_.wlock();  // 寫鎖
        (*locked)[price] += quantity;
    }  // 鎖自動釋放
    
    void add_ask(double price, uint64_t quantity) {
        auto locked = asks_.wlock();
        (*locked)[price] += quantity;
    }
    
    double best_bid() const {
        auto locked = bids_.rlock();  // 讀鎖
        if (locked->empty()) return 0.0;
        return locked->rbegin()->first;
    }
    
    double best_ask() const {
        auto locked = asks_.rlock();
        if (locked->empty()) return 0.0;
        return locked->begin()->first;
    }
    
    // 原子性的複合操作
    std::pair<double, double> get_spread() const {
        // 同時鎖定兩個資源
        auto bids = bids_.rlock();
        auto asks = asks_.rlock();
        
        double bid = bids->empty() ? 0.0 : bids->rbegin()->first;
        double ask = asks->empty() ? 0.0 : asks->begin()->first;
        
        return {bid, ask};
    }
};
```

### 鎖策略選擇

```cpp
#include <folly/Synchronized.h>
#include <folly/SharedMutex.h>
#include <shared_mutex>

// 策略 1: 標準 std::mutex (預設)
folly::Synchronized<int> counter1;

// 策略 2: 讀寫鎖
folly::Synchronized<int, std::shared_mutex> counter2;

// 策略 3: Folly 的 SharedMutex (效能更佳)
folly::Synchronized<int, folly::SharedMutex> counter3;

// 策略 4: 自旋鎖 (低延遲場景)
#include <folly/SpinLock.h>
folly::Synchronized<int, folly::SpinLock> counter4;
```

### 高級操作

```cpp
#include <folly/Synchronized.h>
#include <vector>

class PositionManager {
    struct Position {
        uint32_t symbol_id;
        int64_t quantity;  // 正數: 多頭, 負數: 空頭
        double avg_price;
    };
    
    folly::Synchronized<std::vector<Position>> positions_;
    
public:
    // withWLock: Lambda 自動獲得寫鎖
    void update_position(uint32_t symbol_id, int64_t qty, double price) {
        positions_.withWLock([&](auto& positions) {
            auto it = std::find_if(positions.begin(), positions.end(),
                [symbol_id](const Position& p) { return p.symbol_id == symbol_id; });
            
            if (it != positions.end()) {
                // 更新現有持倉
                double total_cost = it->avg_price * it->quantity + price * qty;
                it->quantity += qty;
                if (it->quantity != 0) {
                    it->avg_price = total_cost / it->quantity;
                }
            } else {
                // 新增持倉
                positions.push_back({symbol_id, qty, price});
            }
        });
    }
    
    // withRLock: Lambda 自動獲得讀鎖
    int64_t get_position(uint32_t symbol_id) const {
        return positions_.withRLock([&](const auto& positions) -> int64_t {
            auto it = std::find_if(positions.begin(), positions.end(),
                [symbol_id](const Position& p) { return p.symbol_id == symbol_id; });
            
            return (it != positions.end()) ? it->quantity : 0;
        });
    }
    
    // copy: 原子性的複製整個結構
    std::vector<Position> get_all_positions() const {
        return positions_.copy();
    }
};
```

### 無拷貝訪問

```cpp
#include <folly/Synchronized.h>

class ConfigManager {
    struct Config {
        double max_position_size;
        double max_order_value;
        bool trading_enabled;
    };
    
    folly::Synchronized<Config> config_;
    
public:
    // 直接訪問成員，無需拷貝整個結構
    bool is_trading_enabled() const {
        return config_.rlock()->trading_enabled;
    }
    
    void set_trading_enabled(bool enabled) {
        config_.wlock()->trading_enabled = enabled;
    }
    
    // 條件式更新
    bool try_enable_trading_if_limits_ok(double pos_size, double order_val) {
        auto locked = config_.wlock();
        
        if (pos_size <= locked->max_position_size && 
            order_val <= locked->max_order_value) {
            locked->trading_enabled = true;
            return true;
        }
        
        return false;
    }
};
```

---

## Memory 組件

### Arena 記憶體池

```cpp
#include <folly/Memory.h>
#include <folly/memory/Arena.h>

class MessageAllocator {
    folly::SysArena arena_;
    
public:
    MessageAllocator() : arena_(4096) {}  // 4KB blocks
    
    template<typename T, typename... Args>
    T* allocate(Args&&... args) {
        void* mem = arena_.allocate(sizeof(T));
        return new (mem) T(std::forward<Args>(args)...);
    }
    
    // 批量分配
    void* allocate_batch(size_t size) {
        return arena_.allocate(size);
    }
    
    // Arena 析構時自動釋放所有記憶體
};

// 使用範例
struct MarketData {
    uint64_t timestamp;
    double price;
    uint64_t volume;
};

void process_market_session() {
    MessageAllocator allocator;
    
    // 高頻分配，無需逐一釋放
    for (int i = 0; i < 100000; ++i) {
        auto* tick = allocator.allocate<MarketData>(
            std::chrono::system_clock::now().time_since_epoch().count(),
            100.5 + i * 0.01,
            1000
        );
        
        // 使用 tick...
    }
    
    // 函數結束時 allocator 析構，所有記憶體一次性釋放
}
```

### 智慧指標工具

```cpp
#include <folly/Memory.h>
#include <memory>

class OrderFactory {
public:
    struct Order {
        uint64_t id;
        double price;
        uint64_t quantity;
        
        Order(uint64_t i, double p, uint64_t q) 
            : id(i), price(p), quantity(q) {}
    };
    
    // make_unique 的 Folly 版本
    static std::unique_ptr<Order> create_order(uint64_t id, double price, uint64_t qty) {
        return folly::make_unique<Order>(id, price, qty);
    }
    
    // 批量創建
    static std::vector<std::unique_ptr<Order>> create_batch(size_t count) {
        std::vector<std::unique_ptr<Order>> orders;
        orders.reserve(count);
        
        for (size_t i = 0; i < count; ++i) {
            orders.push_back(folly::make_unique<Order>(i, 100.0, 100));
        }
        
        return orders;
    }
};
```

---

## String 處理

### fbstring - 高效字串

```cpp
#include <folly/FBString.h>
#include <iostream>

int main() {
    // Small String Optimization (SSO)
    folly::fbstring small = "AAPL";  // 棧上儲存
    
    // 大字串堆分配
    folly::fbstring large = "Very long symbol name that exceeds SSO threshold";
    
    // Copy-on-Write (某些配置)
    folly::fbstring s1 = "Hello";
    folly::fbstring s2 = s1;  // 可能共享底層緩衝區
    
    s2 += " World";  // 觸發 COW
    
    std::cout << "s1: " << s1 << "\n";  // "Hello"
    std::cout << "s2: " << s2 << "\n";  // "Hello World"
    
    return 0;
}
```

### 字串分割

```cpp
#include <folly/String.h>
#include <folly/FBString.h>
#include <vector>

void parse_csv_line(const folly::fbstring& line) {
    std::vector<folly::fbstring> fields;
    
    // 高效分割
    folly::split(',', line, fields);
    
    for (const auto& field : fields) {
        // 處理欄位...
    }
}

// FIX 訊息解析
void parse_fix_message(const folly::fbstring& msg) {
    std::vector<folly::fbstring> tags;
    folly::split('\x01', msg, tags);  // FIX 使用 SOH 分隔符
    
    for (const auto& tag : tags) {
        std::vector<folly::fbstring> kv;
        folly::split('=', tag, kv);
        
        if (kv.size() == 2) {
            // kv[0]: tag number, kv[1]: value
        }
    }
}
```

### 字串格式化

```cpp
#include <folly/Format.h>

void format_examples() {
    uint64_t order_id = 123456;
    double price = 150.75;
    uint64_t quantity = 100;
    
    // 類型安全的格式化
    auto msg = folly::sformat(
        "Order {} @ {} x {}", 
        order_id, price, quantity
    );
    // "Order 123456 @ 150.75 x 100"
    
    // 指定精度
    auto price_str = folly::sformat("{:.2f}", price);  // "150.75"
    
    // 對齊與填充
    auto padded = folly::sformat("{:>10}", order_id);  // "    123456"
}
```

---

## Futures - 異步編程

### 基本 Future/Promise

```cpp
#include <folly/futures/Future.h>
#include <folly/executors/CPUThreadPoolExecutor.h>

using folly::Future;
using folly::Promise;
using folly::makeFuture;

// 創建 Future
Future<int> async_compute() {
    Promise<int> promise;
    auto future = promise.getSemiFuture().via(&folly::InlineExecutor::instance());
    
    // 在另一個線程計算
    std::thread([p = std::move(promise)]() mutable {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        p.setValue(42);
    }).detach();
    
    return future;
}

// 使用 Future
void example_usage() {
    async_compute()
        .thenValue([](int result) {
            std::cout << "Result: " << result << "\n";
            return result * 2;
        })
        .thenValue([](int doubled) {
            std::cout << "Doubled: " << doubled << "\n";
        });
}
```

### 並行執行

```cpp
#include <folly/futures/Future.h>
#include <folly/executors/CPUThreadPoolExecutor.h>
#include <vector>

using folly::Future;
using folly::collectAll;

struct PriceQuote {
    uint32_t symbol_id;
    double price;
};

class MultiExchangePricer {
    folly::CPUThreadPoolExecutor executor_;
    
public:
    MultiExchangePricer() : executor_(4) {}
    
    Future<PriceQuote> fetch_from_exchange(uint32_t symbol_id, int exchange_id) {
        return folly::via(&executor_, [=]() {
            // 模擬網路請求
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
            return PriceQuote{symbol_id, 100.0 + exchange_id};
        });
    }
    
    Future<std::vector<PriceQuote>> fetch_all_exchanges(uint32_t symbol_id) {
        std::vector<Future<PriceQuote>> futures;
        
        // 並行查詢 3 個交易所
        for (int i = 0; i < 3; ++i) {
            futures.push_back(fetch_from_exchange(symbol_id, i));
        }
        
        // 等待所有完成
        return collectAll(futures)
            .thenValue([](std::vector<folly::Try<PriceQuote>>&& results) {
                std::vector<PriceQuote> quotes;
                for (auto& result : results) {
                    if (result.hasValue()) {
                        quotes.push_back(std::move(result.value()));
                    }
                }
                return quotes;
            });
    }
    
    Future<PriceQuote> get_best_price(uint32_t symbol_id) {
        return fetch_all_exchanges(symbol_id)
            .thenValue([](std::vector<PriceQuote>&& quotes) {
                return *std::min_element(quotes.begin(), quotes.end(),
                    [](const PriceQuote& a, const PriceQuote& b) {
                        return a.price < b.price;
                    });
            });
    }
};
```

### 錯誤處理

```cpp
#include <folly/futures/Future.h>

Future<double> risky_operation() {
    return folly::makeFuture()
        .thenValue([](auto&&) -> double {
            if (rand() % 2 == 0) {
                throw std::runtime_error("Random failure");
            }
            return 123.45;
        })
        .thenError(folly::tag_t<std::runtime_error>{}, 
            [](const std::runtime_error& e) -> double {
                std::cerr << "Caught error: " << e.what() << "\n";
                return 0.0;  // 預設值
            });
}
```

---

## 效能基準測試

### Benchmark 框架

```cpp
#include <folly/Benchmark.h>
#include <folly/FBVector.h>
#include <vector>

BENCHMARK(std_vector_reserve_and_push, n) {
    std::vector<int> vec;
    vec.reserve(n);
    for (size_t i = 0; i < n; ++i) {
        vec.push_back(i);
    }
    folly::doNotOptimizeAway(vec.data());
}

BENCHMARK_RELATIVE(fbvector_reserve_and_push, n) {
    folly::fbvector<int> vec;
    vec.reserve(n);
    for (size_t i = 0; i < n; ++i) {
        vec.push_back(i);
    }
    folly::doNotOptimizeAway(vec.data());
}

BENCHMARK_DRAW_LINE();

BENCHMARK(std_vector_emplace, n) {
    std::vector<std::pair<int, int>> vec;
    vec.reserve(n);
    for (size_t i = 0; i < n; ++i) {
        vec.emplace_back(i, i * 2);
    }
    folly::doNotOptimizeAway(vec.data());
}

BENCHMARK_RELATIVE(fbvector_emplace, n) {
    folly::fbvector<std::pair<int, int>> vec;
    vec.reserve(n);
    for (size_t i = 0; i < n; ++i) {
        vec.emplace_back(i, i * 2);
    }
    folly::doNotOptimizeAway(vec.data());
}

int main(int argc, char** argv) {
    folly::runBenchmarks();
    return 0;
}
```

編譯與執行:

```bash
g++ -std=c++17 -O3 benchmark.cpp -lfolly -lglog -lgflags -pthread
./a.out --bm_min_iters=100000
```

輸出範例:

```
============================================================================
std_vector_reserve_and_push                               1.23us    812.52K
fbvector_reserve_and_push                     105.26%      1.17us    855.23K
----------------------------------------------------------------------------
std_vector_emplace                                         2.45us    408.16K
fbvector_emplace                              112.34%      2.18us    458.53K
============================================================================
```

---

## 工程整合

### CMake 配置

```cmake
cmake_minimum_required(VERSION 3.15)
project(HFTSystem)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -march=native")

find_package(folly REQUIRED)
find_package(gflags REQUIRED)
find_package(glog REQUIRED)

add_executable(trading_engine
    main.cpp
    order_book.cpp
    market_data.cpp
)

target_link_libraries(trading_engine
    folly
    glog::glog
    gflags
    pthread
)
```

### 編譯選項

```bash
# 最大優化
g++ -std=c++17 -O3 -march=native \
    -DNDEBUG \
    main.cpp \
    -lfolly -lglog -lgflags -lpthread

# 除錯模式
g++ -std=c++17 -O0 -g \
    -fsanitize=address -fsanitize=undefined \
    main.cpp \
    -lfolly -lglog -lgflags -lpthread
```

---

## 最佳實踐

### 1. 選擇合適的容器

```cpp
// 場景 1: 頻繁尾部插入，偶爾隨機訪問
folly::fbvector<Order> orders;  // 優於 std::vector

// 場景 2: 高並發讀寫，固定容量
folly::AtomicHashMap<uint32_t, Price> symbol_prices(10000);

// 場景 3: 單生產者單消費者
folly::ProducerConsumerQueue<MarketTick> tick_queue(4096);
```

### 2. 記憶體管理策略

```cpp
// 短生命週期批量對象
void process_batch() {
    folly::SysArena arena(8192);
    
    for (int i = 0; i < 10000; ++i) {
        auto* obj = new (arena.allocate(sizeof(MyObject))) MyObject();
        // 使用 obj...
    }
    // arena 析構時統一釋放
}

// 長生命週期對象
auto order = folly::make_unique<Order>(params...);
```

### 3. 並發控制

```cpp
// 讀多寫少: SharedMutex
folly::Synchronized<Config, folly::SharedMutex> config;

// 極低延遲: SpinLock
folly::Synchronized<Counter, folly::SpinLock> counter;

// 無競爭: 無鎖結構
folly::AtomicHashMap<Key, Value> cache(capacity);
```

### 4. 錯誤處理

```cpp
#include <folly/Try.h>

folly::Try<double> safe_divide(double a, double b) {
    if (b == 0.0) {
        return folly::Try<double>(std::runtime_error("Division by zero"));
    }
    return folly::Try<double>(a / b);
}

auto result = safe_divide(10.0, 2.0);
if (result.hasValue()) {
    std::cout << "Result: " << result.value() << "\n";
} else {
    std::cerr << "Error: " << result.exception().what() << "\n";
}
```

---

## 參考資料

1. [Folly 官方文檔](https://github.com/facebook/folly)
2. [Folly: Facebook's C++ Library](https://engineering.fb.com/2012/06/13/core-data/folly-the-facebook-open-source-library/)
3. [AtomicHashMap 設計](https://github.com/facebook/folly/blob/main/folly/AtomicHashMap.h)
4. [ProducerConsumerQueue 實現](https://github.com/facebook/folly/blob/main/folly/ProducerConsumerQueue.h)
5. [Folly Futures](https://github.com/facebook/folly/blob/main/folly/docs/Futures.md)
