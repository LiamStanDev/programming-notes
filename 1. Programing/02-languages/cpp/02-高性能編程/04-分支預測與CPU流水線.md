# 分支預測與 CPU 流水線 (Branch Prediction & CPU Pipeline)

## 概述

現代 CPU 使用流水線 (Pipeline) 並行執行指令:
- **指令獲取 (Fetch)**: 從記憶體讀取指令
- **指令解碼 (Decode)**: 解析指令
- **執行 (Execute)**: 執行運算
- **記憶體訪問 (Memory)**: 讀寫數據
- **寫回 (Write Back)**: 更新暫存器

**問題**: 分支指令 (if, loop) 會打斷流水線

**解決**: 分支預測器 (Branch Predictor) 猜測分支方向

在高頻交易中,分支預測失敗 (Misprediction) 會導致:
- 流水線停頓 10-20 個週期 (~3-10ns)
- 吞吐量下降

---

## 分支預測基礎

### 分支代價

```cpp
// 無分支
int abs_branchless(int x) {
    int mask = x >> 31;  // 全 1 (負數) 或全 0 (非負數)
    return (x + mask) ^ mask;
}
// 性能: ~1 cycle

// 有分支
int abs_branched(int x) {
    if (x < 0) {
        return -x;
    }
    return x;
}
// 性能: 1 cycle (預測正確) or 15-20 cycles (預測錯誤)
```

### 預測器類型

1. **靜態預測**: 編譯器提示
2. **動態預測**: CPU 硬體學習

```cpp
// 靜態預測: 編譯器假設 forward branch 不跳轉
if (unlikely_condition) {  // 很少為真
    // cold path
} else {
    // hot path (預測執行這裡)
}

// C++20: 標準屬性
if (condition) [[likely]] {
    // hot path
} else [[unlikely]] {
    // cold path
}
```

---

## 可預測 vs 不可預測分支

### 可預測模式

```cpp
// ✅ 高度可預測: 總是相同
for (int i = 0; i < 1000; ++i) {
    if (i < 900) {  // 前 900 次為真,預測準確率 ~90%
        // hot path
    }
}

// ✅ 規律模式: 重複的模式
for (int i = 0; i < 1000; ++i) {
    if (i % 2 == 0) {  // 01010101... 模式,可學習
        // even
    } else {
        // odd
    }
}
```

### 不可預測模式

```cpp
// ❌ 隨機數據: 無法預測
std::vector<int> data = generate_random();
for (int x : data) {
    if (x > 50) {  // 50% 概率,完全隨機
        // 預測準確率 ~50%
        // 每次失敗損失 15-20 cycles
    }
}

// ❌ 罕見條件: 低頻但無規律
for (int i = 0; i < 1000000; ++i) {
    if (is_error(data[i])) {  // 0.1% 概率
        handle_error();
        // 預測失敗時付出代價
    }
}
```

---

## 消除分支技巧

### 1. 使用算術替代分支

```cpp
// ❌ 有分支
int max_branched(int a, int b) {
    if (a > b) {
        return a;
    } else {
        return b;
    }
}

// ✅ 無分支
int max_branchless(int a, int b) {
    return a > b ? a : b;  // 編譯器可能優化為 CMOV
}

// 或顯式使用條件移動
int max_cmov(int a, int b) {
    int result = b;
    __asm__("cmpl %1, %0\n\t"
            "cmovg %0, %2"
            : "=r"(result)
            : "r"(a), "r"(b));
    return result;
}
```

### 2. 查找表 (Lookup Table)

```cpp
// ❌ 多分支
char to_upper_branched(char c) {
    if (c >= 'a' && c <= 'z') {
        return c - 32;
    }
    return c;
}

// ✅ 查找表 (編譯期生成)
constexpr std::array<char, 256> make_upper_table() {
    std::array<char, 256> table{};
    for (int i = 0; i < 256; ++i) {
        table[i] = (i >= 'a' && i <= 'z') ? i - 32 : i;
    }
    return table;
}

constexpr auto UPPER_TABLE = make_upper_table();

char to_upper_lut(char c) {
    return UPPER_TABLE[static_cast<unsigned char>(c)];  // 無分支
}
```

### 3. 位運算技巧

```cpp
// 檢查奇偶
bool is_even_branched(int x) {
    if (x % 2 == 0) return true;
    return false;
}

bool is_even_branchless(int x) {
    return (x & 1) == 0;  // 位運算,無分支
}

// 符號判斷
int sign_branched(int x) {
    if (x < 0) return -1;
    if (x > 0) return 1;
    return 0;
}

int sign_branchless(int x) {
    return (x > 0) - (x < 0);  // 布爾值算術
}

// 絕對值
int abs_branchless(int x) {
    int mask = x >> 31;
    return (x + mask) ^ mask;
}
```

### 4. 選擇運算 (Conditional Select)

```cpp
// ❌ 分支
int clamp_branched(int x, int min, int max) {
    if (x < min) return min;
    if (x > max) return max;
    return x;
}

// ✅ 無分支
int clamp_branchless(int x, int min, int max) {
    x = x < min ? min : x;  // CMOV
    x = x > max ? max : x;  // CMOV
    return x;
}

// 或使用標準庫
int clamp_std(int x, int min, int max) {
    return std::clamp(x, min, max);  // 通常無分支實現
}
```

---

## 分支預測提示

### C++20 [[likely]] / [[unlikely]]

```cpp
// 標記熱路徑
int process_order(const Order& order) {
    if (order.is_valid()) [[likely]] {
        // 99% 的訂單有效
        return execute(order);
    } else [[unlikely]] {
        // 1% 的訂單無效
        return reject(order);
    }
}

// switch 語句
switch (msg_type) {
    [[likely]] case MsgType::MARKET_DATA:
        process_market_data(msg);
        break;
    
    [[unlikely]] case MsgType::ERROR:
        handle_error(msg);
        break;
        
    default:
        // ...
}
```

### 編譯器內建函數

```cpp
// GCC/Clang
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)

// 使用
if (likely(order.quantity > 0)) {
    // hot path
} else {
    // cold path
}

// 注意: 錯誤使用會降低性能!
if (unlikely(random_condition)) {  // ❌ 實際 50% 概率
    // 錯誤的提示導致預測準確率下降
}
```

---

## 循環優化

### 循環展開減少分支

```cpp
// ❌ 每次迭代一次分支
for (int i = 0; i < n; ++i) {
    array[i] *= 2;
}
// 分支次數: n

// ✅ 循環展開
for (int i = 0; i < n; i += 4) {
    array[i]   *= 2;
    array[i+1] *= 2;
    array[i+2] *= 2;
    array[i+3] *= 2;
}
// 分支次數: n/4

// 編譯器提示
#pragma GCC unroll 4
for (int i = 0; i < n; ++i) {
    array[i] *= 2;
}
```

### 循環合併

```cpp
// ❌ 多個循環
for (int i = 0; i < n; ++i) {
    a[i] = b[i] + 1;
}
for (int i = 0; i < n; ++i) {
    c[i] = a[i] * 2;
}
// 分支次數: 2n

// ✅ 合併循環
for (int i = 0; i < n; ++i) {
    a[i] = b[i] + 1;
    c[i] = a[i] * 2;
}
// 分支次數: n
```

---

## 高頻交易案例

### 案例 1: 訂單驗證

```cpp
// ❌ 多重分支
bool validate_order_branched(const Order& order) {
    if (order.price <= 0) {
        return false;
    }
    if (order.quantity <= 0) {
        return false;
    }
    if (order.quantity > MAX_QUANTITY) {
        return false;
    }
    return true;
}

// ✅ 無分支 (位運算組合)
bool validate_order_branchless(const Order& order) {
    bool valid_price = order.price > 0;
    bool valid_quantity = order.quantity > 0 && order.quantity <= MAX_QUANTITY;
    return valid_price & valid_quantity;  // 位與,非邏輯與 (避免短路)
}

// 或使用條件組合
bool validate_order_combined(const Order& order) {
    return (order.price > 0) &
           (order.quantity > 0) &
           (order.quantity <= MAX_QUANTITY);
}
```

### 案例 2: 價格過濾

```cpp
// ❌ 不可預測分支
std::vector<Order> filter_by_price_branched(
    const std::vector<Order>& orders, double threshold) {
    
    std::vector<Order> result;
    for (const auto& order : orders) {
        if (order.price > threshold) {  // 不可預測
            result.push_back(order);
        }
    }
    return result;
}

// ✅ 分支消除 + 批量處理
std::vector<Order> filter_by_price_branchless(
    const std::vector<Order>& orders, double threshold) {
    
    std::vector<Order> result;
    result.reserve(orders.size());  // 預分配
    
    for (const auto& order : orders) {
        // 無分支: 使用條件賦值
        result.resize(result.size() + (order.price > threshold));
        if (order.price > threshold) {
            result.back() = order;
        }
    }
    return result;
}

// ✅✅ 更好: 使用 SIMD (後續章節)
```

### 案例 3: 訂單簿更新

```cpp
class OrderBook {
    static constexpr int NUM_LEVELS = 1000;
    std::array<uint32_t, NUM_LEVELS> bids_;
    
public:
    // ❌ 條件更新
    void update_bid_branched(int level, uint32_t quantity) {
        if (quantity > 0) {
            bids_[level] = quantity;
        } else {
            bids_[level] = 0;
        }
    }
    
    // ✅ 無分支
    void update_bid_branchless(int level, uint32_t quantity) {
        // 利用 quantity > 0 的布爾值
        bids_[level] = quantity * (quantity > 0);
    }
    
    // 或使用條件移動
    void update_bid_cmov(int level, uint32_t quantity) {
        uint32_t value = (quantity > 0) ? quantity : 0;
        bids_[level] = value;  // 編譯器生成 CMOV
    }
};
```

---

## 性能測量

### 使用 perf 測量分支預測

```bash
# 測量分支預測統計
perf stat -e branches,branch-misses ./program

# 輸出:
#   1,234,567 branches
#      12,345 branch-misses  # 1% miss rate (好)

# 詳細分支分析
perf record -e branches ./program
perf report
```

### Benchmark 對比

```cpp
#include <benchmark/benchmark.h>
#include <random>

// 可預測模式
static void BM_PredictableBranch(benchmark::State& state) {
    std::vector<int> data(10000, 50);  // 全部 > 0
    
    for (auto _ : state) {
        int sum = 0;
        for (int x : data) {
            if (x > 0) {  // 100% 預測正確
                sum += x;
            }
        }
        benchmark::DoNotOptimize(sum);
    }
}

// 不可預測模式
static void BM_UnpredictableBranch(benchmark::State& state) {
    std::vector<int> data(10000);
    std::mt19937 gen;
    std::uniform_int_distribution<> dis(-100, 100);
    for (auto& x : data) x = dis(gen);  // 隨機
    
    for (auto _ : state) {
        int sum = 0;
        for (int x : data) {
            if (x > 0) {  // 50% 預測正確
                sum += x;
            }
        }
        benchmark::DoNotOptimize(sum);
    }
}

// 無分支版本
static void BM_Branchless(benchmark::State& state) {
    std::vector<int> data(10000);
    std::mt19937 gen;
    std::uniform_int_distribution<> dis(-100, 100);
    for (auto& x : data) x = dis(gen);
    
    for (auto _ : state) {
        int sum = 0;
        for (int x : data) {
            sum += x * (x > 0);  // 無分支
        }
        benchmark::DoNotOptimize(sum);
    }
}

BENCHMARK(BM_PredictableBranch);     // ~100ns
BENCHMARK(BM_UnpredictableBranch);   // ~1000ns (10x 慢)
BENCHMARK(BM_Branchless);            // ~200ns (穩定)
```

---

## 流水線停頓與依賴

### 數據依賴

```cpp
// ❌ 依賴鏈長
int dependent_chain(int x) {
    int a = x * 2;
    int b = a + 3;     // 依賴 a
    int c = b * 4;     // 依賴 b
    int d = c - 1;     // 依賴 c
    return d;
}
// 無法並行,流水線利用率低

// ✅ 獨立運算
int independent_ops(int x, int y, int z, int w) {
    int a = x * 2;     // 獨立
    int b = y + 3;     // 獨立
    int c = z * 4;     // 獨立
    int d = w - 1;     // 獨立
    return a + b + c + d;
}
// 可並行執行,流水線利用率高
```

### 指令級並行 (ILP)

```cpp
// ❌ 低 ILP
void process_array_sequential(float* data, size_t n) {
    float sum = 0;
    for (size_t i = 0; i < n; ++i) {
        sum += data[i];  // 依賴鏈
    }
}

// ✅ 高 ILP: 展開 + 多累加器
void process_array_parallel(float* data, size_t n) {
    float sum1 = 0, sum2 = 0, sum3 = 0, sum4 = 0;
    
    for (size_t i = 0; i < n; i += 4) {
        sum1 += data[i];      // 獨立
        sum2 += data[i+1];    // 獨立
        sum3 += data[i+2];    // 獨立
        sum4 += data[i+3];    // 獨立
    }
    
    float sum = sum1 + sum2 + sum3 + sum4;
}
// 性能提升: 2-4x
```

---

## 編譯器生成的匯編

### 查看分支指令

```cpp
int max(int a, int b) {
    return a > b ? a : b;
}
```

```bash
# 生成匯編
g++ -O2 -S -masm=intel max.cpp

# 查看匯編 (Intel 語法)
cat max.s
```

```asm
; 可能的輸出 (使用 CMOV)
max(int, int):
    mov     eax, edi      ; eax = a
    cmp     edi, esi      ; 比較 a 和 b
    cmovle  eax, esi      ; if (a <= b) eax = b
    ret
; 無分支! 使用條件移動指令
```

---

## 實戰檢查清單

### 分支優化

- [ ] 使用 [[likely]]/[[unlikely]] 標記分支
- [ ] 消除熱路徑中的不可預測分支
- [ ] 使用查找表替代多分支 switch
- [ ] 考慮無分支算術 (bit tricks)

### 循環優化

- [ ] 循環展開減少分支次數
- [ ] 循環合併改善局部性
- [ ] 多累加器提高 ILP
- [ ] 檢查循環是否被向量化

### 測量與驗證

- [ ] 使用 perf 測量 branch-misses
- [ ] Benchmark 可預測 vs 不可預測
- [ ] 查看生成的匯編代碼
- [ ] Profile 熱點分支

### 高頻交易特定

- [ ] 訂單驗證使用位運算組合
- [ ] 避免訂單簿更新中的分支
- [ ] 批量處理減少分支密度
- [ ] 錯誤處理標記為 [[unlikely]]

---

## 參考資料 (References)

1. [Intel Optimization Manual - Branch Prediction](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
2. [Why is processing a sorted array faster than processing an unsorted array?](https://stackoverflow.com/questions/11227809/)
3. Fog, Agner. *Optimizing software in C++*. Chapter on branch prediction.
4. [Performance Analysis with perf](https://perf.wiki.kernel.org/)
5. Lemire, Daniel. [Branch mispredictions don't happen in a vacuum](https://lemire.me/blog/)
