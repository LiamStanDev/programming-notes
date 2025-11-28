# 測試框架 - Google Test 與 Benchmark

## 概述

Google Test (GTest) 是 C++ 單元測試框架，Google Benchmark 是效能基準測試框架。兩者是 Google 開源專案，廣泛應用於工業級軟體開發。

### 核心特點

**Google Test:**
1. **豐富斷言**: 100+ 斷言巨集
2. **測試組織**: Test Fixture, Test Suite
3. **參數化測試**: 數據驅動測試
4. **死亡測試**: 測試崩潰行為
5. **Mock 框架**: Google Mock (GMock) 整合

**Google Benchmark:**
1. **精確計時**: 納秒級精度
2. **統計分析**: 自動計算平均、標準差
3. **比較模式**: 基準對比
4. **自動調整**: 智慧迭代次數

### 安裝與整合

```bash
# Ubuntu/Debian
sudo apt-get install libgtest-dev libgmock-dev libbenchmark-dev

# 從源碼編譯 GTest
git clone https://github.com/google/googletest.git
cd googletest
mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install

# 從源碼編譯 Benchmark
git clone https://github.com/google/benchmark.git
cd benchmark
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
sudo make install
```

CMake 整合:

```cmake
find_package(GTest REQUIRED)
find_package(benchmark REQUIRED)

# 測試執行檔
add_executable(unit_tests test_main.cpp)
target_link_libraries(unit_tests GTest::gtest GTest::gtest_main)

# 基準測試執行檔
add_executable(benchmarks bench_main.cpp)
target_link_libraries(benchmarks benchmark::benchmark benchmark::benchark_main)

# 啟用 CTest
enable_testing()
add_test(NAME unit_tests COMMAND unit_tests)
```

---

## Google Test 基礎

### 第一個測試

```cpp
#include <gtest/gtest.h>

// 簡單測試
TEST(MathTest, Addition) {
    EXPECT_EQ(2 + 2, 4);
    EXPECT_NE(2 + 2, 5);
}

TEST(MathTest, Subtraction) {
    EXPECT_EQ(5 - 3, 2);
    EXPECT_GT(5, 3);
}

// main 函數
int main(int argc, char** argv) {
    ::testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```

編譯與執行:

```bash
g++ -std=c++17 test.cpp -lgtest -lgtest_main -pthread -o test
./test

# 輸出:
# [==========] Running 2 tests from 1 test suite.
# [----------] 2 tests from MathTest
# [ RUN      ] MathTest.Addition
# [       OK ] MathTest.Addition (0 ms)
# [ RUN      ] MathTest.Subtraction
# [       OK ] MathTest.Subtraction (0 ms)
# [----------] 2 tests from MathTest (0 ms total)
```

### 常用斷言

```cpp
#include <gtest/gtest.h>
#include <string>

TEST(AssertionTest, BasicAssertions) {
    // 布林值
    EXPECT_TRUE(true);
    EXPECT_FALSE(false);
    
    // 相等性
    EXPECT_EQ(10, 10);
    EXPECT_NE(10, 20);
    
    // 比較
    EXPECT_LT(5, 10);   // Less Than
    EXPECT_LE(5, 5);    // Less or Equal
    EXPECT_GT(10, 5);   // Greater Than
    EXPECT_GE(10, 10);  // Greater or Equal
    
    // 浮點數 (帶容差)
    EXPECT_FLOAT_EQ(1.0f, 1.0000001f);
    EXPECT_DOUBLE_EQ(1.0, 1.0000000001);
    EXPECT_NEAR(1.0, 1.1, 0.2);  // |1.0 - 1.1| < 0.2
    
    // 字串
    EXPECT_STREQ("hello", "hello");
    EXPECT_STRNE("hello", "world");
    
    // C++ string
    std::string s1 = "test";
    std::string s2 = "test";
    EXPECT_EQ(s1, s2);
}
```

### ASSERT vs EXPECT

```cpp
#include <gtest/gtest.h>

TEST(AssertVsExpect, Difference) {
    // EXPECT: 失敗後繼續執行
    EXPECT_EQ(1, 2);  // 失敗，但繼續
    EXPECT_EQ(3, 3);  // 仍會執行
    
    // ASSERT: 失敗後立即退出當前測試
    ASSERT_EQ(1, 1);  // 成功，繼續
    ASSERT_EQ(1, 2);  // 失敗，以下不執行
    EXPECT_EQ(3, 3);  // 不會執行到這裡
}
```

---

## Test Fixture

### 基本 Fixture

```cpp
#include <gtest/gtest.h>
#include <vector>

class VectorTest : public ::testing::Test {
protected:
    // 每個測試前執行
    void SetUp() override {
        vec_.push_back(1);
        vec_.push_back(2);
        vec_.push_back(3);
    }
    
    // 每個測試後執行
    void TearDown() override {
        vec_.clear();
    }
    
    std::vector<int> vec_;
};

TEST_F(VectorTest, Size) {
    EXPECT_EQ(vec_.size(), 3);
}

TEST_F(VectorTest, PushBack) {
    vec_.push_back(4);
    EXPECT_EQ(vec_.size(), 4);
    EXPECT_EQ(vec_[3], 4);
}

TEST_F(VectorTest, Clear) {
    vec_.clear();
    EXPECT_TRUE(vec_.empty());
}
```

### 交易系統 Fixture

```cpp
#include <gtest/gtest.h>
#include <memory>

class OrderBook {
public:
    void add_order(uint64_t id, double price, uint64_t qty) {
        // 實現...
    }
    
    void cancel_order(uint64_t id) {
        // 實現...
    }
    
    size_t order_count() const {
        // 實現...
        return 0;
    }
    
    double best_bid() const {
        // 實現...
        return 0.0;
    }
};

class OrderBookTest : public ::testing::Test {
protected:
    void SetUp() override {
        order_book_ = std::make_unique<OrderBook>();
        
        // 預填測試數據
        order_book_->add_order(1, 100.0, 100);
        order_book_->add_order(2, 100.5, 200);
        order_book_->add_order(3, 99.5, 150);
    }
    
    void TearDown() override {
        order_book_.reset();
    }
    
    std::unique_ptr<OrderBook> order_book_;
};

TEST_F(OrderBookTest, InitialState) {
    EXPECT_EQ(order_book_->order_count(), 3);
}

TEST_F(OrderBookTest, AddOrder) {
    order_book_->add_order(4, 101.0, 50);
    EXPECT_EQ(order_book_->order_count(), 4);
}

TEST_F(OrderBookTest, CancelOrder) {
    order_book_->cancel_order(1);
    EXPECT_EQ(order_book_->order_count(), 2);
}

TEST_F(OrderBookTest, BestBid) {
    double bid = order_book_->best_bid();
    EXPECT_DOUBLE_EQ(bid, 100.5);
}
```

---

## 參數化測試

### 基本參數化

```cpp
#include <gtest/gtest.h>

// 被測函數
bool is_prime(int n) {
    if (n <= 1) return false;
    for (int i = 2; i * i <= n; ++i) {
        if (n % i == 0) return false;
    }
    return true;
}

class PrimeTest : public ::testing::TestWithParam<int> {};

TEST_P(PrimeTest, CheckPrime) {
    int n = GetParam();
    EXPECT_TRUE(is_prime(n));
}

INSTANTIATE_TEST_SUITE_P(
    KnownPrimes,
    PrimeTest,
    ::testing::Values(2, 3, 5, 7, 11, 13, 17, 19, 23, 29)
);

// 執行 10 個測試案例 (每個數字一個)
```

### 多參數測試

```cpp
#include <gtest/gtest.h>
#include <tuple>

// 訂單驗證函數
bool validate_order(double price, uint64_t quantity) {
    return price > 0 && quantity > 0 && quantity <= 1000000;
}

class OrderValidationTest : public ::testing::TestWithParam<std::tuple<double, uint64_t, bool>> {};

TEST_P(OrderValidationTest, ValidateOrder) {
    auto [price, quantity, expected] = GetParam();
    EXPECT_EQ(validate_order(price, quantity), expected);
}

INSTANTIATE_TEST_SUITE_P(
    OrderTests,
    OrderValidationTest,
    ::testing::Values(
        std::make_tuple(100.0, 100, true),      // 正常
        std::make_tuple(-10.0, 100, false),     // 負價格
        std::make_tuple(100.0, 0, false),       // 零數量
        std::make_tuple(100.0, 2000000, false), // 超量
        std::make_tuple(0.01, 1, true)          // 邊界
    )
);
```

### 組合參數

```cpp
#include <gtest/gtest.h>

class CombinedTest : public ::testing::TestWithParam<std::tuple<int, int>> {};

TEST_P(CombinedTest, Addition) {
    auto [a, b] = GetParam();
    EXPECT_GT(a + b, a);
    EXPECT_GT(a + b, b);
}

INSTANTIATE_TEST_SUITE_P(
    CombinedParams,
    CombinedTest,
    ::testing::Combine(
        ::testing::Values(1, 10, 100),     // 第一個參數
        ::testing::Values(2, 20, 200)      // 第二個參數
    )
    // 產生 3 x 3 = 9 個測試案例
);
```

---

## Google Mock (GMock)

### 基本 Mock

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>

// 介面
class Database {
public:
    virtual ~Database() = default;
    virtual bool connect(const std::string& host) = 0;
    virtual int query(const std::string& sql) = 0;
};

// Mock 類別
class MockDatabase : public Database {
public:
    MOCK_METHOD(bool, connect, (const std::string& host), (override));
    MOCK_METHOD(int, query, (const std::string& sql), (override));
};

// 使用 mock 的類別
class UserService {
    Database* db_;
    
public:
    explicit UserService(Database* db) : db_(db) {}
    
    bool initialize() {
        return db_->connect("localhost");
    }
    
    int get_user_count() {
        return db_->query("SELECT COUNT(*) FROM users");
    }
};

// 測試
TEST(UserServiceTest, Initialize) {
    MockDatabase mock_db;
    
    // 設定期望: connect 會被呼叫一次，並返回 true
    EXPECT_CALL(mock_db, connect("localhost"))
        .Times(1)
        .WillOnce(::testing::Return(true));
    
    UserService service(&mock_db);
    EXPECT_TRUE(service.initialize());
}

TEST(UserServiceTest, GetUserCount) {
    MockDatabase mock_db;
    
    // 設定期望
    EXPECT_CALL(mock_db, query("SELECT COUNT(*) FROM users"))
        .WillOnce(::testing::Return(42));
    
    UserService service(&mock_db);
    EXPECT_EQ(service.get_user_count(), 42);
}
```

### 交易系統 Mock

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>
#include <string>

// 市場數據介面
class MarketDataFeed {
public:
    virtual ~MarketDataFeed() = default;
    virtual bool subscribe(const std::string& symbol) = 0;
    virtual double get_last_price(const std::string& symbol) = 0;
};

// Mock
class MockMarketDataFeed : public MarketDataFeed {
public:
    MOCK_METHOD(bool, subscribe, (const std::string& symbol), (override));
    MOCK_METHOD(double, get_last_price, (const std::string& symbol), (override));
};

// 交易策略
class TradingStrategy {
    MarketDataFeed* feed_;
    
public:
    explicit TradingStrategy(MarketDataFeed* feed) : feed_(feed) {}
    
    bool start(const std::string& symbol) {
        return feed_->subscribe(symbol);
    }
    
    bool should_buy(const std::string& symbol, double threshold) {
        double price = feed_->get_last_price(symbol);
        return price < threshold;
    }
};

// 測試
TEST(TradingStrategyTest, Start) {
    MockMarketDataFeed mock_feed;
    
    EXPECT_CALL(mock_feed, subscribe("AAPL"))
        .WillOnce(::testing::Return(true));
    
    TradingStrategy strategy(&mock_feed);
    EXPECT_TRUE(strategy.start("AAPL"));
}

TEST(TradingStrategyTest, ShouldBuy) {
    MockMarketDataFeed mock_feed;
    
    // 價格低於閾值，應該買入
    EXPECT_CALL(mock_feed, get_last_price("AAPL"))
        .WillOnce(::testing::Return(145.0));
    
    TradingStrategy strategy(&mock_feed);
    EXPECT_TRUE(strategy.should_buy("AAPL", 150.0));
}

TEST(TradingStrategyTest, ShouldNotBuy) {
    MockMarketDataFeed mock_feed;
    
    // 價格高於閾值，不應該買入
    EXPECT_CALL(mock_feed, get_last_price("AAPL"))
        .WillOnce(::testing::Return(155.0));
    
    TradingStrategy strategy(&mock_feed);
    EXPECT_FALSE(strategy.should_buy("AAPL", 150.0));
}
```

### 複雜 Mock 行為

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>

using ::testing::_;
using ::testing::Return;
using ::testing::AtLeast;
using ::testing::Sequence;

class MockDatabase : public Database {
public:
    MOCK_METHOD(bool, connect, (const std::string&), (override));
    MOCK_METHOD(int, query, (const std::string&), (override));
};

TEST(MockBehaviorTest, AnyArgument) {
    MockDatabase db;
    
    // 接受任何參數
    EXPECT_CALL(db, connect(_))
        .WillOnce(Return(true));
    
    EXPECT_TRUE(db.connect("any_host"));
}

TEST(MockBehaviorTest, MultipleReturns) {
    MockDatabase db;
    
    // 多次呼叫，不同返回值
    EXPECT_CALL(db, query(_))
        .WillOnce(Return(10))
        .WillOnce(Return(20))
        .WillRepeatedly(Return(30));
    
    EXPECT_EQ(db.query("q1"), 10);
    EXPECT_EQ(db.query("q2"), 20);
    EXPECT_EQ(db.query("q3"), 30);
    EXPECT_EQ(db.query("q4"), 30);
}

TEST(MockBehaviorTest, CallTimes) {
    MockDatabase db;
    
    // 至少呼叫 2 次
    EXPECT_CALL(db, query(_))
        .Times(AtLeast(2))
        .WillRepeatedly(Return(0));
    
    db.query("q1");
    db.query("q2");
    db.query("q3");
}

TEST(MockBehaviorTest, CallSequence) {
    MockDatabase db;
    Sequence seq;
    
    // 必須按順序呼叫
    EXPECT_CALL(db, connect(_))
        .InSequence(seq)
        .WillOnce(Return(true));
    
    EXPECT_CALL(db, query(_))
        .InSequence(seq)
        .WillOnce(Return(42));
    
    db.connect("host");
    db.query("SELECT ...");
}
```

---

## Google Benchmark 基礎

### 第一個 Benchmark

```cpp
#include <benchmark/benchmark.h>
#include <vector>

// 被測函數
static void vector_push_back(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<int> vec;
        for (int i = 0; i < 1000; ++i) {
            vec.push_back(i);
        }
        benchmark::DoNotOptimize(vec.data());
    }
}

BENCHMARK(vector_push_back);

BENCHMARK_MAIN();
```

編譯與執行:

```bash
g++ -std=c++17 -O3 bench.cpp -lbenchmark -pthread -o bench
./bench

# 輸出:
# Run on (8 X 3600 MHz CPU s)
# CPU Caches:
#   L1 Data 32 KiB (x4)
#   L1 Instruction 32 KiB (x4)
#   L2 Unified 256 KiB (x4)
#   L3 Unified 8192 KiB (x1)
# -----------------------------------------------------
# Benchmark               Time      CPU   Iterations
# -----------------------------------------------------
# vector_push_back     2345 ns  2345 ns      298234
```

### 參數化 Benchmark

```cpp
#include <benchmark/benchmark.h>
#include <vector>

static void vector_reserve(benchmark::State& state) {
    int size = state.range(0);
    
    for (auto _ : state) {
        std::vector<int> vec;
        vec.reserve(size);
        
        for (int i = 0; i < size; ++i) {
            vec.push_back(i);
        }
        
        benchmark::DoNotOptimize(vec.data());
    }
}

// 測試不同大小
BENCHMARK(vector_reserve)->Range(8, 8 << 10);  // 8, 64, 512, 4096, 8192

BENCHMARK_MAIN();
```

### 複雜度分析

```cpp
#include <benchmark/benchmark.h>
#include <algorithm>
#include <vector>

static void sort_benchmark(benchmark::State& state) {
    int size = state.range(0);
    
    for (auto _ : state) {
        state.PauseTiming();  // 暫停計時
        
        std::vector<int> vec(size);
        for (int i = 0; i < size; ++i) {
            vec[i] = size - i;
        }
        
        state.ResumeTiming();  // 恢復計時
        
        std::sort(vec.begin(), vec.end());
        
        benchmark::DoNotOptimize(vec.data());
    }
    
    // 自動計算複雜度
    state.SetComplexityN(size);
}

BENCHMARK(sort_benchmark)
    ->RangeMultiplier(2)
    ->Range(1 << 10, 1 << 18)
    ->Complexity(benchmark::oN);  // O(N) 或 oNLogN, oNSquared

BENCHMARK_MAIN();
```

---

## 實戰基準測試

### 容器效能比較

```cpp
#include <benchmark/benchmark.h>
#include <vector>
#include <list>
#include <deque>

static void vector_insert(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<int> container;
        for (int i = 0; i < 1000; ++i) {
            container.push_back(i);
        }
        benchmark::DoNotOptimize(container.data());
    }
}

static void list_insert(benchmark::State& state) {
    for (auto _ : state) {
        std::list<int> container;
        for (int i = 0; i < 1000; ++i) {
            container.push_back(i);
        }
        benchmark::DoNotOptimize(&container);
    }
}

static void deque_insert(benchmark::State& state) {
    for (auto _ : state) {
        std::deque<int> container;
        for (int i = 0; i < 1000; ++i) {
            container.push_back(i);
        }
        benchmark::DoNotOptimize(&container);
    }
}

BENCHMARK(vector_insert);
BENCHMARK(list_insert);
BENCHMARK(deque_insert);

BENCHMARK_MAIN();
```

### 記憶體分配

```cpp
#include <benchmark/benchmark.h>
#include <memory>

static void raw_new_delete(benchmark::State& state) {
    for (auto _ : state) {
        int* ptr = new int(42);
        benchmark::DoNotOptimize(ptr);
        delete ptr;
    }
}

static void unique_ptr(benchmark::State& state) {
    for (auto _ : state) {
        auto ptr = std::make_unique<int>(42);
        benchmark::DoNotOptimize(ptr.get());
    }
}

static void shared_ptr(benchmark::State& state) {
    for (auto _ : state) {
        auto ptr = std::make_shared<int>(42);
        benchmark::DoNotOptimize(ptr.get());
    }
}

BENCHMARK(raw_new_delete);
BENCHMARK(unique_ptr);
BENCHMARK(shared_ptr);

BENCHMARK_MAIN();
```

### 訂單簿操作

```cpp
#include <benchmark/benchmark.h>
#include <map>
#include <unordered_map>

class OrderBook {
    std::map<double, uint64_t> bids_;
    
public:
    void add_bid(double price, uint64_t qty) {
        bids_[price] += qty;
    }
    
    void remove_bid(double price) {
        bids_.erase(price);
    }
    
    double best_bid() const {
        return bids_.empty() ? 0.0 : bids_.rbegin()->first;
    }
};

static void orderbook_add_remove(benchmark::State& state) {
    OrderBook book;
    
    // 預填數據
    for (int i = 0; i < 100; ++i) {
        book.add_bid(100.0 + i * 0.01, 100);
    }
    
    for (auto _ : state) {
        book.add_bid(150.0, 100);
        benchmark::DoNotOptimize(book.best_bid());
        book.remove_bid(150.0);
    }
}

BENCHMARK(orderbook_add_remove);

static void orderbook_best_bid(benchmark::State& state) {
    OrderBook book;
    
    for (int i = 0; i < 1000; ++i) {
        book.add_bid(100.0 + i * 0.01, 100);
    }
    
    for (auto _ : state) {
        double bid = book.best_bid();
        benchmark::DoNotOptimize(bid);
    }
}

BENCHMARK(orderbook_best_bid);

BENCHMARK_MAIN();
```

### 序列化效能

```cpp
#include <benchmark/benchmark.h>
#include <nlohmann/json.hpp>
#include <sstream>

using json = nlohmann::json;

struct Order {
    uint64_t id;
    std::string symbol;
    double price;
    uint64_t quantity;
};

static void json_serialize(benchmark::State& state) {
    Order order{12345, "AAPL", 150.50, 100};
    
    for (auto _ : state) {
        json j = {
            {"id", order.id},
            {"symbol", order.symbol},
            {"price", order.price},
            {"quantity", order.quantity}
        };
        
        std::string str = j.dump();
        benchmark::DoNotOptimize(str.data());
    }
}

static void manual_serialize(benchmark::State& state) {
    Order order{12345, "AAPL", 150.50, 100};
    
    for (auto _ : state) {
        std::ostringstream oss;
        oss << order.id << "|" 
            << order.symbol << "|"
            << order.price << "|"
            << order.quantity;
        
        std::string str = oss.str();
        benchmark::DoNotOptimize(str.data());
    }
}

BENCHMARK(json_serialize);
BENCHMARK(manual_serialize);

BENCHMARK_MAIN();
```

---

## 整合測試與 CI/CD

### CMake 整合

```cmake
cmake_minimum_required(VERSION 3.15)
project(TradingSystem)

set(CMAKE_CXX_STANDARD 17)

# 尋找庫
find_package(GTest REQUIRED)
find_package(benchmark REQUIRED)

# 主程式
add_library(trading_lib
    src/order_book.cpp
    src/market_data.cpp
)

# 單元測試
add_executable(unit_tests
    tests/test_order_book.cpp
    tests/test_market_data.cpp
)

target_link_libraries(unit_tests
    trading_lib
    GTest::gtest
    GTest::gtest_main
)

# 基準測試
add_executable(benchmarks
    benchmarks/bench_order_book.cpp
    benchmarks/bench_serialization.cpp
)

target_link_libraries(benchmarks
    trading_lib
    benchmark::benchmark
)

# 啟用測試
enable_testing()
add_test(NAME UnitTests COMMAND unit_tests)

# 自定義目標
add_custom_target(test_verbose
    COMMAND ${CMAKE_CTEST_COMMAND} --verbose
    DEPENDS unit_tests
)

add_custom_target(run_benchmarks
    COMMAND benchmarks
    DEPENDS benchmarks
)
```

### GitHub Actions CI

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Install dependencies
      run: |
        sudo apt-get update
        sudo apt-get install -y libgtest-dev libbenchmark-dev
    
    - name: Configure CMake
      run: cmake -B build -DCMAKE_BUILD_TYPE=Release
    
    - name: Build
      run: cmake --build build -j$(nproc)
    
    - name: Run tests
      run: cd build && ctest --output-on-failure
    
    - name: Run benchmarks
      run: ./build/benchmarks --benchmark_out=bench_results.json --benchmark_out_format=json
    
    - name: Upload results
      uses: actions/upload-artifact@v3
      with:
        name: benchmark-results
        path: bench_results.json
```

---

## 測試覆蓋率

### 使用 gcov/lcov

```bash
# 編譯時加入覆蓋率標誌
g++ -std=c++17 --coverage -O0 src/*.cpp tests/*.cpp -lgtest -o test

# 執行測試
./test

# 生成覆蓋率報告
lcov --capture --directory . --output-file coverage.info
lcov --remove coverage.info '/usr/*' --output-file coverage.info
genhtml coverage.info --output-directory coverage_report

# 查看報告
firefox coverage_report/index.html
```

CMake 配置:

```cmake
option(ENABLE_COVERAGE "Enable code coverage" OFF)

if(ENABLE_COVERAGE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
endif()
```

---

## 最佳實踐

### 1. 測試組織

```cpp
// 好的測試結構
// test_order_book.cpp

#include <gtest/gtest.h>
#include "order_book.h"

// 測試分組
namespace {

class OrderBookTest : public ::testing::Test {
protected:
    OrderBook book_;
};

// 基本功能
TEST_F(OrderBookTest, AddBid) { /* ... */ }
TEST_F(OrderBookTest, AddAsk) { /* ... */ }

// 邊界條件
TEST_F(OrderBookTest, EmptyBook) { /* ... */ }
TEST_F(OrderBookTest, MaxPrice) { /* ... */ }

// 錯誤處理
TEST_F(OrderBookTest, InvalidPrice) { /* ... */ }
TEST_F(OrderBookTest, InvalidQuantity) { /* ... */ }

}  // namespace
```

### 2. Benchmark 指南

```cpp
#include <benchmark/benchmark.h>

// 避免編譯器優化
static void proper_benchmark(benchmark::State& state) {
    for (auto _ : state) {
        int result = expensive_computation();
        
        // 防止編譯器優化掉 result
        benchmark::DoNotOptimize(result);
        
        // 防止記憶體被優化
        benchmark::ClobberMemory();
    }
}

// 排除設置時間
static void setup_excluded(benchmark::State& state) {
    for (auto _ : state) {
        state.PauseTiming();
        auto data = prepare_test_data();
        state.ResumeTiming();
        
        process(data);
    }
}

// 報告自定義指標
static void custom_metrics(benchmark::State& state) {
    size_t bytes_processed = 0;
    
    for (auto _ : state) {
        bytes_processed += do_work();
    }
    
    state.SetBytesProcessed(bytes_processed);
    state.SetItemsProcessed(state.iterations());
}

BENCHMARK(proper_benchmark);
BENCHMARK(setup_excluded);
BENCHMARK(custom_metrics);
```

### 3. Mock 設計原則

```cpp
// 好的 Mock 設計
class GoodMock : public Interface {
public:
    // 清晰的方法簽名
    MOCK_METHOD(bool, connect, (const std::string& host), (override));
    
    // 提供預設行為
    GoodMock() {
        ON_CALL(*this, connect(_))
            .WillByDefault(::testing::Return(true));
    }
};

// 避免過度 Mock
TEST(GoodTest, MinimalMocking) {
    // 只 Mock 必要的依賴
    MockDatabase db;
    RealCache cache;  // 使用真實實現
    
    Service service(&db, &cache);
    
    EXPECT_CALL(db, query(_))
        .WillOnce(::testing::Return(42));
    
    // 測試...
}
```

---

## 參考資料

1. [Google Test Primer](https://google.github.io/googletest/primer.html)
2. [Google Test Advanced Guide](https://google.github.io/googletest/advanced.html)
3. [Google Mock for Dummies](https://google.github.io/googletest/gmock_for_dummies.html)
4. [Google Benchmark User Guide](https://github.com/google/benchmark/blob/main/docs/user_guide.md)
5. [Effective Unit Testing](https://martinfowler.com/articles/practical-test-pyramid.html)
