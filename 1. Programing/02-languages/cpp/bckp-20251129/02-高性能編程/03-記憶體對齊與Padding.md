# 記憶體對齊與 Padding (Memory Alignment & Padding)

## 概述

記憶體對齊是 CPU 硬體要求的基本約束:
- **對齊 (Alignment)**: 數據地址必須是特定值的倍數
- **填充 (Padding)**: 編譯器插入的未使用字節

**為什麼重要?**
1. **性能**: 未對齊訪問可能需要多次記憶體讀取
2. **正確性**: 某些架構 (ARM) 未對齊訪問會觸發硬體異常
3. **原子操作**: 需要自然對齊才能保證原子性

在高頻交易中,對齊優化能:
- 減少記憶體訪問次數
- 啟用 SIMD 優化
- 避免 false sharing

---

## 對齊基礎

### 自然對齊 (Natural Alignment)

```cpp
// 自然對齊: 數據大小 = 對齊要求
char c;      // 1 byte,  對齊 1
short s;     // 2 bytes, 對齊 2
int i;       // 4 bytes, 對齊 4
long long l; // 8 bytes, 對齊 8
double d;    // 8 bytes, 對齊 8

// 地址示例:
// 0x1000: char    ✓ (1000 % 1 == 0)
// 0x1002: short   ✓ (1002 % 2 == 0)
// 0x1004: int     ✓ (1004 % 4 == 0)
// 0x1008: double  ✓ (1008 % 8 == 0)

// 未對齊示例:
// 0x1001: short   ✗ (1001 % 2 == 1)
// 0x1003: int     ✗ (1003 % 4 == 3)
```

### 查詢對齊要求

```cpp
#include <iostream>

struct Data {
    char c;
    int i;
};

int main() {
    std::cout << "alignof(char): " << alignof(char) << '\n';       // 1
    std::cout << "alignof(int): " << alignof(int) << '\n';         // 4
    std::cout << "alignof(double): " << alignof(double) << '\n';   // 8
    std::cout << "alignof(Data): " << alignof(Data) << '\n';       // 4 (最大成員)
    
    // C++11
    std::cout << "std::alignment_of<int>::value: " 
              << std::alignment_of<int>::value << '\n';  // 4
}
```

---

## 結構體對齊與填充

### Padding 規則

```cpp
// 示例 1: 基本 padding
struct Example1 {
    char c;      // 1 byte
    // 3 bytes padding
    int i;       // 4 bytes
    char c2;     // 1 byte
    // 3 bytes padding (結尾對齊)
}; // sizeof = 12, alignof = 4

// 內存佈局:
// [c][?][?][?][i][i][i][i][c2][?][?][?]
//  0  1  2  3  4  5  6  7   8  9  A  B

// 示例 2: 優化佈局
struct Example2 {
    int i;       // 4 bytes
    char c;      // 1 byte
    char c2;     // 1 byte
    // 2 bytes padding
}; // sizeof = 8, alignof = 4

// 內存佈局:
// [i][i][i][i][c][c2][?][?]
//  0  1  2  3  4  5   6  7
```

### 計算 sizeof 與對齊

```cpp
struct Data {
    char a;      // offset 0, size 1
    // 3 bytes padding
    int b;       // offset 4, size 4
    short c;     // offset 8, size 2
    // 2 bytes padding (對齊到 4)
}; // sizeof = 12

// 規則:
// 1. 每個成員對齊到自身大小的倍數
// 2. 結構體大小對齊到最大成員的倍數
// 3. 結構體對齊要求 = max(成員對齊要求)
```

### 優化結構體大小

```cpp
// ❌ 未優化 (24 bytes)
struct OrderBad {
    char type;        // 1 byte
    // 7 bytes padding
    double price;     // 8 bytes
    char side;        // 1 byte
    // 3 bytes padding
    int quantity;     // 4 bytes
}; // sizeof = 24

// ✅ 優化: 按大小降序排列 (16 bytes)
struct OrderGood {
    double price;     // 8 bytes
    int quantity;     // 4 bytes
    char type;        // 1 byte
    char side;        // 1 byte
    // 2 bytes padding
}; // sizeof = 16

// 節省: 33% 記憶體!
```

---

## alignas 說明符 (C++11)

### 基本用法

```cpp
// 對齊到 16 bytes
struct alignas(16) Vec4 {
    float x, y, z, w;
}; // sizeof = 16, alignof = 16

// 對齊到 cache line (64 bytes)
struct alignas(64) CacheLineAligned {
    int data[16];
}; // sizeof = 64, alignof = 64

// 陣列中的對齊
alignas(32) float simd_data[1024];
```

### 避免 False Sharing

```cpp
// ❌ False sharing
struct Counters {
    std::atomic<int64_t> counter1;  // 8 bytes
    std::atomic<int64_t> counter2;  // 8 bytes (同一 cache line)
}; // sizeof = 16

// ✅ 使用 alignas 分離
struct Counters {
    alignas(64) std::atomic<int64_t> counter1;
    alignas(64) std::atomic<int64_t> counter2;
}; // sizeof = 128 (每個在獨立 cache line)

// C++17: 標準方式
struct Counters {
    alignas(std::hardware_destructive_interference_size) 
        std::atomic<int64_t> counter1;
    
    alignas(std::hardware_destructive_interference_size) 
        std::atomic<int64_t> counter2;
};
```

### 強制緊湊 (Packed)

```cpp
// GCC/Clang: 禁用 padding
struct __attribute__((packed)) PackedData {
    char c;
    int i;
    char c2;
}; // sizeof = 6 (無 padding)

// 注意: packed 可能導致性能問題
// - 未對齊訪問更慢
// - 某些架構不支持

// MSVC
#pragma pack(push, 1)
struct PackedData {
    char c;
    int i;
    char c2;
};
#pragma pack(pop)
```

---

## C++17 對齊常量

### std::hardware_destructive_interference_size

```cpp
#include <new>

// 避免 false sharing 的最小對齊
constexpr size_t destructive_size = 
    std::hardware_destructive_interference_size;  // 通常 64

// 共享數據的最大對齊 (改善 cache 利用)
constexpr size_t constructive_size = 
    std::hardware_constructive_interference_size;  // 通常 64

// 使用示例
struct alignas(std::hardware_destructive_interference_size) IsolatedCounter {
    std::atomic<int64_t> value;
};

// 多線程計數器數組
std::array<IsolatedCounter, 8> counters;
// 每個計數器在獨立 cache line,無 false sharing
```

---

## 動態記憶體對齊

### aligned_alloc (C++17)

```cpp
#include <cstdlib>

// 分配 256 bytes,對齊到 64 bytes
void* ptr = std::aligned_alloc(64, 256);
if (ptr) {
    // 使用記憶體
    std::free(ptr);
}
```

### 自定義 Allocator

```cpp
template<size_t Alignment>
class AlignedAllocator {
public:
    void* allocate(size_t size) {
        void* ptr = std::aligned_alloc(Alignment, size);
        if (!ptr) throw std::bad_alloc();
        return ptr;
    }
    
    void deallocate(void* ptr) {
        std::free(ptr);
    }
};

// 使用
AlignedAllocator<64> allocator;
void* ptr = allocator.allocate(1024);
// ...
allocator.deallocate(ptr);
```

### std::vector 與對齊

```cpp
// 對齊的 vector 元素
template<typename T, size_t Alignment>
struct AlignedWrapper {
    alignas(Alignment) T value;
};

// 64-byte 對齊的 int vector
std::vector<AlignedWrapper<int, 64>> aligned_vec;

// 或使用自定義 allocator
template<typename T, size_t Alignment>
class AlignedSTLAllocator {
public:
    using value_type = T;
    
    T* allocate(size_t n) {
        void* ptr = std::aligned_alloc(Alignment, n * sizeof(T));
        if (!ptr) throw std::bad_alloc();
        return static_cast<T*>(ptr);
    }
    
    void deallocate(T* ptr, size_t) {
        std::free(ptr);
    }
};

std::vector<int, AlignedSTLAllocator<int, 64>> vec;
```

---

## SIMD 對齊要求

### SSE/AVX 對齊

```cpp
#include <immintrin.h>

// SSE: 128-bit (16 bytes)
alignas(16) float sse_data[4];
__m128 sse_vec = _mm_load_ps(sse_data);  // 需要 16-byte 對齊

// AVX: 256-bit (32 bytes)
alignas(32) float avx_data[8];
__m256 avx_vec = _mm256_load_ps(avx_data);  // 需要 32-byte 對齊

// AVX-512: 512-bit (64 bytes)
alignas(64) float avx512_data[16];
__m512 avx512_vec = _mm512_load_ps(avx512_data);  // 需要 64-byte 對齊

// 未對齊版本 (較慢)
float unaligned_data[4];
__m128 vec = _mm_loadu_ps(unaligned_data);  // 'u' = unaligned
```

### 向量化友好的佈局

```cpp
// ❌ AoS: 難以向量化
struct Particle {
    float x, y, z;
};

std::vector<Particle> particles(1000);

// ✅ SoA + 對齊: SIMD 友好
struct Particles {
    alignas(32) std::vector<float> x;  // AVX 對齊
    alignas(32) std::vector<float> y;
    alignas(32) std::vector<float> z;
    
    Particles(size_t n) {
        x.resize(n);
        y.resize(n);
        z.resize(n);
    }
};

Particles particles(1000);

// 向量化處理
for (size_t i = 0; i < particles.x.size(); i += 8) {
    __m256 vx = _mm256_load_ps(&particles.x[i]);  // 載入 8 個 float
    __m256 vy = _mm256_load_ps(&particles.y[i]);
    __m256 vz = _mm256_load_ps(&particles.z[i]);
    
    // SIMD 運算...
}
```

---

## 高頻交易應用

### 1. 訊息結構對齊

```cpp
// 網路訊息: 需要精確控制佈局
struct __attribute__((packed)) MarketDataMsg {
    uint16_t msg_type;      // 2 bytes
    uint64_t timestamp;     // 8 bytes
    uint32_t symbol_id;     // 4 bytes
    double price;           // 8 bytes
    uint32_t quantity;      // 4 bytes
}; // sizeof = 26 bytes (無 padding)

static_assert(sizeof(MarketDataMsg) == 26);

// 內部處理: 優化對齊
struct MarketDataInternal {
    uint64_t timestamp;     // 8 bytes (先放大的)
    double price;           // 8 bytes
    uint32_t symbol_id;     // 4 bytes
    uint32_t quantity;      // 4 bytes
    uint16_t msg_type;      // 2 bytes
    // 6 bytes padding
}; // sizeof = 32 bytes (對齊優化)
```

### 2. 訂單簿層級

```cpp
// Cache-line 對齊的價格層級
struct alignas(64) PriceLevel {
    double price;                    // 8 bytes
    uint64_t total_quantity;         // 8 bytes
    uint32_t order_count;            // 4 bytes
    uint32_t padding1;               // 4 bytes
    
    // 預留空間給更多數據,填滿 cache line
    uint8_t reserved[40];            // 40 bytes
}; // sizeof = 64 bytes (完整 cache line)

static_assert(sizeof(PriceLevel) == 64);
static_assert(alignof(PriceLevel) == 64);

// 數組中每個元素佔據獨立 cache line
std::array<PriceLevel, 100> price_levels;
```

### 3. 無鎖隊列

```cpp
template<typename T, size_t Capacity>
class LockFreeQueue {
    static_assert((Capacity & (Capacity - 1)) == 0, "Power of 2");
    
    // 寫指針和讀指針分離到不同 cache line
    alignas(std::hardware_destructive_interference_size)
        std::atomic<size_t> write_pos_{0};
    
    alignas(std::hardware_destructive_interference_size)
        std::atomic<size_t> read_pos_{0};
    
    // 數據數組
    alignas(std::hardware_constructive_interference_size)
        std::array<T, Capacity> buffer_;
    
public:
    bool push(const T& item) {
        const size_t write = write_pos_.load(std::memory_order_relaxed);
        const size_t next = (write + 1) & (Capacity - 1);
        
        if (next == read_pos_.load(std::memory_order_acquire)) {
            return false;  // 滿
        }
        
        buffer_[write] = item;
        write_pos_.store(next, std::memory_order_release);
        return true;
    }
    
    bool pop(T& item) {
        const size_t read = read_pos_.load(std::memory_order_relaxed);
        
        if (read == write_pos_.load(std::memory_order_acquire)) {
            return false;  // 空
        }
        
        item = buffer_[read];
        read_pos_.store((read + 1) & (Capacity - 1), std::memory_order_release);
        return true;
    }
};
```

---

## 檢測對齊問題

### 編譯期檢查

```cpp
struct Data {
    char c;
    int i;
};

// 靜態斷言
static_assert(sizeof(Data) == 8, "Unexpected size");
static_assert(alignof(Data) == 4, "Unexpected alignment");

// offsetof 檢查成員偏移
static_assert(offsetof(Data, i) == 4, "Unexpected offset");
```

### 運行時檢查

```cpp
#include <iostream>
#include <cstdint>

void check_alignment(void* ptr, size_t alignment) {
    uintptr_t addr = reinterpret_cast<uintptr_t>(ptr);
    if (addr % alignment != 0) {
        std::cerr << "Misaligned pointer: " << ptr 
                  << " (required: " << alignment << ")\n";
    }
}

int main() {
    alignas(64) int data[16];
    check_alignment(data, 64);  // 應該對齊
    check_alignment(&data[1], 64);  // 可能未對齊
}
```

### Sanitizer 檢測

```bash
# UBSan 檢測未對齊訪問
g++ -fsanitize=alignment program.cpp -o program
./program

# 輸出示例:
# runtime error: load of misaligned address 0x7ffc12345679
```

---

## 跨平台對齊差異

### x86 vs ARM

```cpp
// x86: 允許未對齊訪問 (但較慢)
int* p = reinterpret_cast<int*>(0x1003);  // 未對齊
int value = *p;  // 可能慢 2-3x,但不會崩潰

// ARM: 未對齊訪問可能觸發異常
// 需要使用 memcpy
int value;
std::memcpy(&value, p, sizeof(int));  // 安全
```

### 編譯器差異

```cpp
// GCC/Clang
struct __attribute__((aligned(16))) GCC_Aligned {
    int data;
};

// MSVC
__declspec(align(16)) struct MSVC_Aligned {
    int data;
};

// 跨平台
struct alignas(16) Portable_Aligned {
    int data;
};  // C++11 標準,推薦使用
```

---

## 性能對比

### 對齊 vs 未對齊訪問

```cpp
#include <benchmark/benchmark.h>

// 對齊訪問
alignas(64) int aligned_data[1024];

static void BM_AlignedAccess(benchmark::State& state) {
    for (auto _ : state) {
        int sum = 0;
        for (int i = 0; i < 1024; ++i) {
            sum += aligned_data[i];
        }
        benchmark::DoNotOptimize(sum);
    }
}

// 未對齊訪問 (偏移 1 byte)
char buffer[1024 * 4 + 1];
int* unaligned_data = reinterpret_cast<int*>(buffer + 1);

static void BM_UnalignedAccess(benchmark::State& state) {
    for (auto _ : state) {
        int sum = 0;
        for (int i = 0; i < 1024; ++i) {
            sum += unaligned_data[i];
        }
        benchmark::DoNotOptimize(sum);
    }
}

BENCHMARK(BM_AlignedAccess);    // ~500ns
BENCHMARK(BM_UnalignedAccess);  // ~1500ns (3x 慢)
```

### 結構體佈局對比

```cpp
// 測試不同佈局的遍歷性能

struct Unoptimized {
    char a;
    double b;
    char c;
    int d;
}; // sizeof = 24

struct Optimized {
    double b;
    int d;
    char a;
    char c;
}; // sizeof = 16

static void BM_UnoptimizedLayout(benchmark::State& state) {
    std::vector<Unoptimized> data(10000);
    for (auto _ : state) {
        for (const auto& item : data) {
            benchmark::DoNotOptimize(item.b);
        }
    }
}

static void BM_OptimizedLayout(benchmark::State& state) {
    std::vector<Optimized> data(10000);
    for (auto _ : state) {
        for (const auto& item : data) {
            benchmark::DoNotOptimize(item.b);
        }
    }
}
// Optimized 可能快 30-40% (更好的 cache 利用率)
```

---

## 實戰檢查清單

### 結構體設計

- [ ] 成員按大小降序排列
- [ ] 使用 `static_assert` 驗證大小
- [ ] 檢查是否有不必要的 padding
- [ ] 熱路徑結構體考慮 cache line 對齊

### 對齊優化

- [ ] SIMD 數據使用適當對齊 (16/32/64)
- [ ] 多線程共享數據避免 false sharing
- [ ] 使用 `alignas` 而非編譯器特定屬性
- [ ] 檢查 `offsetof` 確保成員位置

### 性能驗證

- [ ] 使用 `sizeof`/`alignof` 檢查實際大小
- [ ] Benchmark 對比不同佈局
- [ ] Profile 確認 cache miss rate
- [ ] 啟用 UBSan 檢測未對齊訪問

### 高頻交易特定

- [ ] 網路訊息使用 `packed` (精確控制)
- [ ] 內部結構優化對齊 (性能)
- [ ] 無鎖結構分離 cache line
- [ ] 預分配對齊記憶體池

---

## 參考資料 (References)

1. [C++ Object Layout](https://shaharmike.com/cpp/alignment-and-padding/)
2. [Intel Optimization Manual - Data Alignment](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
3. [What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
4. [C++17 std::hardware_destructive_interference_size](https://en.cppreference.com/w/cpp/thread/hardware_destructive_interference_size)
5. Fog, Agner. *Optimizing software in C++*. Chapter on data alignment.
