# Abseil 核心組件

## 概述

Abseil (Abseil C++ Common Libraries) 是 Google 開發的 C++ 基礎庫，源自 Google 內部代碼庫。提供標準庫的擴展與增強，專注於可靠性、效能與可維護性。

### 核心特點

1. **生產驗證**: Google 全線產品使用
2. **標準庫增強**: 填補 C++ 標準庫的空白
3. **向前兼容**: 支援舊版 C++ 標準，提供新特性回溯
4. **高效能**: 針對大規模系統優化

### 安裝與整合

```bash
# 從源碼編譯
git clone https://github.com/abseil/abseil-cpp.git
cd abseil-cpp
mkdir build && cd build
cmake -DCMAKE_CXX_STANDARD=17 ..
make -j$(nproc)
sudo make install
```

CMake 整合:

```cmake
find_package(absl REQUIRED)

add_executable(app main.cpp)

target_link_libraries(app
    absl::strings
    absl::hash
    absl::time
    absl::flags
    absl::synchronization
    absl::container
)
```

---

## 字串處理 (absl::string)

### StrCat - 高效字串拼接

```cpp
#include <absl/strings/str_cat.h>
#include <iostream>

int main() {
    uint64_t order_id = 123456;
    double price = 150.75;
    uint64_t quantity = 100;
    
    // 單次分配，無臨時對象
    std::string msg = absl::StrCat(
        "Order ", order_id, 
        " @ ", price, 
        " x ", quantity
    );
    
    std::cout << msg << "\n";
    // "Order 123456 @ 150.75 x 100"
    
    return 0;
}
```

效能比較:

```cpp
#include <absl/strings/str_cat.h>
#include <sstream>
#include <chrono>

// 方法 1: std::ostringstream
std::string concat_with_stream(uint64_t id, double price, uint64_t qty) {
    std::ostringstream oss;
    oss << "Order " << id << " @ " << price << " x " << qty;
    return oss.str();
}

// 方法 2: operator+
std::string concat_with_plus(uint64_t id, double price, uint64_t qty) {
    return std::string("Order ") + std::to_string(id) + 
           " @ " + std::to_string(price) + 
           " x " + std::to_string(qty);
}

// 方法 3: absl::StrCat
std::string concat_with_strcat(uint64_t id, double price, uint64_t qty) {
    return absl::StrCat("Order ", id, " @ ", price, " x ", qty);
}

// 基準測試結果:
// std::ostringstream:  ~450ns
// operator+:           ~320ns
// absl::StrCat:        ~85ns  (5.3x faster than ostringstream)
```

### StrAppend - 原地拼接

```cpp
#include <absl/strings/str_cat.h>

void build_fix_message(std::string& msg, 
                       const std::string& tag, 
                       const std::string& value) {
    // 直接附加到現有字串，避免重新分配
    absl::StrAppend(&msg, tag, "=", value, "\x01");
}

std::string create_fix_order() {
    std::string msg;
    msg.reserve(256);  // 預留空間
    
    build_fix_message(msg, "35", "D");        // MsgType
    build_fix_message(msg, "11", "ORD001");   // ClOrdID
    build_fix_message(msg, "55", "AAPL");     // Symbol
    build_fix_message(msg, "54", "1");        // Side (Buy)
    build_fix_message(msg, "44", "150.50");   // Price
    build_fix_message(msg, "38", "100");      // OrderQty
    
    return msg;
}
```

### StrSplit - 字串分割

```cpp
#include <absl/strings/str_split.h>
#include <vector>
#include <string>

void parse_csv_line(absl::string_view line) {
    // 分割為 vector
    std::vector<std::string> fields = absl::StrSplit(line, ',');
    
    for (const auto& field : fields) {
        // 處理欄位...
    }
}

void parse_fix_message(absl::string_view msg) {
    // 自定義分隔符
    for (absl::string_view tag : absl::StrSplit(msg, '\x01')) {
        std::vector<std::string> kv = absl::StrSplit(tag, '=');
        
        if (kv.size() == 2) {
            // kv[0]: tag number, kv[1]: value
        }
    }
}

// 跳過空白欄位
void parse_with_skip_empty(absl::string_view input) {
    std::vector<std::string> tokens = 
        absl::StrSplit(input, ',', absl::SkipEmpty());
    
    // "a,,b,c" -> ["a", "b", "c"]
}
```

### StrFormat - 類型安全格式化

```cpp
#include <absl/strings/str_format.h>

void format_examples() {
    uint64_t order_id = 123456;
    double price = 150.755;
    
    // 類型安全，編譯期檢查
    std::string msg = absl::StrFormat(
        "Order %d @ %.2f", 
        order_id, price
    );
    // "Order 123456 @ 150.76"
    
    // 寬度與對齊
    std::string padded = absl::StrFormat("%10d", order_id);
    // "    123456"
    
    // 十六進位
    std::string hex = absl::StrFormat("0x%08X", order_id);
    // "0x0001E240"
}

// 直接輸出到檔案
void log_to_file(std::FILE* file, uint64_t ts, const char* event) {
    absl::FPrintF(file, "[%lu] %s\n", ts, event);
}
```

### string_view - 零拷貝字串視圖

```cpp
#include <absl/strings/string_view.h>
#include <string>

class MessageParser {
public:
    // 接受任何字串類型，無需拷貝
    static bool is_valid_symbol(absl::string_view symbol) {
        return symbol.size() >= 1 && symbol.size() <= 10;
    }
    
    static absl::string_view extract_field(
        absl::string_view msg, 
        size_t start, 
        size_t len) {
        // 零拷貝子字串
        return msg.substr(start, len);
    }
};

void usage_examples() {
    // 可以從多種來源構造
    std::string std_str = "AAPL";
    const char* c_str = "GOOGL";
    
    MessageParser::is_valid_symbol(std_str);   // 無拷貝
    MessageParser::is_valid_symbol(c_str);     // 無拷貝
    MessageParser::is_valid_symbol("MSFT");    // 無拷貝
    
    // 子字串視圖
    absl::string_view msg = "35=D\x0111=ORD001\x01";
    absl::string_view tag1 = msg.substr(0, 4);  // "35=D"
}
```

---

## 容器 (absl::container)

### flat_hash_map - 高效能雜湊表

```cpp
#include <absl/container/flat_hash_map.h>
#include <string>

class SymbolCache {
    // 比 std::unordered_map 更快的雜湊表
    absl::flat_hash_map<std::string, uint32_t> symbol_to_id_;
    absl::flat_hash_map<uint32_t, std::string> id_to_symbol_;
    
    uint32_t next_id_ = 1;
    
public:
    uint32_t register_symbol(const std::string& symbol) {
        auto [it, inserted] = symbol_to_id_.try_emplace(symbol, next_id_);
        
        if (inserted) {
            id_to_symbol_[next_id_] = symbol;
            return next_id_++;
        }
        
        return it->second;
    }
    
    std::string get_symbol(uint32_t id) const {
        auto it = id_to_symbol_.find(id);
        return (it != id_to_symbol_.end()) ? it->second : "";
    }
    
    uint32_t get_id(const std::string& symbol) const {
        auto it = symbol_to_id_.find(symbol);
        return (it != symbol_to_id_.end()) ? it->second : 0;
    }
};
```

效能比較:

```cpp
#include <absl/container/flat_hash_map.h>
#include <unordered_map>
#include <chrono>

template<typename MapType>
void benchmark_insert(size_t n) {
    MapType map;
    
    auto start = std::chrono::steady_clock::now();
    
    for (size_t i = 0; i < n; ++i) {
        map[i] = i * 2;
    }
    
    auto end = std::chrono::steady_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
    
    std::cout << "Insert " << n << " elements: " 
              << duration.count() << " us\n";
}

int main() {
    const size_t N = 1'000'000;
    
    benchmark_insert<std::unordered_map<int, int>>(N);
    // ~450ms
    
    benchmark_insert<absl::flat_hash_map<int, int>>(N);
    // ~280ms (1.6x faster)
    
    return 0;
}
```

### flat_hash_set - 高效能集合

```cpp
#include <absl/container/flat_hash_set.h>

class TradedSymbols {
    absl::flat_hash_set<uint32_t> active_symbols_;
    
public:
    void mark_traded(uint32_t symbol_id) {
        active_symbols_.insert(symbol_id);
    }
    
    bool is_traded_today(uint32_t symbol_id) const {
        return active_symbols_.contains(symbol_id);
    }
    
    size_t count() const {
        return active_symbols_.size();
    }
    
    void reset_session() {
        active_symbols_.clear();
    }
};
```

### node_hash_map - 指標穩定性

```cpp
#include <absl/container/node_hash_map.h>

struct OrderData {
    uint64_t order_id;
    double price;
    uint64_t quantity;
};

class OrderBook {
    // node_hash_map 保證元素地址不變 (即使擴容)
    absl::node_hash_map<uint64_t, OrderData> orders_;
    
public:
    OrderData* add_order(uint64_t id, double price, uint64_t qty) {
        auto [it, inserted] = orders_.try_emplace(id, OrderData{id, price, qty});
        
        // 返回的指標在 map 生命週期內始終有效
        return &it->second;
    }
    
    void update_quantity(OrderData* order, uint64_t new_qty) {
        // order 指標始終有效，即使 map 擴容
        order->quantity = new_qty;
    }
    
    void remove_order(uint64_t id) {
        orders_.erase(id);
    }
};
```

### btree_map - 有序容器

```cpp
#include <absl/container/btree_map.h>

class LimitOrderBook {
    // B-tree 實現，比 std::map (紅黑樹) 更高效
    absl::btree_map<double, uint64_t> bids_;   // price -> quantity
    absl::btree_map<double, uint64_t> asks_;
    
public:
    void add_bid(double price, uint64_t quantity) {
        bids_[price] += quantity;
    }
    
    void add_ask(double price, uint64_t quantity) {
        asks_[price] += quantity;
    }
    
    // O(1) 訪問最佳價格
    double best_bid() const {
        return bids_.empty() ? 0.0 : bids_.rbegin()->first;
    }
    
    double best_ask() const {
        return asks_.empty() ? 0.0 : asks_.begin()->first;
    }
    
    // 範圍查詢: 取得價格區間內的所有掛單
    std::vector<std::pair<double, uint64_t>> get_bids_in_range(
        double low, double high) const {
        
        std::vector<std::pair<double, uint64_t>> result;
        
        auto it_low = bids_.lower_bound(low);
        auto it_high = bids_.upper_bound(high);
        
        for (auto it = it_low; it != it_high; ++it) {
            result.push_back(*it);
        }
        
        return result;
    }
};
```

---

## 時間處理 (absl::time)

### 基本操作

```cpp
#include <absl/time/time.h>
#include <absl/time/clock.h>
#include <iostream>

void time_examples() {
    // 獲取當前時間
    absl::Time now = absl::Now();
    
    // 時間運算
    absl::Duration one_second = absl::Seconds(1);
    absl::Duration one_ms = absl::Milliseconds(1);
    absl::Duration one_us = absl::Microseconds(1);
    absl::Duration one_ns = absl::Nanoseconds(1);
    
    absl::Time future = now + absl::Seconds(60);
    absl::Duration elapsed = future - now;
    
    // 轉換為數值
    int64_t ns = absl::ToInt64Nanoseconds(elapsed);
    double seconds = absl::ToDoubleSeconds(elapsed);
    
    std::cout << "Nanoseconds: " << ns << "\n";
    std::cout << "Seconds: " << seconds << "\n";
}
```

### 高精度計時

```cpp
#include <absl/time/time.h>
#include <absl/time/clock.h>

class LatencyTracker {
    absl::Time start_time_;
    
public:
    void start() {
        start_time_ = absl::Now();
    }
    
    int64_t elapsed_ns() const {
        absl::Duration elapsed = absl::Now() - start_time_;
        return absl::ToInt64Nanoseconds(elapsed);
    }
    
    double elapsed_us() const {
        absl::Duration elapsed = absl::Now() - start_time_;
        return absl::ToDoubleMicroseconds(elapsed);
    }
};

void measure_operation() {
    LatencyTracker tracker;
    tracker.start();
    
    // 執行操作...
    
    std::cout << "Operation took " << tracker.elapsed_us() << " us\n";
}
```

### 時間格式化

```cpp
#include <absl/time/time.h>
#include <absl/time/clock.h>
#include <iostream>

void format_examples() {
    absl::Time now = absl::Now();
    
    // ISO 8601 格式
    std::string iso = absl::FormatTime(now);
    std::cout << "ISO: " << iso << "\n";
    // "2024-01-15T14:30:45.123456+00:00"
    
    // 自定義格式
    std::string custom = absl::FormatTime("%Y-%m-%d %H:%M:%S", now, absl::LocalTimeZone());
    std::cout << "Custom: " << custom << "\n";
    // "2024-01-15 14:30:45"
    
    // Unix 時間戳
    int64_t unix_ns = absl::ToUnixNanos(now);
    std::cout << "Unix nanos: " << unix_ns << "\n";
}
```

### 延遲統計

```cpp
#include <absl/time/time.h>
#include <absl/time/clock.h>
#include <vector>
#include <algorithm>

class LatencyCollector {
    std::vector<absl::Duration> samples_;
    
public:
    void record(absl::Duration latency) {
        samples_.push_back(latency);
    }
    
    void print_statistics() const {
        if (samples_.empty()) return;
        
        auto sorted = samples_;
        std::sort(sorted.begin(), sorted.end());
        
        auto p50 = sorted[sorted.size() * 50 / 100];
        auto p99 = sorted[sorted.size() * 99 / 100];
        auto p999 = sorted[sorted.size() * 999 / 1000];
        
        std::cout << "P50:  " << absl::ToDoubleMicroseconds(p50) << " us\n";
        std::cout << "P99:  " << absl::ToDoubleMicroseconds(p99) << " us\n";
        std::cout << "P999: " << absl::ToDoubleMicroseconds(p999) << " us\n";
    }
};

void benchmark_with_collector() {
    LatencyCollector collector;
    
    for (int i = 0; i < 10000; ++i) {
        absl::Time start = absl::Now();
        
        // 執行操作...
        
        absl::Duration latency = absl::Now() - start;
        collector.record(latency);
    }
    
    collector.print_statistics();
}
```

---

## 並發同步 (absl::synchronization)

### Mutex - 互斥鎖

```cpp
#include <absl/synchronization/mutex.h>
#include <vector>

class OrderQueue {
    absl::Mutex mutex_;
    std::vector<uint64_t> orders_;
    
public:
    void add_order(uint64_t order_id) {
        absl::MutexLock lock(&mutex_);
        orders_.push_back(order_id);
    }
    
    bool try_get_order(uint64_t& order_id) {
        absl::MutexLock lock(&mutex_);
        
        if (orders_.empty()) {
            return false;
        }
        
        order_id = orders_.back();
        orders_.pop_back();
        return true;
    }
    
    size_t size() const {
        absl::MutexLock lock(&mutex_);
        return orders_.size();
    }
};
```

### 條件等待

```cpp
#include <absl/synchronization/mutex.h>
#include <queue>

template<typename T>
class BlockingQueue {
    mutable absl::Mutex mutex_;
    std::queue<T> queue_;
    
public:
    void push(T item) {
        absl::MutexLock lock(&mutex_);
        queue_.push(std::move(item));
    }
    
    T pop() {
        absl::MutexLock lock(&mutex_);
        
        // 等待直到佇列非空
        mutex_.Await(absl::Condition(
            +[](std::queue<T>* q) { return !q->empty(); },
            &queue_
        ));
        
        T item = std::move(queue_.front());
        queue_.pop();
        return item;
    }
    
    bool try_pop(T& item, absl::Duration timeout) {
        absl::MutexLock lock(&mutex_);
        
        // 帶超時的等待
        bool ready = mutex_.AwaitWithTimeout(
            absl::Condition(
                +[](std::queue<T>* q) { return !q->empty(); },
                &queue_
            ),
            timeout
        );
        
        if (ready) {
            item = std::move(queue_.front());
            queue_.pop();
            return true;
        }
        
        return false;
    }
};

// 使用範例
void worker_thread(BlockingQueue<int>& queue) {
    while (true) {
        int item = queue.pop();  // 阻塞直到有數據
        // 處理 item...
    }
}
```

### ReaderWriterMutex 模式

```cpp
#include <absl/synchronization/mutex.h>
#include <absl/container/flat_hash_map.h>

class ConfigStore {
    mutable absl::Mutex mutex_;
    absl::flat_hash_map<std::string, std::string> config_;
    
public:
    void set(const std::string& key, const std::string& value) {
        absl::MutexLock lock(&mutex_);
        config_[key] = value;
    }
    
    std::string get(const std::string& key) const {
        // 讀鎖
        absl::ReaderMutexLock lock(&mutex_);
        
        auto it = config_.find(key);
        return (it != config_.end()) ? it->second : "";
    }
    
    bool try_update(const std::string& key, 
                    const std::string& old_value,
                    const std::string& new_value) {
        absl::MutexLock lock(&mutex_);
        
        auto it = config_.find(key);
        if (it != config_.end() && it->second == old_value) {
            it->second = new_value;
            return true;
        }
        
        return false;
    }
};
```

### Notification - 一次性通知

```cpp
#include <absl/synchronization/notification.h>
#include <thread>

class StartupCoordinator {
    absl::Notification initialization_complete_;
    
public:
    void run_async_initialization() {
        std::thread([this]() {
            // 執行初始化...
            std::this_thread::sleep_for(std::chrono::seconds(2));
            
            // 標記完成
            initialization_complete_.Notify();
        }).detach();
    }
    
    void wait_for_initialization() {
        // 阻塞直到初始化完成
        initialization_complete_.WaitForNotification();
    }
    
    bool is_initialized() const {
        return initialization_complete_.HasBeenNotified();
    }
};

void main_thread() {
    StartupCoordinator coordinator;
    
    coordinator.run_async_initialization();
    
    // 執行其他工作...
    
    coordinator.wait_for_initialization();
    
    // 現在可以安全使用已初始化的資源
}
```

---

## 數值計算 (absl::numeric)

### 安全整數運算

```cpp
#include <absl/numeric/int128.h>
#include <iostream>

void int128_examples() {
    // 128 位整數
    absl::int128 large1 = absl::MakeInt128(1, 0);  // 2^64
    absl::int128 large2 = absl::MakeInt128(0, std::numeric_limits<uint64_t>::max());
    
    absl::int128 sum = large1 + large2;
    absl::int128 product = large1 * large2;
    
    std::cout << "Sum: " << absl::Int128High64(sum) << " (high), "
              << absl::Int128Low64(sum) << " (low)\n";
}

// 金融計算: 避免浮點誤差
class PrecisePrice {
    absl::int128 value_in_cents_;  // 以分為單位
    
public:
    explicit PrecisePrice(double dollars) {
        value_in_cents_ = static_cast<absl::int128>(dollars * 100);
    }
    
    PrecisePrice operator+(const PrecisePrice& other) const {
        PrecisePrice result(0);
        result.value_in_cents_ = value_in_cents_ + other.value_in_cents_;
        return result;
    }
    
    double to_dollars() const {
        return static_cast<double>(value_in_cents_) / 100.0;
    }
};
```

---

## 雜湊函式 (absl::hash)

### 自定義雜湊

```cpp
#include <absl/hash/hash.h>
#include <absl/container/flat_hash_map.h>

struct OrderKey {
    uint32_t symbol_id;
    uint64_t client_id;
    
    template <typename H>
    friend H AbslHashValue(H h, const OrderKey& key) {
        return H::combine(std::move(h), key.symbol_id, key.client_id);
    }
    
    bool operator==(const OrderKey& other) const {
        return symbol_id == other.symbol_id && 
               client_id == other.client_id;
    }
};

// 直接使用於 Abseil 容器
using OrderMap = absl::flat_hash_map<OrderKey, std::string>;

void usage() {
    OrderMap orders;
    
    OrderKey key{1001, 999};
    orders[key] = "Order details";
    
    auto it = orders.find(key);
    if (it != orders.end()) {
        std::cout << "Found: " << it->second << "\n";
    }
}
```

### 組合雜湊

```cpp
#include <absl/hash/hash.h>

class MarketDataKey {
    std::string exchange_;
    std::string symbol_;
    uint64_t timestamp_;
    
public:
    MarketDataKey(std::string ex, std::string sym, uint64_t ts)
        : exchange_(std::move(ex)), symbol_(std::move(sym)), timestamp_(ts) {}
    
    template <typename H>
    friend H AbslHashValue(H h, const MarketDataKey& key) {
        return H::combine(
            std::move(h), 
            key.exchange_, 
            key.symbol_, 
            key.timestamp_
        );
    }
    
    bool operator==(const MarketDataKey& other) const {
        return exchange_ == other.exchange_ &&
               symbol_ == other.symbol_ &&
               timestamp_ == other.timestamp_;
    }
};
```

---

## 命令列標誌 (absl::flags)

### 定義與使用

```cpp
#include <absl/flags/flag.h>
#include <absl/flags/parse.h>
#include <iostream>

// 定義命令列標誌
ABSL_FLAG(std::string, config_file, "config.json", "Configuration file path");
ABSL_FLAG(int, port, 8080, "Server port");
ABSL_FLAG(bool, enable_logging, true, "Enable logging");
ABSL_FLAG(double, risk_limit, 1000000.0, "Risk limit in dollars");

int main(int argc, char* argv[]) {
    // 解析命令列
    absl::ParseCommandLine(argc, argv);
    
    // 讀取標誌值
    std::string config = absl::GetFlag(FLAGS_config_file);
    int port = absl::GetFlag(FLAGS_port);
    bool logging = absl::GetFlag(FLAGS_enable_logging);
    double risk = absl::GetFlag(FLAGS_risk_limit);
    
    std::cout << "Config: " << config << "\n";
    std::cout << "Port: " << port << "\n";
    std::cout << "Logging: " << (logging ? "ON" : "OFF") << "\n";
    std::cout << "Risk limit: " << risk << "\n";
    
    return 0;
}
```

執行範例:

```bash
./app --config_file=/etc/app.conf --port=9090 --enable_logging=false

# 輸出:
# Config: /etc/app.conf
# Port: 9090
# Logging: OFF
# Risk limit: 1000000
```

### 動態更新標誌

```cpp
#include <absl/flags/flag.h>

ABSL_FLAG(double, max_order_size, 10000.0, "Maximum order size");

class RiskManager {
public:
    bool check_order_size(double size) const {
        double limit = absl::GetFlag(FLAGS_max_order_size);
        return size <= limit;
    }
    
    void update_limit(double new_limit) {
        absl::SetFlag(&FLAGS_max_order_size, new_limit);
    }
};
```

---

## 狀態碼 (absl::Status)

### 基本錯誤處理

```cpp
#include <absl/status/status.h>
#include <absl/status/statusor.h>
#include <iostream>

absl::Status validate_order(double price, uint64_t quantity) {
    if (price <= 0) {
        return absl::InvalidArgumentError("Price must be positive");
    }
    
    if (quantity == 0) {
        return absl::InvalidArgumentError("Quantity must be non-zero");
    }
    
    if (quantity > 1000000) {
        return absl::OutOfRangeError("Quantity exceeds maximum limit");
    }
    
    return absl::OkStatus();
}

void process_order() {
    absl::Status status = validate_order(-10.0, 100);
    
    if (!status.ok()) {
        std::cerr << "Order validation failed: " 
                  << status.message() << "\n";
        return;
    }
    
    // 處理訂單...
}
```

### StatusOr - 返回值或錯誤

```cpp
#include <absl/status/statusor.h>

struct OrderResult {
    uint64_t order_id;
    double filled_price;
    uint64_t filled_quantity;
};

absl::StatusOr<OrderResult> execute_order(
    const std::string& symbol,
    double price,
    uint64_t quantity) {
    
    // 驗證
    if (symbol.empty()) {
        return absl::InvalidArgumentError("Symbol cannot be empty");
    }
    
    if (price <= 0) {
        return absl::InvalidArgumentError("Invalid price");
    }
    
    // 模擬訂單執行
    if (rand() % 10 == 0) {
        return absl::UnavailableError("Market temporarily unavailable");
    }
    
    // 成功
    OrderResult result{12345, price, quantity};
    return result;
}

void trading_logic() {
    absl::StatusOr<OrderResult> result = execute_order("AAPL", 150.0, 100);
    
    if (!result.ok()) {
        std::cerr << "Order failed: " << result.status().message() << "\n";
        return;
    }
    
    // 訪問結果
    const OrderResult& order = result.value();
    std::cout << "Order " << order.order_id 
              << " filled @ " << order.filled_price << "\n";
}
```

### 錯誤傳播

```cpp
#include <absl/status/status.h>
#include <absl/status/statusor.h>

class TradingEngine {
public:
    absl::StatusOr<double> get_account_balance(uint64_t account_id) {
        if (account_id == 0) {
            return absl::InvalidArgumentError("Invalid account ID");
        }
        
        // 模擬查詢
        return 1000000.0;
    }
    
    absl::StatusOr<bool> validate_order_against_balance(
        uint64_t account_id,
        double order_value) {
        
        // 獲取餘額 (可能失敗)
        absl::StatusOr<double> balance_or = get_account_balance(account_id);
        
        if (!balance_or.ok()) {
            // 錯誤傳播
            return balance_or.status();
        }
        
        double balance = balance_or.value();
        return balance >= order_value;
    }
};
```

---

## 實戰應用

### 高效能訂單處理系統

```cpp
#include <absl/container/flat_hash_map.h>
#include <absl/strings/str_cat.h>
#include <absl/time/time.h>
#include <absl/time/clock.h>
#include <absl/synchronization/mutex.h>

struct Order {
    uint64_t order_id;
    uint32_t symbol_id;
    double price;
    uint64_t quantity;
    absl::Time created_at;
};

class OrderManager {
    absl::Mutex mutex_;
    absl::flat_hash_map<uint64_t, Order> active_orders_;
    uint64_t next_order_id_ = 1;
    
public:
    absl::StatusOr<uint64_t> create_order(
        uint32_t symbol_id, 
        double price, 
        uint64_t quantity) {
        
        if (price <= 0) {
            return absl::InvalidArgumentError("Invalid price");
        }
        
        if (quantity == 0) {
            return absl::InvalidArgumentError("Invalid quantity");
        }
        
        absl::MutexLock lock(&mutex_);
        
        uint64_t order_id = next_order_id_++;
        
        Order order{
            order_id,
            symbol_id,
            price,
            quantity,
            absl::Now()
        };
        
        active_orders_[order_id] = order;
        
        return order_id;
    }
    
    absl::StatusOr<Order> get_order(uint64_t order_id) const {
        absl::ReaderMutexLock lock(&mutex_);
        
        auto it = active_orders_.find(order_id);
        if (it == active_orders_.end()) {
            return absl::NotFoundError(
                absl::StrCat("Order ", order_id, " not found")
            );
        }
        
        return it->second;
    }
    
    absl::Status cancel_order(uint64_t order_id) {
        absl::MutexLock lock(&mutex_);
        
        auto it = active_orders_.find(order_id);
        if (it == active_orders_.end()) {
            return absl::NotFoundError("Order not found");
        }
        
        active_orders_.erase(it);
        return absl::OkStatus();
    }
    
    size_t active_count() const {
        absl::ReaderMutexLock lock(&mutex_);
        return active_orders_.size();
    }
};
```

### 市場數據聚合器

```cpp
#include <absl/container/flat_hash_map.h>
#include <absl/container/btree_map.h>
#include <absl/time/time.h>

struct Tick {
    double price;
    uint64_t volume;
    absl::Time timestamp;
};

class MarketDataAggregator {
    struct SymbolData {
        absl::btree_map<absl::Time, Tick> ticks;
        double vwap = 0.0;  // Volume-Weighted Average Price
        uint64_t total_volume = 0;
    };
    
    absl::flat_hash_map<uint32_t, SymbolData> symbol_data_;
    
public:
    void add_tick(uint32_t symbol_id, double price, uint64_t volume) {
        absl::Time now = absl::Now();
        
        auto& data = symbol_data_[symbol_id];
        
        Tick tick{price, volume, now};
        data.ticks[now] = tick;
        
        // 更新 VWAP
        data.total_volume += volume;
        data.vwap = ((data.vwap * (data.total_volume - volume)) + 
                     (price * volume)) / data.total_volume;
    }
    
    double get_vwap(uint32_t symbol_id) const {
        auto it = symbol_data_.find(symbol_id);
        return (it != symbol_data_.end()) ? it->second.vwap : 0.0;
    }
    
    std::vector<Tick> get_recent_ticks(
        uint32_t symbol_id, 
        absl::Duration lookback) const {
        
        auto it = symbol_data_.find(symbol_id);
        if (it == symbol_data_.end()) {
            return {};
        }
        
        absl::Time cutoff = absl::Now() - lookback;
        std::vector<Tick> result;
        
        auto lower = it->second.ticks.lower_bound(cutoff);
        for (auto tick_it = lower; tick_it != it->second.ticks.end(); ++tick_it) {
            result.push_back(tick_it->second);
        }
        
        return result;
    }
};
```

---

## 參考資料

1. [Abseil 官方文檔](https://abseil.io/)
2. [Abseil C++ Tips of the Week](https://abseil.io/tips/)
3. [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)
4. [Abseil Containers](https://abseil.io/docs/cpp/guides/container)
5. [Abseil Time Library](https://abseil.io/docs/cpp/guides/time)
6. [Abseil Synchronization](https://abseil.io/docs/cpp/guides/synchronization)
