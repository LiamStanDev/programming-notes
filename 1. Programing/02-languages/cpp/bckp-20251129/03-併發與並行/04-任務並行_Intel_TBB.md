# 任務並行: Intel TBB (Task Parallelism with Intel TBB)

## 概述

Intel Threading Building Blocks (TBB) 是高階並行程式庫,提供任務式並行 (Task-Based Parallelism) 抽象,讓開發者專注於並行邏輯而非執行緒管理。相較於手動管理執行緒,TBB 提供更高的抽象層次與更好的效能。

## TBB 核心概念

### 任務 vs 執行緒

```cpp
// 傳統方式: 手動管理執行緒
std::vector<std::thread> threads;
for (int i = 0; i < std::thread::hardware_concurrency(); ++i) {
    threads.emplace_back([i]() {
        process_chunk(i);
    });
}
for (auto& t : threads) t.join();

// TBB 方式: 任務式並行
tbb::parallel_for(0, num_chunks, [](int i) {
    process_chunk(i);
});
// TBB 自動管理執行緒池與工作竊取
```

**優勢**:
- 自動負載平衡 (Work Stealing)
- 避免過度訂閱 (Over-subscription)
- 支援巢狀並行 (Nested Parallelism)
- Cache 友好的排程器

## 安裝與設定

```bash
# Ubuntu/Debian
sudo apt install libtbb-dev

# macOS
brew install tbb

# CMake 整合
find_package(TBB REQUIRED)
target_link_libraries(my_app TBB::tbb)
```

```cpp
// 基本引入
#include <tbb/parallel_for.h>
#include <tbb/parallel_reduce.h>
#include <tbb/parallel_invoke.h>
#include <tbb/blocked_range.h>
#include <tbb/task_arena.h>
```

## parallel_for - 並行迴圈

### 基礎用法

```cpp
#include <tbb/parallel_for.h>
#include <vector>

// 範例 1: 簡單並行迴圈
void process_array(std::vector<double>& data) {
    tbb::parallel_for(size_t(0), data.size(), [&](size_t i) {
        data[i] = std::sqrt(data[i]) * 2.0;
    });
}

// 範例 2: 使用 blocked_range (更高效)
void process_array_blocked(std::vector<double>& data) {
    tbb::parallel_for(
        tbb::blocked_range<size_t>(0, data.size()),
        [&](const tbb::blocked_range<size_t>& r) {
            for (size_t i = r.begin(); i != r.end(); ++i) {
                data[i] = std::sqrt(data[i]) * 2.0;
            }
        }
    );
}
```

### Grain Size 調整

```cpp
// Grain Size: 每個任務處理的最小元素數
// 太小: 排程開銷大
// 太大: 負載不均

// 自動 grain size
tbb::parallel_for(
    tbb::blocked_range<size_t>(0, data.size()),  // 預設 grain size
    [&](const auto& r) { /* ... */ }
);

// 手動指定 grain size
tbb::parallel_for(
    tbb::blocked_range<size_t>(0, data.size(), 1000),  // grain size = 1000
    [&](const auto& r) { /* ... */ }
);

// 經驗法則: grain size 應讓每個任務執行 10,000+ 個 CPU cycles
constexpr size_t GRAIN_SIZE = 10000 / CYCLES_PER_ELEMENT;
```

### HFT 應用: 批次訂單處理

```cpp
#include <tbb/parallel_for.h>

struct Order {
    uint64_t order_id;
    uint32_t symbol_id;
    double price;
    int64_t quantity;
};

class OrderValidator {
public:
    // 並行驗證大量訂單
    void validate_orders(std::vector<Order>& orders) {
        tbb::parallel_for(
            tbb::blocked_range<size_t>(0, orders.size(), 100),  // grain size = 100
            [&](const tbb::blocked_range<size_t>& r) {
                for (size_t i = r.begin(); i != r.end(); ++i) {
                    validate_single_order(orders[i]);
                }
            }
        );
    }
    
private:
    void validate_single_order(Order& order) {
        // 檢查價格、數量、帳戶餘額等
        if (order.price <= 0 || order.quantity <= 0) {
            order.order_id = 0;  // 標記為無效
        }
    }
};
```

## parallel_reduce - 並行規約

### 求和範例

```cpp
#include <tbb/parallel_reduce.h>
#include <tbb/blocked_range.h>

// 計算陣列總和
double parallel_sum(const std::vector<double>& data) {
    return tbb::parallel_reduce(
        tbb::blocked_range<size_t>(0, data.size()),
        0.0,  // 初始值
        
        // 局部規約 (每個執行緒)
        [&](const tbb::blocked_range<size_t>& r, double init) -> double {
            for (size_t i = r.begin(); i != r.end(); ++i) {
                init += data[i];
            }
            return init;
        },
        
        // 合併結果
        [](double a, double b) -> double {
            return a + b;
        }
    );
}
```

### 複雜規約: 尋找最大值與索引

```cpp
struct MaxResult {
    double value = -std::numeric_limits<double>::infinity();
    size_t index = 0;
};

MaxResult find_max_parallel(const std::vector<double>& data) {
    return tbb::parallel_reduce(
        tbb::blocked_range<size_t>(0, data.size()),
        MaxResult{},
        
        // 局部規約
        [&](const tbb::blocked_range<size_t>& r, MaxResult init) -> MaxResult {
            for (size_t i = r.begin(); i != r.end(); ++i) {
                if (data[i] > init.value) {
                    init.value = data[i];
                    init.index = i;
                }
            }
            return init;
        },
        
        // 合併
        [](MaxResult a, MaxResult b) -> MaxResult {
            return a.value > b.value ? a : b;
        }
    );
}
```

### HFT 應用: 計算 VWAP

```cpp
#include <tbb/parallel_reduce.h>

struct Trade {
    double price;
    int64_t volume;
};

struct VWAPAccumulator {
    double price_volume_sum = 0.0;
    int64_t total_volume = 0;
    
    void operator+=(const VWAPAccumulator& other) {
        price_volume_sum += other.price_volume_sum;
        total_volume += other.total_volume;
    }
    
    double vwap() const {
        return total_volume > 0 ? price_volume_sum / total_volume : 0.0;
    }
};

double calculate_vwap_parallel(const std::vector<Trade>& trades) {
    auto result = tbb::parallel_reduce(
        tbb::blocked_range<size_t>(0, trades.size()),
        VWAPAccumulator{},
        
        // 局部累積
        [&](const tbb::blocked_range<size_t>& r, VWAPAccumulator acc) {
            for (size_t i = r.begin(); i != r.end(); ++i) {
                acc.price_volume_sum += trades[i].price * trades[i].volume;
                acc.total_volume += trades[i].volume;
            }
            return acc;
        },
        
        // 合併
        [](VWAPAccumulator a, VWAPAccumulator b) {
            a += b;
            return a;
        }
    );
    
    return result.vwap();
}
```

## parallel_invoke - 並行執行異質任務

### 基礎用法

```cpp
#include <tbb/parallel_invoke.h>

void process_market_data() {
    std::vector<double> prices, volumes;
    
    // 並行執行三個獨立任務
    tbb::parallel_invoke(
        [&] { load_prices(prices); },
        [&] { load_volumes(volumes); },
        [&] { load_metadata(); }
    );
    
    // 所有任務完成後繼續
    process_data(prices, volumes);
}
```

### 遞迴並行: 快速排序

```cpp
#include <tbb/parallel_invoke.h>
#include <algorithm>

template<typename RandomIt>
void parallel_quicksort(RandomIt first, RandomIt last) {
    constexpr size_t SEQUENTIAL_THRESHOLD = 1000;
    
    if (std::distance(first, last) < SEQUENTIAL_THRESHOLD) {
        std::sort(first, last);  // 小範圍用循序排序
        return;
    }
    
    auto pivot = *std::next(first, std::distance(first, last) / 2);
    auto middle1 = std::partition(first, last, [pivot](const auto& x) {
        return x < pivot;
    });
    auto middle2 = std::partition(middle1, last, [pivot](const auto& x) {
        return !(pivot < x);
    });
    
    // 並行遞迴
    tbb::parallel_invoke(
        [=] { parallel_quicksort(first, middle1); },
        [=] { parallel_quicksort(middle2, last); }
    );
}
```

## parallel_pipeline - 流水線並行

### 三階段流水線

```cpp
#include <tbb/parallel_pipeline.h>

struct RawData { std::string content; };
struct ParsedData { std::vector<double> values; };
struct ProcessedData { double result; };

void pipeline_example() {
    tbb::parallel_pipeline(
        8,  // 最大並行度
        
        // Stage 1: 輸入 (串列)
        tbb::make_filter<void, RawData>(
            tbb::filter_mode::serial_in_order,
            [](tbb::flow_control& fc) -> RawData {
                auto data = read_next_chunk();
                if (data.empty()) {
                    fc.stop();  // 無資料,停止流水線
                    return {};
                }
                return {data};
            }
        ) &
        
        // Stage 2: 解析 (並行)
        tbb::make_filter<RawData, ParsedData>(
            tbb::filter_mode::parallel,
            [](RawData raw) -> ParsedData {
                return parse_data(raw.content);
            }
        ) &
        
        // Stage 3: 輸出 (串列)
        tbb::make_filter<ParsedData, void>(
            tbb::filter_mode::serial_in_order,
            [](ParsedData parsed) {
                write_results(parsed.values);
            }
        )
    );
}
```

### HFT 應用: 市場資料流水線

```cpp
#include <tbb/parallel_pipeline.h>

struct RawMarketData {
    uint8_t buffer[1024];
    size_t size;
};

struct DecodedData {
    uint32_t symbol_id;
    double price;
    int64_t volume;
};

void market_data_pipeline() {
    tbb::parallel_pipeline(
        16,  // 最大並行度
        
        // Stage 1: 接收網路封包 (串列)
        tbb::make_filter<void, RawMarketData>(
            tbb::filter_mode::serial_in_order,
            [](tbb::flow_control& fc) -> RawMarketData {
                RawMarketData raw;
                if (!receive_packet(raw.buffer, raw.size)) {
                    fc.stop();
                    return {};
                }
                return raw;
            }
        ) &
        
        // Stage 2: 解碼 (並行)
        tbb::make_filter<RawMarketData, DecodedData>(
            tbb::filter_mode::parallel,
            [](RawMarketData raw) -> DecodedData {
                return decode_market_data(raw.buffer, raw.size);
            }
        ) &
        
        // Stage 3: 策略處理 (並行)
        tbb::make_filter<DecodedData, void>(
            tbb::filter_mode::parallel,
            [](DecodedData data) {
                update_order_book(data);
                execute_strategy(data);
            }
        )
    );
}
```

## task_arena - 執行緒池控制

### 限制並行度

```cpp
#include <tbb/task_arena.h>

// 建立專用執行緒池
tbb::task_arena arena(4);  // 4 個執行緒

arena.execute([&] {
    tbb::parallel_for(0, 1000, [](int i) {
        // 僅使用 4 個執行緒
        process(i);
    });
});
```

### CPU 綁定 (Affinity)

```cpp
#include <tbb/task_arena.h>
#include <tbb/task_scheduler_observer.h>

// HFT: 將關鍵執行緒綁定到特定 CPU
class PinningObserver : public tbb::task_scheduler_observer {
    std::vector<int> cpus_;
    
public:
    PinningObserver(const std::vector<int>& cpus) 
        : cpus_(cpus) {
        observe(true);
    }
    
    void on_scheduler_entry(bool) override {
        // 綁定執行緒到指定 CPU
        cpu_set_t cpuset;
        CPU_ZERO(&cpuset);
        
        int slot = tbb::this_task_arena::current_thread_index();
        if (slot < cpus_.size()) {
            CPU_SET(cpus_[slot], &cpuset);
            pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);
        }
    }
};

// 使用
tbb::task_arena arena(2);
PinningObserver observer({0, 1});  // 綁定到 CPU 0, 1

arena.execute([&] {
    tbb::parallel_for(0, 1000, [](int i) {
        // 在 CPU 0, 1 上執行
        latency_sensitive_work(i);
    });
});
```

## concurrent_vector - 並行容器

### 基礎用法

```cpp
#include <tbb/concurrent_vector.h>

tbb::concurrent_vector<int> vec;

// 多執行緒安全的 push_back
tbb::parallel_for(0, 1000, [&](int i) {
    vec.push_back(i);  // 執行緒安全
});

// 成長過程中可安全讀取 (但不保證順序)
for (const auto& x : vec) {
    std::cout << x << " ";
}
```

### HFT 應用: 收集交易訊號

```cpp
#include <tbb/concurrent_vector.h>

struct Signal {
    uint64_t timestamp;
    uint32_t symbol_id;
    double score;
};

class SignalCollector {
    tbb::concurrent_vector<Signal> signals_;
    
public:
    // 多策略並行產生訊號
    void generate_signals(const std::vector<MarketData>& data) {
        tbb::parallel_for(
            tbb::blocked_range<size_t>(0, data.size()),
            [&](const auto& r) {
                for (size_t i = r.begin(); i != r.end(); ++i) {
                    auto signal = evaluate_strategy(data[i]);
                    
                    if (signal.score > threshold_) {
                        signals_.push_back(signal);  // 執行緒安全
                    }
                }
            }
        );
    }
    
    // 排序並執行前 N 個訊號
    void execute_top_signals(size_t n) {
        std::vector<Signal> sorted(signals_.begin(), signals_.end());
        std::partial_sort(sorted.begin(), sorted.begin() + n, sorted.end(),
                         [](const Signal& a, const Signal& b) {
                             return a.score > b.score;
                         });
        
        for (size_t i = 0; i < n; ++i) {
            execute_order(sorted[i]);
        }
    }
};
```

## concurrent_hash_map - 並行哈希表

```cpp
#include <tbb/concurrent_hash_map.h>

// 定義哈希函數
struct SymbolHashCompare {
    static size_t hash(uint32_t key) {
        return std::hash<uint32_t>{}(key);
    }
    
    static bool equal(uint32_t a, uint32_t b) {
        return a == b;
    }
};

using SymbolMap = tbb::concurrent_hash_map<uint32_t, double, SymbolHashCompare>;

// 並行更新
SymbolMap price_map;

tbb::parallel_for(0, updates.size(), [&](size_t i) {
    SymbolMap::accessor acc;
    
    if (price_map.insert(acc, updates[i].symbol_id)) {
        // 新插入
        acc->second = updates[i].price;
    } else {
        // 已存在,更新
        acc->second = updates[i].price;
    }
    // acc 解構時自動釋放鎖
});

// 並行讀取
tbb::parallel_for(0, queries.size(), [&](size_t i) {
    SymbolMap::const_accessor acc;
    
    if (price_map.find(acc, queries[i])) {
        std::cout << "Price: " << acc->second << "\n";
    }
});
```

## 效能調校

### 測量並行效能

```cpp
#include <tbb/tick_count.h>

void benchmark_parallel() {
    std::vector<double> data(10000000);
    
    // 循序版本
    auto t0 = tbb::tick_count::now();
    for (auto& x : data) {
        x = expensive_computation(x);
    }
    auto sequential_time = (tbb::tick_count::now() - t0).seconds();
    
    // 並行版本
    t0 = tbb::tick_count::now();
    tbb::parallel_for(size_t(0), data.size(), [&](size_t i) {
        data[i] = expensive_computation(data[i]);
    });
    auto parallel_time = (tbb::tick_count::now() - t0).seconds();
    
    std::cout << "Speedup: " << sequential_time / parallel_time << "x\n";
}
```

### 避免常見陷阱

```cpp
// 陷阱 1: False Sharing
// 錯誤: 多執行緒寫入相鄰記憶體
std::vector<int> counters(num_threads);  // 可能在同一 cache line

tbb::parallel_for(0, num_threads, [&](int i) {
    counters[i]++;  // False sharing!
});

// 修正: Cache-line 對齊
struct alignas(64) Counter {
    int value = 0;
};
std::vector<Counter> counters(num_threads);

// 陷阱 2: 過小的 Grain Size
tbb::parallel_for(0, 1000, [](int i) {
    ++global_counter;  // 排程開銷 > 計算
});

// 修正: 調整 grain size
tbb::parallel_for(
    tbb::blocked_range<int>(0, 1000, 100),  // grain size = 100
    [](const auto& r) {
        for (int i = r.begin(); i != r.end(); ++i) {
            ++global_counter;
        }
    }
);
```

## 檢查清單

- [ ] 優先使用 `parallel_for` 處理資料並行
- [ ] 使用 `parallel_reduce` 處理聚合操作
- [ ] 使用 `parallel_invoke` 處理異質任務
- [ ] 使用 `parallel_pipeline` 處理流式資料
- [ ] 調整 grain size 平衡排程開銷與負載均衡
- [ ] 使用 `concurrent_vector` / `concurrent_hash_map` 避免手動鎖
- [ ] 使用 `task_arena` 控制執行緒數量與 CPU affinity
- [ ] 避免 false sharing (使用 `alignas(64)`)
- [ ] 測量並行 speedup,確認效益
- [ ] HFT 系統考慮 CPU 綁定與 NUMA 配置

---

## 參考資料 (References)

1. [Intel TBB Documentation](https://www.intel.com/content/www/us/en/docs/onetbb/developer-guide/current/overview.html)
2. Reinders, James. *Intel Threading Building Blocks* (2007)
3. [oneTBB GitHub Repository](https://github.com/oneapi-src/oneTBB)
4. [TBB Tutorial](https://www.threadingbuildingblocks.org/tutorial-intel-tbb-tutorial)
5. Robison, Arch D., et al. *Structured Parallel Programming* (2012)
