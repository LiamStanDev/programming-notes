# SIMD 與向量化 (SIMD & Vectorization)

## 1. 核心概念

**SIMD (Single Instruction, Multiple Data)** 是一種並行計算技術，允許一條 CPU 指令同時處理多個數據。

*   **SISD (Scalar)**: `a + b` (一次處理 1 個數據)
*   **SIMD (Vector)**: `[a1, a2, a3, a4] + [b1, b2, b3, b4]` (一次處理 4 個數據)

在 HFT 中，SIMD 是提升吞吐量的關鍵武器，常用於行情解碼、指標計算 (VWAP, MA) 和風險檢查。

### 1.1 x86 SIMD 指令集演進

| 指令集 | 位寬 | float (32-bit) | double (64-bit) | 引入年份 | 普及度 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **SSE** | 128-bit | 4 | 2 | 1999 | 100% |
| **AVX/AVX2** | 256-bit | 8 | 4 | 2011 | 高 (HFT 標配) |
| **AVX-512** | 512-bit | 16 | 8 | 2017 | 服務器級 (Skylake-X+) |

---

## 2. SIMD 基礎 (Intrinsics)

C++ 通過編譯器內建函數 (Intrinsics) 直接調用 SIMD 指令。需包含頭文件 `<immintrin.h>`。

### 2.1 數據類型

*   `__m128` / `__m128d` / `__m128i`: 128-bit (float / double / integer)
*   `__m256` / `__m256d` / `__m256i`: 256-bit
*   `__m512` / `__m512d` / `__m512i`: 512-bit

### 2.2 基本操作 (AVX2 示例)

```cpp
#include <immintrin.h>
#include <iostream>

void simd_add_example() {
    // 1. 初始化 (Set)
    // 注意: set_ps 是逆序的 (e7, e6, ..., e0)
    __m256 a = _mm256_set_ps(8.0, 7.0, 6.0, 5.0, 4.0, 3.0, 2.0, 1.0);
    __m256 b = _mm256_set1_ps(10.0); // 廣播: {10, 10, ...}

    // 2. 運算 (Add)
    __m256 c = _mm256_add_ps(a, b);

    // 3. 存儲 (Store)
    alignas(32) float result[8];
    _mm256_store_ps(result, c); // 需要 32-byte 對齊

    for (float x : result) std::cout << x << " "; 
    // 輸出: 11 12 13 14 15 16 17 18
}
```

### 2.3 Load / Store 與對齊

SIMD 對內存對齊非常敏感。

*   `_mm256_load_ps` / `_mm256_store_ps`: **必須對齊** (32-byte)。如果地址未對齊，程序會崩潰 (Segfault)。
*   `_mm256_loadu_ps` / `_mm256_storeu_ps`: **允許未對齊** (Unaligned)。現代 CPU (Haswell+) 上性能損耗極小，推薦默認使用。

```cpp
alignas(32) float aligned_buf[8]; // 棧上對齊
float* ptr = (float*)_mm_malloc(1024, 32); // 堆上對齊分配
_mm_free(ptr);
```

---

## 3. 常用向量化模式

### 3.1 垂直計算 (Vertical Processing)
最自然的 SIMD 模式，對數組進行逐元素運算。

```cpp
void add_arrays(float* a, float* b, float* c, size_t n) {
    size_t i = 0;
    // 每次處理 8 個 float
    for (; i + 8 <= n; i += 8) {
        __m256 va = _mm256_loadu_ps(&a[i]);
        __m256 vb = _mm256_loadu_ps(&b[i]);
        __m256 vc = _mm256_add_ps(va, vb);
        _mm256_storeu_ps(&c[i], vc);
    }
    // 處理剩餘元素 (Epilogue)
    for (; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}
```

### 3.2 水平計算 (Horizontal Processing)
將向量內部的元素進行歸約 (Reduction)，如求和。

```cpp
float hsum_avx(__m256 v) {
    // 步驟: 256 -> 128 -> 64 -> 32
    __m128 lo = _mm256_castps256_ps128(v);
    __m128 hi = _mm256_extractf128_ps(v, 1);
    __m128 sum = _mm_add_ps(lo, hi);
    
    __m128 shuf = _mm_movehdup_ps(sum);
    sum = _mm_add_ps(sum, shuf);
    shuf = _mm_movehl_ps(shuf, sum);
    sum = _mm_add_ss(sum, shuf);
    
    return _mm_cvtss_f32(sum);
}
```

### 3.3 條件掩碼 (Masking)
SIMD 沒有 `if-else`，使用掩碼 (Mask) 進行條件選擇。

```cpp
// 相當於: c[i] = (a[i] > b[i]) ? a[i] : b[i];
__m256 va = ...;
__m256 vb = ...;
__m256 mask = _mm256_cmp_ps(va, vb, _CMP_GT_OQ); // 生成掩碼
__m256 result = _mm256_blendv_ps(vb, va, mask);  // 根據掩碼選擇
```

---

## 4. HFT 實戰案例

### 4.1 VWAP (成交量加權平均價) 計算

```cpp
double calculate_vwap_avx(const double* prices, const int64_t* volumes, size_t n) {
    __m256d v_sum_pv = _mm256_setzero_pd(); // Price * Volume 累加器
    __m256d v_sum_v = _mm256_setzero_pd();  // Volume 累加器 (轉為 double)

    for (size_t i = 0; i + 4 <= n; i += 4) {
        __m256d p = _mm256_loadu_pd(&prices[i]);
        
        // int64 -> double 轉換
        // 注意: AVX2 沒有直接 load int64 到 double 的指令，需先 load int 再 convert
        __m256i v_int = _mm256_loadu_si256((__m256i*)&volumes[i]);
        // 這裡簡化假設 volumes 適合 double 精度 (53-bit)
        // 實際可能需要更複雜的處理或使用 AVX-512
        // 這裡演示邏輯: 假設 volumes 已經是 double 數組
        // __m256d v = _mm256_loadu_pd(&volumes_double[i]); 
        
        // 為了演示完整性，我們假設輸入 volumes 也是 double* 
        // (實際系統中通常會預處理數據格式以適應 SIMD)
        __m256d v = _mm256_cvtepi32_pd(_mm_loadu_si128((__m128i*)&volumes[i])); // 僅演示 load 4個 int32

        v_sum_pv = _mm256_add_pd(v_sum_pv, _mm256_mul_pd(p, v));
        v_sum_v = _mm256_add_pd(v_sum_v, v);
    }

    // 水平求和
    double total_pv = hsum_pd(v_sum_pv);
    double total_v = hsum_pd(v_sum_v);
    return total_pv / total_v;
}
```

### 4.2 訂單簿快速掃描
尋找第一個非空價格層級 (用於 Best Bid/Ask)。

```cpp
// 假設 quantities 是 32-byte 對齊的 uint32_t 數組
int find_best_level(const uint32_t* quantities, int max_levels) {
    __m256i zero = _mm256_setzero_si256();
    
    for (int i = 0; i < max_levels; i += 8) {
        __m256i q = _mm256_load_si256((__m256i*)&quantities[i]);
        
        // 比較是否等於 0 (0xFFFFFFFF if equal, 0 otherwise)
        __m256i cmp = _mm256_cmpeq_epi32(q, zero);
        
        // 提取符號位生成 8-bit mask
        int mask = _mm256_movemask_ps(_mm256_castsi256_ps(cmp));
        
        // 如果 mask 不是全 1 (0xFF)，說明有非零元素
        if (mask != 0xFF) {
            // 找到第一個 0 bit 的位置 (即第一個非零 quantity)
            // __builtin_ctz 計算 trailing zeros
            int idx = __builtin_ctz(~mask); 
            return i + idx;
        }
    }
    return -1;
}
```

---

## 5. 自動向量化 (Auto-Vectorization)

現代編譯器 (GCC/Clang) 非常聰明，能自動將簡單循環轉換為 SIMD 代碼。

### 5.1 啟用條件
1.  **開啟優化**: `-O2` 或 `-O3`。
2.  **指定架構**: `-march=native` 或 `-mavx2`。
3.  **代碼結構**: 循環次數已知或可計算，無複雜依賴，無函數調用。

### 5.2 幫助編譯器
*   **`__restrict`**: 告訴編譯器指針不重疊 (Aliasing)。
*   **`#pragma GCC ivdep`**: 忽略向量依賴。
*   **對齊提示**: `__builtin_assume_aligned(ptr, 32)`.

```cpp
void scale_array(float* __restrict a, float s, size_t n) {
    a = (float*)__builtin_assume_aligned(a, 32);
    for (size_t i = 0; i < n; ++i) {
        a[i] *= s; // 編譯器極大機率生成 AVX 指令
    }
}
```

---

## 6. 總結與最佳實踐

1.  **優先使用庫**: 對於數學運算，優先使用 Intel MKL, IPP 或 Vc 庫，而不是手寫 Intrinsics。
2.  **數據佈局**: **SoA (Structure of Arrays)** 比 AoS 更適合 SIMD。
3.  **對齊**: 始終保持數據對齊 (32-byte for AVX)，雖然 `loadu` 很快，但對齊能防止跨 Cache Line 拆分。
4.  **混合精度**: 避免在循環中頻繁進行 float/double/int 轉換。
5.  **測試**: 使用 Benchmark 驗證 SIMD 是否真的變快了 (有時數據傳輸開銷大於計算收益)。

## 參考資料
*   [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) (必備工具)
*   [Agner Fog's Vector Class Library](https://www.agner.org/optimize/#vectorclass)
*   [CppCon 2017: "Postmodern C++ and SIMD"](https://www.youtube.com/watch?v=XX975Q2jV_g)
