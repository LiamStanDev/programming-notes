# Day 2: C++ 現代特性 Part 2

## 學習目標

本課程深入探討 C++17/20 現代特性、RAII 原則、記憶體管理與 STL 容器選擇。這些技術是編寫高效、安全、可維護 C++ 代碼的基礎。

**核心主題:**
- C++17 特性 (optional, variant, string_view)
- C++20 特性 (Concepts, Ranges, span)
- RAII 與記憶體管理
- 記憶體池與對象池
- STL 容器性能特性

---

## 2.1 C++17 核心特性

### 2.1.1 結構化綁定 (Structured Bindings)

```cpp
#include <iostream>
#include <map>
#include <tuple>

// 解構 pair
void pair_example() {
    std::pair<int, std::string> data{42, "answer"};
    auto [number, text] = data;
    
    std::cout << "Number: " << number << ", Text: " << text << "\n";
}

// 解構 tuple
std::tuple<int, double, std::string> get_data() {
    return {1, 3.14, "pi"};
}

void tuple_example() {
    auto [id, value, name] = get_data();
    std::cout << "ID: " << id << ", Value: " << value << ", Name: " << name << "\n";
}

// 遍歷 map
void map_example() {
    std::map<std::string, int> prices{{"AAPL", 150}, {"GOOGL", 2800}};
    
    for (const auto& [symbol, price] : prices) {
        std::cout << symbol << ": $" << price << "\n";
    }
}

// 解構自定義結構
struct MarketData {
    double bid;
    double ask;
    int volume;
};

void struct_example() {
    MarketData data{150.0, 150.05, 1000};
    auto [bid, ask, vol] = data;
    
    std::cout << "Bid: " << bid << ", Ask: " << ask << ", Volume: " << vol << "\n";
}
```

### 2.1.2 std::optional - 可選值

```cpp
#include <optional>
#include <iostream>
#include <string>

// 替代指針表示"可能不存在"
std::optional<std::string> find_user(int id) {
    if (id == 42) {
        return "Alice";
    }
    return std::nullopt;  // 表示無值
}

void optional_basic() {
    auto user = find_user(42);
    
    if (user.has_value()) {
        std::cout << "Found: " << user.value() << "\n";
    }
    
    // 或使用簡潔語法
    if (user) {
        std::cout << "Found: " << *user << "\n";
    }
    
    // 提供默認值
    std::cout << "User: " << user.value_or("Unknown") << "\n";
}

// HFT 應用: 查找訂單
struct Order {
    int id;
    double price;
    int quantity;
};

class OrderBook {
private:
    std::map<int, Order> orders_;
    
public:
    std::optional<Order> find_order(int order_id) const {
        auto it = orders_.find(order_id);
        if (it != orders_.end()) {
            return it->second;
        }
        return std::nullopt;
    }
    
    void add_order(const Order& order) {
        orders_[order.id] = order;
    }
};

void optional_hft_example() {
    OrderBook book;
    book.add_order({1001, 150.0, 100});
    
    if (auto order = book.find_order(1001)) {
        std::cout << "Order found: price=" << order->price 
                  << ", qty=" << order->quantity << "\n";
    } else {
        std::cout << "Order not found\n";
    }
}
```

### 2.1.3 std::variant - 類型安全的 union

```cpp
#include <variant>
#include <iostream>
#include <string>

// 替代傳統 union
using Value = std::variant<int, double, std::string>;

void variant_basic() {
    Value v1 = 42;
    Value v2 = 3.14;
    Value v3 = "hello";
    
    // 檢查當前類型
    if (std::holds_alternative<int>(v1)) {
        std::cout << "v1 is int: " << std::get<int>(v1) << "\n";
    }
    
    // 安全訪問 (拋出異常如果類型不匹配)
    try {
        std::cout << std::get<double>(v1) << "\n";
    } catch (const std::bad_variant_access& e) {
        std::cout << "Wrong type access\n";
    }
    
    // 訪問通過索引
    std::cout << "v2 type index: " << v2.index() << "\n";
}

// HFT 應用: 消息分發
struct NewOrderMessage {
    int order_id;
    double price;
    int quantity;
};

struct CancelOrderMessage {
    int order_id;
};

struct MarketDataMessage {
    std::string symbol;
    double bid;
    double ask;
};

using Message = std::variant<NewOrderMessage, CancelOrderMessage, MarketDataMessage>;

// 使用 visitor 模式處理
class MessageHandler {
public:
    void operator()(const NewOrderMessage& msg) {
        std::cout << "New Order: ID=" << msg.order_id 
                  << ", Price=" << msg.price << "\n";
    }
    
    void operator()(const CancelOrderMessage& msg) {
        std::cout << "Cancel Order: ID=" << msg.order_id << "\n";
    }
    
    void operator()(const MarketDataMessage& msg) {
        std::cout << "Market Data: " << msg.symbol 
                  << " Bid=" << msg.bid << " Ask=" << msg.ask << "\n";
    }
};

void variant_hft_example() {
    std::vector<Message> messages = {
        NewOrderMessage{1001, 150.0, 100},
        CancelOrderMessage{1002},
        MarketDataMessage{"AAPL", 149.95, 150.05}
    };
    
    MessageHandler handler;
    
    for (const auto& msg : messages) {
        std::visit(handler, msg);
    }
}
```

### 2.1.4 std::string_view - 零拷貝字串視圖

```cpp
#include <string_view>
#include <iostream>
#include <string>

// 避免字串拷貝
void print_symbol(std::string_view symbol) {
    // 不會拷貝字串,僅持有指針和長度
    std::cout << "Symbol: " << symbol << "\n";
}

void string_view_basic() {
    std::string s = "AAPL";
    const char* cstr = "GOOGL";
    
    print_symbol(s);      // 不拷貝
    print_symbol(cstr);   // 不拷貝
    print_symbol("MSFT"); // 不拷貝
}

// 字串解析優化
struct FIXField {
    std::string_view tag;
    std::string_view value;
};

std::vector<FIXField> parse_fix_message(std::string_view message) {
    std::vector<FIXField> fields;
    
    size_t pos = 0;
    while (pos < message.size()) {
        size_t tag_end = message.find('=', pos);
        if (tag_end == std::string_view::npos) break;
        
        size_t value_end = message.find('\x01', tag_end);
        if (value_end == std::string_view::npos) value_end = message.size();
        
        fields.push_back({
            message.substr(pos, tag_end - pos),
            message.substr(tag_end + 1, value_end - tag_end - 1)
        });
        
        pos = value_end + 1;
    }
    
    return fields;
}

void string_view_fix_example() {
    std::string fix_msg = "35=D\x01""49=SENDER\x01""56=TARGET\x01";
    
    // 零拷貝解析
    auto fields = parse_fix_message(fix_msg);
    
    for (const auto& [tag, value] : fields) {
        std::cout << "Tag: " << tag << ", Value: " << value << "\n";
    }
}
```

**注意事項:** `string_view` 不擁有數據,必須確保底層字串生命週期有效。

### 2.1.5 if/switch 初始化語句

```cpp
#include <map>
#include <iostream>

void if_init_example() {
    std::map<std::string, double> prices{{"AAPL", 150.0}};
    
    // C++17: 初始化語句在 if 中
    if (auto it = prices.find("AAPL"); it != prices.end()) {
        std::cout << "Price: " << it->second << "\n";
    }
    // it 的作用域僅限於 if 語句
}

void switch_init_example() {
    enum class OrderStatus { NEW, FILLED, CANCELLED };
    
    auto get_status = []() { return OrderStatus::FILLED; };
    
    switch (auto status = get_status(); status) {
        case OrderStatus::NEW:
            std::cout << "New order\n";
            break;
        case OrderStatus::FILLED:
            std::cout << "Order filled\n";
            break;
        case OrderStatus::CANCELLED:
            std::cout << "Order cancelled\n";
            break;
    }
}
```

### 2.1.6 Fold Expressions - 可變參數模板簡化

```cpp
#include <iostream>

// 舊方法: 遞歸展開
template<typename T>
T sum_old(T value) {
    return value;
}

template<typename T, typename... Args>
T sum_old(T first, Args... args) {
    return first + sum_old(args...);
}

// C++17: Fold expression
template<typename... Args>
auto sum(Args... args) {
    return (args + ...);  // 一元右折疊
}

template<typename... Args>
auto product(Args... args) {
    return (args * ...);
}

// 邏輯操作
template<typename... Args>
bool all_true(Args... args) {
    return (args && ...);
}

template<typename... Args>
bool any_true(Args... args) {
    return (args || ...);
}

// 打印所有參數
template<typename... Args>
void print_all(Args... args) {
    ((std::cout << args << " "), ...);  // 二元左折疊
    std::cout << "\n";
}

void fold_examples() {
    std::cout << "Sum: " << sum(1, 2, 3, 4, 5) << "\n";
    std::cout << "Product: " << product(2, 3, 4) << "\n";
    std::cout << "All true: " << all_true(true, true, false) << "\n";
    
    print_all(1, "hello", 3.14, 'x');
}
```

---

## 2.2 C++20 核心特性

### 2.2.1 Concepts - 約束模板參數

```cpp
#include <concepts>
#include <iostream>
#include <type_traits>

// 使用標準 concepts
template<std::integral T>
T add(T a, T b) {
    return a + b;
}

template<std::floating_point T>
T multiply(T a, T b) {
    return a * b;
}

// 自定義 concept
template<typename T>
concept Numeric = std::is_arithmetic_v<T>;

template<Numeric T>
T square(T value) {
    return value * value;
}

// 複雜 concept
template<typename T>
concept Hashable = requires(T a) {
    { std::hash<T>{}(a) } -> std::convertible_to<std::size_t>;
};

template<Hashable T>
void print_hash(T value) {
    std::cout << "Hash: " << std::hash<T>{}(value) << "\n";
}

// HFT 應用: 訂單概念
template<typename T>
concept Order = requires(T order) {
    { order.get_price() } -> std::convertible_to<double>;
    { order.get_quantity() } -> std::convertible_to<int>;
    { order.get_symbol() } -> std::convertible_to<std::string>;
};

template<Order T>
double calculate_notional(const T& order) {
    return order.get_price() * order.get_quantity();
}

struct LimitOrder {
    double get_price() const { return price; }
    int get_quantity() const { return quantity; }
    std::string get_symbol() const { return symbol; }
    
    double price;
    int quantity;
    std::string symbol;
};

void concept_examples() {
    std::cout << add(1, 2) << "\n";           // OK: int is integral
    std::cout << multiply(3.14, 2.0) << "\n"; // OK: double is floating_point
    
    print_hash(42);
    print_hash(std::string("hello"));
    
    LimitOrder order{150.0, 100, "AAPL"};
    std::cout << "Notional: " << calculate_notional(order) << "\n";
}
```

### 2.2.2 Ranges - 函數式編程

```cpp
#include <ranges>
#include <vector>
#include <iostream>
#include <algorithm>

namespace rng = std::ranges;
namespace views = std::views;

void ranges_basic() {
    std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // 管道語法: 過濾 + 轉換
    auto result = numbers 
        | views::filter([](int n) { return n % 2 == 0; })
        | views::transform([](int n) { return n * n; });
    
    for (int n : result) {
        std::cout << n << " ";
    }
    std::cout << "\n";
}

// HFT 應用: 處理訂單列表
struct OrderData {
    std::string symbol;
    double price;
    int quantity;
    bool is_buy;
};

void ranges_hft_example() {
    std::vector<OrderData> orders = {
        {"AAPL", 150.0, 100, true},
        {"AAPL", 149.5, 200, false},
        {"GOOGL", 2800.0, 50, true},
        {"AAPL", 151.0, 150, true}
    };
    
    // 查找 AAPL 的買單,按價格排序
    auto aapl_buys = orders
        | views::filter([](const OrderData& o) { 
            return o.symbol == "AAPL" && o.is_buy; 
          })
        | views::transform([](const OrderData& o) { 
            return std::make_pair(o.price, o.quantity); 
          });
    
    std::vector<std::pair<double, int>> result(aapl_buys.begin(), aapl_buys.end());
    rng::sort(result, rng::greater{});
    
    std::cout << "AAPL Buy Orders:\n";
    for (const auto& [price, qty] : result) {
        std::cout << "  Price: " << price << ", Qty: " << qty << "\n";
    }
}
```

### 2.2.3 Three-way Comparison (<=>)

```cpp
#include <compare>
#include <iostream>

struct Price {
    double value;
    
    // 自動生成所有比較運算符
    auto operator<=>(const Price&) const = default;
};

void spaceship_basic() {
    Price p1{150.0};
    Price p2{149.5};
    
    if (p1 > p2) {
        std::cout << "p1 > p2\n";
    }
    
    if (p1 == p2) {
        std::cout << "p1 == p2\n";
    }
}

// 自定義比較邏輯
struct Order {
    int priority;
    double price;
    
    std::strong_ordering operator<=>(const Order& other) const {
        // 先比較優先級,再比較價格
        if (auto cmp = priority <=> other.priority; cmp != 0) {
            return cmp;
        }
        return price <=> other.price;
    }
    
    bool operator==(const Order&) const = default;
};

void spaceship_custom() {
    Order o1{1, 150.0};
    Order o2{1, 149.5};
    Order o3{2, 148.0};
    
    std::cout << "o1 > o2: " << (o1 > o2) << "\n";
    std::cout << "o3 > o1: " << (o3 > o1) << "\n";
}
```

### 2.2.4 std::span - 非擁有陣列視圖

```cpp
#include <span>
#include <vector>
#include <array>
#include <iostream>

// 統一處理各種連續容器
void process_data(std::span<const int> data) {
    for (int value : data) {
        std::cout << value << " ";
    }
    std::cout << "\n";
}

void span_basic() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    std::array<int, 3> arr = {6, 7, 8};
    int c_array[] = {9, 10, 11};
    
    process_data(vec);
    process_data(arr);
    process_data(c_array);
    
    // 部分視圖
    process_data(std::span(vec).subspan(1, 3));  // {2, 3, 4}
}

// HFT 應用: 零拷貝消息處理
struct MarketDataBatch {
    std::vector<double> prices;
    std::vector<int> volumes;
};

void process_batch(std::span<const double> prices, std::span<const int> volumes) {
    for (size_t i = 0; i < prices.size(); ++i) {
        // 處理市場數據,無需拷貝
        std::cout << "Price: " << prices[i] << ", Volume: " << volumes[i] << "\n";
    }
}

void span_hft_example() {
    MarketDataBatch batch;
    batch.prices = {150.0, 150.05, 149.95};
    batch.volumes = {100, 200, 150};
    
    process_batch(batch.prices, batch.volumes);
}
```

---

## 2.3 RAII 與記憶體管理

### 2.3.1 RAII 原則

```cpp
#include <mutex>
#include <fstream>
#include <memory>

// RAII: Resource Acquisition Is Initialization
// 資源獲取即初始化,在構造函數獲取資源,析構函數釋放

// 示例 1: 文件處理
class FileHandler {
private:
    std::FILE* file_;
    
public:
    explicit FileHandler(const char* filename) {
        file_ = std::fopen(filename, "r");
        if (!file_) {
            throw std::runtime_error("Failed to open file");
        }
    }
    
    ~FileHandler() {
        if (file_) {
            std::fclose(file_);
        }
    }
    
    // 禁用拷貝
    FileHandler(const FileHandler&) = delete;
    FileHandler& operator=(const FileHandler&) = delete;
    
    // 支持移動
    FileHandler(FileHandler&& other) noexcept : file_(other.file_) {
        other.file_ = nullptr;
    }
    
    std::FILE* get() { return file_; }
};

// 示例 2: 鎖管理 (標準庫已提供 lock_guard)
class ScopedLock {
private:
    std::mutex& mutex_;
    
public:
    explicit ScopedLock(std::mutex& mtx) : mutex_(mtx) {
        mutex_.lock();
    }
    
    ~ScopedLock() {
        mutex_.unlock();
    }
    
    ScopedLock(const ScopedLock&) = delete;
    ScopedLock& operator=(const ScopedLock&) = delete;
};

// 示例 3: 計時器
class ScopedTimer {
private:
    std::string name_;
    std::chrono::high_resolution_clock::time_point start_;
    
public:
    explicit ScopedTimer(std::string name) 
        : name_(std::move(name))
        , start_(std::chrono::high_resolution_clock::now()) {}
    
    ~ScopedTimer() {
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start_);
        std::cout << name_ << " took " << duration.count() << " μs\n";
    }
};

void raii_examples() {
    // 文件自動關閉
    {
        FileHandler file("data.txt");
        // 使用 file...
    }  // file 自動關閉
    
    // 鎖自動釋放
    std::mutex mtx;
    {
        ScopedLock lock(mtx);
        // 臨界區...
    }  // lock 自動釋放
    
    // 計時自動輸出
    {
        ScopedTimer timer("Operation");
        // 執行操作...
    }  // timer 自動輸出耗時
}
```

### 2.3.2 自定義內存分配

```cpp
#include <cstdlib>
#include <new>
#include <iostream>

// 重載全局 new/delete (謹慎使用!)
void* operator new(size_t size) {
    std::cout << "Allocating " << size << " bytes\n";
    void* ptr = std::malloc(size);
    if (!ptr) throw std::bad_alloc();
    return ptr;
}

void operator delete(void* ptr) noexcept {
    std::cout << "Deallocating\n";
    std::free(ptr);
}

// Placement new - 在指定內存構造對象
void placement_new_example() {
    alignas(alignof(std::string)) char buffer[sizeof(std::string)];
    
    // 在 buffer 中構造 string
    std::string* str = new (buffer) std::string("Hello");
    
    std::cout << "String: " << *str << "\n";
    
    // 必須手動調用析構函數
    str->~string();
}

// 自定義分配器
template<typename T>
class LoggingAllocator {
public:
    using value_type = T;
    
    LoggingAllocator() = default;
    
    template<typename U>
    LoggingAllocator(const LoggingAllocator<U>&) {}
    
    T* allocate(size_t n) {
        std::cout << "Allocating " << n << " objects of size " << sizeof(T) << "\n";
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }
    
    void deallocate(T* ptr, size_t n) {
        std::cout << "Deallocating " << n << " objects\n";
        ::operator delete(ptr);
    }
};

template<typename T, typename U>
bool operator==(const LoggingAllocator<T>&, const LoggingAllocator<U>&) {
    return true;
}

template<typename T, typename U>
bool operator!=(const LoggingAllocator<T>&, const LoggingAllocator<U>&) {
    return false;
}

void allocator_example() {
    std::vector<int, LoggingAllocator<int>> vec;
    vec.push_back(1);
    vec.push_back(2);
    vec.push_back(3);
}
```

### 2.3.3 記憶體池 (Memory Pool)

```cpp
#include <vector>
#include <memory>
#include <cstddef>

template<typename T, size_t BlockSize = 4096>
class MemoryPool {
private:
    union Node {
        Node* next;
        alignas(alignof(T)) char data[sizeof(T)];
    };
    
    Node* free_list_ = nullptr;
    std::vector<std::unique_ptr<Node[]>> blocks_;
    
    void allocate_block() {
        auto block = std::make_unique<Node[]>(BlockSize);
        
        // 將新塊加入空閒鏈表
        for (size_t i = 0; i < BlockSize - 1; ++i) {
            block[i].next = &block[i + 1];
        }
        block[BlockSize - 1].next = free_list_;
        free_list_ = block.get();
        
        blocks_.push_back(std::move(block));
    }
    
public:
    MemoryPool() {
        allocate_block();
    }
    
    T* allocate() {
        if (!free_list_) {
            allocate_block();
        }
        
        Node* node = free_list_;
        free_list_ = node->next;
        
        return reinterpret_cast<T*>(node->data);
    }
    
    void deallocate(T* ptr) {
        Node* node = reinterpret_cast<Node*>(ptr);
        node->next = free_list_;
        free_list_ = node;
    }
    
    template<typename... Args>
    T* construct(Args&&... args) {
        T* ptr = allocate();
        new (ptr) T(std::forward<Args>(args)...);
        return ptr;
    }
    
    void destroy(T* ptr) {
        ptr->~T();
        deallocate(ptr);
    }
};

// 使用示例
struct Order {
    int id;
    double price;
    int quantity;
    
    Order(int i, double p, int q) : id(i), price(p), quantity(q) {
        std::cout << "Order " << id << " constructed\n";
    }
    
    ~Order() {
        std::cout << "Order " << id << " destroyed\n";
    }
};

void memory_pool_example() {
    MemoryPool<Order> pool;
    
    // 快速分配
    Order* o1 = pool.construct(1, 150.0, 100);
    Order* o2 = pool.construct(2, 149.5, 200);
    
    // 使用訂單...
    
    // 快速回收
    pool.destroy(o1);
    pool.destroy(o2);
}
```

### 2.3.4 對象池 (Object Pool)

```cpp
#include <vector>
#include <memory>
#include <queue>

template<typename T>
class ObjectPool {
private:
    std::vector<std::unique_ptr<T>> objects_;
    std::queue<T*> available_;
    size_t block_size_;
    
public:
    explicit ObjectPool(size_t block_size = 64) : block_size_(block_size) {
        allocate_block();
    }
    
    void allocate_block() {
        size_t old_size = objects_.size();
        objects_.resize(old_size + block_size_);
        
        for (size_t i = old_size; i < objects_.size(); ++i) {
            objects_[i] = std::make_unique<T>();
            available_.push(objects_[i].get());
        }
    }
    
    T* acquire() {
        if (available_.empty()) {
            allocate_block();
        }
        
        T* obj = available_.front();
        available_.pop();
        return obj;
    }
    
    void release(T* obj) {
        // 可選: 重置對象狀態
        available_.push(obj);
    }
    
    size_t size() const { return objects_.size(); }
    size_t available_count() const { return available_.size(); }
};

// HFT 應用
struct MarketDataMessage {
    std::string symbol;
    double bid;
    double ask;
    uint64_t timestamp;
    
    void reset() {
        symbol.clear();
        bid = ask = 0.0;
        timestamp = 0;
    }
};

void object_pool_hft_example() {
    ObjectPool<MarketDataMessage> pool(1000);
    
    std::vector<MarketDataMessage*> active_messages;
    
    // 獲取對象
    for (int i = 0; i < 100; ++i) {
        auto* msg = pool.acquire();
        msg->symbol = "AAPL";
        msg->bid = 150.0;
        msg->ask = 150.05;
        active_messages.push_back(msg);
    }
    
    std::cout << "Active: " << active_messages.size() 
              << ", Available: " << pool.available_count() << "\n";
    
    // 釋放對象
    for (auto* msg : active_messages) {
        msg->reset();
        pool.release(msg);
    }
    
    std::cout << "After release - Available: " << pool.available_count() << "\n";
}
```

---

## 2.4 STL 容器選擇

### 2.4.1 容器性能特性

```cpp
#include <vector>
#include <list>
#include <deque>
#include <map>
#include <unordered_map>
#include <chrono>
#include <iostream>

template<typename Container>
void benchmark_insert(const std::string& name, int n) {
    Container cont;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < n; ++i) {
        cont.insert(cont.end(), i);
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
    
    std::cout << name << " insert: " << duration.count() << " μs\n";
}

void container_benchmark() {
    const int N = 10000;
    
    benchmark_insert<std::vector<int>>("vector", N);
    benchmark_insert<std::list<int>>("list", N);
    benchmark_insert<std::deque<int>>("deque", N);
}
```

**容器選擇指南:**

| 容器 | 隨機訪問 | 插入/刪除 (尾部) | 插入/刪除 (任意) | Cache 友好 | HFT 推薦場景 |
|------|----------|------------------|------------------|------------|--------------|
| `vector` | O(1) | O(1) 攤銷 | O(n) | ✅ 優秀 | 訂單列表、市場數據 |
| `deque` | O(1) | O(1) | O(n) | ⚠️ 中等 | 雙端隊列 |
| `list` | O(n) | O(1) | O(1) | ❌ 差 | 頻繁插入/刪除 |
| `array` | O(1) | N/A | N/A | ✅ 優秀 | 固定大小緩衝區 |

### 2.4.2 關聯容器選擇

```cpp
#include <map>
#include <unordered_map>
#include <set>
#include <iostream>
#include <string>

// map vs unordered_map
void associative_benchmark() {
    const int N = 10000;
    
    // map: 有序,基於紅黑樹
    {
        std::map<int, std::string> m;
        
        auto start = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < N; ++i) {
            m[i] = "value";
        }
        auto end = std::chrono::high_resolution_clock::now();
        
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "map insert: " << duration.count() << " μs\n";
    }
    
    // unordered_map: 無序,基於哈希表
    {
        std::unordered_map<int, std::string> um;
        
        auto start = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < N; ++i) {
            um[i] = "value";
        }
        auto end = std::chrono::high_resolution_clock::now();
        
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "unordered_map insert: " << duration.count() << " μs\n";
    }
}

// HFT: 訂單簿價格層 (需要有序)
using PriceLevel = std::map<double, int>;  // price -> quantity

void order_book_example() {
    PriceLevel bids;
    
    bids[150.00] = 100;
    bids[149.95] = 200;
    bids[150.05] = 150;
    
    // 遍歷是有序的
    std::cout << "Best bid levels:\n";
    auto it = bids.rbegin();  // 從高到低
    for (int i = 0; i < 3 && it != bids.rend(); ++i, ++it) {
        std::cout << "  " << it->first << ": " << it->second << "\n";
    }
}

// HFT: 訂單查找 (不需有序)
using OrderMap = std::unordered_map<int, Order>;  // order_id -> Order

void order_lookup_example() {
    OrderMap orders;
    
    orders[1001] = {1001, 150.0, 100};
    orders[1002] = {1002, 149.5, 200};
    
    // O(1) 查找
    if (auto it = orders.find(1001); it != orders.end()) {
        std::cout << "Order 1001 found\n";
    }
}
```

**關聯容器選擇:**

| 容器 | 複雜度 | 有序 | 使用場景 |
|------|--------|------|----------|
| `map` | O(log n) | ✅ | 需要有序遍歷 (訂單簿價格層) |
| `unordered_map` | O(1) 平均 | ❌ | 快速查找 (訂單 ID 映射) |
| `set` | O(log n) | ✅ | 去重 + 有序 |
| `unordered_set` | O(1) 平均 | ❌ | 去重 (無序) |

### 2.4.3 自定義哈希函數

```cpp
#include <unordered_map>
#include <string>

struct Symbol {
    std::string exchange;
    std::string ticker;
    
    bool operator==(const Symbol& other) const {
        return exchange == other.exchange && ticker == other.ticker;
    }
};

// 自定義哈希函數
struct SymbolHash {
    size_t operator()(const Symbol& s) const {
        size_t h1 = std::hash<std::string>{}(s.exchange);
        size_t h2 = std::hash<std::string>{}(s.ticker);
        return h1 ^ (h2 << 1);
    }
};

void custom_hash_example() {
    std::unordered_map<Symbol, double, SymbolHash> prices;
    
    prices[{"NYSE", "AAPL"}] = 150.0;
    prices[{"NASDAQ", "GOOGL"}] = 2800.0;
    
    Symbol aapl{"NYSE", "AAPL"};
    std::cout << "AAPL price: " << prices[aapl] << "\n";
}
```

---

## 實作練習

### 練習 1: 使用 std::variant 實作消息分發

**需求:** 實現類型安全的消息處理系統。

```cpp
#include <variant>
#include <vector>
#include <iostream>

struct OrderNew { int order_id; double price; int quantity; };
struct OrderCancel { int order_id; };
struct OrderModify { int order_id; double new_price; };

using Message = std::variant<OrderNew, OrderCancel, OrderModify>;

class MessageDispatcher {
public:
    void operator()(const OrderNew& msg) {
        std::cout << "New Order: " << msg.order_id << "\n";
    }
    
    void operator()(const OrderCancel& msg) {
        std::cout << "Cancel Order: " << msg.order_id << "\n";
    }
    
    void operator()(const OrderModify& msg) {
        std::cout << "Modify Order: " << msg.order_id 
                  << " to price " << msg.new_price << "\n";
    }
};

int main() {
    std::vector<Message> messages = {
        OrderNew{1001, 150.0, 100},
        OrderCancel{1002},
        OrderModify{1001, 151.0}
    };
    
    MessageDispatcher dispatcher;
    
    for (const auto& msg : messages) {
        std::visit(dispatcher, msg);
    }
    
    return 0;
}
```

### 練習 2: 實作固定大小記憶體池

見前文 `MemoryPool` 實現。

### 練習 3: 解決 False Sharing 問題

```cpp
#include <thread>
#include <vector>
#include <chrono>
#include <iostream>

// 錯誤: false sharing
struct BadCounter {
    int value1;
    int value2;
};

// 正確: cache line 對齊
struct GoodCounter {
    alignas(64) int value1;
    alignas(64) int value2;
};

template<typename Counter>
void benchmark_counter() {
    Counter counter{0, 0};
    
    auto start = std::chrono::high_resolution_clock::now();
    
    std::thread t1([&]() {
        for (int i = 0; i < 10000000; ++i) {
            ++counter.value1;
        }
    });
    
    std::thread t2([&]() {
        for (int i = 0; i < 10000000; ++i) {
            ++counter.value2;
        }
    });
    
    t1.join();
    t2.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "Time: " << duration.count() << " ms\n";
}

int main() {
    std::cout << "With false sharing:\n";
    benchmark_counter<BadCounter>();
    
    std::cout << "\nWithout false sharing:\n";
    benchmark_counter<GoodCounter>();
    
    return 0;
}
```

---

## 關鍵要點總結

### C++17/20 特性選擇

| 特性 | 替代方案 | 性能提升 | 推薦場景 |
|------|----------|----------|----------|
| `std::optional` | 指針/特殊值 | 無 | 可選返回值 |
| `std::variant` | union | 類型安全 | 消息分發 |
| `std::string_view` | `const std::string&` | 零拷貝 | 只讀字串 |
| `std::span` | 指針+長度 | 零拷貝 | 陣列視圖 |
| Concepts | SFINAE | 可讀性 | 模板約束 |
| Ranges | 循環 | 可讀性 | 數據處理 |

### RAII 最佳實踐

- 資源獲取即初始化
- 析構函數自動釋放資源
- 禁用拷貝,支持移動
- 使用智能指針管理動態內存

### 容器選擇建議 (HFT)

| 數據結構 | 推薦容器 | 理由 |
|----------|----------|------|
| 訂單列表 | `std::vector` | Cache 友好,連續內存 |
| 訂單簿價格層 | `std::map` | 有序,快速範圍查詢 |
| 訂單 ID 映射 | `std::unordered_map` | O(1) 查找 |
| 固定緩衝區 | `std::array` | 無動態分配 |
| 消息隊列 | 自定義環形緩衝 | 無鎖,固定大小 |

---

## 參考資料 (References)

1. Bjarne Stroustrup, *A Tour of C++*, 2nd Edition (2018)
2. [cppreference.com](https://en.cppreference.com/)
3. [C++17 Features](https://github.com/AnthonyCalandra/modern-cpp-features#cpp17)
4. [C++20 Features](https://github.com/AnthonyCalandra/modern-cpp-features#cpp20)
5. Scott Meyers, *Effective Modern C++* (2014)
6. Nicolai Josuttis, *C++17 - The Complete Guide* (2019)
7. [Ranges Library](https://en.cppreference.com/w/cpp/ranges)
8. [Memory Order](https://en.cppreference.com/w/cpp/atomic/memory_order)
