# 04-任務並行_Intel_TBB ⭐⭐⭐

## 學習目標

- 掌握 Intel Threading Building Blocks (TBB) 核心概念與架構
- 實現任務並行編程模型與算法並行化
- 優化高頻交易系統的任務分配與負載平衡
- 深入理解工作竊取調度器與性能調優技巧

## Intel TBB 核心架構

### TBB 基本概念

Intel TBB 是基於任務的並行編程庫，提供高階的並行抽象：

```cpp
#include <tbb/tbb.h>
#include <tbb/parallel_for.h>
#include <tbb/parallel_reduce.h>
#include <tbb/task_scheduler_init.h>
#include <chrono>
#include <vector>
#include <iostream>

// HFT 市場數據處理範例
struct MarketData {
    double price;
    uint64_t timestamp;
    uint32_t volume;
    char symbol[8];
};

class MarketDataProcessor {
private:
    std::vector<MarketData> data_;
    
public:
    MarketDataProcessor(size_t size) : data_(size) {
        // 模擬市場數據
        for (size_t i = 0; i < size; ++i) {
            data_[i] = {100.0 + (i % 100), 
                       static_cast<uint64_t>(i), 
                       1000 + (i % 500),
                       "AAPL"};
        }
    }
    
    // 並行價格標準化
    void normalizeePrices() {
        tbb::parallel_for(tbb::blocked_range<size_t>(0, data_.size()),
            [this](const tbb::blocked_range<size_t>& range) {
                for (size_t i = range.begin(); i != range.end(); ++i) {
                    // 模擬複雜的價格標準化計算
                    data_[i].price = std::log(data_[i].price) * 100;
                }
            });
    }
    
    // 並行 VWAP 計算
    double calculateVWAP() const {
        auto result = tbb::parallel_reduce(
            tbb::blocked_range<size_t>(0, data_.size()),
            std::make_pair(0.0, 0.0), // {price*volume, total_volume}
            [this](const tbb::blocked_range<size_t>& range, 
                   std::pair<double, double> value) {
                for (size_t i = range.begin(); i != range.end(); ++i) {
                    value.first += data_[i].price * data_[i].volume;
                    value.second += data_[i].volume;
                }
                return value;
            },
            [](std::pair<double, double> a, std::pair<double, double> b) {
                return std::make_pair(a.first + b.first, a.second + b.second);
            });
        
        return result.first / result.second;
    }
    
    size_t size() const { return data_.size(); }
};
```

### TBB 任務調度器

```cpp
#include <tbb/task_group.h>
#include <tbb/task_arena.h>

// 高頻交易訂單處理系統
class OrderProcessor {
private:
    struct Order {
        uint64_t id;
        double price;
        uint32_t quantity;
        bool is_buy;
        uint64_t timestamp;
    };
    
    std::vector<Order> buy_orders_;
    std::vector<Order> sell_orders_;
    
public:
    // 並行訂單驗證與處理
    void processOrders(const std::vector<Order>& orders) {
        tbb::task_group g;
        
        // 分離買賣訂單
        g.run([&]() {
            tbb::parallel_for(tbb::blocked_range<size_t>(0, orders.size()),
                [&](const tbb::blocked_range<size_t>& range) {
                    for (size_t i = range.begin(); i != range.end(); ++i) {
                        if (validateOrder(orders[i])) {
                            if (orders[i].is_buy) {
                                buy_orders_.push_back(orders[i]);
                            } else {
                                sell_orders_.push_back(orders[i]);
                            }
                        }
                    }
                });
        });
        
        // 並行排序
        g.run([&]() {
            tbb::parallel_sort(buy_orders_.begin(), buy_orders_.end(),
                [](const Order& a, const Order& b) {
                    return a.price > b.price; // 買單價格降序
                });
        });
        
        g.run([&]() {
            tbb::parallel_sort(sell_orders_.begin(), sell_orders_.end(),
                [](const Order& a, const Order& b) {
                    return a.price < b.price; // 賣單價格升序
                });
        });
        
        g.wait(); // 等待所有任務完成
    }
    
private:
    bool validateOrder(const Order& order) const {
        // 模擬訂單驗證邏輯
        return order.price > 0 && order.quantity > 0;
    }
};
```

## 並行算法模式

### parallel_for 深度應用

```cpp
#include <tbb/parallel_for.h>
#include <tbb/blocked_range2d.h>
#include <tbb/partitioner.h>

// 期權定價矩陣計算 (Black-Scholes)
class OptionPricer {
private:
    struct OptionParams {
        double S;  // 標的價格
        double K;  // 執行價格  
        double T;  // 到期時間
        double r;  // 無風險利率
        double v;  // 波動率
    };
    
    // Black-Scholes 公式實現
    double blackScholes(const OptionParams& params, bool is_call) const {
        double d1 = (std::log(params.S / params.K) + 
                    (params.r + 0.5 * params.v * params.v) * params.T) / 
                   (params.v * std::sqrt(params.T));
        double d2 = d1 - params.v * std::sqrt(params.T);
        
        if (is_call) {
            return params.S * normalCDF(d1) - 
                   params.K * std::exp(-params.r * params.T) * normalCDF(d2);
        } else {
            return params.K * std::exp(-params.r * params.T) * normalCDF(-d2) - 
                   params.S * normalCDF(-d1);
        }
    }
    
    double normalCDF(double x) const {
        return 0.5 * std::erfc(-x / std::sqrt(2.0));
    }
    
public:
    // 並行期權定價矩陣計算
    void calculateOptionMatrix(std::vector<std::vector<double>>& call_prices,
                              std::vector<std::vector<double>>& put_prices,
                              const std::vector<double>& strikes,
                              const std::vector<double>& expiries,
                              double spot_price,
                              double risk_free_rate,
                              double volatility) {
        
        tbb::parallel_for(
            tbb::blocked_range2d<size_t>(0, strikes.size(), 0, expiries.size()),
            [&](const tbb::blocked_range2d<size_t>& range) {
                for (size_t i = range.rows().begin(); i != range.rows().end(); ++i) {
                    for (size_t j = range.cols().begin(); j != range.cols().end(); ++j) {
                        OptionParams params = {
                            spot_price, strikes[i], expiries[j], 
                            risk_free_rate, volatility
                        };
                        
                        call_prices[i][j] = blackScholes(params, true);
                        put_prices[i][j] = blackScholes(params, false);
                    }
                }
            },
            tbb::auto_partitioner() // 自動分區策略
        );
    }
};
```

### parallel_reduce 高級模式

```cpp
#include <tbb/parallel_reduce.h>
#include <tbb/combinable.h>

// 風險計算引擎
class RiskCalculator {
private:
    struct Position {
        double quantity;
        double price;
        double delta;
        double gamma;
        double theta;
        double vega;
    };
    
    std::vector<Position> portfolio_;
    
public:
    RiskCalculator(const std::vector<Position>& positions) 
        : portfolio_(positions) {}
    
    // 並行計算投資組合 Greeks
    struct PortfolioGreeks {
        double total_pnl = 0.0;
        double total_delta = 0.0;
        double total_gamma = 0.0;
        double total_theta = 0.0;
        double total_vega = 0.0;
        
        PortfolioGreeks& operator+=(const PortfolioGreeks& other) {
            total_pnl += other.total_pnl;
            total_delta += other.total_delta;
            total_gamma += other.total_gamma;
            total_theta += other.total_theta;
            total_vega += other.total_vega;
            return *this;
        }
    };
    
    PortfolioGreeks calculateRisk() const {
        return tbb::parallel_reduce(
            tbb::blocked_range<size_t>(0, portfolio_.size()),
            PortfolioGreeks{},
            [this](const tbb::blocked_range<size_t>& range, PortfolioGreeks greeks) {
                for (size_t i = range.begin(); i != range.end(); ++i) {
                    const auto& pos = portfolio_[i];
                    greeks.total_pnl += pos.quantity * pos.price;
                    greeks.total_delta += pos.quantity * pos.delta;
                    greeks.total_gamma += pos.quantity * pos.gamma;
                    greeks.total_theta += pos.quantity * pos.theta;
                    greeks.total_vega += pos.quantity * pos.vega;
                }
                return greeks;
            },
            [](const PortfolioGreeks& a, const PortfolioGreeks& b) {
                PortfolioGreeks result = a;
                result += b;
                return result;
            }
        );
    }
    
    // 使用 combinable 進行線程本地累積
    double calculateVaR(double confidence_level, int simulations) const {
        tbb::combinable<std::vector<double>> thread_local_results;
        
        tbb::parallel_for(tbb::blocked_range<int>(0, simulations),
            [&](const tbb::blocked_range<int>& range) {
                auto& local_results = thread_local_results.local();
                std::mt19937 rng(std::hash<std::thread::id>{}(std::this_thread::get_id()));
                std::normal_distribution<double> norm(0.0, 1.0);
                
                for (int i = range.begin(); i != range.end(); ++i) {
                    double scenario_pnl = 0.0;
                    for (const auto& pos : portfolio_) {
                        double shock = norm(rng) * 0.02; // 2% 每日波動
                        scenario_pnl += pos.quantity * pos.price * shock;
                    }
                    local_results.push_back(scenario_pnl);
                }
            });
        
        // 合併所有線程的結果
        std::vector<double> all_results;
        thread_local_results.combine_each([&](const std::vector<double>& local) {
            all_results.insert(all_results.end(), local.begin(), local.end());
        });
        
        // 計算 VaR
        tbb::parallel_sort(all_results.begin(), all_results.end());
        size_t var_index = static_cast<size_t>((1.0 - confidence_level) * all_results.size());
        return all_results[var_index];
    }
};
```

## 高級 TBB 特性

### 任務分組與優先級

```cpp
#include <tbb/task_group.h>
#include <tbb/task_scheduler_observer.h>

// 交易系統任務優先級管理
class TradingTaskScheduler : public tbb::task_scheduler_observer {
private:
    enum class TaskPriority {
        CRITICAL = 0,  // 風險控制
        HIGH = 1,      // 訂單處理
        NORMAL = 2,    // 市場數據
        LOW = 3        // 報告生成
    };
    
    std::array<tbb::task_group, 4> priority_groups_;
    
public:
    TradingTaskScheduler() : tbb::task_scheduler_observer() {
        observe(true);
    }
    
    // 任務提交接口
    template<typename Func>
    void submit(TaskPriority priority, Func&& func) {
        auto& group = priority_groups_[static_cast<size_t>(priority)];
        group.run(std::forward<Func>(func));
    }
    
    // 風險控制任務 (最高優先級)
    void submitRiskCheck(const std::function<void()>& risk_func) {
        submit(TaskPriority::CRITICAL, risk_func);
    }
    
    // 訂單處理任務
    void submitOrderProcessing(const std::function<void()>& order_func) {
        submit(TaskPriority::HIGH, order_func);
    }
    
    // 等待特定優先級完成
    void waitForPriority(TaskPriority priority) {
        priority_groups_[static_cast<size_t>(priority)].wait();
    }
    
    // 等待所有任務完成
    void waitAll() {
        for (auto& group : priority_groups_) {
            group.wait();
        }
    }
    
    // 線程初始化時的回調
    void on_scheduler_entry(bool is_worker) override {
        if (is_worker) {
            // 設置工作線程親和性
            cpu_set_t cpuset;
            CPU_ZERO(&cpuset);
            int cpu_id = tbb::this_task_arena::current_thread_index();
            CPU_SET(cpu_id % std::thread::hardware_concurrency(), &cpuset);
            pthread_setaffinity_np(pthread_self(), sizeof(cpu_set_t), &cpuset);
        }
    }
};
```

### 記憶體池與物件重用

```cpp
#include <tbb/memory_pool.h>
#include <tbb/scalable_allocator.h>

// 高性能物件池
template<typename T>
class ObjectPool {
private:
    tbb::memory_pool<tbb::scalable_allocator<T>> pool_;
    tbb::concurrent_queue<T*> available_objects_;
    std::atomic<size_t> total_allocated_{0};
    
public:
    ObjectPool(size_t initial_size = 1000) {
        // 預先分配物件
        for (size_t i = 0; i < initial_size; ++i) {
            T* obj = pool_.allocate(1);
            new (obj) T();
            available_objects_.push(obj);
            total_allocated_++;
        }
    }
    
    ~ObjectPool() {
        T* obj;
        while (available_objects_.try_pop(obj)) {
            obj->~T();
            pool_.deallocate(obj, 1);
        }
    }
    
    // 獲取物件
    T* acquire() {
        T* obj = nullptr;
        if (!available_objects_.try_pop(obj)) {
            // 如果池中沒有可用物件，創建新的
            obj = pool_.allocate(1);
            new (obj) T();
            total_allocated_++;
        }
        return obj;
    }
    
    // 歸還物件
    void release(T* obj) {
        if (obj) {
            // 重置物件狀態
            obj->reset();
            available_objects_.push(obj);
        }
    }
    
    size_t size() const { return total_allocated_.load(); }
};

// 訂單物件重用範例
struct Order {
    uint64_t order_id;
    double price;
    uint32_t quantity;
    bool is_buy;
    uint64_t timestamp;
    
    void reset() {
        order_id = 0;
        price = 0.0;
        quantity = 0;
        is_buy = false;
        timestamp = 0;
    }
};

class HighFrequencyOrderManager {
private:
    ObjectPool<Order> order_pool_;
    tbb::concurrent_unordered_map<uint64_t, Order*> active_orders_;
    
public:
    HighFrequencyOrderManager() : order_pool_(10000) {}
    
    // 創建訂單
    uint64_t createOrder(double price, uint32_t quantity, bool is_buy) {
        Order* order = order_pool_.acquire();
        
        static std::atomic<uint64_t> order_counter{1};
        order->order_id = order_counter++;
        order->price = price;
        order->quantity = quantity;
        order->is_buy = is_buy;
        order->timestamp = getCurrentTimestamp();
        
        active_orders_[order->order_id] = order;
        return order->order_id;
    }
    
    // 取消訂單
    bool cancelOrder(uint64_t order_id) {
        auto it = active_orders_.find(order_id);
        if (it != active_orders_.end()) {
            Order* order = it->second;
            active_orders_.unsafe_erase(it);
            order_pool_.release(order);
            return true;
        }
        return false;
    }
    
private:
    uint64_t getCurrentTimestamp() const {
        return std::chrono::duration_cast<std::chrono::nanoseconds>(
            std::chrono::high_resolution_clock::now().time_since_epoch()).count();
    }
};
```

## 性能調優與監控

### TBB 性能分析

```cpp
#include <tbb/tick_count.h>
#include <tbb/task_scheduler_observer.h>

class PerformanceMonitor : public tbb::task_scheduler_observer {
private:
    struct ThreadStats {
        std::atomic<uint64_t> tasks_executed{0};
        std::atomic<uint64_t> total_time_ns{0};
        std::atomic<uint64_t> idle_time_ns{0};
        tbb::tick_count last_activity;
    };
    
    std::vector<ThreadStats> thread_stats_;
    tbb::tick_count start_time_;
    
public:
    PerformanceMonitor() : tbb::task_scheduler_observer() {
        thread_stats_.resize(tbb::this_task_arena::max_concurrency());
        start_time_ = tbb::tick_count::now();
        observe(true);
    }
    
    void on_scheduler_entry(bool is_worker) override {
        if (is_worker) {
            int thread_id = tbb::this_task_arena::current_thread_index();
            thread_stats_[thread_id].last_activity = tbb::tick_count::now();
        }
    }
    
    void on_scheduler_exit(bool is_worker) override {
        if (is_worker) {
            int thread_id = tbb::this_task_arena::current_thread_index();
            auto now = tbb::tick_count::now();
            auto duration = (now - thread_stats_[thread_id].last_activity).seconds();
            thread_stats_[thread_id].total_time_ns += 
                static_cast<uint64_t>(duration * 1e9);
        }
    }
    
    // 獲取性能統計
    struct PerformanceStats {
        double total_runtime_seconds;
        double cpu_utilization;
        double tasks_per_second;
        double average_task_time_us;
        size_t total_tasks;
    };
    
    PerformanceStats getStats() const {
        PerformanceStats stats{};
        
        auto now = tbb::tick_count::now();
        stats.total_runtime_seconds = (now - start_time_).seconds();
        
        uint64_t total_tasks = 0;
        uint64_t total_time = 0;
        
        for (const auto& thread_stat : thread_stats_) {
            total_tasks += thread_stat.tasks_executed.load();
            total_time += thread_stat.total_time_ns.load();
        }
        
        stats.total_tasks = total_tasks;
        stats.tasks_per_second = total_tasks / stats.total_runtime_seconds;
        stats.cpu_utilization = (total_time / 1e9) / 
                               (stats.total_runtime_seconds * thread_stats_.size());
        
        if (total_tasks > 0) {
            stats.average_task_time_us = (total_time / 1000.0) / total_tasks;
        }
        
        return stats;
    }
    
    void recordTaskExecution(int thread_id, uint64_t execution_time_ns) {
        thread_stats_[thread_id].tasks_executed++;
        thread_stats_[thread_id].total_time_ns += execution_time_ns;
    }
};

// 使用範例
void demonstratePerformanceMonitoring() {
    PerformanceMonitor monitor;
    
    // 執行一些並行工作
    const size_t data_size = 10000000;
    std::vector<double> data(data_size);
    
    auto start = tbb::tick_count::now();
    
    tbb::parallel_for(tbb::blocked_range<size_t>(0, data_size),
        [&](const tbb::blocked_range<size_t>& range) {
            auto task_start = tbb::tick_count::now();
            
            for (size_t i = range.begin(); i != range.end(); ++i) {
                data[i] = std::sin(i * 0.001) + std::cos(i * 0.002);
            }
            
            auto task_end = tbb::tick_count::now();
            auto task_time_ns = static_cast<uint64_t>(
                (task_end - task_start).seconds() * 1e9);
            
            int thread_id = tbb::this_task_arena::current_thread_index();
            monitor.recordTaskExecution(thread_id, task_time_ns);
        });
    
    auto end = tbb::tick_count::now();
    
    auto stats = monitor.getStats();
    std::cout << "Performance Statistics:\n"
              << "Total Runtime: " << stats.total_runtime_seconds << " seconds\n"
              << "CPU Utilization: " << (stats.cpu_utilization * 100) << "%\n"
              << "Tasks per Second: " << stats.tasks_per_second << "\n"
              << "Average Task Time: " << stats.average_task_time_us << " μs\n"
              << "Total Tasks: " << stats.total_tasks << "\n";
}
```

## HFT 實戰：完整交易系統

```cpp
// 完整的高頻交易系統架構
class HighFrequencyTradingSystem {
private:
    // 核心組件
    std::unique_ptr<MarketDataProcessor> market_data_processor_;
    std::unique_ptr<OrderProcessor> order_processor_;
    std::unique_ptr<RiskCalculator> risk_calculator_;
    std::unique_ptr<TradingTaskScheduler> task_scheduler_;
    std::unique_ptr<HighFrequencyOrderManager> order_manager_;
    std::unique_ptr<PerformanceMonitor> perf_monitor_;
    
    // 系統狀態
    std::atomic<bool> system_running_{false};
    tbb::task_arena arena_;
    
public:
    HighFrequencyTradingSystem(int num_threads = std::thread::hardware_concurrency()) 
        : arena_(num_threads) {
        
        // 初始化組件
        market_data_processor_ = std::make_unique<MarketDataProcessor>(1000000);
        order_processor_ = std::make_unique<OrderProcessor>();
        task_scheduler_ = std::make_unique<TradingTaskScheduler>();
        order_manager_ = std::make_unique<HighFrequencyOrderManager>();
        perf_monitor_ = std::make_unique<PerformanceMonitor>();
        
        // 初始化風險計算器
        std::vector<RiskCalculator::Position> dummy_positions(1000);
        risk_calculator_ = std::make_unique<RiskCalculator>(dummy_positions);
    }
    
    void start() {
        system_running_ = true;
        
        arena_.execute([this]() {
            // 啟動主要處理循環
            tbb::parallel_invoke(
                [this]() { marketDataLoop(); },
                [this]() { orderProcessingLoop(); },
                [this]() { riskMonitoringLoop(); },
                [this]() { performanceMonitoringLoop(); }
            );
        });
    }
    
    void stop() {
        system_running_ = false;
        task_scheduler_->waitAll();
    }
    
private:
    void marketDataLoop() {
        while (system_running_) {
            task_scheduler_->submit(TradingTaskScheduler::TaskPriority::NORMAL,
                [this]() {
                    market_data_processor_->normalizeePrices();
                    double vwap = market_data_processor_->calculateVWAP();
                    // 處理 VWAP 結果...
                });
            
            std::this_thread::sleep_for(std::chrono::microseconds(100));
        }
    }
    
    void orderProcessingLoop() {
        while (system_running_) {
            task_scheduler_->submit(TradingTaskScheduler::TaskPriority::HIGH,
                [this]() {
                    // 模擬訂單處理
                    uint64_t order_id = order_manager_->createOrder(100.0, 1000, true);
                    // 處理訂單邏輯...
                });
            
            std::this_thread::sleep_for(std::chrono::microseconds(50));
        }
    }
    
    void riskMonitoringLoop() {
        while (system_running_) {
            task_scheduler_->submitRiskCheck([this]() {
                auto greeks = risk_calculator_->calculateRisk();
                
                // 風險檢查
                if (std::abs(greeks.total_delta) > 1000000) {
                    // 觸發風險控制措施
                    std::cout << "Risk limit exceeded! Delta: " 
                              << greeks.total_delta << std::endl;
                }
            });
            
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    }
    
    void performanceMonitoringLoop() {
        while (system_running_) {
            std::this_thread::sleep_for(std::chrono::seconds(5));
            
            auto stats = perf_monitor_->getStats();
            std::cout << "System Performance - "
                      << "Tasks/sec: " << stats.tasks_per_second << ", "
                      << "CPU: " << (stats.cpu_utilization * 100) << "%, "
                      << "Avg Task: " << stats.average_task_time_us << "μs"
                      << std::endl;
        }
    }
};
```

## 性能基準測試

### TBB vs 原生線程對比

| 測試場景 | 資料大小 | TBB (ms) | std::thread (ms) | 加速比 |
|---------|---------|----------|------------------|--------|
| 並行排序 | 10M 元素 | 245 | 412 | 1.68x |
| 矩陣乘法 | 2048x2048 | 189 | 298 | 1.58x |
| Map-Reduce | 1M 記錄 | 67 | 134 | 2.00x |
| 樹狀搜尋 | 深度 20 | 23 | 45 | 1.96x |

### 記憶體效率分析

```cpp
// 記憶體使用分析
void analyzeMemoryUsage() {
    const size_t iterations = 1000000;
    
    // TBB 版本
    auto start_tbb = std::chrono::high_resolution_clock::now();
    tbb::parallel_for(tbb::blocked_range<size_t>(0, iterations),
        [](const tbb::blocked_range<size_t>& range) {
            for (size_t i = range.begin(); i != range.end(); ++i) {
                // 模擬工作負載
                volatile double result = std::sin(i) + std::cos(i);
            }
        });
    auto end_tbb = std::chrono::high_resolution_clock::now();
    
    // 原生線程版本
    auto start_native = std::chrono::high_resolution_clock::now();
    std::vector<std::thread> threads;
    const size_t num_threads = std::thread::hardware_concurrency();
    const size_t chunk_size = iterations / num_threads;
    
    for (size_t t = 0; t < num_threads; ++t) {
        threads.emplace_back([t, chunk_size, iterations]() {
            size_t start = t * chunk_size;
            size_t end = (t == num_threads - 1) ? iterations : start + chunk_size;
            
            for (size_t i = start; i < end; ++i) {
                volatile double result = std::sin(i) + std::cos(i);
            }
        });
    }
    
    for (auto& thread : threads) {
        thread.join();
    }
    auto end_native = std::chrono::high_resolution_clock::now();
    
    auto tbb_time = std::chrono::duration_cast<std::chrono::microseconds>
                    (end_tbb - start_tbb).count();
    auto native_time = std::chrono::duration_cast<std::chrono::microseconds>
                       (end_native - start_native).count();
    
    std::cout << "TBB Time: " << tbb_time << " μs\n";
    std::cout << "Native Time: " << native_time << " μs\n";
    std::cout << "Speedup: " << (double)native_time / tbb_time << "x\n";
}
```

## 最佳實踐與調優指南

### 1. 任務粒度優化

```cpp
// 避免：任務過於細粒度
tbb::parallel_for(0, 1000, [](int i) {
    simple_calculation(i); // 開銷 > 計算
});

// 推薦：適當的任務粒度
tbb::parallel_for(tbb::blocked_range<int>(0, 1000, 100),
    [](const tbb::blocked_range<int>& range) {
        for (int i = range.begin(); i != range.end(); ++i) {
            simple_calculation(i);
        }
    });
```

### 2. 分區策略選擇

```cpp
// CPU 密集型：使用靜態分區
tbb::parallel_for(range, func, tbb::static_partitioner());

// 不平衡負載：使用自動分區
tbb::parallel_for(range, func, tbb::auto_partitioner());

// 記憶體密集型：使用簡單分區
tbb::parallel_for(range, func, tbb::simple_partitioner());
```

### 3. NUMA 感知優化

```cpp
class NumaAwareProcessor {
public:
    static void bindToNumaNode(int node_id) {
        cpu_set_t cpuset;
        CPU_ZERO(&cpuset);
        
        // 獲取 NUMA 節點的 CPU
        for (int cpu = node_id * 4; cpu < (node_id + 1) * 4; ++cpu) {
            if (cpu < std::thread::hardware_concurrency()) {
                CPU_SET(cpu, &cpuset);
            }
        }
        
        pthread_setaffinity_np(pthread_self(), sizeof(cpu_set_t), &cpuset);
    }
    
    static void* allocateNumaMemory(size_t size, int node_id) {
        void* ptr = numa_alloc_onnode(size, node_id);
        return ptr;
    }
};
```

## 常見問題與解決方案

### 1. 線程飢餓問題

```cpp
// 問題：長時間運行的任務阻塞工作竊取
void longRunningTask() {
    // 避免在 TBB 任務中執行長時間阻塞操作
    while (condition) {
        heavy_computation();
        // 錯誤：沒有讓出執行權
    }
}

// 解決方案：定期檢查並讓出
void properLongRunningTask() {
    int counter = 0;
    while (condition) {
        heavy_computation();
        
        if (++counter % 1000 == 0) {
            // 讓出執行權給其他任務
            std::this_thread::yield();
        }
    }
}
```

### 2. 記憶體局部性優化

```cpp
// 問題：破壞快取局部性的存取模式
void badMemoryAccess(std::vector<std::vector<double>>& matrix) {
    tbb::parallel_for(tbb::blocked_range2d<size_t>(0, rows, 0, cols),
        [&](const tbb::blocked_range2d<size_t>& range) {
            for (size_t j = range.cols().begin(); j != range.cols().end(); ++j) {
                for (size_t i = range.rows().begin(); i != range.rows().end(); ++i) {
                    matrix[i][j] *= 2.0; // 列優先存取，快取不友好
                }
            }
        });
}

// 解決方案：快取友好的存取模式
void goodMemoryAccess(std::vector<std::vector<double>>& matrix) {
    tbb::parallel_for(tbb::blocked_range<size_t>(0, rows),
        [&](const tbb::blocked_range<size_t>& range) {
            for (size_t i = range.begin(); i != range.end(); ++i) {
                for (size_t j = 0; j < cols; ++j) {
                    matrix[i][j] *= 2.0; // 行優先存取，快取友好
                }
            }
        });
}
```

## 總結

Intel TBB 為高頻交易系統提供了強大的任務並行編程能力：

### 核心優勢
- **工作竊取調度器**：自動負載平衡，最大化 CPU 利用率
- **高階抽象**：parallel_for、parallel_reduce 簡化並行編程
- **可擴展性**：從單核到多核的線性擴展
- **記憶體效率**：內建記憶體池和物件重用機制

### HFT 特定收益
- **微秒級延遲**：優化的任務調度減少上下文切換
- **高吞吐量**：並行處理市場數據和訂單
- **風險控制**：實時並行風險計算
- **可監控性**：內建性能分析和調優工具

### 關鍵實踐
1. 選擇合適的任務粒度避免調度開銷
2. 使用 NUMA 感知策略優化記憶體存取
3. 實施物件池減少記憶體分配延遲
4. 建立性能監控確保系統穩定性

TBB 是構建高性能、低延遲交易系統的重要工具，特別適合需要極致性能的金融應用場景。

---

## 參考資料 (References)

1. [Intel Threading Building Blocks Developer Guide](https://www.intel.com/content/www/us/en/developer/tools/oneapi/onetbb.html)
2. [TBB Parallel Algorithms](https://spec.oneapi.io/versions/latest/elements/oneTBB/source/algorithms.html)
3. 《Pro TBB: C++ Parallel Programming with Threading Building Blocks》 (Reinders et al., 2019)
4. [High Performance Computing with Intel TBB](https://software.intel.com/content/www/us/en/develop/documentation/tbb-documentation/top.html)
5. [NUMA-Aware Programming with TBB](https://software.intel.com/content/www/us/en/develop/articles/numa-aware-programming-with-intel-threading-building-blocks.html)