# Cache 友好與數據局部性 (Cache Friendliness & Data Locality)

## 1. 核心概念

在現代計算機架構中，CPU 與主存 (RAM) 之間的速度差異巨大（約 100x）。為了彌補這一差距，引入了多級緩存 (L1, L2, L3)。編寫 "Cache Friendly" 的代碼是高性能編程（特別是 HFT）的關鍵。

### 1.1 CPU 緩存架構

典型的緩存層級 (Latency 僅供參考):

| 層級 | 大小 | 延遲 (Cycles) | 共享性 |
| :--- | :--- | :--- | :--- |
| **L1 Cache** | 32KB - 64KB | ~4 | Core 獨佔 |
| **L2 Cache** | 256KB - 1MB | ~10-12 | Core 獨佔/共享 |
| **L3 Cache** | 10MB - 64MB | ~40-70 | 所有 Core 共享 |
| **Main Memory** | GB - TB | ~100-200 | 全局 |

> **關鍵**: 數據是以 **Cache Line** (通常 64 bytes) 為單位從內存加載到緩存的。讀取 1 byte 和讀取 64 bytes 的開銷幾乎相同。

### 1.2 數據局部性 (Data Locality)

*   **空間局部性 (Spatial Locality)**: 如果訪問了某個內存位置，附近的內存位置很可能即將被訪問。
    *   *優化*: 使用連續內存容器 (`std::vector`, `std::array`)，避免鏈表 (`std::list`)。
*   **時間局部性 (Temporal Locality)**: 如果訪問了某個內存位置，該位置很可能在不久後再次被訪問。
    *   *優化*: 將頻繁使用的數據保留在緩存中，避免頻繁驅逐。

---

## 2. Cache 友好的數據結構

### 2.1 連續內存 vs 指針追逐 (Pointer Chasing)

指針追逐是緩存殺手。每次解引用指針都可能導致 Cache Miss。

```cpp
// ❌ Cache Unfriendly: std::list, std::map, std::unordered_map
// 節點分散在堆內存中，遍歷需要跳轉，無法利用預取
std::list<int> list; 

// ✅ Cache Friendly: std::vector, std::array
// 數據連續存儲，CPU 預取器 (Hardware Prefetcher) 能高效工作
std::vector<int> vec;
```

**HFT 應用**: 訂單簿 (Order Book) 的價格層級 (Price Levels) 應盡量使用 `std::vector` 或預分配的數組，而不是 `std::map`。雖然 `std::map` 查找是 O(log N)，但在 N 較小時，線性掃描數組 (O(N)) 往往更快，因為全是 Cache Hit。

### 2.2 AoS vs SoA

*   **AoS (Array of Structures)**: 面向對象的自然寫法，但可能導致緩存浪費（如果只訪問部分成員）且不利於 SIMD。
*   **SoA (Structure of Arrays)**: 將結構體的成員拆分成獨立的數組。

```cpp
// AoS: 適合隨機訪問單個對象的所有屬性
struct Particle {
    float x, y, z;
    float r, g, b;
    bool active;
    // Padding...
};
std::vector<Particle> particles;

// SoA: 適合批量處理特定屬性 (SIMD 友好)
struct ParticleSystem {
    std::vector<float> x, y, z; // 緊湊存儲，適合 AVX/SSE 計算位置
    std::vector<float> r, g, b;
    std::vector<bool> active;
};
```

**SIMD 示例 (SoA)**:
```cpp
// 使用 AVX2 批量更新 X 坐標
for (size_t i = 0; i < n; i += 8) {
    __m256 vx = _mm256_load_ps(&x[i]);
    __m256 vvel = _mm256_load_ps(&vx_vel[i]);
    vx = _mm256_add_ps(vx, vvel);
    _mm256_store_ps(&x[i], vx);
}
```

### 2.3 冷熱數據分離 (Hot/Cold Splitting)

將頻繁訪問的數據 ("Hot") 和不常訪問的數據 ("Cold") 分開，以提高 Cache Line 利用率。

```cpp
// ❌ 混合冷熱數據
struct Order {
    double price;           // Hot (用於撮合)
    uint64_t id;            // Cold (僅用於查詢)
    uint32_t quantity;      // Hot
    char customer_name[64]; // Cold (佔用大量緩存空間)
    // ...
}; 
// sizeof(Order) 可能超過 64 bytes，加載一個 Order 佔用多個 Cache Lines

// ✅ 冷熱分離
struct OrderHot {
    double price;
    uint32_t quantity;
    uint32_t cold_ptr_idx; // 指向 Cold 數據的索引
}; // 緊湊，一個 Cache Line 可以存多個 OrderHot

struct OrderCold {
    uint64_t id;
    char customer_name[64];
};
```

---

## 3. 內存對齊與 Padding

### 3.1 為什麼需要對齊？

*   **原子性**: 許多 CPU 架構要求原子操作的地址必須自然對齊。
*   **性能**: 未對齊訪問可能導致兩次內存讀取 (跨 Cache Line) 甚至硬件異常 (ARM, SPARC)。
*   **SIMD**: AVX/SSE 指令通常要求數據對齊到 16/32/64 字節。

### 3.2 `alignas` 與 `alignof`

```cpp
// 強制對齊到 64 bytes (Cache Line 大小)
struct alignas(64) AlignedData {
    double values[4];
};

// 檢查對齊
static_assert(alignof(AlignedData) == 64);
```

### 3.3 結構體 Padding 優化

編譯器會插入 Padding 以滿足成員的對齊要求。通過調整成員順序可以減少 Padding，節省內存。

```cpp
// ❌ Size = 24 (Padding 浪費嚴重)
struct Bad {
    char c;     // 1 byte
    // 7 bytes padding
    double d;   // 8 bytes
    int i;      // 4 bytes
    // 4 bytes padding
};

// ✅ Size = 16 (按大小降序排列)
struct Good {
    double d;   // 8 bytes
    int i;      // 4 bytes
    char c;     // 1 byte
    // 3 bytes padding
};
```

---

## 4. False Sharing (偽共享)

當多個線程修改位於**同一個 Cache Line** 的不同變量時，會導致緩存一致性協議 (MESI) 頻繁在核心間傳遞 Cache Line 所有權，嚴重降低性能。

### 4.1 現象與檢測

```cpp
// ❌ 典型的 False Sharing
struct SharedData {
    std::atomic<int> a; // Thread 1 修改
    std::atomic<int> b; // Thread 2 修改
}; 
// a 和 b 極可能在同一個 64-byte Cache Line 中
```

### 4.2 解決方案

使用 `alignas` 或填充將變量隔離在不同的 Cache Line 中。

```cpp
#include <new> // for std::hardware_destructive_interference_size

// ✅ C++17 標準解法
struct SharedData {
    alignas(std::hardware_destructive_interference_size) std::atomic<int> a;
    alignas(std::hardware_destructive_interference_size) std::atomic<int> b;
};

// 手動填充 (如果不支持 C++17)
struct PaddingData {
    std::atomic<int> a;
    char pad1[60]; // 假設 int 是 4 bytes，填充到 64
    std::atomic<int> b;
    char pad2[60];
};
```

**HFT 實戰**: 在無鎖隊列 (SPSC Queue) 中，`head` (消費者寫) 和 `tail` (生產者寫) 必須強制隔離，否則吞吐量會暴跌。

---

## 5. 預取 (Prefetching)

### 5.1 硬件預取
現代 CPU 會自動識別順序訪問模式並預取數據。這就是為什麼 `std::vector` 遍歷如此之快。

### 5.2 軟件預取
對於非連續訪問 (如 Hash Map, 跳表)，可以使用指令提示 CPU 預取。

```cpp
#include <xmmintrin.h> // _mm_prefetch

void process_data(int* indices, Data* data_array, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        // 預取將來要訪問的數據 (例如 16 個元素之後)
        // _MM_HINT_T0: 預取到 L1 緩存
        if (i + 16 < n) {
            _mm_prefetch((const char*)&data_array[indices[i+16]], _MM_HINT_T0);
        }
        
        // 處理當前數據
        process(data_array[indices[i]]);
    }
}
```

> **注意**: 軟件預取是一把雙刃劍。預取過早會被驅逐，預取過晚沒效果，預取錯誤數據會浪費帶寬。必須通過 Benchmark 驗證。

---

## 6. HFT 實戰案例

### 6.1 訂單簿 (Order Book) 的 Flat Map 實現

標準的 `std::map<Price, Level>` 是紅黑樹，節點分散。HFT 系統通常使用固定大小的數組或 `std::vector` 作為 "Flat Map"。

```cpp
struct PriceLevel {
    double price;
    uint32_t quantity;
    // ...
};

// 假設價格是離散的 tick，可以直接映射到數組索引
// 或者對於稀疏價格，使用排序的 vector + 二分查找 (std::lower_bound)
class FlatOrderBook {
    std::vector<PriceLevel> bids; // 保持排序
    std::vector<PriceLevel> asks;

public:
    void on_order(double price, uint32_t qty) {
        // 二分查找位置
        auto it = std::lower_bound(bids.begin(), bids.end(), price, ...);
        // 插入或更新 (vector 插入涉及內存移動，但對於小規模數據，
        // 內存移動 (memmove) 比指針追逐更快)
    }
};
```

### 6.2 環形緩衝區 (Ring Buffer)

Disruptor 模式的核心。使用數組實現隊列，保證內存連續。

```cpp
template<typename T, size_t Size>
class RingBuffer {
    alignas(64) std::array<T, Size> buffer_; // 數據連續
    alignas(64) std::atomic<size_t> head_{0};
    alignas(64) std::atomic<size_t> tail_{0};
    
    // ... push/pop 實現 ...
};
```

---

## 7. 性能分析工具

1.  **Linux Perf**:
    ```bash
    # 查看 Cache Misses
    perf stat -e cache-references,cache-misses,L1-dcache-load-misses ./app
    ```
2.  **Valgrind (Cachegrind)**:
    ```bash
    valgrind --tool=cachegrind ./app
    ```
    模擬 CPU 緩存行為，提供詳細的 Miss Rate 報告。

## 總結

| 優化策略 | 關鍵點 | 適用場景 |
| :--- | :--- | :--- |
| **數據結構** | 優先 `vector`/`array`，避免 `list`/`map` | 所有場景 |
| **數據佈局** | AoS -> SoA | SIMD 計算，粒子系統 |
| **冷熱分離** | 將頻繁訪問成員聚合 | 複雜對象，訂單結構 |
| **對齊** | `alignas(64)` 避免 False Sharing | 多線程共享變量 |
| **預取** | `_mm_prefetch` | 指針追逐，非連續訪問 |
