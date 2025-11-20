# Day 3: 編譯與優化

## 學習目標

本課程深入探討 C++ 編譯過程、編譯器優化技術與構建系統。掌握編譯優化對於構建高性能應用(尤其是高頻交易系統)至關重要。

**核心主題:**
- 編譯過程與中間產物
- 編譯器優化等級與選項
- Link Time Optimization (LTO)
- Profile-Guided Optimization (PGO)
- 函數內聯與分支預測
- CMake 構建系統

---

## 3.1 編譯過程

### 3.1.1 編譯四階段

```cpp
// example.cpp
#include <iostream>

#define SQUARE(x) ((x) * (x))

int main() {
    std::cout << SQUARE(5) << std::endl;
    return 0;
}
```

**編譯階段:**

```bash
# 1. 預處理 (Preprocessing): 展開宏、include
g++ -E example.cpp -o example.i
# 輸出: 展開所有宏和頭文件的源代碼

# 2. 編譯 (Compilation): 生成彙編代碼
g++ -S example.cpp -o example.s
# 輸出: x86-64 彙編代碼

# 3. 彙編 (Assembly): 生成機器碼
g++ -c example.cpp -o example.o
# 輸出: 目標文件 (二進制)

# 4. 鏈接 (Linking): 合併目標文件與庫
g++ example.o -o example
# 輸出: 可執行文件
```

### 3.1.2 查看預處理結果

```bash
# 查看宏展開
g++ -E example.cpp | grep -A 5 "main"
# 輸出:
# int main() {
#     std::cout << ((5) * (5)) << std::endl;
#     return 0;
# }
```

### 3.1.3 查看彙編代碼

```bash
# 生成彙編 (帶註釋)
g++ -S -fverbose-asm example.cpp -o example.s

# 查看優化後的彙編
g++ -S -O3 example.cpp -o example_O3.s
```

### 3.1.4 符號解析

```cpp
// lib.cpp
int add(int a, int b) {
    return a + b;
}

// main.cpp
extern int add(int, int);

int main() {
    return add(1, 2);
}
```

```bash
# 編譯目標文件
g++ -c lib.cpp -o lib.o
g++ -c main.cpp -o main.o

# 查看符號表
nm lib.o
# 輸出:
# 0000000000000000 T _Z3addii  (add 函數,mangled name)

nm main.o
# 輸出:
#                  U _Z3addii  (未定義符號,需鏈接)

# 鏈接
g++ main.o lib.o -o program
```

---

## 3.2 編譯器優化

### 3.2.1 優化等級

```cpp
// benchmark.cpp
#include <chrono>
#include <iostream>

int compute(int n) {
    int sum = 0;
    for (int i = 0; i < n; ++i) {
        sum += i * i;
    }
    return sum;
}

int main() {
    auto start = std::chrono::high_resolution_clock::now();
    
    volatile int result = compute(10000000);
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "Result: " << result << ", Time: " << duration.count() << " ms\n";
    return 0;
}
```

**優化等級對比:**

```bash
# -O0: 無優化 (默認)
g++ -O0 benchmark.cpp -o bench_O0
./bench_O0
# 時間: ~120 ms

# -O1: 基本優化
g++ -O1 benchmark.cpp -o bench_O1
./bench_O1
# 時間: ~80 ms

# -O2: 推薦優化
g++ -O2 benchmark.cpp -o bench_O2
./bench_O2
# 時間: ~50 ms

# -O3: 激進優化
g++ -O3 benchmark.cpp -o bench_O3
./bench_O3
# 時間: ~40 ms

# -Ofast: 最快 (可能違反標準)
g++ -Ofast benchmark.cpp -o bench_Ofast
./bench_Ofast
# 時間: ~35 ms
```

**優化等級說明:**

| 等級 | 說明 | 特性 | HFT 推薦 |
|------|------|------|----------|
| `-O0` | 無優化 | 快速編譯、易調試 | 開發 |
| `-O1` | 基本優化 | 減少代碼大小和執行時間 | ❌ |
| `-O2` | 推薦優化 | 平衡編譯時間與性能 | ✅ 測試 |
| `-O3` | 激進優化 | 更多內聯、循環展開 | ✅ 生產 |
| `-Ofast` | 最激進 | 可能違反 IEEE 浮點標準 | ⚠️ 謹慎 |
| `-Os` | 大小優化 | 優先減少代碼大小 | ❌ |

### 3.2.2 特定 CPU 優化

```bash
# -march=native: 針對當前 CPU 架構
g++ -O3 -march=native benchmark.cpp -o bench_native
# 啟用: AVX2, SSE4.2 等指令集

# 查看當前 CPU 支持的指令集
cat /proc/cpuinfo | grep flags

# 指定特定架構
g++ -O3 -march=skylake benchmark.cpp -o bench_skylake
g++ -O3 -march=haswell benchmark.cpp -o bench_haswell

# 可移植性 vs 性能
g++ -O3 -march=x86-64 benchmark.cpp -o bench_generic  # 通用
```

**HFT 實踐:**
- 開發環境: `-O0` (快速迭代)
- 測試環境: `-O2 -march=native` (接近生產)
- 生產環境: `-O3 -march=native -DNDEBUG` (最高性能)

### 3.2.3 Link Time Optimization (LTO)

```cpp
// module1.cpp
int expensive_function(int x) {
    int sum = 0;
    for (int i = 0; i < x; ++i) {
        sum += i;
    }
    return sum;
}

// module2.cpp
extern int expensive_function(int);

int main() {
    return expensive_function(1000);
}
```

```bash
# 傳統編譯 (無跨模塊優化)
g++ -O3 -c module1.cpp -o module1.o
g++ -O3 -c module2.cpp -o module2.o
g++ module1.o module2.o -o program

# 使用 LTO (跨模塊內聯)
g++ -O3 -flto -c module1.cpp -o module1.o
g++ -O3 -flto -c module2.cpp -o module2.o
g++ -O3 -flto module1.o module2.o -o program_lto

# LTO 可能內聯 expensive_function 到 main
```

**LTO 優勢:**
- 跨編譯單元內聯
- 移除未使用代碼
- 更好的常量傳播
- 性能提升: 5-15%

**LTO 限制:**
- 編譯時間增加
- 內存使用增加
- 調試困難

### 3.2.4 Profile-Guided Optimization (PGO)

```cpp
// pgo_example.cpp
#include <iostream>
#include <random>

int process_data(int x) {
    if (x % 2 == 0) {  // 熱路徑
        return x * x;
    } else {           // 冷路徑
        return x + 1;
    }
}

int main() {
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> dis(0, 99);
    
    int sum = 0;
    for (int i = 0; i < 10000000; ++i) {
        int x = dis(gen);
        sum += process_data(x);
    }
    
    std::cout << "Sum: " << sum << "\n";
    return 0;
}
```

```bash
# 步驟 1: 生成插樁版本
g++ -O3 -fprofile-generate pgo_example.cpp -o pgo_gen

# 步驟 2: 運行收集性能數據
./pgo_gen
# 生成: *.gcda 文件

# 步驟 3: 使用性能數據優化
g++ -O3 -fprofile-use pgo_example.cpp -o pgo_opt

# 對比
time ./pgo_gen
time ./pgo_opt  # 通常快 5-10%
```

**PGO 應用場景:**
- 分支預測優化
- 熱代碼優先內聯
- 緩存局部性優化
- HFT: 使用真實市場數據收集 profile

---

## 3.3 函數內聯

### 3.3.1 inline 關鍵字

```cpp
// inline_example.cpp
#include <iostream>

// 內聯建議 (編譯器可能忽略)
inline int add(int a, int b) {
    return a + b;
}

// 非內聯
int multiply(int a, int b) {
    return a * b;
}

int main() {
    int x = add(1, 2);        // 可能被內聯
    int y = multiply(3, 4);   // 需要函數調用
    
    std::cout << x + y << "\n";
    return 0;
}
```

```bash
# 查看是否內聯 (彙編中無 call 指令即已內聯)
g++ -O3 -S inline_example.cpp -o inline.s
grep "call" inline.s
```

### 3.3.2 強制內聯

```cpp
// 強制內聯 (GCC/Clang)
__attribute__((always_inline)) inline int force_inline(int x) {
    return x * x;
}

// 禁止內聯
__attribute__((noinline)) int no_inline(int x) {
    return x + 1;
}

// MSVC
__forceinline int force_inline_msvc(int x) {
    return x * x;
}

__declspec(noinline) int no_inline_msvc(int x) {
    return x + 1;
}
```

### 3.3.3 模板與內聯

```cpp
// 模板函數默認內聯
template<typename T>
T max_value(T a, T b) {
    return (a > b) ? a : b;
}

// 外部模板 (避免多次實例化)
// header.h
template<typename T>
class MyClass {
    void method();
};

// source.cpp
template class MyClass<int>;  // 顯式實例化

// other.cpp
extern template class MyClass<int>;  // 外部模板聲明
```

---

## 3.4 編譯器擴展

### 3.4.1 分支預測提示

```cpp
#include <iostream>

// GCC/Clang 內建
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)

int process_order(int order_id) {
    if (unlikely(order_id < 0)) {  // 冷路徑
        std::cerr << "Invalid order ID\n";
        return -1;
    }
    
    if (likely(order_id < 10000)) {  // 熱路徑
        // 正常處理
        return 0;
    }
    
    // 罕見情況
    std::cerr << "Order ID out of range\n";
    return -2;
}

// HFT 應用: 風控檢查
bool risk_check(double price, int quantity) {
    if (unlikely(price < 0 || quantity < 0)) {
        return false;  // 異常情況,快速拒絕
    }
    
    if (likely(quantity < 10000)) {
        return true;   // 正常訂單,熱路徑
    }
    
    // 大單額外檢查
    return price * quantity < 1000000;
}
```

### 3.4.2 編譯器屬性

```cpp
#include <iostream>

// C++17 標準屬性

// [[nodiscard]]: 警告忽略返回值
[[nodiscard]] int get_order_id() {
    return 42;
}

void test_nodiscard() {
    get_order_id();  // 編譯警告: 忽略返回值
}

// [[maybe_unused]]: 抑制未使用警告
void debug_print([[maybe_unused]] const char* msg) {
#ifdef DEBUG
    std::cout << msg << "\n";
#endif
}

// [[fallthrough]]: 顯式 switch fall-through
void process_status(int status) {
    switch (status) {
        case 0:
            std::cout << "Success\n";
            [[fallthrough]];
        case 1:
            std::cout << "Partially filled\n";
            break;
        default:
            std::cout << "Error\n";
    }
}

// [[likely]] / [[unlikely]] (C++20)
int process_message(int type) {
    if (type == 0) [[likely]] {
        return 1;  // 熱路徑
    } else [[unlikely]] {
        return 0;  // 冷路徑
    }
}
```

### 3.4.3 內建函數

```cpp
#include <iostream>

void builtin_examples() {
    // 預取 (prefetch)
    int arr[1000];
    for (int i = 0; i < 1000; ++i) {
        __builtin_prefetch(&arr[i + 10], 0, 3);  // 預取未來數據
        // 參數: 地址, 0=讀/1=寫, 0-3=局部性級別
        arr[i] = i;
    }
    
    // 位操作
    int x = 0b11010110;
    std::cout << "Popcount: " << __builtin_popcount(x) << "\n";         // 1 的個數
    std::cout << "CLZ: " << __builtin_clz(x) << "\n";                   // 前導零
    std::cout << "CTZ: " << __builtin_ctz(x) << "\n";                   // 尾隨零
    
    // 預測
    int a = 10, b = 20;
    if (__builtin_expect(a < b, 1)) {  // 預測為真
        std::cout << "Expected path\n";
    }
    
    // 不可達代碼
    switch (x) {
        case 1: return;
        case 2: return;
        default:
            __builtin_unreachable();  // 告訴編譯器此處不可達
    }
}
```

---

## 3.5 靜態庫與動態庫

### 3.5.1 創建靜態庫

```cpp
// math_lib.h
#pragma once

int add(int a, int b);
int multiply(int a, int b);

// math_lib.cpp
#include "math_lib.h"

int add(int a, int b) {
    return a + b;
}

int multiply(int a, int b) {
    return a * b;
}
```

```bash
# 編譯目標文件
g++ -c math_lib.cpp -o math_lib.o

# 創建靜態庫
ar rcs libmath.a math_lib.o

# 查看庫內容
ar -t libmath.a

# 使用靜態庫
g++ main.cpp -L. -lmath -o program
# 或
g++ main.cpp libmath.a -o program
```

### 3.5.2 創建動態庫

```bash
# 編譯位置無關代碼 (PIC)
g++ -fPIC -c math_lib.cpp -o math_lib.o

# 創建動態庫
g++ -shared -o libmath.so math_lib.o

# 使用動態庫
g++ main.cpp -L. -lmath -o program

# 運行時指定庫路徑
LD_LIBRARY_PATH=. ./program
```

### 3.5.3 符號可見性

```cpp
// visibility.h
#ifdef _WIN32
    #define EXPORT __declspec(dllexport)
#else
    #define EXPORT __attribute__((visibility("default")))
#endif

// 導出符號
EXPORT int public_function();

// 隱藏符號
__attribute__((visibility("hidden"))) int private_function();
```

```bash
# 編譯時設置默認隱藏
g++ -fPIC -fvisibility=hidden -c lib.cpp -o lib.o
g++ -shared -o libmylib.so lib.o

# 查看導出符號
nm -D libmylib.so | grep " T "
```

**HFT 選擇: 靜態鏈接**
- 無動態庫加載開銷
- 更好的內聯優化 (配合 LTO)
- 減少依賴問題
- 二進制文件更大 (可接受)

---

## 3.6 CMake 構建系統

### 3.6.1 基本 CMakeLists.txt

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.15)
project(HFTSystem VERSION 1.0 LANGUAGES CXX)

# C++ 標準
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 編譯選項
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -march=native -flto")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -flto")
elseif(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O0 -g")
endif()

# 可執行文件
add_executable(trading_system
    src/main.cpp
    src/order_book.cpp
    src/strategy.cpp
)

# 包含目錄
target_include_directories(trading_system PRIVATE include)

# 鏈接庫
target_link_libraries(trading_system PRIVATE pthread)
```

### 3.6.2 進階 CMake

```cmake
# 庫
add_library(order_book STATIC
    src/order_book.cpp
    src/market_data.cpp
)

target_include_directories(order_book PUBLIC include)

# 主程序
add_executable(trading_system src/main.cpp)
target_link_libraries(trading_system PRIVATE order_book pthread)

# 測試
enable_testing()
add_executable(order_book_test test/order_book_test.cpp)
target_link_libraries(order_book_test PRIVATE order_book gtest gtest_main)
add_test(NAME order_book_test COMMAND order_book_test)

# 安裝
install(TARGETS trading_system DESTINATION bin)
install(DIRECTORY include/ DESTINATION include)

# 編譯選項 (更細粒度)
target_compile_options(trading_system PRIVATE
    -Wall -Wextra -Wpedantic
    $<$<CONFIG:Release>:-O3 -march=native -flto>
    $<$<CONFIG:Debug>:-O0 -g -fsanitize=address>
)

target_link_options(trading_system PRIVATE
    $<$<CONFIG:Debug>:-fsanitize=address>
)
```

### 3.6.3 使用 CMake

```bash
# 配置 (Release)
cmake -B build -DCMAKE_BUILD_TYPE=Release

# 編譯
cmake --build build -j$(nproc)

# 安裝
cmake --install build --prefix /usr/local

# 配置 (Debug + AddressSanitizer)
cmake -B build_debug -DCMAKE_BUILD_TYPE=Debug

# 清理
rm -rf build
```

### 3.6.4 ccache 加速

```bash
# 安裝 ccache
sudo apt install ccache

# 在 CMakeLists.txt 中啟用
find_program(CCACHE_PROGRAM ccache)
if(CCACHE_PROGRAM)
    set(CMAKE_CXX_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
    set(CMAKE_C_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
endif()

# 或通過環境變量
export CC="ccache gcc"
export CXX="ccache g++"
cmake -B build

# 查看統計
ccache -s
```

---

## 實作練習

### 練習 1: 優化等級對比

```cpp
// optimize_benchmark.cpp
#include <chrono>
#include <iostream>
#include <vector>

double compute_expensive() {
    double sum = 0;
    for (int i = 0; i < 10000000; ++i) {
        sum += std::sqrt(i) * std::sin(i);
    }
    return sum;
}

int main() {
    auto start = std::chrono::high_resolution_clock::now();
    
    volatile double result = compute_expensive();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "Time: " << duration.count() << " ms\n";
    return 0;
}
```

```bash
# 測試
g++ -O0 optimize_benchmark.cpp -o bench_O0 && time ./bench_O0
g++ -O2 optimize_benchmark.cpp -o bench_O2 && time ./bench_O2
g++ -O3 optimize_benchmark.cpp -o bench_O3 && time ./bench_O3
g++ -O3 -march=native optimize_benchmark.cpp -o bench_native && time ./bench_native
```

### 練習 2: LTO 效果測試

```cpp
// module1.cpp
int calculate(int n) {
    int sum = 0;
    for (int i = 0; i < n; ++i) {
        sum += i;
    }
    return sum;
}

// module2.cpp
extern int calculate(int);

int main() {
    return calculate(1000);
}
```

```bash
# 無 LTO
g++ -O3 -c module1.cpp -o module1.o
g++ -O3 -c module2.cpp -o module2.o
g++ module1.o module2.o -o program

# 有 LTO
g++ -O3 -flto -c module1.cpp -o module1_lto.o
g++ -O3 -flto -c module2.cpp -o module2_lto.o
g++ -O3 -flto module1_lto.o module2_lto.o -o program_lto

# 比較大小
ls -lh program program_lto

# 性能測試
time ./program
time ./program_lto
```

### 練習 3: 分支預測優化

```cpp
#include <chrono>
#include <iostream>
#include <random>

#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)

int process_without_hint(int x) {
    if (x % 100 == 0) {  // 1% 機率
        return x * 2;
    }
    return x + 1;
}

int process_with_hint(int x) {
    if (unlikely(x % 100 == 0)) {
        return x * 2;
    }
    return x + 1;
}

template<typename Func>
void benchmark(const char* name, Func f) {
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> dis(0, 999);
    
    auto start = std::chrono::high_resolution_clock::now();
    
    volatile int sum = 0;
    for (int i = 0; i < 100000000; ++i) {
        sum += f(dis(gen));
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << name << ": " << duration.count() << " ms\n";
}

int main() {
    benchmark("Without hint", process_without_hint);
    benchmark("With hint", process_with_hint);
    return 0;
}
```

### 練習 4: 完整 CMake 項目

```
project/
├── CMakeLists.txt
├── include/
│   └── order_book.h
├── src/
│   ├── main.cpp
│   └── order_book.cpp
└── test/
    └── order_book_test.cpp
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.15)
project(TradingSystem CXX)

set(CMAKE_CXX_STANDARD 20)

# 優化選項
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    add_compile_options(-O3 -march=native -flto)
    add_link_options(-flto)
endif()

# 庫
add_library(order_book src/order_book.cpp)
target_include_directories(order_book PUBLIC include)

# 可執行文件
add_executable(trading src/main.cpp)
target_link_libraries(trading PRIVATE order_book)

# 測試
enable_testing()
find_package(GTest)
if(GTest_FOUND)
    add_executable(test_order_book test/order_book_test.cpp)
    target_link_libraries(test_order_book PRIVATE order_book GTest::gtest_main)
    add_test(NAME test_order_book COMMAND test_order_book)
endif()
```

---

### 練習 5: 使用 Compiler Explorer 分析彙編輸出

**任務:** 使用 Compiler Explorer (godbolt.org) 分析不同優化級別下的彙編輸出差異。

```cpp
// 分析以下函數在 -O0, -O2, -O3 下的彙編差異
int sum_array(const int* arr, int n) {
    int sum = 0;
    for (int i = 0; i < n; ++i) {
        sum += arr[i];
    }
    return sum;
}

// 啟用 auto-vectorization 後的版本
// 觀察是否生成 SIMD 指令 (如 vpaddq, vpaddd)
int sum_array_unrolled(const int* arr, int n) {
    int sum = 0;
    int i = 0;
    
    // 手動展開
    for (; i + 4 <= n; i += 4) {
        sum += arr[i] + arr[i+1] + arr[i+2] + arr[i+3];
    }
    
    for (; i < n; ++i) {
        sum += arr[i];
    }
    
    return sum;
}
```

**觀察要點:**
1. 循環展開 (Loop Unrolling)
2. SIMD 向量化
3. 寄存器分配
4. 內聯展開

---

### 練習 6: Profile-Guided Optimization (PGO)

**任務:** 實作 PGO 並測量性能提升。

```cpp
// pgo_benchmark.cpp
#include <iostream>
#include <vector>
#include <random>
#include <chrono>

// 模擬訂單處理
enum class OrderType { MARKET, LIMIT, STOP, STOP_LIMIT };

struct Order {
    OrderType type;
    double price;
    int quantity;
};

void process_order(const Order& order) {
    // 不同訂單類型的處理路徑
    switch (order.type) {
        case OrderType::LIMIT:
            // 最常見: 80%
            // 模擬處理
            volatile double x = order.price * order.quantity;
            break;
        case OrderType::MARKET:
            // 次常見: 15%
            volatile double y = order.quantity;
            break;
        case OrderType::STOP:
        case OrderType::STOP_LIMIT:
            // 罕見: 5%
            volatile double z = order.price;
            break;
    }
}

int main() {
    // 生成符合真實分佈的測試數據
    std::vector<Order> orders;
    std::mt19937 gen(42);
    std::uniform_real_distribution<> dist(0, 1);
    
    for (int i = 0; i < 10000000; ++i) {
        Order order;
        double r = dist(gen);
        
        if (r < 0.80) {
            order.type = OrderType::LIMIT;
        } else if (r < 0.95) {
            order.type = OrderType::MARKET;
        } else {
            order.type = r < 0.975 ? OrderType::STOP : OrderType::STOP_LIMIT;
        }
        
        order.price = 100.0 + dist(gen) * 10;
        order.quantity = 100 + gen() % 900;
        orders.push_back(order);
    }
    
    auto start = std::chrono::high_resolution_clock::now();
    
    for (const auto& order : orders) {
        process_order(order);
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "Time: " << duration.count() << " ms\n";
    return 0;
}
```

**PGO 步驟:**

```bash
# 1. 生成 instrumented binary
g++ -O3 -fprofile-generate pgo_benchmark.cpp -o pgo_gen

# 2. 運行收集 profile 數據
./pgo_gen

# 3. 使用 profile 數據重新編譯
g++ -O3 -fprofile-use pgo_benchmark.cpp -o pgo_opt

# 4. 對比性能
time ./pgo_gen
time ./pgo_opt
```

---

### 練習 7: 函數屬性優化

**任務:** 使用各種函數屬性優化關鍵路徑。

```cpp
#include <iostream>
#include <cmath>

// 1. hot/cold 屬性
__attribute__((hot))
double critical_calculation(double x) {
    return std::sqrt(x) * std::log(x);
}

__attribute__((cold))
void log_error(const char* msg) {
    std::cerr << "Error: " << msg << std::endl;
}

// 2. pure/const 屬性 (允許更激進優化)
__attribute__((pure))
int calculate_fee(int quantity, int rate) {
    return quantity * rate / 100;
}

__attribute__((const))
int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

// 3. flatten 屬性 (內聯所有調用)
__attribute__((flatten))
double process_data(double* data, int n) {
    double sum = 0;
    for (int i = 0; i < n; ++i) {
        sum += critical_calculation(data[i]);
    }
    return sum;
}

// 4. noinline (阻止內聯,用於調試)
__attribute__((noinline))
void debug_checkpoint(int line) {
    // 調試檢查點
}

// 5. aligned (對齊優化)
struct __attribute__((aligned(64))) CacheAlignedData {
    double values[8];
};

int main() {
    double data[] = {1.0, 2.0, 3.0, 4.0, 5.0};
    
    double result = process_data(data, 5);
    int fee = calculate_fee(1000, 5);
    
    std::cout << "Result: " << result << "\n";
    std::cout << "Fee: " << fee << "\n";
    std::cout << "5! = " << factorial(5) << "\n";
    
    return 0;
}
```

---

## 關鍵要點總結

### 編譯優化檢查清單 (HFT)

**編譯選項:**
- [ ] `-O3` (激進優化)
- [ ] `-march=native` (CPU 特定指令)
- [ ] `-flto` (鏈接時優化)
- [ ] `-DNDEBUG` (禁用 assert)
- [ ] `-fno-exceptions` (可選,禁用異常)
- [ ] `-fno-rtti` (可選,禁用 RTTI)

**代碼優化:**
- [ ] 使用 `likely`/`unlikely` 提示熱路徑
- [ ] 關鍵函數標記 `__attribute__((always_inline))`
- [ ] 使用 `__builtin_prefetch` 預取數據
- [ ] `[[nodiscard]]` 避免忽略重要返回值

**構建策略:**
- [ ] 靜態鏈接 (避免動態庫開銷)
- [ ] PGO (使用真實數據優化)
- [ ] ccache 加速迭代
- [ ] 分離調試符號 (`-g -Og` for debug, strip for release)

### 優化效果預估

| 技術 | 性能提升 | 編譯時間 | 適用場景 |
|------|----------|----------|----------|
| `-O3` vs `-O0` | 3-10x | +20% | 所有 |
| `-march=native` | 5-20% | +0% | CPU 密集 |
| LTO | 5-15% | +50-100% | 多模塊 |
| PGO | 5-10% | +100% | 分支預測 |
| `likely`/`unlikely` | 1-5% | +0% | 熱路徑 |

---

## 參考資料 (References)

1. [GCC Optimization Options](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)
2. [Clang Optimization Flags](https://clang.llvm.org/docs/CommandGuide/clang.html#optimization-options)
3. [CMake Documentation](https://cmake.org/documentation/)
4. [Link Time Optimization](https://gcc.gnu.org/wiki/LinkTimeOptimization)
5. [Profile-Guided Optimization](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html#index-fprofile-generate)
6. Agner Fog, [Optimizing software in C++](https://www.agner.org/optimize/optimizing_cpp.pdf)
7. [Compiler Explorer (godbolt.org)](https://godbolt.org/) - 在線查看編譯器輸出
8. [Build Systems à la Carte](https://www.microsoft.com/en-us/research/uploads/prod/2018/03/build-systems.pdf)
