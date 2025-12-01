# Lambda 與函數對象

> **學習優先級**: ⭐⭐ 建議
>
> Lambda 表達式是現代 C++ 函數式編程的基礎,與 STL 算法完美結合。

---

## 目錄

1. [Lambda 基本語法](#1-lambda-基本語法)
2. [捕獲機制](#2-捕獲機制)
3. [泛型 Lambda](#3-泛型-lambda)
4. [Lambda 與 STL](#4-lambda-與-stl)
5. [函數對象](#5-函數對象)
6. [高頻交易應用](#6-高頻交易應用)

---

## 1. Lambda 基本語法

### 1.1 基本形式

```cpp
// [捕獲列表](參數列表) -> 返回類型 { 函數體 }

// 基本 lambda
auto add = [](int a, int b) -> int {
    return a + b;
};

// 返回類型推導
auto multiply = [](int a, int b) { return a * b; };

// 無參數
auto greet = []() { std::cout << "Hello\n"; };

// 使用
int result = add(3, 4);  // 7
greet();
```

### 1.2 Lambda 的本質

```cpp
// Lambda 本質是匿名函數對象
auto add = [](int a, int b) { return a + b; };

// 等價於:
struct __lambda_add {
    auto operator()(int a, int b) const { return a + b; }
};
__lambda_add add;

// 編譯後內聯展開,無函數調用開銷
```

### 1.3 立即調用 Lambda (IIFE)

```cpp
// 初始化複雜常量
const auto config = [&]() {
    Config cfg;
    cfg.load_from_file("config.json");
    cfg.validate();
    return cfg;
}();

// 條件初始化
const int value = [](bool condition) {
    if (condition) {
        return expensive_computation();
    } else {
        return default_value();
    }
}(some_condition);
```

---

## 2. 捕獲機制

### 2.1 值捕獲

```cpp
int x = 10;
int y = 20;

// 值捕獲
auto f1 = [x, y]() { return x + y; };

// 捕獲所有 (值)
auto f2 = [=]() { return x + y; };

// ⚠️ 值捕獲是拷貝,修改不影響外部
auto f3 = [x]() mutable {
    x = 100;  // 只修改拷貝
};
f3();
std::cout << x << "\n";  // 10 (未改變)
```

### 2.2 引用捕獲

```cpp
int x = 10;
int y = 20;

// 引用捕獲
auto f1 = [&x, &y]() { x++; y++; };

// 捕獲所有 (引用)
auto f2 = [&]() { x++; y++; };

// 混合捕獲
auto f3 = [x, &y]() { y = x; };

f1();
std::cout << x << ", " << y << "\n";  // 11, 21
```

### 2.3 初始化捕獲 (C++14)

```cpp
// 初始化捕獲
auto f1 = [z = x + y]() { return z * 2; };

// 移動捕獲
auto ptr = std::make_unique<int>(42);
auto f2 = [p = std::move(ptr)]() { return *p; };

// ptr 已失效
// std::cout << *ptr << "\n";  // Error!

// 捕獲成員
struct Widget {
    int value = 42;

    auto get_lambda() {
        return [v = value]() { return v; };
    }
};
```

### 2.4 捕獲 this

```cpp
class Counter {
    int count_ = 0;

public:
    auto get_incrementer() {
        // ❌ 捕獲 this 指針 (危險)
        return [this]() { return ++count_; };
    }

    auto get_incrementer_safe() {
        // ✅ 拷貝捕獲 (C++17)
        return [*this]() mutable { return ++count_; };
    }

    auto get_incrementer_ref() {
        // ✅ 捕獲成員變量
        return [&count = count_]() { return ++count; };
    }
};
```

### 2.5 捕獲陷阱

```cpp
// ❌ 懸空引用
std::function<int()> create_lambda() {
    int x = 42;
    return [&x]() { return x; };  // UB! x 已銷毀
}

// ✅ 值捕獲
std::function<int()> create_lambda_safe() {
    int x = 42;
    return [x]() { return x; };  // OK
}

// ❌ 大對象值捕獲
std::vector<int> data(1000);
auto lambda = [data]() {  // 拷貝整個 vector!
    return data.size();
};

// ✅ 引用捕獲或移動捕獲
auto lambda_ref = [&data]() { return data.size(); };
auto lambda_move = [data = std::move(data)]() { return data.size(); };
```

---

## 3. 泛型 Lambda

### 3.1 Auto 參數 (C++14)

```cpp
// auto 參數
auto print = [](const auto& x) {
    std::cout << x << "\n";
};

print(42);
print(3.14);
print("Hello");

// 多個 auto 參數
auto add = [](auto a, auto b) {
    return a + b;
};

add(1, 2);      // int
add(1.5, 2.5);  // double
```

### 3.2 模板 Lambda (C++20)

```cpp
// 顯式模板參數
auto print = []<typename T>(const T& x) {
    std::cout << typeid(T).name() << ": " << x << "\n";
};

// 約束模板參數
auto add = []<typename T>(T a, T b) requires std::is_arithmetic_v<T> {
    return a + b;
};
```

### 3.3 完美轉發

```cpp
// 完美轉發參數
auto forward_call = [](auto&& func, auto&&... args) {
    return std::forward<decltype(func)>(func)(
        std::forward<decltype(args)>(args)...);
};

void process(int& x) { std::cout << "Lvalue\n"; }
void process(int&& x) { std::cout << "Rvalue\n"; }

int x = 10;
forward_call(process, x);   // Lvalue
forward_call(process, 20);  // Rvalue
```

---

## 4. Lambda 與 STL

### 4.1 排序與查找

```cpp
#include <algorithm>
#include <vector>

std::vector<int> vec = {5, 2, 8, 1, 9};

// 排序
std::sort(vec.begin(), vec.end(),
    [](int a, int b) { return a > b; }); // 前一個要大於後一個

// 查找
auto it = std::find_if(vec.begin(), vec.end(),
    [](int x) { return x > 5; });

// 計數
int count = std::count_if(vec.begin(), vec.end(),
    [](int x) { return x % 2 == 0; });
```

### 4.2 過濾與轉換

```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};
std::vector<int> result;

// 過濾
std::copy_if(vec.begin(), vec.end(),
    std::back_inserter(result),
    [](int x) { return x > 2; });

// 轉換
std::transform(vec.begin(), vec.end(), vec.begin(),
    [](int x) { return x * 2; });

// 累加
int sum = std::accumulate(vec.begin(), vec.end(), 0,
    [](int acc, int x) { return acc + x; });
```

### 4.3 自定義比較器

```cpp
struct Order {
    int id;
    double price;
    int quantity;
};

std::vector<Order> orders;

// 按價格排序
std::sort(orders.begin(), orders.end(),
    [](const Order& a, const Order& b) {
        return a.price < b.price;
    });

// 按多個條件排序
std::sort(orders.begin(), orders.end(),
    [](const Order& a, const Order& b) {
        if (a.price != b.price) return a.price < b.price;
        return a.quantity > b.quantity;
    });

// 使用 map 自定義排序
auto cmp = [](const std::string& a, const std::string& b) {
    return a.size() < b.size();
};
std::map<std::string, int, decltype(cmp)> map(cmp);
```

---

## 5. 函數對象

### 5.1 函數對象 vs Lambda

```cpp
// 函數對象
struct Adder {
    int offset;

    Adder(int o) : offset(o) {}

    int operator()(int x) const {
        return x + offset;
    }
};

Adder add10(10);
int result = add10(5);  // 15

// 等價 Lambda
int offset = 10;
auto add10_lambda = [offset](int x) { return x + offset; };
```

### 5.2 std::function

```cpp
#include <functional>

// 存儲各種可調用對象
std::function<int(int, int)> func;

// 函數指針
int add(int a, int b) { return a + b; }
func = add;

// Lambda
func = [](int a, int b) { return a * b; };

// 函數對象
struct Multiplier {
    int operator()(int a, int b) const { return a * b; }
};
func = Multiplier{};

// ⚠️ std::function 有開銷 (類型擦除)
// 優先使用 auto 或模板
```

### 5.3 無狀態 Lambda 轉函數指針

```cpp
// 無捕獲 Lambda 可轉換為函數指針
using Predicate = bool(*)(const Order&);

Predicate is_valid = [](const Order& order) {
    return order.price > 0 && order.quantity > 0;
};

// 可內聯,無間接調用
bool check_order(const Order& order, Predicate pred) {
    return pred(order);
}
```

---

## 6. 高頻交易應用

### 6.1 市場數據過濾

```cpp
class MarketDataFilter {
    std::vector<std::function<bool(const Quote&)>> filters_;

public:
    void add_filter(std::function<bool(const Quote&)> filter) {
        filters_.push_back(std::move(filter));
    }

    bool should_process(const Quote& quote) const {
        return std::all_of(filters_.begin(), filters_.end(),
            [&quote](const auto& filter) {
                return filter(quote);
            });
    }
};

// 使用
MarketDataFilter filter;

// 價格範圍過濾
filter.add_filter([](const Quote& q) {
    return q.price >= 100.0 && q.price <= 200.0;
});

// 數量過濾
filter.add_filter([](const Quote& q) {
    return q.quantity >= 100;
});

// 符號過濾
std::set<std::string> symbols = {"AAPL", "GOOGL"};
filter.add_filter([&symbols](const Quote& q) {
    return symbols.count(q.symbol) > 0;
});
```

### 6.2 訂單簿操作

```cpp
class OrderBook {
    std::map<Price, std::vector<Order>> bids_;
    std::map<Price, std::vector<Order>> asks_;

public:
    // 查找最佳買價
    std::optional<Price> best_bid() const {
        if (bids_.empty()) return std::nullopt;
        return bids_.rbegin()->first;
    }

    // 移除滿足條件的訂單
    void remove_if(std::function<bool(const Order&)> pred) {
        auto remove_from_map = [&pred](auto& map) {
            for (auto& [price, orders] : map) {
                orders.erase(
                    std::remove_if(orders.begin(), orders.end(), pred),
                    orders.end()
                );
            }
        };

        remove_from_map(bids_);
        remove_from_map(asks_);
    }

    // 統計訂單
    int count_orders(std::function<bool(const Order&)> pred) const {
        int count = 0;

        auto count_in_map = [&](const auto& map) {
            for (const auto& [price, orders] : map) {
                count += std::count_if(orders.begin(), orders.end(), pred);
            }
        };

        count_in_map(bids_);
        count_in_map(asks_);

        return count;
    }
};

// 使用
OrderBook book;

// 移除過期訂單
auto now = std::chrono::system_clock::now();
book.remove_if([now](const Order& o) {
    return o.expire_time < now;
});

// 統計大單
int large_orders = book.count_orders([](const Order& o) {
    return o.quantity >= 10000;
});
```

### 6.3 性能監控

```cpp
class PerformanceMonitor {
    using Callback = std::function<void(const std::string&, double)>;
    std::vector<Callback> callbacks_;

public:
    void add_callback(Callback cb) {
        callbacks_.push_back(std::move(cb));
    }

    void report_latency(const std::string& operation, double latency_us) {
        for (const auto& cb : callbacks_) {
            cb(operation, latency_us);
        }
    }
};

// 使用
PerformanceMonitor monitor;

// 日誌回調
monitor.add_callback([](const std::string& op, double latency) {
    if (latency > 1000.0) {  // > 1ms
        std::cout << "WARNING: " << op << " took " << latency << "us\n";
    }
});

// 統計回調
std::map<std::string, std::vector<double>> latencies;
monitor.add_callback([&latencies](const std::string& op, double latency) {
    latencies[op].push_back(latency);
});

// 告警回調
monitor.add_callback([](const std::string& op, double latency) {
    if (latency > 10000.0) {  // > 10ms
        send_alert("High latency detected: " + op);
    }
});
```

### 6.4 策略模式

```cpp
class TradingStrategy {
public:
    using SignalGenerator = std::function<Signal(const MarketData&)>;

private:
    SignalGenerator signal_gen_;

public:
    explicit TradingStrategy(SignalGenerator gen)
        : signal_gen_(std::move(gen)) {}

    Signal generate_signal(const MarketData& data) {
        return signal_gen_(data);
    }
};

// 簡單移動平均策略
auto sma_strategy = TradingStrategy([](const MarketData& data) {
    double sma = calculate_sma(data, 20);
    if (data.last_price > sma) return Signal::BUY;
    if (data.last_price < sma) return Signal::SELL;
    return Signal::HOLD;
});

// 動量策略
auto momentum_strategy = TradingStrategy([](const MarketData& data) {
    double momentum = calculate_momentum(data, 10);
    if (momentum > 0.02) return Signal::BUY;
    if (momentum < -0.02) return Signal::SELL;
    return Signal::HOLD;
});
```

---

## 參考資料

1. Meyers, Scott. _Effective Modern C++_. O'Reilly, 2014. (Item 31-34)
2. [C++ Reference - Lambda Expressions](https://en.cppreference.com/w/cpp/language/lambda)
3. [C++ Core Guidelines - F: Functions](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-functions)
4. [Lambda Expressions in C++](https://docs.microsoft.com/en-us/cpp/cpp/lambda-expressions-in-cpp)
5. [CppCon - Lambdas from First Principles](https://www.youtube.com/watch?v=3jCOwajNch0)
