# 任務並行: Intel TBB

> **優先級**: ⭐⭐ 建議
> **適用場景**: 高性能計算 / 數據並行處理 / 流水線架構
> **前置知識**: C++ Lambda 表達式, 多線程基礎

## 目錄

- [核心概念](#核心概念)
- [並行迴圈 (parallel_for)](#並行迴圈-parallel_for)
- [並行規約 (parallel_reduce)](#並行規約-parallel_reduce)
- [流水線並行 (parallel_pipeline)](#流水線並行-parallel_pipeline)
- [並行容器](#並行容器)
- [任務調度與優化](#任務調度與優化)
- [HFT 實戰案例](#hft-實戰案例)
- [參考資料](#參考資料)

## 核心概念

### 1. 為什麼選擇 TBB?

**Intel Threading Building Blocks (TBB)** 是一個 C++ 模板庫,專注於**任務 (Task)** 而非執行緒。

- **抽象層次高**: 開發者只需描述"做什麼" (任務),而非"怎麼做" (執行緒管理)。
- **工作竊取 (Work Stealing)**: 自動平衡負載,空閒的執行緒會主動從繁忙執行緒的隊列中竊取任務。
- **可組合性**: TBB 組件可以無縫組合,不會導致過度訂閱 (Over-subscription)。

### 2. 任務 vs 執行緒

| 特性 | 原始執行緒 (std::thread) | TBB 任務 |
|------|--------------------------|----------|
| **創建開銷** | 高 (系統調用) | 極低 (用戶空間) |
| **調度方式** | OS 搶佔式調度 | 用戶空間協作式調度 |
| **負載平衡** | 手動管理 | 自動 (工作竊取) |
| **適用粒度** | 粗粒度 (毫秒級) | 細粒度 (微秒級) |

---

## 並行迴圈 (parallel_for)

最常用的並行模式,適用於數據並行。

### 1. 基礎用法

```cpp
#include <tbb/parallel_for.h>
#include <vector>
#include <cmath>

void process_data(std::vector<double>& data) {
    // 自動將迴圈拆分為多個任務並行執行
    tbb::parallel_for(size_t(0), data.size(), [&](size_t i) {
        data[i] = std::sqrt(data[i]) * 2.0;
    });
}
```

### 2. Blocked Range 與 Grain Size

為了減少排程開銷,TBB 將迭代空間切分為塊 (Block)。

```cpp
#include <tbb/blocked_range.h>

void process_optimized(std::vector<double>& data) {
    // Grain Size = 1000: 每個任務至少處理 1000 個元素
    tbb::parallel_for(
        tbb::blocked_range<size_t>(0, data.size(), 1000),
        [&](const tbb::blocked_range<size_t>& r) {
            // 內部使用普通迴圈處理一個塊
            for (size_t i = r.begin(); i != r.end(); ++i) {
                data[i] = std::sqrt(data[i]) * 2.0;
            }
        }
    );
}
```

> [!TIP]
> **Grain Size 選擇原則**: 每個任務的執行時間應至少為 10,000-100,000 個 CPU 週期,以攤銷排程開銷。

---

## 並行規約 (parallel_reduce)

用於將並行計算的結果匯總 (如求和、找最大值)。

### 1. 並行求和

```cpp
#include <tbb/parallel_reduce.h>

double parallel_sum(const std::vector<double>& data) {
    return tbb::parallel_reduce(
        tbb::blocked_range<size_t>(0, data.size()),
        0.0, // 初始值 (Identity)
        
        // 1. 局部規約 (Reduction)
        [&](const tbb::blocked_range<size_t>& r, double local_sum) {
            for (size_t i = r.begin(); i != r.end(); ++i) {
                local_sum += data[i];
            }
            return local_sum;
        },
        
        // 2. 合併結果 (Join)
        [](double a, double b) {
            return a + b;
        }
    );
}
```

---

## 流水線並行 (parallel_pipeline)

適用於多階段處理,且各階段吞吐量不一致的場景。

```mermaid
graph LR
    Input[輸入 (串列)] -->|Buffer| Decode[解碼 (並行)]
    Decode -->|Buffer| Process[處理 (並行)]
    Process -->|Buffer| Output[輸出 (串列)]
```

```cpp
#include <tbb/parallel_pipeline.h>

void market_data_pipeline() {
    tbb::parallel_pipeline(
        16, // 最大並行 token 數 (In-flight tasks)
        
        // Stage 1: 接收數據 (串列)
        tbb::make_filter<void, RawData>(
            tbb::filter_mode::serial_in_order,
            [](tbb::flow_control& fc) -> RawData {
                auto data = receive_packet();
                if (!data) fc.stop();
                return *data;
            }
        ) &
        
        // Stage 2: 解碼 (並行)
        tbb::make_filter<RawData, DecodedData>(
            tbb::filter_mode::parallel,
            [](RawData raw) {
                return decode(raw);
            }
        ) &
        
        // Stage 3: 寫入日誌 (串列)
        tbb::make_filter<DecodedData, void>(
            tbb::filter_mode::serial_in_order,
            [](DecodedData data) {
                write_log(data);
            }
        )
    );
}
```

---

## 並行容器

TBB 提供了線程安全的容器,支援高並發訪問。

### 1. concurrent_vector

動態增長的數組,支援並發 `push_back`。

```cpp
#include <tbb/concurrent_vector.h>

tbb::concurrent_vector<int> vec;

tbb::parallel_for(0, 10000, [&](int i) {
    vec.push_back(i); // 安全, 無需鎖
});
```

### 2. concurrent_hash_map

細粒度鎖的哈希表,支援高並發讀寫。

```cpp
#include <tbb/concurrent_hash_map.h>

tbb::concurrent_hash_map<int, int> map;

// 並發插入/更新
tbb::parallel_for(0, 1000, [&](int i) {
    tbb::concurrent_hash_map<int, int>::accessor acc;
    if (map.insert(acc, i)) {
        acc->second = 1;
    } else {
        acc->second += 1;
    }
}); // acc 解構時釋放鎖
```

---

## 任務調度與優化

### 1. task_arena (隔離與控制)

用於限制並發度或綁定 CPU。

```cpp
#include <tbb/task_arena.h>

// 限制只使用 2 個執行緒
tbb::task_arena limited_arena(2);

limited_arena.execute([&] {
    tbb::parallel_for(0, 1000, [](int i) {
        // 這裡只會由 2 個執行緒執行
        heavy_work(i);
    });
});
```

### 2. CPU 親和性 (Affinity)

在 HFT 中,通常需要將關鍵任務綁定到特定 CPU 核心。TBB 可以通過 `task_scheduler_observer` 實現。

```cpp
class PinningObserver : public tbb::task_scheduler_observer {
public:
    void on_scheduler_entry(bool) override {
        // 獲取當前執行緒索引並綁定 CPU
        int slot = tbb::this_task_arena::current_thread_index();
        pin_thread_to_cpu(slot);
    }
};
```

---

## HFT 實戰案例

### 批次訂單驗證

```cpp
struct Order {
    int id;
    double price;
    int quantity;
    bool valid;
};

void validate_orders_parallel(std::vector<Order>& orders) {
    // 使用 parallel_for 並行驗證
    // Grain size 設為 100, 避免過細粒度
    tbb::parallel_for(
        tbb::blocked_range<size_t>(0, orders.size(), 100),
        [&](const tbb::blocked_range<size_t>& r) {
            for (size_t i = r.begin(); i != r.end(); ++i) {
                // 複雜驗證邏輯
                orders[i].valid = check_risk_limits(orders[i]) && 
                                  check_account_balance(orders[i]);
            }
        }
    );
}
```

### VWAP 計算 (Volume Weighted Average Price)

```cpp
struct Trade { double price; int volume; };

double calculate_vwap(const std::vector<Trade>& trades) {
    struct Accumulator {
        double total_pv = 0.0;
        long total_vol = 0;
    };

    auto result = tbb::parallel_reduce(
        tbb::blocked_range<size_t>(0, trades.size()),
        Accumulator{},
        [](const tbb::blocked_range<size_t>& r, Accumulator acc) {
            for (size_t i = r.begin(); i != r.end(); ++i) {
                acc.total_pv += trades[i].price * trades[i].volume;
                acc.total_vol += trades[i].volume;
            }
            return acc;
        },
        [](Accumulator a, Accumulator b) {
            a.total_pv += b.total_pv;
            a.total_vol += b.total_vol;
            return a;
        }
    );

    return result.total_vol ? result.total_pv / result.total_vol : 0.0;
}
```

---

## 參考資料

1. [oneTBB Documentation](https://oneapi-src.github.io/oneTBB/)
2. [Intel TBB Tutorial](https://www.threadingbuildingblocks.org/tutorial-intel-tbb-tutorial)
3. *Structured Parallel Programming* - Michael McCool et al.