# Cache 友好的資料結構 (Cache-Friendly Data Structures)

## 概述

現代 CPU 的記憶體層次結構:
- **L1 Cache**: ~1ns, 32-64KB (每核心獨立)
- **L2 Cache**: ~3-5ns, 256KB-1MB (每核心獨立)
- **L3 Cache**: ~10-20ns, 8-32MB (多核心共享)
- **主記憶體 (RAM)**: ~100ns, GB 級別

**關鍵洞察**: L1 cache 訪問比主記憶體快 **100 倍**

在高頻交易中,cache miss 可能導致:
- 單次操作延遲從 1ns 增加到 100ns
- 吞吐量下降 10-100 倍

---

## Cache 基礎概念

### Cache Line

```cpp
// 典型 cache line 大小: 64 bytes
constexpr size_t CACHE_LINE_SIZE = 64;

struct alignas(CACHE_LINE_SIZE) CacheLineAligned {
    int data[16];  // 64 bytes = 16 * 4
};

// 讀取一個元素會載入整個 cache line
int arr[100];
int x = arr[0];   // 載入 arr[0..15] 到 cache
int y = arr[1];   // cache hit! (已在 cache 中)
int z = arr[20];  // 可能 cache miss (不同 cache line)
```

### 空間局部性 (Spatial Locality)

相鄰記憶體位置傾向於一起被訪問:

```cpp
// ✅ 好: 順序訪問 (高空間局部性)
for (int i = 0; i < n; ++i) {
    sum += arr[i];
}

// ❌ 差: 隨機訪問 (低空間局部性)
for (int i = 0; i < n; ++i) {
    sum += arr[random_indices[i]];
}
```

### 時間局部性 (Temporal Locality)

最近訪問的數據傾向於再次被訪問:

```cpp
// ✅ 好: 重複訪問相同數據
int total = 0;
for (int i = 0; i < 1000; ++i) {
    total += global_config.threshold;  // config 在 cache 中
}

// ❌ 差: 數據被訪問後不再使用
for (int i = 0; i < n; ++i) {
    process(large_array[i]);  // 若 n 很大,前面的數據已被驅逐
}
```

---

## False Sharing (偽共享)

### 問題

多個線程訪問不同變量,但它們在同一 cache line:

```cpp
// ❌ False Sharing
struct Counters {
    std::atomic<int64_t> counter1;  // 8 bytes
    std::atomic<int64_t> counter2;  // 8 bytes (在同一 cache line!)
};

Counters counters;

// 線程 1
void thread1() {
    for (int i = 0; i < 1000000; ++i) {
        counters.counter1.fetch_add(1);  // 寫入使整個 cache line 失效
    }
}

// 線程 2
void thread2() {
    for (int i = 0; i < 1000000; ++i) {
        counters.counter2.fetch_add(1);  // 必須重新載入 cache line
    }
}
// 性能: ~1000ms (大量 cache line 爭用)
```

### 解決方案

```cpp
// ✅ 使用 padding 分離到不同 cache line
struct alignas(64) Counters {
    std::atomic<int64_t> counter1;
    char padding1[64 - sizeof(std::atomic<int64_t>)];
    
    std::atomic<int64_t> counter2;
    char padding2[64 - sizeof(std::atomic<int64_t>)];
};
// 性能: ~100ms (10x 提升!)

// 或使用 C++17 std::hardware_destructive_interference_size
struct Counters {
    alignas(std::hardware_destructive_interference_size) 
        std::atomic<int64_t> counter1;
    
    alignas(std::hardware_destructive_interference_size) 
        std::atomic<int64_t> counter2;
};
```

---

## 數組遍歷模式

### 行主序 vs 列主序

```cpp
constexpr int N = 1024;
int matrix[N][N];

// ✅ 行主序 (Row-major): C++ 默認
for (int i = 0; i < N; ++i) {
    for (int j = 0; j < N; ++j) {
        sum += matrix[i][j];  // 連續訪問
    }
}
// 性能: ~10ms

// ❌ 列主序 (Column-major): cache miss 多
for (int j = 0; j < N; ++j) {
    for (int i = 0; i < N; ++i) {
        sum += matrix[i][j];  // 跳躍訪問 (stride = N)
    }
}
// 性能: ~100ms (10x 慢!)
```

### 矩陣乘法優化

```cpp
// ❌ 未優化版本
void matmul_naive(const float A[][N], const float B[][N], float C[][N]) {
    for (int i = 0; i < N; ++i) {
        for (int j = 0; j < N; ++j) {
            float sum = 0;
            for (int k = 0; k < N; ++k) {
                sum += A[i][k] * B[k][j];  // B[k][j] 跳躍訪問
            }
            C[i][j] = sum;
        }
    }
}

// ✅ 優化: 轉置 B 矩陣
void matmul_optimized(const float A[][N], const float B[][N], float C[][N]) {
    float B_transposed[N][N];
    
    // 轉置 B
    for (int i = 0; i < N; ++i) {
        for (int j = 0; j < N; ++j) {
            B_transposed[j][i] = B[i][j];
        }
    }
    
    // 現在兩個矩陣都是連續訪問
    for (int i = 0; i < N; ++i) {
        for (int j = 0; j < N; ++j) {
            float sum = 0;
            for (int k = 0; k < N; ++k) {
                sum += A[i][k] * B_transposed[j][k];  // 連續訪問!
            }
            C[i][j] = sum;
        }
    }
}
// 性能提升: 2-3x
```

---

## 結構體佈局優化 (SoA vs AoS)

### Array of Structures (AoS)

```cpp
// AoS: 傳統面向對象風格
struct Particle {
    float x, y, z;      // 位置
    float vx, vy, vz;   // 速度
    float mass;
};

std::vector<Particle> particles(10000);

// 只需要更新位置
for (auto& p : particles) {
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.z += p.vz * dt;
    // 每次迭代載入整個 Particle (28 bytes)
    // 但只使用 24 bytes (位置和速度)
    // mass 被載入但未使用 → cache 浪費
}
```

### Structure of Arrays (SoA)

```cpp
// SoA: 數據導向設計
struct Particles {
    std::vector<float> x, y, z;       // 位置
    std::vector<float> vx, vy, vz;    // 速度
    std::vector<float> mass;
};

Particles particles;
particles.x.resize(10000);
particles.y.resize(10000);
// ... 初始化其他數組

// 只需要更新位置
for (size_t i = 0; i < particles.x.size(); ++i) {
    particles.x[i] += particles.vx[i] * dt;
    particles.y[i] += particles.vy[i] * dt;
    particles.z[i] += particles.vz[i] * dt;
    // 只載入需要的數據
    // 更好的 cache 利用率
}
// 性能提升: 2-4x
```

### 高頻交易應用: 訂單簿

```cpp
// ❌ AoS: 每個訂單是一個對象
struct Order {
    uint64_t id;
    double price;
    uint32_t quantity;
    uint32_t timestamp;
};

std::vector<Order> orders;

// 查找特定價格的訂單
for (const auto& order : orders) {
    if (order.price == target_price) {
        // 載入了 id, quantity, timestamp (未使用)
    }
}

// ✅ SoA: 按字段分組
struct OrderBook {
    std::vector<uint64_t> ids;
    std::vector<double> prices;
    std::vector<uint32_t> quantities;
    std::vector<uint32_t> timestamps;
};

OrderBook book;

// 查找特定價格的訂單
for (size_t i = 0; i < book.prices.size(); ++i) {
    if (book.prices[i] == target_price) {
        // 只載入 prices 數組,其他字段不在 cache 中
    }
}
```

---

## 鏈表 vs 數組

### 鏈表的 Cache 問題

```cpp
// ❌ 鏈表: cache miss 多
struct Node {
    int data;
    Node* next;
};

Node* head = /* ... */;
int sum = 0;

for (Node* p = head; p != nullptr; p = p->next) {
    sum += p->data;  // 每次迭代都可能 cache miss
}
// 原因: 節點在記憶體中不連續
```

### 數組的優勢

```cpp
// ✅ 數組/vector: cache 友好
std::vector<int> data = /* ... */;
int sum = 0;

for (int x : data) {
    sum += x;  // 連續記憶體,高 cache hit rate
}
// 性能: 可能快 10-100x
```

### 當必須使用鏈表時

```cpp
// 使用記憶體池改善局部性
template<typename T>
class PoolAllocator {
    static constexpr size_t POOL_SIZE = 1024;
    
    struct Pool {
        std::array<T, POOL_SIZE> data;
        size_t used = 0;
        Pool* next = nullptr;
    };
    
    Pool* current_pool_ = nullptr;
    
public:
    T* allocate() {
        if (!current_pool_ || current_pool_->used >= POOL_SIZE) {
            Pool* new_pool = new Pool;
            new_pool->next = current_pool_;
            current_pool_ = new_pool;
        }
        return &current_pool_->data[current_pool_->used++];
    }
};

// 節點分配在連續記憶體中,改善 cache 局部性
```

---

## 高頻交易數據結構

### 1. 固定大小環形緩衝

```cpp
template<typename T, size_t Capacity>
class RingBuffer {
    static_assert((Capacity & (Capacity - 1)) == 0, "Capacity must be power of 2");
    
    alignas(64) std::array<T, Capacity> buffer_;
    alignas(64) size_t write_pos_ = 0;
    alignas(64) size_t read_pos_ = 0;
    
public:
    bool push(const T& item) {
        size_t next = (write_pos_ + 1) & (Capacity - 1);
        if (next == read_pos_) return false;  // 滿
        
        buffer_[write_pos_] = item;
        write_pos_ = next;
        return true;
    }
    
    bool pop(T& item) {
        if (read_pos_ == write_pos_) return false;  // 空
        
        item = buffer_[read_pos_];
        read_pos_ = (read_pos_ + 1) & (Capacity - 1);
        return true;
    }
};

// 優勢:
// - 固定大小,無動態分配
// - 環形索引使用位運算 (快於取模)
// - write_pos_ 和 read_pos_ 在不同 cache line (避免 false sharing)
```

### 2. 價格層級數組 (Price Level Array)

```cpp
// 交易所訂單簿: 價格離散化
class OrderBook {
    static constexpr int TICK_SIZE_CENTS = 1;  // 0.01 美元
    static constexpr int MIN_PRICE_CENTS = 10000;  // 100 美元
    static constexpr int MAX_PRICE_CENTS = 20000;  // 200 美元
    static constexpr int NUM_LEVELS = (MAX_PRICE_CENTS - MIN_PRICE_CENTS) / TICK_SIZE_CENTS;
    
    struct PriceLevel {
        uint32_t bid_quantity = 0;
        uint32_t ask_quantity = 0;
    };
    
    std::array<PriceLevel, NUM_LEVELS> levels_;
    
    int price_to_index(double price) const {
        int cents = static_cast<int>(price * 100);
        return (cents - MIN_PRICE_CENTS) / TICK_SIZE_CENTS;
    }
    
public:
    void add_bid(double price, uint32_t quantity) {
        int idx = price_to_index(price);
        levels_[idx].bid_quantity += quantity;  // O(1), cache 友好
    }
    
    std::optional<double> get_best_bid() const {
        // 從高到低掃描
        for (int i = NUM_LEVELS - 1; i >= 0; --i) {
            if (levels_[i].bid_quantity > 0) {
                return (MIN_PRICE_CENTS + i * TICK_SIZE_CENTS) / 100.0;
            }
        }
        return std::nullopt;
    }
};

// 優勢:
// - O(1) 更新
// - 所有數據在連續記憶體中
// - 適合向量化 (SIMD)
```

### 3. 熱數據冷數據分離

```cpp
// ❌ 混合熱冷數據
struct Order {
    // 熱數據 (頻繁訪問)
    double price;
    uint32_t quantity;
    
    // 冷數據 (偶爾訪問)
    std::string client_id;      // 32+ bytes
    std::string description;    // 32+ bytes
    uint64_t timestamp;
};

// ✅ 分離熱冷數據
struct OrderHot {
    double price;
    uint32_t quantity;
    uint32_t cold_data_index;  // 指向冷數據
};

struct OrderCold {
    std::string client_id;
    std::string description;
    uint64_t timestamp;
};

class OrderManager {
    std::vector<OrderHot> hot_data_;   // 緊湊,cache 友好
    std::vector<OrderCold> cold_data_; // 分離存儲
    
public:
    void process_orders() {
        // 只訪問熱數據,cache 利用率高
        for (const auto& order : hot_data_) {
            if (order.price > threshold) {
                execute(order);
            }
        }
    }
    
    void log_order_details(size_t index) {
        // 需要時才訪問冷數據
        const auto& cold = cold_data_[hot_data_[index].cold_data_index];
        log(cold.client_id, cold.description);
    }
};
```

---

## 預取 (Prefetching)

### 軟體預取

```cpp
// 手動預取未來數據
void process_array(const int* data, size_t n) {
    constexpr size_t PREFETCH_DISTANCE = 8;
    
    for (size_t i = 0; i < n; ++i) {
        // 預取未來的數據
        if (i + PREFETCH_DISTANCE < n) {
            __builtin_prefetch(&data[i + PREFETCH_DISTANCE]);
        }
        
        // 處理當前數據
        int result = expensive_compute(data[i]);
        use(result);
    }
}
```

### 鏈表預取

```cpp
// 鏈表遍歷時預取
struct Node {
    int data;
    Node* next;
};

void process_list(Node* head) {
    Node* current = head;
    Node* next = current ? current->next : nullptr;
    
    while (current) {
        // 預取下一個節點
        if (next) {
            __builtin_prefetch(next);
            if (next->next) {
                __builtin_prefetch(next->next);  // 預取下下個
            }
        }
        
        // 處理當前節點
        process(current->data);
        
        current = next;
        next = current ? current->next : nullptr;
    }
}
```

---

## 數據對齊優化

### 自然對齊

```cpp
// ❌ 未對齊: 可能跨越 cache line
struct Data {
    char c;       // 1 byte
    double d;     // 8 bytes (可能未對齊)
    int i;        // 4 bytes
}; // sizeof = 24 (有 padding)

// ✅ 對齊優化
struct alignas(64) Data {
    double d;     // 8 bytes
    int i;        // 4 bytes
    char c;       // 1 byte
    char padding[51];  // 填充到 64 bytes
}; // sizeof = 64 (完整 cache line)
```

### SIMD 對齊

```cpp
// 256-bit SIMD 需要 32-byte 對齊
alignas(32) float data[1024];

void process_simd(float* __restrict__ data, size_t n) {
    // 編譯器可安全使用 AVX 指令
    for (size_t i = 0; i < n; i += 8) {
        // 處理 8 個 float (256 bits)
    }
}
```

---

## Cache 性能測量

### 使用 perf 工具

```bash
# 測量 cache miss
perf stat -e cache-references,cache-misses ./program

# 輸出示例:
# 1,234,567 cache-references
#   123,456 cache-misses  # 10% miss rate

# 詳細 cache 事件
perf stat -e L1-dcache-load-misses,L1-dcache-loads ./program
```

### 代碼測量

```cpp
#include <x86intrin.h>

uint64_t rdtsc() {
    return __rdtsc();
}

void measure_cache_effect() {
    constexpr size_t N = 1024 * 1024;
    std::vector<int> data(N);
    
    // 預熱 cache
    for (int& x : data) x = 0;
    
    // 測量順序訪問
    uint64_t start = rdtsc();
    for (int& x : data) {
        x += 1;
    }
    uint64_t sequential_cycles = rdtsc() - start;
    
    // 測量隨機訪問
    std::vector<size_t> indices(N);
    std::iota(indices.begin(), indices.end(), 0);
    std::shuffle(indices.begin(), indices.end(), std::mt19937{});
    
    start = rdtsc();
    for (size_t idx : indices) {
        data[idx] += 1;
    }
    uint64_t random_cycles = rdtsc() - start;
    
    std::cout << "Sequential: " << sequential_cycles << " cycles\n";
    std::cout << "Random: " << random_cycles << " cycles\n";
    std::cout << "Ratio: " << static_cast<double>(random_cycles) / sequential_cycles << "x\n";
    // 典型結果: Random 可能慢 10-20x
}
```

---

## 實戰檢查清單

### 數據結構選擇

- [ ] 優先使用連續記憶體容器 (vector, array)
- [ ] 避免鏈表,除非有充分理由
- [ ] 考慮 SoA 而非 AoS (特別是大量對象)
- [ ] 熱數據與冷數據分離

### Cache 優化

- [ ] 順序訪問優於隨機訪問
- [ ] 批量處理相同操作 (改善時間局部性)
- [ ] 避免 false sharing (多線程)
- [ ] 關鍵結構體 cache line 對齊

### 測量與驗證

- [ ] 使用 perf 測量 cache miss rate
- [ ] 使用 benchmark 對比不同佈局
- [ ] 檢查結構體大小與對齊
- [ ] Profile 熱點代碼的 cache 行為

### 高頻交易特定

- [ ] 固定大小數據結構 (避免動態分配)
- [ ] 價格/時間離散化為數組索引
- [ ] 預分配環形緩衝
- [ ] 預取預測性訪問模式

---

## 參考資料 (References)

1. Drepper, Ulrich. *What Every Programmer Should Know About Memory*. Red Hat, 2007.
2. [Intel Optimization Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
3. [Gallery of Processor Cache Effects](http://igoro.com/archive/gallery-of-processor-cache-effects/)
4. Fog, Agner. *Optimizing software in C++*. Technical University of Denmark, 2004.
5. [Performance Analysis Tools](https://perf.wiki.kernel.org/)
