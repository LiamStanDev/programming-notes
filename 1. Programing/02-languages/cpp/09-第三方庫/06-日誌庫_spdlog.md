# 日誌庫 - spdlog

## 概述

spdlog 是高效能、header-only 的 C++ 日誌庫，提供簡潔 API 與豐富功能。廣泛應用於需要低延遲日誌記錄的系統，如高頻交易、遊戲引擎、實時系統。

### 核心特點

1. **極致效能**: 異步日誌、零拷貝、記憶體池
2. **Header-Only**: 可選編譯版本
3. **格式化**: 支援 fmt 庫語法
4. **多 Sink**: 同時輸出到多個目標 (檔案、console、網路)
5. **執行緒安全**: 內建同步機制

### 安裝與整合

```bash
# Header-only 模式 (最簡單)
git clone https://github.com/gabime/spdlog.git
# 將 include/spdlog 複製到專案

# 或使用套件管理器
sudo apt-get install libspdlog-dev  # Ubuntu

# vcpkg
vcpkg install spdlog

# conan
conan install spdlog/1.12.0@
```

CMake 整合:

```cmake
# Header-only
include_directories(path/to/spdlog/include)

# 或使用編譯版本 (更快的編譯速度)
find_package(spdlog REQUIRED)
target_link_libraries(your_target spdlog::spdlog)
```

---

## 基本使用

### 第一個日誌

```cpp
#include <spdlog/spdlog.h>

int main() {
    // 直接使用預設 logger
    spdlog::info("Hello, {}!", "spdlog");
    spdlog::warn("Warning message");
    spdlog::error("Error message");
    spdlog::critical("Critical error");
    
    // 格式化輸出
    int value = 42;
    spdlog::info("The answer is {}", value);
    
    // 多個參數
    spdlog::info("{} + {} = {}", 1, 2, 3);
    
    return 0;
}
```

輸出:

```
[2024-01-15 10:30:45.123] [info] Hello, spdlog!
[2024-01-15 10:30:45.124] [warning] Warning message
[2024-01-15 10:30:45.125] [error] Error message
[2024-01-15 10:30:45.126] [critical] Critical error
[2024-01-15 10:30:45.127] [info] The answer is 42
[2024-01-15 10:30:45.128] [info] 1 + 2 = 3
```

### 日誌等級

```cpp
#include <spdlog/spdlog.h>

void log_level_examples() {
    // 設定全局日誌等級
    spdlog::set_level(spdlog::level::debug);
    
    // 各級別日誌
    spdlog::trace("Trace message");      // 最詳細
    spdlog::debug("Debug message");
    spdlog::info("Info message");
    spdlog::warn("Warning message");
    spdlog::error("Error message");
    spdlog::critical("Critical message"); // 最嚴重
    
    // 動態調整等級
    spdlog::set_level(spdlog::level::warn);
    
    spdlog::info("This won't show");   // 被過濾
    spdlog::warn("This will show");    // 顯示
}
```

### 自定義 Logger

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/sinks/stdout_color_sinks.h>

int main() {
    // 創建 console logger
    auto console = spdlog::stdout_color_mt("console");
    
    console->info("Hello from custom logger");
    console->warn("Warning message");
    
    // 通過名稱獲取 logger
    auto logger = spdlog::get("console");
    if (logger) {
        logger->error("Error message");
    }
    
    return 0;
}
```

---

## Sink 系統

### Console Sink

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/sinks/stdout_color_sinks.h>

void console_sink_examples() {
    // 彩色 console (多執行緒安全)
    auto console_mt = spdlog::stdout_color_mt("console_mt");
    
    // 彩色 console (單執行緒)
    auto console_st = spdlog::stdout_color_st("console_st");
    
    // stderr
    auto error_logger = spdlog::stderr_color_mt("stderr");
    
    console_mt->info("Info message");
    console_mt->warn("Warning message");
    console_mt->error("Error message");
}
```

### 檔案 Sink

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/sinks/basic_file_sink.h>

void file_sink_examples() {
    // 基本檔案 logger (多執行緒安全)
    auto file_logger = spdlog::basic_logger_mt("file_logger", "logs/basic.log");
    
    file_logger->info("Log to file");
    
    // 單執行緒版本 (更快)
    auto fast_logger = spdlog::basic_logger_st("fast", "logs/fast.log");
    
    fast_logger->info("Fast logging");
}
```

### 循環檔案 Sink

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/sinks/rotating_file_sink.h>

void rotating_file_examples() {
    // 單檔最大 10MB, 最多保留 3 個檔案
    auto max_size = 1024 * 1024 * 10;  // 10 MB
    auto max_files = 3;
    
    auto rotating_logger = spdlog::rotating_logger_mt(
        "rotating",
        "logs/rotating.log",
        max_size,
        max_files
    );
    
    for (int i = 0; i < 100000; ++i) {
        rotating_logger->info("Message number {}", i);
    }
    
    // 檔案會自動輪替: rotating.log, rotating.1.log, rotating.2.log
}
```

### 每日檔案 Sink

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/sinks/daily_file_sink.h>

void daily_file_examples() {
    // 每天 02:30 輪替
    auto daily_logger = spdlog::daily_logger_mt(
        "daily",
        "logs/daily.log",
        2,   // 小時
        30   // 分鐘
    );
    
    daily_logger->info("Daily log message");
    
    // 檔案命名: daily_2024-01-15.log
}
```

### 多 Sink Logger

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/sinks/stdout_color_sinks.h>
#include <spdlog/sinks/basic_file_sink.h>
#include <vector>

void multi_sink_examples() {
    // 同時輸出到 console 和檔案
    auto console_sink = std::make_shared<spdlog::sinks::stdout_color_sink_mt>();
    auto file_sink = std::make_shared<spdlog::sinks::basic_file_sink_mt>(
        "logs/multi.log", true);
    
    std::vector<spdlog::sink_ptr> sinks{console_sink, file_sink};
    
    auto logger = std::make_shared<spdlog::logger>(
        "multi_sink", 
        sinks.begin(), 
        sinks.end()
    );
    
    spdlog::register_logger(logger);
    
    logger->info("Logged to both console and file");
}
```

---

## 異步日誌

### 基本異步

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/async.h>
#include <spdlog/sinks/basic_file_sink.h>

void async_logging_examples() {
    // 初始化異步日誌
    // 參數: 佇列大小, 執行緒數
    spdlog::init_thread_pool(8192, 1);
    
    // 創建異步 logger
    auto async_file = spdlog::basic_logger_mt<spdlog::async_factory>(
        "async_file",
        "logs/async.log"
    );
    
    // 使用方式同同步 logger
    for (int i = 0; i < 100000; ++i) {
        async_file->info("Async message {}", i);
    }
    
    // 確保所有日誌寫入
    spdlog::shutdown();
}
```

### 高頻交易場景

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/async.h>
#include <spdlog/sinks/basic_file_sink.h>

class TradeLogger {
    std::shared_ptr<spdlog::logger> logger_;
    
public:
    TradeLogger() {
        // 大佇列 (128K entries), 2 個背景執行緒
        spdlog::init_thread_pool(131072, 2);
        
        auto sink = std::make_shared<spdlog::sinks::basic_file_sink_mt>(
            "logs/trades.log", 
            true  // truncate
        );
        
        logger_ = std::make_shared<spdlog::async_logger>(
            "trade",
            sink,
            spdlog::thread_pool(),
            spdlog::async_overflow_policy::overrun_oldest  // 佇列滿時覆蓋舊訊息
        );
        
        // 設定格式
        logger_->set_pattern("[%Y-%m-%d %H:%M:%S.%f] [%l] %v");
        
        spdlog::register_logger(logger_);
    }
    
    void log_trade(uint64_t order_id, 
                   const std::string& symbol,
                   double price,
                   uint64_t quantity) {
        logger_->info("TRADE order_id={} symbol={} price={:.2f} qty={}",
                     order_id, symbol, price, quantity);
    }
    
    void log_order(uint64_t order_id,
                   const std::string& symbol,
                   const std::string& side,
                   double price,
                   uint64_t quantity) {
        logger_->info("ORDER order_id={} symbol={} side={} price={:.2f} qty={}",
                     order_id, symbol, side, price, quantity);
    }
    
    void flush() {
        logger_->flush();
    }
};
```

---

## 格式化

### 自定義格式

```cpp
#include <spdlog/spdlog.h>

void pattern_examples() {
    auto logger = spdlog::stdout_color_mt("pattern");
    
    // 預設格式: [2024-01-15 10:30:45.123] [logger_name] [level] message
    
    // 自定義格式
    logger->set_pattern("[%Y-%m-%d %H:%M:%S.%f] [%n] [%^%l%$] %v");
    // %Y: 年, %m: 月, %d: 日
    // %H: 時, %M: 分, %S: 秒, %f: 微秒
    // %n: logger 名稱
    // %l: 日誌等級
    // %^%l%$: 彩色的日誌等級
    // %v: 訊息內容
    
    logger->info("Custom format");
    
    // 極簡格式
    logger->set_pattern("%v");
    logger->info("Just the message");
    
    // 完整資訊
    logger->set_pattern("[%Y-%m-%d %H:%M:%S.%f] [%t] [%n] [%l] [%s:%#] %v");
    // %t: 執行緒 ID
    // %s: 源檔案名稱
    // %#: 行號
    logger->info("Full info format");
}
```

### 交易系統格式

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/sinks/basic_file_sink.h>

void trading_log_format() {
    auto logger = spdlog::basic_logger_mt("trade", "logs/trades.log");
    
    // 交易日誌格式: timestamp|thread|level|message
    logger->set_pattern("%Y%m%d-%H:%M:%S.%f|%t|%l|%v");
    
    logger->info("ORDER|12345|AAPL|BUY|150.50|100");
    logger->info("FILL|12345|AAPL|150.48|100");
    
    // 輸出:
    // 20240115-10:30:45.123456|140234|info|ORDER|12345|AAPL|BUY|150.50|100
    // 20240115-10:30:45.234567|140234|info|FILL|12345|AAPL|150.48|100
}
```

---

## 效能優化

### 編譯期優化

```cpp
// 定義在包含 spdlog 之前
#define SPDLOG_ACTIVE_LEVEL SPDLOG_LEVEL_INFO  // 編譯期過濾

#include <spdlog/spdlog.h>

void compile_time_optimization() {
    // TRACE 和 DEBUG 會被完全編譯掉 (零開銷)
    SPDLOG_TRACE("This won't be compiled");
    SPDLOG_DEBUG("This won't be compiled either");
    
    SPDLOG_INFO("This will be compiled");
    SPDLOG_WARN("This will be compiled");
}
```

### 減少分配

```cpp
#include <spdlog/spdlog.h>

void reduce_allocations() {
    auto logger = spdlog::stdout_color_mt("fast");
    
    // 避免: 每次都構造臨時 string
    for (int i = 0; i < 10000; ++i) {
        logger->info("Message {}", std::to_string(i));  // 慢
    }
    
    // 推薦: 直接格式化
    for (int i = 0; i < 10000; ++i) {
        logger->info("Message {}", i);  // 快
    }
}
```

### 條件式日誌

```cpp
#include <spdlog/spdlog.h>

void conditional_logging() {
    auto logger = spdlog::stdout_color_mt("cond");
    
    int expensive_value = 0;
    
    // 避免: 總是計算
    logger->debug("Value: {}", compute_expensive());  // compute_expensive 總是執行
    
    // 推薦: 先檢查等級
    if (logger->should_log(spdlog::level::debug)) {
        logger->debug("Value: {}", compute_expensive());  // 只在需要時執行
    }
}

int compute_expensive() {
    // 昂貴的計算...
    return 42;
}
```

### 批次 Flush

```cpp
#include <spdlog/spdlog.h>

void batch_flush() {
    auto logger = spdlog::basic_logger_mt("batch", "logs/batch.log");
    
    // 預設每條日誌都 flush (安全但慢)
    logger->flush_on(spdlog::level::err);  // 只在 error 及以上 flush
    
    // 手動 flush
    for (int i = 0; i < 10000; ++i) {
        logger->info("Message {}", i);
    }
    logger->flush();  // 一次性寫入
}
```

---

## 實戰應用

### 交易引擎日誌系統

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/async.h>
#include <spdlog/sinks/rotating_file_sink.h>
#include <spdlog/sinks/stdout_color_sinks.h>

class TradingEngineLogger {
    std::shared_ptr<spdlog::logger> trade_logger_;
    std::shared_ptr<spdlog::logger> order_logger_;
    std::shared_ptr<spdlog::logger> market_data_logger_;
    std::shared_ptr<spdlog::logger> error_logger_;
    
public:
    TradingEngineLogger() {
        // 異步執行緒池
        spdlog::init_thread_pool(32768, 2);
        
        // 1. 交易日誌 (輪替, 50MB, 10 個檔案)
        auto trade_sink = std::make_shared<spdlog::sinks::rotating_file_sink_mt>(
            "logs/trades.log", 1024 * 1024 * 50, 10);
        trade_logger_ = std::make_shared<spdlog::async_logger>(
            "trade", trade_sink, spdlog::thread_pool());
        trade_logger_->set_pattern("%Y%m%d-%H:%M:%S.%f|%v");
        
        // 2. 訂單日誌
        auto order_sink = std::make_shared<spdlog::sinks::rotating_file_sink_mt>(
            "logs/orders.log", 1024 * 1024 * 50, 10);
        order_logger_ = std::make_shared<spdlog::async_logger>(
            "order", order_sink, spdlog::thread_pool());
        order_logger_->set_pattern("%Y%m%d-%H:%M:%S.%f|%v");
        
        // 3. 行情日誌 (高頻, 只記錄警告以上)
        auto md_sink = std::make_shared<spdlog::sinks::rotating_file_sink_mt>(
            "logs/market_data.log", 1024 * 1024 * 100, 5);
        market_data_logger_ = std::make_shared<spdlog::async_logger>(
            "market_data", md_sink, spdlog::thread_pool());
        market_data_logger_->set_level(spdlog::level::warn);
        market_data_logger_->set_pattern("%Y%m%d-%H:%M:%S.%f|%l|%v");
        
        // 4. 錯誤日誌 (console + file)
        auto console_sink = std::make_shared<spdlog::sinks::stderr_color_sink_mt>();
        auto error_file_sink = std::make_shared<spdlog::sinks::rotating_file_sink_mt>(
            "logs/errors.log", 1024 * 1024 * 10, 3);
        
        std::vector<spdlog::sink_ptr> error_sinks{console_sink, error_file_sink};
        error_logger_ = std::make_shared<spdlog::async_logger>(
            "error", error_sinks.begin(), error_sinks.end(), spdlog::thread_pool());
        error_logger_->set_level(spdlog::level::err);
        error_logger_->set_pattern("[%Y-%m-%d %H:%M:%S.%f] [%^%l%$] %v");
        
        // 註冊所有 logger
        spdlog::register_logger(trade_logger_);
        spdlog::register_logger(order_logger_);
        spdlog::register_logger(market_data_logger_);
        spdlog::register_logger(error_logger_);
    }
    
    ~TradingEngineLogger() {
        spdlog::shutdown();
    }
    
    void log_trade(uint64_t order_id, const std::string& symbol,
                   const std::string& side, double price, uint64_t qty) {
        trade_logger_->info("TRADE|{}|{}|{}|{:.2f}|{}", 
                           order_id, symbol, side, price, qty);
    }
    
    void log_order_new(uint64_t order_id, const std::string& symbol,
                       const std::string& side, double price, uint64_t qty) {
        order_logger_->info("NEW|{}|{}|{}|{:.2f}|{}", 
                           order_id, symbol, side, price, qty);
    }
    
    void log_order_cancel(uint64_t order_id) {
        order_logger_->info("CANCEL|{}", order_id);
    }
    
    void log_order_fill(uint64_t order_id, double fill_price, uint64_t fill_qty) {
        order_logger_->info("FILL|{}|{:.2f}|{}", order_id, fill_price, fill_qty);
    }
    
    void log_market_data_gap(const std::string& symbol, uint64_t expected_seq, 
                            uint64_t actual_seq) {
        market_data_logger_->warn("GAP|{}|expected:{}|actual:{}", 
                                 symbol, expected_seq, actual_seq);
    }
    
    void log_error(const std::string& component, const std::string& message) {
        error_logger_->error("[{}] {}", component, message);
    }
    
    void log_critical(const std::string& component, const std::string& message) {
        error_logger_->critical("[{}] {}", component, message);
        error_logger_->flush();  // 立即寫入
    }
};
```

### 延遲敏感系統

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/async.h>
#include <spdlog/sinks/basic_file_sink.h>

class LowLatencyLogger {
    std::shared_ptr<spdlog::logger> logger_;
    
public:
    LowLatencyLogger() {
        // 超大佇列, 降低阻塞機率
        spdlog::init_thread_pool(1048576, 1);  // 1M entries, 1 thread
        
        auto sink = std::make_shared<spdlog::sinks::basic_file_sink_mt>(
            "logs/low_latency.log", true);
        
        logger_ = std::make_shared<spdlog::async_logger>(
            "low_latency",
            sink,
            spdlog::thread_pool(),
            spdlog::async_overflow_policy::overrun_oldest  // 永不阻塞
        );
        
        // 極簡格式
        logger_->set_pattern("%f|%v");
        
        // 只記錄 warning 以上
        logger_->set_level(spdlog::level::warn);
        
        spdlog::register_logger(logger_);
    }
    
    // 關鍵路徑: 只在異常時記錄
    void log_if_slow(uint64_t latency_ns, uint64_t threshold_ns) {
        if (latency_ns > threshold_ns) {
            logger_->warn("SLOW|latency:{}ns|threshold:{}ns", 
                         latency_ns, threshold_ns);
        }
    }
    
    void log_error_fast(const char* msg) {
        // 使用 const char* 避免 string 構造
        logger_->error(msg);
    }
};
```

### 結構化日誌

```cpp
#include <spdlog/spdlog.h>
#include <nlohmann/json.hpp>

using json = nlohmann::json;

class StructuredLogger {
    std::shared_ptr<spdlog::logger> logger_;
    
public:
    StructuredLogger() {
        logger_ = spdlog::basic_logger_mt("structured", "logs/structured.log");
        logger_->set_pattern("%v");  // 只輸出訊息內容
    }
    
    void log_event(const std::string& event_type, const json& data) {
        json log_entry = {
            {"timestamp", std::chrono::system_clock::now().time_since_epoch().count()},
            {"event", event_type},
            {"data", data}
        };
        
        logger_->info(log_entry.dump());
    }
    
    void log_order_event(uint64_t order_id, const std::string& status,
                        const std::string& symbol, double price, uint64_t qty) {
        json data = {
            {"order_id", order_id},
            {"status", status},
            {"symbol", symbol},
            {"price", price},
            {"quantity", qty}
        };
        
        log_event("order", data);
    }
};

// 使用範例
void structured_logging_example() {
    StructuredLogger logger;
    
    logger.log_order_event(12345, "NEW", "AAPL", 150.50, 100);
    logger.log_order_event(12345, "FILLED", "AAPL", 150.48, 100);
    
    // 輸出 (每行是有效的 JSON):
    // {"timestamp":1705315845123456,"event":"order","data":{"order_id":12345,"status":"NEW",...}}
    // {"timestamp":1705315845234567,"event":"order","data":{"order_id":12345,"status":"FILLED",...}}
}
```

---

## 延遲測量

### 同步 vs 異步

```cpp
#include <spdlog/spdlog.h>
#include <spdlog/async.h>
#include <spdlog/sinks/basic_file_sink.h>
#include <chrono>
#include <vector>
#include <algorithm>

void benchmark_sync() {
    auto logger = spdlog::basic_logger_st("sync", "logs/bench_sync.log");
    
    std::vector<uint64_t> latencies;
    latencies.reserve(10000);
    
    for (int i = 0; i < 10000; ++i) {
        auto start = std::chrono::steady_clock::now();
        
        logger->info("Log message {}", i);
        
        auto end = std::chrono::steady_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(
            end - start);
        
        latencies.push_back(duration.count());
    }
    
    std::sort(latencies.begin(), latencies.end());
    
    std::cout << "Sync logging latency:\n";
    std::cout << "  P50: " << latencies[5000] / 1000.0 << " us\n";
    std::cout << "  P99: " << latencies[9900] / 1000.0 << " us\n";
}

void benchmark_async() {
    spdlog::init_thread_pool(8192, 1);
    
    auto logger = spdlog::basic_logger_mt<spdlog::async_factory>(
        "async", "logs/bench_async.log");
    
    std::vector<uint64_t> latencies;
    latencies.reserve(10000);
    
    for (int i = 0; i < 10000; ++i) {
        auto start = std::chrono::steady_clock::now();
        
        logger->info("Log message {}", i);
        
        auto end = std::chrono::steady_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(
            end - start);
        
        latencies.push_back(duration.count());
    }
    
    spdlog::shutdown();
    
    std::sort(latencies.begin(), latencies.end());
    
    std::cout << "Async logging latency:\n";
    std::cout << "  P50: " << latencies[5000] / 1000.0 << " us\n";
    std::cout << "  P99: " << latencies[9900] / 1000.0 << " us\n";
}

// 典型結果:
// Sync:  P50 ~8 us,  P99 ~25 us
// Async: P50 ~0.2 us, P99 ~1.5 us
```

---

## 除錯技巧

### Source Location

```cpp
#include <spdlog/spdlog.h>

void source_location_logging() {
    auto logger = spdlog::stdout_color_mt("debug");
    
    // 包含檔案名稱與行號
    logger->set_pattern("[%s:%#] %v");
    
    logger->info("This message shows source location");
    
    // 輸出: [main.cpp:42] This message shows source location
}
```

### Backtrace

```cpp
#include <spdlog/spdlog.h>

void backtrace_example() {
    auto logger = spdlog::stdout_color_mt("backtrace");
    
    // 啟用 backtrace (保留最後 32 條訊息)
    logger->enable_backtrace(32);
    
    logger->debug("Debug 1");
    logger->debug("Debug 2");
    logger->debug("Debug 3");
    
    // 觸發錯誤時，dump backtrace
    logger->error("Error occurred!");
    logger->dump_backtrace();
    
    // 輸出包含之前的 debug 訊息
}
```

---

## 最佳實踐

### 1. Logger 管理

```cpp
// 不推薦: 每次獲取 logger
void bad_practice() {
    for (int i = 0; i < 10000; ++i) {
        auto logger = spdlog::get("mylogger");  // 慢! 每次查找
        if (logger) {
            logger->info("Message {}", i);
        }
    }
}

// 推薦: 快取 logger
void good_practice() {
    auto logger = spdlog::get("mylogger");
    if (!logger) return;
    
    for (int i = 0; i < 10000; ++i) {
        logger->info("Message {}", i);  // 快!
    }
}
```

### 2. 生產環境配置

```cpp
void production_setup() {
    // 1. 異步日誌
    spdlog::init_thread_pool(131072, 2);
    
    // 2. 只記錄 info 及以上
    spdlog::set_level(spdlog::level::info);
    
    // 3. 輪替檔案
    auto logger = spdlog::rotating_logger_mt<spdlog::async_factory>(
        "prod", "logs/app.log", 1024 * 1024 * 100, 10);
    
    // 4. 簡化格式 (減少開銷)
    logger->set_pattern("%Y%m%d-%H:%M:%S.%f|%l|%v");
    
    // 5. 批次 flush
    logger->flush_on(spdlog::level::err);
}
```

### 3. 除錯環境配置

```cpp
void debug_setup() {
    // 1. 同步日誌 (更容易追蹤)
    auto logger = spdlog::stdout_color_mt("debug");
    
    // 2. 記錄所有等級
    logger->set_level(spdlog::level::trace);
    
    // 3. 詳細格式
    logger->set_pattern("[%Y-%m-%d %H:%M:%S.%f] [%t] [%s:%#] [%^%l%$] %v");
    
    // 4. 每條都 flush
    logger->flush_on(spdlog::level::trace);
    
    // 5. 啟用 backtrace
    logger->enable_backtrace(64);
}
```

---

## 參考資料

1. [spdlog GitHub](https://github.com/gabime/spdlog)
2. [spdlog Wiki](https://github.com/gabime/spdlog/wiki)
3. [fmt 格式化語法](https://fmt.dev/latest/syntax.html)
4. [spdlog Benchmarks](https://github.com/gabime/spdlog#benchmarks)
5. [C++ Logging Performance](https://www.codeproject.com/Articles/1272619/spdlog-Fast-Cplusplus-Logging-Library)
