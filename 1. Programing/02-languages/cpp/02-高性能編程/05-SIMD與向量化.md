# SIMD 與向量化 (SIMD & Vectorization)

## 概述

SIMD (Single Instruction, Multiple Data) 允許單一指令處理多個數據:
- **標量運算**: 一次處理一個數據
- **向量運算**: 一次處理 4/8/16 個數據

**x86 SIMD 指令集進化**:
- **SSE**: 128-bit (4 × float 或 2 × double)
- **AVX**: 256-bit (8 × float 或 4 × double)
- **AVX-512**: 512-bit (16 × float 或 8 × double)

在高頻交易中,SIMD 能:
- 批量處理市場數據 (4-16x 吞吐量)
- 並行計算價格/統計指標
- 加速訂單簿操作

---

## SIMD 基礎

### 內建類型 (Intrinsics)

```cpp
#include <immintrin.h>  // Intel intrinsics

// SSE: 128-bit = 4 × 32-bit float
__m128 sse_vec = _mm_set_ps(1.0f, 2.0f, 3.0f, 4.0f);

// AVX: 256-bit = 8 × 32-bit float
__m256 avx_vec = _mm256_set_ps(1.0f, 2.0f, 3.0f, 4.0f,
                                5.0f, 6.0f, 7.0f, 8.0f);

// AVX-512: 512-bit = 16 × 32-bit float
__m512 avx512_vec = _mm512_set_ps(/* 16 個 float */);
```

### 基本運算

```cpp
// 向量加法
__m256 a = _mm256_set1_ps(1.0f);  // {1, 1, 1, 1, 1, 1, 1, 1}
__m256 b = _mm256_set1_ps(2.0f);  // {2, 2, 2, 2, 2, 2, 2, 2}
__m256 c = _mm256_add_ps(a, b);   // {3, 3, 3, 3, 3, 3, 3, 3}

// 向量乘法
__m256 d = _mm256_mul_ps(a, b);   // {2, 2, 2, 2, 2, 2, 2, 2}

// 向量減法
__m256 e = _mm256_sub_ps(b, a);   // {1, 1, 1, 1, 1, 1, 1, 1}

// 向量除法
__m256 f = _mm256_div_ps(b, a);   // {2, 2, 2, 2, 2, 2, 2, 2}
```

---

## 載入與存儲

### 對齊載入 vs 未對齊載入

```cpp
// 對齊載入 (更快,要求 32-byte 對齊)
alignas(32) float aligned_data[8] = {1, 2, 3, 4, 5, 6, 7, 8};
__m256 vec = _mm256_load_ps(aligned_data);

// 未對齊載入 (較慢,但無對齊要求)
float unaligned_data[8] = {1, 2, 3, 4, 5, 6, 7, 8};
__m256 vec2 = _mm256_loadu_ps(unaligned_data);

// 存儲
alignas(32) float result[8];
_mm256_store_ps(result, vec);  // 對齊存儲

float result2[8];
_mm256_storeu_ps(result2, vec);  // 未對齊存儲
```

### 廣播 (Broadcast)

```cpp
// 將單一值複製到所有位置
float value = 3.14f;
__m256 broadcast = _mm256_set1_ps(value);
// {3.14, 3.14, 3.14, 3.14, 3.14, 3.14, 3.14, 3.14}

// 或使用 broadcast_ss
__m256 broadcast2 = _mm256_broadcast_ss(&value);
```

---

## 常見模式

### 1. 向量化循環

```cpp
// ❌ 標量版本
void add_arrays_scalar(const float* a, const float* b, float* c, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}

// ✅ AVX 向量化
void add_arrays_simd(const float* a, const float* b, float* c, size_t n) {
    size_t i = 0;
    
    // 處理 8 的倍數
    for (; i + 8 <= n; i += 8) {
        __m256 va = _mm256_loadu_ps(&a[i]);
        __m256 vb = _mm256_loadu_ps(&b[i]);
        __m256 vc = _mm256_add_ps(va, vb);
        _mm256_storeu_ps(&c[i], vc);
    }
    
    // 處理剩餘元素
    for (; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}
// 性能提升: ~8x
```

### 2. 水平求和 (Horizontal Sum)

```cpp
// 將向量中的所有元素相加
float horizontal_sum_avx(__m256 vec) {
    // vec = {a, b, c, d, e, f, g, h}
    
    // 步驟 1: 將高低 128 位相加
    __m128 low = _mm256_castps256_ps128(vec);          // {a, b, c, d}
    __m128 high = _mm256_extractf128_ps(vec, 1);       // {e, f, g, h}
    __m128 sum128 = _mm_add_ps(low, high);             // {a+e, b+f, c+g, d+h}
    
    // 步驟 2: 水平加法
    __m128 shuf = _mm_movehdup_ps(sum128);             // {b+f, b+f, d+h, d+h}
    __m128 sum64 = _mm_add_ps(sum128, shuf);           // {a+e+b+f, *, c+g+d+h, *}
    
    shuf = _mm_movehl_ps(shuf, sum64);                 // {c+g+d+h, *}
    __m128 result = _mm_add_ss(sum64, shuf);
    
    return _mm_cvtss_f32(result);
}

// 使用案例
float sum_array_simd(const float* data, size_t n) {
    __m256 acc = _mm256_setzero_ps();  // 累加器
    
    for (size_t i = 0; i + 8 <= n; i += 8) {
        __m256 vec = _mm256_loadu_ps(&data[i]);
        acc = _mm256_add_ps(acc, vec);
    }
    
    return horizontal_sum_avx(acc);  // 最後合併
}
```

### 3. 條件運算 (Masking)

```cpp
// 使用掩碼進行條件運算
void clamp_simd(float* data, size_t n, float min_val, float max_val) {
    __m256 vmin = _mm256_set1_ps(min_val);
    __m256 vmax = _mm256_set1_ps(max_val);
    
    for (size_t i = 0; i + 8 <= n; i += 8) {
        __m256 vec = _mm256_loadu_ps(&data[i]);
        
        // vec = max(vec, vmin)
        vec = _mm256_max_ps(vec, vmin);
        
        // vec = min(vec, vmax)
        vec = _mm256_min_ps(vec, vmax);
        
        _mm256_storeu_ps(&data[i], vec);
    }
}
```

---

## 高頻交易應用

### 1. VWAP 計算 (成交量加權平均價)

```cpp
// 標量版本
double calculate_vwap_scalar(const double* prices, const uint64_t* volumes, size_t n) {
    double total_value = 0.0;
    uint64_t total_volume = 0;
    
    for (size_t i = 0; i < n; ++i) {
        total_value += prices[i] * volumes[i];
        total_volume += volumes[i];
    }
    
    return total_value / total_volume;
}

// AVX 向量化 (double)
double calculate_vwap_simd(const double* prices, const uint64_t* volumes, size_t n) {
    __m256d acc_value = _mm256_setzero_pd();  // 4 × double
    __m256i acc_volume = _mm256_setzero_si256();  // 4 × int64
    
    for (size_t i = 0; i + 4 <= n; i += 4) {
        __m256d vprice = _mm256_loadu_pd(&prices[i]);
        
        // 載入並轉換 volume (int64 -> double)
        __m256i vvolume_int = _mm256_loadu_si256((__m256i*)&volumes[i]);
        __m256d vvolume = _mm256_cvtepi64_pd(vvolume_int);
        
        // price * volume
        __m256d vvalue = _mm256_mul_pd(vprice, vvolume);
        
        // 累加
        acc_value = _mm256_add_pd(acc_value, vvalue);
        acc_volume = _mm256_add_epi64(acc_volume, vvolume_int);
    }
    
    // 水平求和
    double total_value = horizontal_sum_pd(acc_value);
    uint64_t total_volume = horizontal_sum_epi64(acc_volume);
    
    return total_value / total_volume;
}
// 性能提升: ~4x
```

### 2. 批量訂單驗證

```cpp
// 驗證 8 個訂單的價格是否在範圍內
struct Order {
    double price;
    uint32_t quantity;
};

void validate_prices_simd(const Order* orders, bool* valid, size_t n,
                          double min_price, double max_price) {
    __m256d vmin = _mm256_set1_pd(min_price);
    __m256d vmax = _mm256_set1_pd(max_price);
    
    for (size_t i = 0; i + 4 <= n; i += 4) {
        // 載入價格 (假設 Order 佈局允許)
        __m256d prices = _mm256_set_pd(
            orders[i+3].price, orders[i+2].price,
            orders[i+1].price, orders[i].price
        );
        
        // 檢查範圍
        __m256d ge_min = _mm256_cmp_pd(prices, vmin, _CMP_GE_OQ);  // >= min
        __m256d le_max = _mm256_cmp_pd(prices, vmax, _CMP_LE_OQ);  // <= max
        
        // 合併條件
        __m256d mask = _mm256_and_pd(ge_min, le_max);
        
        // 轉換為整數掩碼
        int result = _mm256_movemask_pd(mask);
        
        // 存儲結果
        valid[i]   = (result & 1) != 0;
        valid[i+1] = (result & 2) != 0;
        valid[i+2] = (result & 4) != 0;
        valid[i+3] = (result & 8) != 0;
    }
}
```

### 3. 價格層級掃描

```cpp
// 在訂單簿中找到第一個非零數量的價格層級
class OrderBook {
    static constexpr size_t NUM_LEVELS = 1024;
    alignas(32) std::array<uint32_t, NUM_LEVELS> quantities_;
    
public:
    // AVX2: 掃描 8 個 int32
    std::optional<size_t> find_first_nonzero() const {
        __m256i zero = _mm256_setzero_si256();
        
        for (size_t i = 0; i < NUM_LEVELS; i += 8) {
            __m256i levels = _mm256_load_si256((__m256i*)&quantities_[i]);
            
            // 比較是否不等於 0
            __m256i cmp = _mm256_cmpeq_epi32(levels, zero);
            
            // 取得掩碼 (bit 位表示相等)
            int mask = _mm256_movemask_epi8(cmp);
            
            // 若不是全 0,找到第一個非零
            if (mask != 0xFFFFFFFF) {
                // 找到第一個 0 bit (表示非零元素)
                int first_zero = __builtin_ctz(~mask) / 4;
                return i + first_zero;
            }
        }
        
        return std::nullopt;
    }
};
```

---

## 自動向量化

### 編譯器自動向量化

```cpp
// 簡單循環,編譯器可能自動向量化
void simple_loop(const float* a, const float* b, float* c, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}

// 編譯選項
// g++ -O3 -march=native -ftree-vectorize program.cpp
```

```bash
# 查看向量化報告
g++ -O3 -march=native -fopt-info-vec program.cpp

# 輸出示例:
# program.cpp:5:5: optimized: loop vectorized using 32 byte vectors
```

### 阻礙自動向量化的因素

```cpp
// ❌ 函數調用
for (size_t i = 0; i < n; ++i) {
    c[i] = expensive_function(a[i]);  // 無法向量化
}

// ❌ 複雜控制流
for (size_t i = 0; i < n; ++i) {
    if (a[i] > threshold) {
        c[i] = a[i] * 2;
    } else if (a[i] < -threshold) {
        c[i] = a[i] / 2;
    } else {
        c[i] = 0;
    }
}

// ❌ 數據依賴
for (size_t i = 1; i < n; ++i) {
    c[i] = c[i-1] + a[i];  // 依賴前一個元素
}

// ✅ 簡單,可向量化
for (size_t i = 0; i < n; ++i) {
    c[i] = a[i] * 2.0f + b[i];  // 獨立運算
}
```

### 幫助編譯器向量化

```cpp
// 1. 使用 __restrict (告訴編譯器無別名)
void add_arrays(const float* __restrict a, 
                const float* __restrict b,
                float* __restrict c, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}

// 2. 明確循環界限
void process(float* data) {
    for (size_t i = 0; i < 1024; ++i) {  // 常量界限
        data[i] *= 2.0f;
    }
}

// 3. 對齊提示
void process_aligned(float* __attribute__((aligned(32))) data, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        data[i] *= 2.0f;
    }
}

// 4. pragma 提示
void process_with_pragma(float* data, size_t n) {
    #pragma GCC ivdep  // 忽略向量依賴
    for (size_t i = 0; i < n; ++i) {
        data[i] *= 2.0f;
    }
}
```

---

## SIMD 數學函數

### 使用 SVML (Short Vector Math Library)

```cpp
#include <immintrin.h>

// Intel SVML: 向量化數學函數
__m256 vec = _mm256_set_ps(1.0f, 2.0f, 3.0f, 4.0f, 5.0f, 6.0f, 7.0f, 8.0f);

__m256 sin_vec = _mm256_sin_ps(vec);   // 向量 sin
__m256 cos_vec = _mm256_cos_ps(vec);   // 向量 cos
__m256 exp_vec = _mm256_exp_ps(vec);   // 向量 exp
__m256 log_vec = _mm256_log_ps(vec);   // 向量 log

// 需要編譯選項: -fveclib=SVML (ICC) 或 -ffast-math (GCC)
```

### 手動實現 (近似)

```cpp
// 快速近似 1/sqrt(x) - Quake III 算法改進
__m256 fast_inv_sqrt(__m256 x) {
    const __m256 three = _mm256_set1_ps(3.0f);
    const __m256 half = _mm256_set1_ps(0.5f);
    
    __m256 half_x = _mm256_mul_ps(half, x);
    __m256i i = _mm256_cvtps_epi32(x);
    
    // Magic constant
    i = _mm256_sub_epi32(_mm256_set1_epi32(0x5f3759df), _mm256_srli_epi32(i, 1));
    
    __m256 y = _mm256_cvtepi32_ps(i);
    
    // Newton-Raphson 迭代
    y = _mm256_mul_ps(y, _mm256_sub_ps(three, _mm256_mul_ps(half_x, _mm256_mul_ps(y, y))));
    
    return y;
}
```

---

## 性能測量

### Benchmark 對比

```cpp
#include <benchmark/benchmark.h>

constexpr size_t N = 10000;
alignas(32) float data_a[N];
alignas(32) float data_b[N];
alignas(32) float result[N];

static void BM_Scalar(benchmark::State& state) {
    for (auto _ : state) {
        for (size_t i = 0; i < N; ++i) {
            result[i] = data_a[i] + data_b[i];
        }
        benchmark::DoNotOptimize(result);
    }
}

static void BM_SIMD_AVX(benchmark::State& state) {
    for (auto _ : state) {
        for (size_t i = 0; i < N; i += 8) {
            __m256 a = _mm256_load_ps(&data_a[i]);
            __m256 b = _mm256_load_ps(&data_b[i]);
            __m256 c = _mm256_add_ps(a, b);
            _mm256_store_ps(&result[i], c);
        }
        benchmark::DoNotOptimize(result);
    }
}

BENCHMARK(BM_Scalar);      // ~1000ns
BENCHMARK(BM_SIMD_AVX);    // ~125ns (8x faster)
```

---

## 可移植性與跨平台

### 檢測 CPU 特性

```cpp
#include <cpuid.h>

bool has_avx() {
    unsigned int eax, ebx, ecx, edx;
    if (__get_cpuid(1, &eax, &ebx, &ecx, &edx)) {
        return (ecx & bit_AVX) != 0;
    }
    return false;
}

bool has_avx2() {
    unsigned int eax, ebx, ecx, edx;
    if (__get_cpuid_count(7, 0, &eax, &ebx, &ecx, &edx)) {
        return (ebx & bit_AVX2) != 0;
    }
    return false;
}
```

### 運行時分發

```cpp
void add_arrays(const float* a, const float* b, float* c, size_t n) {
    static auto impl = []() {
        if (has_avx2()) {
            return add_arrays_avx2;
        } else if (has_avx()) {
            return add_arrays_avx;
        } else {
            return add_arrays_scalar;
        }
    }();
    
    impl(a, b, c, n);
}
```

### GCC 函數多版本

```cpp
__attribute__((target("avx2")))
void process_avx2(float* data, size_t n) {
    // 使用 AVX2
}

__attribute__((target("sse4.2")))
void process_sse(float* data, size_t n) {
    // 使用 SSE4.2
}

__attribute__((target("default")))
void process(float* data, size_t n) {
    // 通用版本
}

// 編譯器自動選擇合適版本
```

---

## 實戰檢查清單

### SIMD 使用

- [ ] 批量數據處理使用 SIMD
- [ ] 數據對齊到 16/32/64 bytes
- [ ] 使用 `_loadu` 處理未對齊數據
- [ ] 水平運算使用專門函數

### 優化策略

- [ ] 先嘗試編譯器自動向量化
- [ ] 查看向量化報告 (`-fopt-info-vec`)
- [ ] 手動 intrinsics 用於關鍵路徑
- [ ] 考慮 SoA 佈局改善向量化

### 跨平台

- [ ] 運行時檢測 CPU 特性
- [ ] 提供標量後備實現
- [ ] 使用 `__attribute__((target))`
- [ ] 測試不同 SIMD 等級性能

### 高頻交易特定

- [ ] 市場數據解析向量化
- [ ] 批量訂單驗證使用 SIMD
- [ ] 統計計算 (VWAP, TWAP) 向量化
- [ ] 訂單簿掃描使用 SIMD

---

## 參考資料 (References)

1. [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html)
2. [GCC Vector Extensions](https://gcc.gnu.org/onlinedocs/gcc/Vector-Extensions.html)
3. Fog, Agner. *Optimizing software in C++*. Chapter on SIMD.
4. [SIMD for C++ Developers](http://const.me/articles/simd/simd.pdf)
5. Lemire, Daniel. [Faster code through SIMD](https://lemire.me/blog/)
