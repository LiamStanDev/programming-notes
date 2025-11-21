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

### 練習 4: 使用 std::optional 實現安全的查找

**任務**: 實現一個安全的資料庫查詢模擬,使用 `std::optional` 處理查找失敗情況。

```cpp
#include <optional>
#include <unordered_map>
#include <string>
#include <iostream>
#include <vector>

// 用戶資料
struct User {
    int id;
    std::string name;
    std::string email;
    double balance;
};

// 模擬數據庫
class UserDatabase {
private:
    std::unordered_map<int, User> users_;
    std::unordered_map<std::string, int> email_index_;  // email -> user_id
    
public:
    void addUser(const User& user) {
        users_[user.id] = user;
        email_index_[user.email] = user.id;
    }
    
    // 按 ID 查找
    std::optional<User> findById(int id) const {
        auto it = users_.find(id);
        if (it != users_.end()) {
            return it->second;
        }
        return std::nullopt;
    }
    
    // 按 Email 查找
    std::optional<User> findByEmail(const std::string& email) const {
        auto it = email_index_.find(email);
        if (it != email_index_.end()) {
            return findById(it->second);
        }
        return std::nullopt;
    }
    
    // 嘗試更新餘額
    std::optional<double> updateBalance(int id, double amount) {
        auto it = users_.find(id);
        if (it == users_.end()) {
            return std::nullopt;
        }
        
        if (it->second.balance + amount < 0) {
            return std::nullopt;  // 餘額不足
        }
        
        it->second.balance += amount;
        return it->second.balance;
    }
};

int main() {
    UserDatabase db;
    
    db.addUser({1, "Alice", "alice@example.com", 1000.0});
    db.addUser({2, "Bob", "bob@example.com", 500.0});
    
    // 查找存在的用戶
    if (auto user = db.findById(1)) {
        std::cout << "Found: " << user->name 
                  << ", Balance: $" << user->balance << "\n";
    }
    
    // 查找不存在的用戶
    auto notFound = db.findById(999);
    std::cout << "User 999: " << notFound.value_or(User{0, "Unknown", "", 0}).name << "\n";
    
    // 按 Email 查找
    if (auto user = db.findByEmail("bob@example.com")) {
        std::cout << "Found by email: " << user->name << "\n";
    }
    
    // 更新餘額
    if (auto newBalance = db.updateBalance(1, -200)) {
        std::cout << "New balance for Alice: $" << *newBalance << "\n";
    }
    
    // 嘗試透支
    if (auto result = db.updateBalance(2, -1000)) {
        std::cout << "Withdrawal successful\n";
    } else {
        std::cout << "Insufficient funds\n";
    }
    
    return 0;
}
```

---

### 練習 5: 使用 string_view 優化字串解析

**任務**: 使用 `std::string_view` 實現高效的 CSV 解析器。

```cpp
#include <string_view>
#include <vector>
#include <iostream>
#include <string>
#include <charconv>

class CSVParser {
public:
    // 解析一行 CSV,返回字段視圖 (零拷貝)
    static std::vector<std::string_view> parseLine(std::string_view line) {
        std::vector<std::string_view> fields;
        
        size_t start = 0;
        while (start < line.size()) {
            size_t end = line.find(',', start);
            if (end == std::string_view::npos) {
                end = line.size();
            }
            
            // 去除首尾空格
            auto field = line.substr(start, end - start);
            field = trim(field);
            
            fields.push_back(field);
            start = end + 1;
        }
        
        return fields;
    }
    
    // 安全地解析數字
    static std::optional<double> parseDouble(std::string_view sv) {
        double value;
        auto result = std::from_chars(sv.data(), sv.data() + sv.size(), value);
        if (result.ec == std::errc()) {
            return value;
        }
        return std::nullopt;
    }
    
    static std::optional<int> parseInt(std::string_view sv) {
        int value;
        auto result = std::from_chars(sv.data(), sv.data() + sv.size(), value);
        if (result.ec == std::errc()) {
            return value;
        }
        return std::nullopt;
    }
    
private:
    static std::string_view trim(std::string_view sv) {
        size_t start = 0;
        while (start < sv.size() && std::isspace(sv[start])) ++start;
        
        size_t end = sv.size();
        while (end > start && std::isspace(sv[end - 1])) --end;
        
        return sv.substr(start, end - start);
    }
};

// 市場數據結構
struct MarketData {
    std::string symbol;
    double bid;
    double ask;
    int volume;
};

// 解析市場數據 CSV
std::optional<MarketData> parseMarketData(std::string_view line) {
    auto fields = CSVParser::parseLine(line);
    
    if (fields.size() != 4) {
        return std::nullopt;
    }
    
    auto bid = CSVParser::parseDouble(fields[1]);
    auto ask = CSVParser::parseDouble(fields[2]);
    auto volume = CSVParser::parseInt(fields[3]);
    
    if (!bid || !ask || !volume) {
        return std::nullopt;
    }
    
    return MarketData{
        std::string(fields[0]),
        *bid,
        *ask,
        *volume
    };
}

int main() {
    // 模擬 CSV 數據
    std::vector<std::string> csvData = {
        "AAPL, 150.00, 150.05, 1000",
        "GOOGL, 2800.50, 2801.00, 500",
        "MSFT, 299.95, 300.00, 750",
        "INVALID, abc, 100.00, xyz"  // 無效數據
    };
    
    std::cout << "Parsing market data:\n";
    
    for (const auto& line : csvData) {
        if (auto data = parseMarketData(line)) {
            std::cout << "Symbol: " << data->symbol
                      << ", Bid: " << data->bid
                      << ", Ask: " << data->ask
                      << ", Volume: " << data->volume << "\n";
        } else {
            std::cout << "Failed to parse: " << line << "\n";
        }
    }
    
    return 0;
}
```

---

### 練習 6: 使用 Concepts 約束模板

**任務**: 使用 C++20 Concepts 實現類型安全的金融計算庫。

```cpp
#include <concepts>
#include <iostream>
#include <string>
#include <cmath>

// 定義可交易資產的 Concept
template<typename T>
concept Tradable = requires(T asset) {
    { asset.getSymbol() } -> std::convertible_to<std::string>;
    { asset.getPrice() } -> std::floating_point;
    { asset.getQuantity() } -> std::integral;
};

// 定義期權的 Concept
template<typename T>
concept Option = Tradable<T> && requires(T opt) {
    { opt.getStrike() } -> std::floating_point;
    { opt.getExpiry() } -> std::convertible_to<double>;
    { opt.isCall() } -> std::same_as<bool>;
};

// 股票類
class Stock {
public:
    Stock(std::string symbol, double price, int quantity)
        : symbol_(std::move(symbol)), price_(price), quantity_(quantity) {}
    
    std::string getSymbol() const { return symbol_; }
    double getPrice() const { return price_; }
    int getQuantity() const { return quantity_; }
    
private:
    std::string symbol_;
    double price_;
    int quantity_;
};

// 期權類
class StockOption {
public:
    StockOption(std::string symbol, double price, int quantity,
                double strike, double expiry, bool isCall)
        : symbol_(std::move(symbol)), price_(price), quantity_(quantity),
          strike_(strike), expiry_(expiry), isCall_(isCall) {}
    
    std::string getSymbol() const { return symbol_; }
    double getPrice() const { return price_; }
    int getQuantity() const { return quantity_; }
    double getStrike() const { return strike_; }
    double getExpiry() const { return expiry_; }
    bool isCall() const { return isCall_; }
    
private:
    std::string symbol_;
    double price_;
    int quantity_;
    double strike_;
    double expiry_;
    bool isCall_;
};

// 使用 Concept 約束的函數
template<Tradable T>
double calculateNotional(const T& asset) {
    return asset.getPrice() * asset.getQuantity();
}

template<Option T>
double calculateIntrinsicValue(const T& opt, double spotPrice) {
    if (opt.isCall()) {
        return std::max(0.0, spotPrice - opt.getStrike());
    } else {
        return std::max(0.0, opt.getStrike() - spotPrice);
    }
}

// 僅適用於浮點數的數學函數
template<std::floating_point T>
T blackScholesD1(T spot, T strike, T rate, T vol, T time) {
    return (std::log(spot / strike) + (rate + vol * vol / 2) * time) 
           / (vol * std::sqrt(time));
}

int main() {
    Stock stock("AAPL", 150.0, 100);
    StockOption option("AAPL", 5.0, 10, 155.0, 0.25, true);
    
    std::cout << "Stock notional: $" << calculateNotional(stock) << "\n";
    std::cout << "Option notional: $" << calculateNotional(option) << "\n";
    
    double spotPrice = 160.0;
    std::cout << "Option intrinsic value: $" 
              << calculateIntrinsicValue(option, spotPrice) << "\n";
    
    // Black-Scholes d1
    double d1 = blackScholesD1(150.0, 155.0, 0.05, 0.2, 0.25);
    std::cout << "Black-Scholes d1: " << d1 << "\n";
    
    return 0;
}
```

---

### 練習 7: 使用 Ranges 處理數據流

**任務**: 使用 C++20 Ranges 庫實現數據處理管道。

```cpp
#include <ranges>
#include <vector>
#include <iostream>
#include <algorithm>
#include <numeric>
#include <string>

namespace rng = std::ranges;
namespace views = std::views;

// 交易記錄
struct Trade {
    std::string symbol;
    double price;
    int quantity;
    bool isBuy;
    long timestamp;
};

// 使用 Ranges 進行數據分析
class TradeAnalyzer {
public:
    explicit TradeAnalyzer(std::vector<Trade> trades)
        : trades_(std::move(trades)) {}
    
    // 計算總成交量
    int totalVolume() const {
        return std::accumulate(trades_.begin(), trades_.end(), 0,
            [](int sum, const Trade& t) { return sum + t.quantity; });
    }
    
    // 獲取特定股票的交易
    auto tradesForSymbol(const std::string& symbol) const {
        return trades_ | views::filter([symbol](const Trade& t) {
            return t.symbol == symbol;
        });
    }
    
    // 計算 VWAP (Volume Weighted Average Price)
    double calculateVWAP(const std::string& symbol) const {
        auto symbolTrades = trades_ | views::filter([&](const Trade& t) {
            return t.symbol == symbol;
        });
        
        double totalValue = 0;
        int totalVolume = 0;
        
        for (const auto& trade : symbolTrades) {
            totalValue += trade.price * trade.quantity;
            totalVolume += trade.quantity;
        }
        
        return totalVolume > 0 ? totalValue / totalVolume : 0;
    }
    
    // 獲取前 N 大交易
    std::vector<Trade> topNByVolume(int n) const {
        std::vector<Trade> sorted = trades_;
        rng::partial_sort(sorted, sorted.begin() + n, 
            [](const Trade& a, const Trade& b) {
                return a.quantity > b.quantity;
            });
        
        return std::vector<Trade>(sorted.begin(), sorted.begin() + n);
    }
    
    // 獲取時間範圍內的交易
    auto tradesInTimeRange(long start, long end) const {
        return trades_ | views::filter([=](const Trade& t) {
            return t.timestamp >= start && t.timestamp <= end;
        });
    }
    
    // 轉換為價值列表
    auto tradeValues() const {
        return trades_ | views::transform([](const Trade& t) {
            return t.price * t.quantity;
        });
    }
    
private:
    std::vector<Trade> trades_;
};

int main() {
    std::vector<Trade> trades = {
        {"AAPL", 150.0, 100, true, 1000},
        {"AAPL", 150.5, 200, false, 1001},
        {"GOOGL", 2800.0, 50, true, 1002},
        {"AAPL", 149.5, 300, true, 1003},
        {"MSFT", 300.0, 150, false, 1004},
        {"AAPL", 151.0, 250, true, 1005}
    };
    
    TradeAnalyzer analyzer(trades);
    
    // VWAP 計算
    std::cout << "AAPL VWAP: $" << analyzer.calculateVWAP("AAPL") << "\n";
    
    // 總成交量
    std::cout << "Total volume: " << analyzer.totalVolume() << "\n";
    
    // 前 3 大交易
    std::cout << "\nTop 3 trades by volume:\n";
    for (const auto& trade : analyzer.topNByVolume(3)) {
        std::cout << "  " << trade.symbol << ": " << trade.quantity << " shares\n";
    }
    
    // 特定股票的交易
    std::cout << "\nAAPL trades:\n";
    for (const auto& trade : analyzer.tradesForSymbol("AAPL")) {
        std::cout << "  Price: $" << trade.price 
                  << ", Qty: " << trade.quantity << "\n";
    }
    
    // 使用管道計算總交易額
    auto values = analyzer.tradeValues();
    double totalValue = std::accumulate(values.begin(), values.end(), 0.0);
    std::cout << "\nTotal trade value: $" << totalValue << "\n";
    
    return 0;
}
```

---

### 練習 8: 實現 RAII 資源守衛

**任務**: 實現通用的 RAII 資源守衛類,支援自定義清理操作。

```cpp
#include <iostream>
#include <functional>
#include <utility>

// 通用資源守衛
template<typename CleanupFunc>
class ScopeGuard {
private:
    CleanupFunc cleanup_;
    bool active_ = true;
    
public:
    explicit ScopeGuard(CleanupFunc func)
        : cleanup_(std::move(func)) {}
    
    ~ScopeGuard() {
        if (active_) {
            cleanup_();
        }
    }
    
    // 禁止拷貝
    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;
    
    // 支持移動
    ScopeGuard(ScopeGuard&& other) noexcept
        : cleanup_(std::move(other.cleanup_)), active_(other.active_) {
        other.active_ = false;
    }
    
    // 取消清理
    void dismiss() { active_ = false; }
    
    // 立即執行清理
    void execute() {
        if (active_) {
            cleanup_();
            active_ = false;
        }
    }
};

// 輔助函數
template<typename F>
auto makeScopeGuard(F&& f) {
    return ScopeGuard<std::decay_t<F>>(std::forward<F>(f));
}

// 使用宏簡化 (可選)
#define SCOPE_EXIT auto CONCAT(scope_exit_, __LINE__) = makeScopeGuard
#define CONCAT_IMPL(a, b) a##b
#define CONCAT(a, b) CONCAT_IMPL(a, b)

// 模擬資源
class Connection {
public:
    explicit Connection(const std::string& name) : name_(name) {
        std::cout << "Opening connection: " << name_ << "\n";
    }
    
    void close() {
        std::cout << "Closing connection: " << name_ << "\n";
    }
    
    void send(const std::string& data) {
        std::cout << "Sending via " << name_ << ": " << data << "\n";
    }
    
private:
    std::string name_;
};

// 使用示例
void processData() {
    Connection conn("Database");
    
    // 確保連接在退出時關閉
    auto guard = makeScopeGuard([&conn]() {
        conn.close();
    });
    
    conn.send("SELECT * FROM orders");
    
    // 模擬錯誤
    bool hasError = false;
    if (hasError) {
        throw std::runtime_error("Processing failed");
    }
    
    conn.send("UPDATE orders SET status = 'processed'");
    
    // guard 會在退出時自動調用 conn.close()
}

// 事務模擬
class Transaction {
public:
    void begin() { std::cout << "Transaction started\n"; }
    void commit() { std::cout << "Transaction committed\n"; committed_ = true; }
    void rollback() { std::cout << "Transaction rolled back\n"; }
    bool isCommitted() const { return committed_; }
    
private:
    bool committed_ = false;
};

void performTransaction() {
    Transaction tx;
    tx.begin();
    
    auto guard = makeScopeGuard([&tx]() {
        if (!tx.isCommitted()) {
            tx.rollback();
        }
    });
    
    // 執行操作...
    std::cout << "Performing operations...\n";
    
    // 成功則提交
    tx.commit();
    
    // 如果到這裡,guard 的 rollback 不會執行
    // 因為 isCommitted() 返回 true
}

int main() {
    std::cout << "=== Basic scope guard ===\n";
    processData();
    
    std::cout << "\n=== Transaction with guard ===\n";
    performTransaction();
    
    std::cout << "\n=== Guard with dismiss ===\n";
    {
        auto guard = makeScopeGuard([]() {
            std::cout << "This should not print\n";
        });
        
        guard.dismiss();  // 取消清理
    }
    std::cout << "Guard dismissed\n";
    
    return 0;
}
```

---

### 練習 9: 使用 span 實現零拷貝緩衝區

**任務**: 使用 `std::span` 實現高效的消息緩衝區處理。

```cpp
#include <span>
#include <vector>
#include <array>
#include <iostream>
#include <cstring>
#include <cstdint>

// 消息頭
struct MessageHeader {
    uint32_t type;
    uint32_t length;
    uint64_t timestamp;
};

// 消息解析器 (零拷貝)
class MessageParser {
public:
    // 從緩衝區解析消息頭
    static std::optional<MessageHeader> parseHeader(std::span<const std::byte> buffer) {
        if (buffer.size() < sizeof(MessageHeader)) {
            return std::nullopt;
        }
        
        MessageHeader header;
        std::memcpy(&header, buffer.data(), sizeof(MessageHeader));
        return header;
    }
    
    // 獲取消息體
    static std::span<const std::byte> getBody(std::span<const std::byte> buffer) {
        if (buffer.size() <= sizeof(MessageHeader)) {
            return {};
        }
        return buffer.subspan(sizeof(MessageHeader));
    }
    
    // 解析整數
    template<std::integral T>
    static std::optional<T> parseInteger(std::span<const std::byte> buffer, size_t offset = 0) {
        if (offset + sizeof(T) > buffer.size()) {
            return std::nullopt;
        }
        
        T value;
        std::memcpy(&value, buffer.data() + offset, sizeof(T));
        return value;
    }
};

// 環形緩衝區
template<size_t Size>
class RingBuffer {
private:
    std::array<std::byte, Size> buffer_;
    size_t head_ = 0;
    size_t tail_ = 0;
    size_t count_ = 0;
    
public:
    // 寫入數據
    bool write(std::span<const std::byte> data) {
        if (data.size() > available()) {
            return false;
        }
        
        for (auto byte : data) {
            buffer_[tail_] = byte;
            tail_ = (tail_ + 1) % Size;
            ++count_;
        }
        return true;
    }
    
    // 讀取數據到輸出緩衝區
    size_t read(std::span<std::byte> output) {
        size_t toRead = std::min(output.size(), count_);
        
        for (size_t i = 0; i < toRead; ++i) {
            output[i] = buffer_[head_];
            head_ = (head_ + 1) % Size;
            --count_;
        }
        
        return toRead;
    }
    
    // 獲取可讀視圖 (不移除數據)
    std::span<const std::byte> peek(size_t maxSize) const {
        size_t available = std::min(maxSize, count_);
        size_t contiguous = std::min(available, Size - head_);
        return std::span<const std::byte>(buffer_.data() + head_, contiguous);
    }
    
    size_t available() const { return Size - count_; }
    size_t size() const { return count_; }
    bool empty() const { return count_ == 0; }
};

int main() {
    // 創建測試消息
    std::vector<std::byte> rawMessage(sizeof(MessageHeader) + 8);
    
    MessageHeader header{1, 8, 1234567890};
    std::memcpy(rawMessage.data(), &header, sizeof(header));
    
    uint64_t payload = 42;
    std::memcpy(rawMessage.data() + sizeof(header), &payload, sizeof(payload));
    
    // 使用 span 解析
    std::span<const std::byte> messageSpan(rawMessage);
    
    if (auto hdr = MessageParser::parseHeader(messageSpan)) {
        std::cout << "Message type: " << hdr->type << "\n";
        std::cout << "Message length: " << hdr->length << "\n";
        std::cout << "Timestamp: " << hdr->timestamp << "\n";
        
        auto body = MessageParser::getBody(messageSpan);
        if (auto value = MessageParser::parseInteger<uint64_t>(body)) {
            std::cout << "Payload: " << *value << "\n";
        }
    }
    
    // 測試環形緩衝區
    std::cout << "\n=== Ring Buffer Test ===\n";
    RingBuffer<64> ring;
    
    std::array<std::byte, 16> testData;
    for (size_t i = 0; i < testData.size(); ++i) {
        testData[i] = static_cast<std::byte>(i);
    }
    
    ring.write(testData);
    std::cout << "Written " << testData.size() << " bytes\n";
    std::cout << "Buffer size: " << ring.size() << "\n";
    
    auto peeked = ring.peek(8);
    std::cout << "Peeked " << peeked.size() << " bytes\n";
    
    std::array<std::byte, 8> output;
    size_t read = ring.read(output);
    std::cout << "Read " << read << " bytes\n";
    std::cout << "Remaining: " << ring.size() << " bytes\n";
    
    return 0;
}
```

---

### 練習 10: 綜合練習 - 訂單管理系統

**任務**: 結合本課所學,實現一個小型訂單管理系統。

```cpp
#include <variant>
#include <optional>
#include <string_view>
#include <span>
#include <unordered_map>
#include <vector>
#include <iostream>
#include <memory>
#include <chrono>

// 訂單類型
enum class OrderType { MARKET, LIMIT };
enum class OrderSide { BUY, SELL };
enum class OrderStatus { NEW, FILLED, CANCELLED, REJECTED };

// 訂單結構
struct Order {
    int id;
    std::string symbol;
    OrderType type;
    OrderSide side;
    double price;
    int quantity;
    int filledQty = 0;
    OrderStatus status = OrderStatus::NEW;
    
    double filledValue = 0.0;
    
    bool isFilled() const { return filledQty >= quantity; }
    double avgPrice() const { return filledQty > 0 ? filledValue / filledQty : 0; }
};

// 事件類型
struct OrderAccepted { int orderId; };
struct OrderRejected { int orderId; std::string reason; };
struct OrderFilled { int orderId; int qty; double price; };
struct OrderCancelled { int orderId; };

using OrderEvent = std::variant<OrderAccepted, OrderRejected, OrderFilled, OrderCancelled>;

// 訂單管理器
class OrderManager {
private:
    std::unordered_map<int, Order> orders_;
    std::vector<OrderEvent> events_;
    int nextOrderId_ = 1000;
    
public:
    // 提交訂單
    std::optional<int> submitOrder(std::string_view symbol, OrderType type,
                                   OrderSide side, double price, int quantity) {
        if (quantity <= 0) {
            return std::nullopt;
        }
        
        int orderId = nextOrderId_++;
        Order order{
            orderId,
            std::string(symbol),
            type,
            side,
            price,
            quantity
        };
        
        orders_[orderId] = std::move(order);
        events_.push_back(OrderAccepted{orderId});
        
        return orderId;
    }
    
    // 取消訂單
    bool cancelOrder(int orderId) {
        auto it = orders_.find(orderId);
        if (it == orders_.end() || it->second.status != OrderStatus::NEW) {
            return false;
        }
        
        it->second.status = OrderStatus::CANCELLED;
        events_.push_back(OrderCancelled{orderId});
        return true;
    }
    
    // 成交回報
    bool fillOrder(int orderId, int qty, double price) {
        auto it = orders_.find(orderId);
        if (it == orders_.end() || it->second.status != OrderStatus::NEW) {
            return false;
        }
        
        auto& order = it->second;
        int actualQty = std::min(qty, order.quantity - order.filledQty);
        
        order.filledQty += actualQty;
        order.filledValue += actualQty * price;
        
        if (order.isFilled()) {
            order.status = OrderStatus::FILLED;
        }
        
        events_.push_back(OrderFilled{orderId, actualQty, price});
        return true;
    }
    
    // 查詢訂單
    std::optional<Order> getOrder(int orderId) const {
        auto it = orders_.find(orderId);
        if (it != orders_.end()) {
            return it->second;
        }
        return std::nullopt;
    }
    
    // 獲取所有事件
    std::span<const OrderEvent> getEvents() const {
        return events_;
    }
    
    // 清除事件
    void clearEvents() {
        events_.clear();
    }
};

// 事件處理器
class EventProcessor {
public:
    void operator()(const OrderAccepted& e) {
        std::cout << "Order " << e.orderId << " accepted\n";
    }
    
    void operator()(const OrderRejected& e) {
        std::cout << "Order " << e.orderId << " rejected: " << e.reason << "\n";
    }
    
    void operator()(const OrderFilled& e) {
        std::cout << "Order " << e.orderId << " filled " 
                  << e.qty << " @ " << e.price << "\n";
    }
    
    void operator()(const OrderCancelled& e) {
        std::cout << "Order " << e.orderId << " cancelled\n";
    }
};

int main() {
    OrderManager manager;
    EventProcessor processor;
    
    // 提交訂單
    auto orderId1 = manager.submitOrder("AAPL", OrderType::LIMIT, 
                                        OrderSide::BUY, 150.0, 100);
    auto orderId2 = manager.submitOrder("GOOGL", OrderType::MARKET,
                                        OrderSide::SELL, 0, 50);
    
    // 處理事件
    std::cout << "=== Initial Events ===\n";
    for (const auto& event : manager.getEvents()) {
        std::visit(processor, event);
    }
    manager.clearEvents();
    
    // 成交
    if (orderId1) {
        manager.fillOrder(*orderId1, 50, 149.95);
        manager.fillOrder(*orderId1, 50, 150.05);
    }
    
    // 取消
    if (orderId2) {
        manager.cancelOrder(*orderId2);
    }
    
    // 處理新事件
    std::cout << "\n=== Trading Events ===\n";
    for (const auto& event : manager.getEvents()) {
        std::visit(processor, event);
    }
    
    // 查詢訂單狀態
    std::cout << "\n=== Order Status ===\n";
    if (orderId1) {
        if (auto order = manager.getOrder(*orderId1)) {
            std::cout << "Order " << order->id << ": "
                      << order->filledQty << "/" << order->quantity
                      << " filled, avg price: $" << order->avgPrice() << "\n";
        }
    }
    
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
