# Day 10: 整合實戰項目 - 構建高頻交易系統

> 本章整合前9天所學,構建完整的高頻交易系統原型。

---

## 10.1 系統需求與架構設計

### 10.1.1 功能需求

**核心功能:**
- 接收市場數據 (UDP Multicast)
- 維護訂單簿
- 執行交易策略 (市場製造)
- 訂單管理與發送
- 前置風控檢查
- 性能監控與日誌

**非功能需求:**
- 端到端延遲 < 100 微秒
- 吞吐量 > 100,000 msg/s
- 低抖動 (jitter)
- 高可用性

---

### 10.1.2 系統架構

```
                            +-----------------+
                            |   Config File   |
                            +-----------------+
                                     |
                                     v
+----------------+    SPSC    +----------------+    SPSC    +----------------+
|  Market Data   | ---------> |   Strategy     | ---------> |  Order Send    |
|   Receiver     |   Queue    |    Engine      |   Queue    |    Thread      |
+----------------+            +----------------+            +----------------+
       |                             |                             |
       | UDP Multicast               |                             | TCP/FIX
       v                             v                             v
+----------------+            +----------------+            +----------------+
|   Exchange     |            |  Order Book    |            |   Exchange     |
|   (MD Feed)    |            |  + Risk Mgr    |            |   (Gateway)    |
+----------------+            +----------------+            +----------------+
```

**執行緒設計:**
| 執行緒 | 職責 | CPU | 優先級 |
|--------|------|-----|--------|
| MarketData | UDP 接收、解析、訂單簿更新 | Core 1 | SCHED_FIFO 99 |
| Strategy | 策略邏輯、訊號生成、風控 | Core 2 | SCHED_FIFO 98 |
| OrderSend | FIX 編碼、TCP 發送 | Core 3 | SCHED_FIFO 97 |
| Monitor | 日誌、統計、監控 | Core 0 | SCHED_OTHER |

---

### 10.1.3 數據流

```
1. Market Data 接收
   UDP packet → 解析 → 訂單簿更新 → BBO 事件

2. Strategy 處理
   BBO 事件 → 策略計算 → 風控檢查 → 訂單請求

3. Order 發送
   訂單請求 → FIX 編碼 → TCP 發送 → 等待回報

4. 回報處理
   FIX 回報 → 解析 → 訂單狀態更新 → 策略通知
```

---

## 10.2 核心數據結構

### 10.2.1 共享類型定義

```cpp
// common/types.h
#pragma once

#include <cstdint>
#include <array>
#include <chrono>

namespace hft {

// 時間戳 (奈秒)
using Timestamp = uint64_t;

inline Timestamp now() {
    return std::chrono::duration_cast<std::chrono::nanoseconds>(
        std::chrono::steady_clock::now().time_since_epoch()
    ).count();
}

// 高精度時間 (RDTSC)
inline uint64_t rdtsc() {
    uint32_t lo, hi;
    __asm__ volatile ("rdtsc" : "=a" (lo), "=d" (hi));
    return ((uint64_t)hi << 32) | lo;
}

// 標的代碼
using Symbol = std::array<char, 8>;

// 方向
enum class Side : char {
    Buy = '1',
    Sell = '2'
};

// 訂單類型
enum class OrderType : char {
    Market = '1',
    Limit = '2'
};

// 價格 (定點數,避免浮點運算)
// 價格 = rawPrice / 10000 (支援4位小數)
using Price = int64_t;

constexpr int64_t PRICE_MULTIPLIER = 10000;

inline Price toPrice(double d) {
    return static_cast<Price>(d * PRICE_MULTIPLIER + 0.5);
}

inline double fromPrice(Price p) {
    return static_cast<double>(p) / PRICE_MULTIPLIER;
}

// 數量
using Quantity = int32_t;

// 訂單 ID
using OrderId = uint64_t;

} // namespace hft
```

---

### 10.2.2 市場數據事件

```cpp
// market_data/events.h
#pragma once

#include "common/types.h"

namespace hft {

// BBO 更新事件
struct BBOEvent {
    Symbol symbol;
    Price bidPrice;
    Quantity bidSize;
    Price askPrice;
    Quantity askSize;
    Timestamp exchangeTime;
    Timestamp receiveTime;
} __attribute__((packed));

// 成交事件
struct TradeEvent {
    Symbol symbol;
    Price price;
    Quantity size;
    Side aggressor;
    Timestamp exchangeTime;
    Timestamp receiveTime;
} __attribute__((packed));

} // namespace hft
```

---

### 10.2.3 訂單請求與回報

```cpp
// order/messages.h
#pragma once

#include "common/types.h"

namespace hft {

// 訂單請求 (內部使用)
struct OrderRequest {
    enum class Type : uint8_t {
        New,
        Cancel,
        Replace
    };
    
    Type type;
    OrderId clientOrderId;
    Symbol symbol;
    Side side;
    OrderType orderType;
    Price price;
    Quantity quantity;
    Timestamp createTime;
} __attribute__((packed));

// 訂單狀態
enum class OrderStatus : uint8_t {
    New,
    PartiallyFilled,
    Filled,
    Canceled,
    Rejected
};

// 執行回報
struct ExecutionReport {
    OrderId clientOrderId;
    OrderId exchangeOrderId;
    OrderStatus status;
    Quantity filledQty;
    Quantity leavesQty;
    Price avgPrice;
    Timestamp exchangeTime;
} __attribute__((packed));

} // namespace hft
```

---

## 10.3 SPSC 無鎖佇列

```cpp
// core/spsc_queue.h
#pragma once

#include <atomic>
#include <array>
#include <new>

namespace hft {

template<typename T, size_t Capacity>
class SPSCQueue {
    static_assert((Capacity & (Capacity - 1)) == 0, 
                  "Capacity must be power of 2");
    
    // Cache line 對齊避免 false sharing
    alignas(64) std::atomic<size_t> head_{0};
    alignas(64) std::atomic<size_t> tail_{0};
    alignas(64) std::array<T, Capacity> buffer_;
    
public:
    bool push(const T& item) {
        const size_t tail = tail_.load(std::memory_order_relaxed);
        const size_t next = (tail + 1) & (Capacity - 1);
        
        if (next == head_.load(std::memory_order_acquire)) {
            return false;  // 佇列已滿
        }
        
        buffer_[tail] = item;
        tail_.store(next, std::memory_order_release);
        return true;
    }
    
    bool pop(T& item) {
        const size_t head = head_.load(std::memory_order_relaxed);
        
        if (head == tail_.load(std::memory_order_acquire)) {
            return false;  // 佇列為空
        }
        
        item = buffer_[head];
        head_.store((head + 1) & (Capacity - 1), std::memory_order_release);
        return true;
    }
    
    bool empty() const {
        return head_.load(std::memory_order_acquire) == 
               tail_.load(std::memory_order_acquire);
    }
    
    size_t size() const {
        const size_t head = head_.load(std::memory_order_acquire);
        const size_t tail = tail_.load(std::memory_order_acquire);
        return (tail - head + Capacity) & (Capacity - 1);
    }
};

} // namespace hft
```

---

## 10.4 市場數據接收模組

### 10.4.1 UDP 接收器

```cpp
// market_data/receiver.h
#pragma once

#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <unistd.h>

#include "common/types.h"
#include "market_data/events.h"
#include "core/spsc_queue.h"

namespace hft {

class MarketDataReceiver {
    int socket_ = -1;
    char buffer_[65536];
    
    // 輸出佇列
    SPSCQueue<BBOEvent, 65536>& outputQueue_;
    
    // 訂單簿
    // 簡化版本,實際應該用 Day 9 的完整實作
    struct SimpleOrderBook {
        Price bidPrice = 0;
        Quantity bidSize = 0;
        Price askPrice = 0;
        Quantity askSize = 0;
    };
    SimpleOrderBook book_;
    
public:
    explicit MarketDataReceiver(SPSCQueue<BBOEvent, 65536>& queue)
        : outputQueue_(queue) {}
    
    ~MarketDataReceiver() {
        if (socket_ >= 0) close(socket_);
    }
    
    bool init(const char* multicastAddr, uint16_t port,
              const char* interfaceAddr) {
        socket_ = socket(AF_INET, SOCK_DGRAM, 0);
        if (socket_ < 0) return false;
        
        // 設置非阻塞 (用於 busy poll)
        int flags = fcntl(socket_, F_GETFL, 0);
        fcntl(socket_, F_SETFL, flags | O_NONBLOCK);
        
        // 允許位址重用
        int reuse = 1;
        setsockopt(socket_, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));
        
        // 增大接收緩衝區
        int bufSize = 4 * 1024 * 1024;
        setsockopt(socket_, SOL_SOCKET, SO_RCVBUF, &bufSize, sizeof(bufSize));
        
        // 綁定
        sockaddr_in addr{};
        addr.sin_family = AF_INET;
        addr.sin_addr.s_addr = htonl(INADDR_ANY);
        addr.sin_port = htons(port);
        
        if (bind(socket_, (sockaddr*)&addr, sizeof(addr)) < 0) {
            return false;
        }
        
        // 加入多播組
        ip_mreq mreq{};
        mreq.imr_multiaddr.s_addr = inet_addr(multicastAddr);
        mreq.imr_interface.s_addr = inet_addr(interfaceAddr);
        
        if (setsockopt(socket_, IPPROTO_IP, IP_ADD_MEMBERSHIP,
                       &mreq, sizeof(mreq)) < 0) {
            return false;
        }
        
        return true;
    }
    
    // 主循環 (busy poll)
    void run(std::atomic<bool>& running) {
        while (running.load(std::memory_order_relaxed)) {
            ssize_t len = recv(socket_, buffer_, sizeof(buffer_), 0);
            
            if (len > 0) {
                Timestamp recvTime = now();
                processPacket(buffer_, len, recvTime);
            }
            // 非阻塞,繼續輪詢
        }
    }
    
private:
    void processPacket(const char* data, size_t len, Timestamp recvTime) {
        // 解析簡化的二進制協定
        // 實際應使用 ITCH 或自定義協定
        
        if (len < sizeof(SimpleMDMessage)) return;
        
        auto* msg = reinterpret_cast<const SimpleMDMessage*>(data);
        
        // 更新訂單簿
        book_.bidPrice = msg->bidPrice;
        book_.bidSize = msg->bidSize;
        book_.askPrice = msg->askPrice;
        book_.askSize = msg->askSize;
        
        // 發布 BBO 事件
        BBOEvent event;
        event.symbol = msg->symbol;
        event.bidPrice = book_.bidPrice;
        event.bidSize = book_.bidSize;
        event.askPrice = book_.askPrice;
        event.askSize = book_.askSize;
        event.exchangeTime = msg->timestamp;
        event.receiveTime = recvTime;
        
        outputQueue_.push(event);
    }
    
    // 簡化的市場數據訊息格式
    struct SimpleMDMessage {
        Symbol symbol;
        Price bidPrice;
        Quantity bidSize;
        Price askPrice;
        Quantity askSize;
        Timestamp timestamp;
    } __attribute__((packed));
};

} // namespace hft
```

---

## 10.5 策略引擎模組

### 10.5.1 策略引擎

```cpp
// strategy/engine.h
#pragma once

#include "common/types.h"
#include "market_data/events.h"
#include "order/messages.h"
#include "core/spsc_queue.h"
#include "risk/manager.h"

namespace hft {

class StrategyEngine {
    // 輸入佇列 (來自 MD)
    SPSCQueue<BBOEvent, 65536>& inputQueue_;
    // 輸出佇列 (到 Order Send)
    SPSCQueue<OrderRequest, 8192>& outputQueue_;
    
    // 風控
    RiskManager risk_;
    
    // 策略參數
    struct Parameters {
        Price spreadTicks = 10;     // 價差 (tick)
        Quantity quoteSize = 100;
        Quantity positionLimit = 1000;
        Price tickSize = 1;         // 最小價格單位
    } params_;
    
    // 狀態
    Quantity position_ = 0;
    OrderId nextOrderId_ = 1;
    
    // 當前報價
    OrderId currentBidId_ = 0;
    OrderId currentAskId_ = 0;
    
public:
    StrategyEngine(SPSCQueue<BBOEvent, 65536>& in,
                   SPSCQueue<OrderRequest, 8192>& out)
        : inputQueue_(in), outputQueue_(out) {}
    
    void run(std::atomic<bool>& running) {
        BBOEvent event;
        
        while (running.load(std::memory_order_relaxed)) {
            if (inputQueue_.pop(event)) {
                onBBO(event);
            }
            // 可以加入 _mm_pause() 減少 CPU 消耗
        }
    }
    
    void onExecutionReport(const ExecutionReport& report) {
        // 更新持倉
        // 實際應該追蹤每個訂單的方向
        if (report.status == OrderStatus::Filled ||
            report.status == OrderStatus::PartiallyFilled) {
            // 假設買單
            // position_ += report.filledQty;
        }
    }
    
private:
    void onBBO(const BBOEvent& event) {
        // 計算中間價
        Price midPrice = (event.bidPrice + event.askPrice) / 2;
        
        // 生成報價
        Price halfSpread = params_.spreadTicks * params_.tickSize / 2;
        
        // 持倉調整 (持倉多則降低買價、提高賣價)
        Price skew = position_ * params_.tickSize / 100;
        
        Price bidPrice = midPrice - halfSpread - skew;
        Price askPrice = midPrice + halfSpread - skew;
        
        // 取消舊報價
        if (currentBidId_ != 0) {
            sendCancel(currentBidId_);
        }
        if (currentAskId_ != 0) {
            sendCancel(currentAskId_);
        }
        
        // 發送新報價
        Quantity bidSize = params_.quoteSize;
        Quantity askSize = params_.quoteSize;
        
        // 持倉限制
        if (position_ > params_.positionLimit * 80 / 100) {
            bidSize = 0;
        }
        if (position_ < -params_.positionLimit * 80 / 100) {
            askSize = 0;
        }
        
        // 風控檢查並發送
        if (bidSize > 0 && risk_.check(bidSize, bidPrice, Side::Buy)) {
            currentBidId_ = sendOrder(event.symbol, Side::Buy, 
                                      bidPrice, bidSize);
        }
        
        if (askSize > 0 && risk_.check(askSize, askPrice, Side::Sell)) {
            currentAskId_ = sendOrder(event.symbol, Side::Sell,
                                      askPrice, askSize);
        }
    }
    
    OrderId sendOrder(const Symbol& symbol, Side side, 
                      Price price, Quantity qty) {
        OrderRequest req;
        req.type = OrderRequest::Type::New;
        req.clientOrderId = nextOrderId_++;
        req.symbol = symbol;
        req.side = side;
        req.orderType = OrderType::Limit;
        req.price = price;
        req.quantity = qty;
        req.createTime = now();
        
        outputQueue_.push(req);
        return req.clientOrderId;
    }
    
    void sendCancel(OrderId orderId) {
        OrderRequest req;
        req.type = OrderRequest::Type::Cancel;
        req.clientOrderId = orderId;
        req.createTime = now();
        
        outputQueue_.push(req);
    }
};

} // namespace hft
```

---

## 10.6 風控模組

```cpp
// risk/manager.h
#pragma once

#include <atomic>
#include "common/types.h"

namespace hft {

class RiskManager {
    struct Limits {
        Quantity maxOrderQty = 10000;
        Price maxPriceDeviation = 500;  // 5%
        Quantity maxPosition = 100000;
        int maxOrdersPerSecond = 100;
    } limits_;
    
    std::atomic<Quantity> position_{0};
    std::atomic<int> ordersInWindow_{0};
    std::atomic<Timestamp> windowStart_{0};
    
public:
    bool check(Quantity qty, Price price, Side side) {
        // 1. 數量檢查
        if (qty > limits_.maxOrderQty) {
            return false;
        }
        
        // 2. 持倉檢查
        Quantity delta = (side == Side::Buy) ? qty : -qty;
        Quantity newPos = position_.load(std::memory_order_relaxed) + delta;
        if (std::abs(newPos) > limits_.maxPosition) {
            return false;
        }
        
        // 3. 頻率檢查
        if (!checkRateLimit()) {
            return false;
        }
        
        return true;
    }
    
    void updatePosition(Quantity delta) {
        position_.fetch_add(delta, std::memory_order_relaxed);
    }
    
    Quantity getPosition() const {
        return position_.load(std::memory_order_relaxed);
    }
    
private:
    bool checkRateLimit() {
        Timestamp current = now();
        Timestamp windowStart = windowStart_.load(std::memory_order_relaxed);
        
        constexpr Timestamp WINDOW_NS = 1000000000;  // 1 秒
        
        if (current - windowStart > WINDOW_NS) {
            windowStart_.store(current, std::memory_order_relaxed);
            ordersInWindow_.store(1, std::memory_order_relaxed);
            return true;
        }
        
        int orders = ordersInWindow_.fetch_add(1, std::memory_order_relaxed);
        return orders < limits_.maxOrdersPerSecond;
    }
};

} // namespace hft
```

---

## 10.7 訂單發送模組

```cpp
// order/sender.h
#pragma once

#include <sys/socket.h>
#include <netinet/tcp.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <cstring>

#include "common/types.h"
#include "order/messages.h"
#include "core/spsc_queue.h"

namespace hft {

class OrderSender {
    int socket_ = -1;
    char sendBuffer_[4096];
    
    SPSCQueue<OrderRequest, 8192>& inputQueue_;
    
public:
    explicit OrderSender(SPSCQueue<OrderRequest, 8192>& queue)
        : inputQueue_(queue) {}
    
    ~OrderSender() {
        if (socket_ >= 0) close(socket_);
    }
    
    bool connect(const char* host, uint16_t port) {
        socket_ = socket(AF_INET, SOCK_STREAM, 0);
        if (socket_ < 0) return false;
        
        // TCP_NODELAY - 禁用 Nagle
        int nodelay = 1;
        setsockopt(socket_, IPPROTO_TCP, TCP_NODELAY, 
                   &nodelay, sizeof(nodelay));
        
        // 增大發送緩衝區
        int bufSize = 1024 * 1024;
        setsockopt(socket_, SOL_SOCKET, SO_SNDBUF, &bufSize, sizeof(bufSize));
        
        sockaddr_in addr{};
        addr.sin_family = AF_INET;
        addr.sin_port = htons(port);
        inet_pton(AF_INET, host, &addr.sin_addr);
        
        if (::connect(socket_, (sockaddr*)&addr, sizeof(addr)) < 0) {
            return false;
        }
        
        return true;
    }
    
    void run(std::atomic<bool>& running) {
        OrderRequest req;
        
        while (running.load(std::memory_order_relaxed)) {
            if (inputQueue_.pop(req)) {
                processRequest(req);
            }
        }
    }
    
private:
    void processRequest(const OrderRequest& req) {
        switch (req.type) {
            case OrderRequest::Type::New:
                sendNewOrder(req);
                break;
            case OrderRequest::Type::Cancel:
                sendCancelOrder(req);
                break;
            case OrderRequest::Type::Replace:
                sendReplaceOrder(req);
                break;
        }
    }
    
    void sendNewOrder(const OrderRequest& req) {
        // 構建簡化的 FIX 訊息
        // 實際應使用完整的 FIX 編碼器
        int len = snprintf(sendBuffer_, sizeof(sendBuffer_),
            "8=FIX.4.4\x01"
            "35=D\x01"
            "11=%lu\x01"
            "55=%.8s\x01"
            "54=%c\x01"
            "38=%d\x01"
            "40=%c\x01"
            "44=%.4f\x01"
            "10=000\x01",
            req.clientOrderId,
            req.symbol.data(),
            static_cast<char>(req.side),
            req.quantity,
            static_cast<char>(req.orderType),
            fromPrice(req.price)
        );
        
        send(socket_, sendBuffer_, len, 0);
    }
    
    void sendCancelOrder(const OrderRequest& req) {
        int len = snprintf(sendBuffer_, sizeof(sendBuffer_),
            "8=FIX.4.4\x01"
            "35=F\x01"
            "11=%lu\x01"
            "10=000\x01",
            req.clientOrderId
        );
        
        send(socket_, sendBuffer_, len, 0);
    }
    
    void sendReplaceOrder(const OrderRequest& req) {
        // 類似 NewOrder
    }
};

} // namespace hft
```

---

## 10.8 主程式整合

```cpp
// main.cpp
#include <thread>
#include <atomic>
#include <csignal>

#include <sched.h>
#include <pthread.h>
#include <sys/mman.h>

#include "common/types.h"
#include "core/spsc_queue.h"
#include "market_data/receiver.h"
#include "strategy/engine.h"
#include "order/sender.h"

using namespace hft;

std::atomic<bool> g_running{true};

void signalHandler(int) {
    g_running.store(false, std::memory_order_relaxed);
}

// 設置 CPU 親和性
void setAffinity(int cpu) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu, &cpuset);
    pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);
}

// 設置即時優先級
void setRealtimePriority(int priority) {
    sched_param param;
    param.sched_priority = priority;
    pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);
}

int main(int argc, char* argv[]) {
    // 信號處理
    signal(SIGINT, signalHandler);
    signal(SIGTERM, signalHandler);
    
    // 鎖定記憶體
    if (mlockall(MCL_CURRENT | MCL_FUTURE) != 0) {
        perror("mlockall failed");
        // 繼續執行,但可能有頁面錯誤
    }
    
    // 建立佇列
    SPSCQueue<BBOEvent, 65536> mdQueue;
    SPSCQueue<OrderRequest, 8192> orderQueue;
    
    // 建立組件
    MarketDataReceiver mdReceiver(mdQueue);
    StrategyEngine strategy(mdQueue, orderQueue);
    OrderSender orderSender(orderQueue);
    
    // 初始化
    if (!mdReceiver.init("239.255.0.1", 5001, "0.0.0.0")) {
        fprintf(stderr, "Failed to init market data receiver\n");
        return 1;
    }
    
    if (!orderSender.connect("127.0.0.1", 5002)) {
        fprintf(stderr, "Failed to connect to exchange\n");
        return 1;
    }
    
    printf("HFT System starting...\n");
    
    // 啟動執行緒
    std::thread mdThread([&] {
        setAffinity(1);
        setRealtimePriority(99);
        mdReceiver.run(g_running);
    });
    
    std::thread strategyThread([&] {
        setAffinity(2);
        setRealtimePriority(98);
        strategy.run(g_running);
    });
    
    std::thread orderThread([&] {
        setAffinity(3);
        setRealtimePriority(97);
        orderSender.run(g_running);
    });
    
    printf("HFT System running. Press Ctrl+C to stop.\n");
    
    // 等待結束
    mdThread.join();
    strategyThread.join();
    orderThread.join();
    
    printf("HFT System stopped.\n");
    return 0;
}
```

---

## 10.9 性能優化技巧

### 10.9.1 記憶體優化

```cpp
// 物件池避免動態分配
template<typename T, size_t Size>
class ObjectPool {
    alignas(64) std::array<T, Size> pool_;
    alignas(64) std::array<T*, Size> freeList_;
    size_t freeCount_ = Size;
    
public:
    ObjectPool() {
        for (size_t i = 0; i < Size; ++i) {
            freeList_[i] = &pool_[i];
        }
    }
    
    T* allocate() {
        return freeCount_ > 0 ? freeList_[--freeCount_] : nullptr;
    }
    
    void deallocate(T* p) {
        freeList_[freeCount_++] = p;
    }
};

// Cache line 對齊避免 false sharing
struct alignas(64) AlignedCounter {
    std::atomic<uint64_t> value{0};
    char padding[64 - sizeof(std::atomic<uint64_t>)];
};
```

---

### 10.9.2 分支預測優化

```cpp
// 使用 likely/unlikely
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)

// 熱路徑優化
void processMessage(const Message& msg) {
    if (likely(msg.type == MessageType::MarketData)) {
        // 最常見的情況
        processMarketData(msg);
    } else if (unlikely(msg.type == MessageType::Error)) {
        // 罕見情況
        processError(msg);
    } else {
        processOther(msg);
    }
}
```

---

### 10.9.3 編譯器優化

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(hft_system)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Release 編譯選項
set(CMAKE_CXX_FLAGS_RELEASE 
    "-O3 -march=native -flto -DNDEBUG -fno-exceptions")

# 連結時優化
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "-flto")

add_executable(hft_system
    main.cpp
    # 其他源文件
)

# 靜態連結
target_link_options(hft_system PRIVATE -static-libgcc -static-libstdc++)
```

---

## 10.10 性能測量

### 10.10.1 延遲測量器

```cpp
// perf/latency_meter.h
#pragma once

#include <array>
#include <algorithm>
#include <cmath>
#include "common/types.h"

namespace hft {

class LatencyMeter {
    static constexpr size_t MAX_SAMPLES = 100000;
    std::array<uint64_t, MAX_SAMPLES> samples_;
    size_t count_ = 0;
    
public:
    void record(uint64_t latency) {
        if (count_ < MAX_SAMPLES) {
            samples_[count_++] = latency;
        }
    }
    
    struct Stats {
        uint64_t min;
        uint64_t max;
        uint64_t avg;
        uint64_t p50;
        uint64_t p99;
        uint64_t p999;
    };
    
    Stats calculate() {
        if (count_ == 0) return {};
        
        std::sort(samples_.begin(), samples_.begin() + count_);
        
        Stats stats;
        stats.min = samples_[0];
        stats.max = samples_[count_ - 1];
        
        uint64_t sum = 0;
        for (size_t i = 0; i < count_; ++i) {
            sum += samples_[i];
        }
        stats.avg = sum / count_;
        
        stats.p50 = samples_[count_ * 50 / 100];
        stats.p99 = samples_[count_ * 99 / 100];
        stats.p999 = samples_[count_ * 999 / 1000];
        
        return stats;
    }
    
    void reset() {
        count_ = 0;
    }
};

// 使用 RDTSC 的高精度計時
class RDTSCTimer {
    uint64_t start_;
    
public:
    void start() {
        start_ = rdtsc();
    }
    
    uint64_t elapsed() const {
        return rdtsc() - start_;
    }
    
    // 將 TSC 週期轉換為奈秒 (需要校準)
    static uint64_t toNanos(uint64_t cycles, uint64_t tscFreq) {
        return cycles * 1000000000ULL / tscFreq;
    }
};

} // namespace hft
```

---

### 10.10.2 吞吐量測量

```cpp
// perf/throughput_meter.h
#pragma once

#include <atomic>
#include "common/types.h"

namespace hft {

class ThroughputMeter {
    std::atomic<uint64_t> count_{0};
    Timestamp startTime_ = 0;
    
public:
    void start() {
        startTime_ = now();
        count_.store(0, std::memory_order_relaxed);
    }
    
    void increment() {
        count_.fetch_add(1, std::memory_order_relaxed);
    }
    
    void add(uint64_t n) {
        count_.fetch_add(n, std::memory_order_relaxed);
    }
    
    double getRate() const {
        uint64_t elapsed = now() - startTime_;
        if (elapsed == 0) return 0;
        
        uint64_t cnt = count_.load(std::memory_order_relaxed);
        return static_cast<double>(cnt) * 1e9 / elapsed;
    }
};

} // namespace hft
```

---

## 10.11 日誌系統

```cpp
// log/logger.h
#pragma once

#include <cstdio>
#include <cstdarg>
#include "common/types.h"
#include "core/spsc_queue.h"

namespace hft {

// 非同步日誌,避免阻塞熱路徑
class AsyncLogger {
    struct LogEntry {
        Timestamp time;
        int level;
        char message[256];
    };
    
    SPSCQueue<LogEntry, 8192> queue_;
    FILE* file_ = nullptr;
    std::atomic<bool> running_{true};
    
public:
    enum Level { DEBUG, INFO, WARN, ERROR };
    
    bool init(const char* filename) {
        file_ = fopen(filename, "a");
        return file_ != nullptr;
    }
    
    ~AsyncLogger() {
        if (file_) fclose(file_);
    }
    
    void log(int level, const char* fmt, ...) {
        LogEntry entry;
        entry.time = now();
        entry.level = level;
        
        va_list args;
        va_start(args, fmt);
        vsnprintf(entry.message, sizeof(entry.message), fmt, args);
        va_end(args);
        
        queue_.push(entry);
    }
    
    // 日誌寫入執行緒
    void writerLoop() {
        LogEntry entry;
        while (running_.load(std::memory_order_relaxed)) {
            while (queue_.pop(entry)) {
                fprintf(file_, "[%lu][%d] %s\n", 
                        entry.time, entry.level, entry.message);
            }
            fflush(file_);
            usleep(1000);  // 1ms
        }
    }
};

// 全局 logger
extern AsyncLogger g_logger;

#define LOG_DEBUG(fmt, ...) g_logger.log(AsyncLogger::DEBUG, fmt, ##__VA_ARGS__)
#define LOG_INFO(fmt, ...)  g_logger.log(AsyncLogger::INFO, fmt, ##__VA_ARGS__)
#define LOG_WARN(fmt, ...)  g_logger.log(AsyncLogger::WARN, fmt, ##__VA_ARGS__)
#define LOG_ERROR(fmt, ...) g_logger.log(AsyncLogger::ERROR, fmt, ##__VA_ARGS__)

} // namespace hft
```

---

## 10.12 系統配置與部署

### 10.12.1 系統調優腳本

```bash
#!/bin/bash
# tune_system.sh

# 1. CPU 隔離 (需要在 grub 設置)
# GRUB_CMDLINE_LINUX="isolcpus=1,2,3 nohz_full=1,2,3"

# 2. 設置 CPU 為 performance 模式
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance > $cpu
done

# 3. 禁用 C-states
echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo 2>/dev/null || true

# 4. 設置 Huge Pages
echo 1024 > /proc/sys/vm/nr_hugepages

# 5. 網路調優
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sysctl -w net.ipv4.tcp_wmem="4096 87380 16777216"
sysctl -w net.core.netdev_max_backlog=30000
sysctl -w net.ipv4.tcp_max_syn_backlog=4096
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_fin_timeout=15

# 6. 禁用透明大頁 (避免延遲峰值)
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# 7. 設置中斷親和性 (將網卡中斷綁定到 CPU 0)
# 需要知道網卡的中斷號
# echo 1 > /proc/irq/<IRQ>/smp_affinity

echo "System tuning complete"
```

---

### 10.12.2 資源限制配置

```bash
# /etc/security/limits.conf

# HFT 用戶配置
hft_user soft memlock unlimited
hft_user hard memlock unlimited
hft_user soft rtprio 99
hft_user hard rtprio 99
hft_user soft nofile 65536
hft_user hard nofile 65536
```

---

### 10.12.3 啟動腳本

```bash
#!/bin/bash
# start_hft.sh

# 設置環境
export LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so

# 啟動系統
sudo numactl --cpunodebind=0 --membind=0 ./hft_system \
    --config=/etc/hft/config.json \
    --log=/var/log/hft/hft.log

# numactl: 綁定到 NUMA 節點 0
```

---

## 10.13 測試框架

### 10.13.1 單元測試

```cpp
// tests/test_orderbook.cpp
#include <gtest/gtest.h>
#include "market_data/orderbook.h"

TEST(OrderBookTest, AddOrder) {
    hft::OrderBook book("AAPL");
    
    book.addOrder(1, hft::toPrice(100.00), 100, true);  // Buy
    book.addOrder(2, hft::toPrice(100.01), 200, false); // Sell
    
    auto bbo = book.getBBO();
    EXPECT_EQ(bbo.bidPrice, hft::toPrice(100.00));
    EXPECT_EQ(bbo.bidSize, 100);
    EXPECT_EQ(bbo.askPrice, hft::toPrice(100.01));
    EXPECT_EQ(bbo.askSize, 200);
}

TEST(OrderBookTest, DeleteOrder) {
    hft::OrderBook book("AAPL");
    
    book.addOrder(1, hft::toPrice(100.00), 100, true);
    book.deleteOrder(1);
    
    auto bbo = book.getBBO();
    EXPECT_EQ(bbo.bidSize, 0);
}
```

---

### 10.13.2 性能基準測試

```cpp
// benchmarks/bench_queue.cpp
#include <benchmark/benchmark.h>
#include "core/spsc_queue.h"

static void BM_SPSCQueue_PushPop(benchmark::State& state) {
    hft::SPSCQueue<int, 65536> queue;
    int value = 42;
    
    for (auto _ : state) {
        queue.push(value);
        queue.pop(value);
    }
}

BENCHMARK(BM_SPSCQueue_PushPop);

static void BM_OrderBook_Update(benchmark::State& state) {
    hft::OrderBook book("AAPL");
    uint64_t orderId = 1;
    
    for (auto _ : state) {
        book.addOrder(orderId++, hft::toPrice(100.00), 100, true);
        book.deleteOrder(orderId - 1);
    }
}

BENCHMARK(BM_OrderBook_Update);

BENCHMARK_MAIN();
```

---

## 10.14 最佳實踐總結

### 10.14.1 開發原則

1. **先正確,再快速**: 優化前確保邏輯正確
2. **測量驅動**: 每次優化都要測量效果
3. **避免過早優化**: 專注於關鍵路徑
4. **保持簡單**: 複雜度是性能的敵人

### 10.14.2 常見陷阱

| 陷阱 | 問題 | 解決方案 |
|------|------|----------|
| 動態分配 | malloc 延遲不可預測 | 物件池、預分配 |
| 虛函數 | 間接呼叫開銷 | CRTP、模板 |
| 異常處理 | 展開開銷 | 錯誤碼、-fno-exceptions |
| 鎖競爭 | 上下文切換 | 無鎖結構 |
| False sharing | Cache 失效 | alignas(64) |
| 系統呼叫 | 模式切換開銷 | 批量處理、busy poll |

### 10.14.3 持續改進

- 建立性能基線
- 定期執行基準測試
- 監控生產環境延遲
- 跟蹤技術債務

---

## 10.15 延伸學習

### 進階主題

- **FPGA 加速**: 使用硬體實現更低延遲
- **更複雜策略**: 統計套利、事件驅動
- **機器學習**: 訊號生成與風控

### 推薦資源

**開源專案:**
- [QuickFIX](https://quickfixengine.org/) - FIX 引擎
- [LMAX Disruptor](https://github.com/LMAX-Exchange/disruptor) - 高性能佇列
- [Folly](https://github.com/facebook/folly) - Facebook C++ 庫
- [Abseil](https://github.com/abseil/abseil-cpp) - Google C++ 庫

**書籍:**
- 《High-Frequency Trading》 (Irene Aldridge)
- 《C++ High Performance》 (Björn Andrist)
- 《Systems Performance》 (Brendan Gregg)
- 《The Art of Multiprocessor Programming》 (Herlihy & Shavit)

**線上資源:**
- [Mechanical Sympathy](https://mechanical-sympathy.blogspot.com/)
- [Brendan Gregg's Blog](https://www.brendangregg.com/)
- [Agner Fog's Optimization Resources](https://www.agner.org/optimize/)

---

## 實作專案清單

完成本課程後,你應該能夠:

- [ ] 設計並實作完整的 HFT 系統架構
- [ ] 實作高性能 SPSC 佇列
- [ ] 實作微秒級訂單簿
- [ ] 實作簡單市場製造策略
- [ ] 實作前置風控系統
- [ ] 使用 RDTSC 測量延遲
- [ ] 配置 CPU 親和性與即時排程
- [ ] 調優系統達到 P99 < 50μs
- [ ] 使用 perf 分析性能瓶頸
- [ ] 撰寫單元測試與基準測試

---

## 參考資料 (References)

1. 《High-Frequency Trading: A Practical Guide to Algorithmic Strategies》 (Irene Aldridge, 2013)
2. 《C++ High Performance》 (Björn Andrist & Viktor Sehr, 2018)
3. 《Systems Performance: Enterprise and the Cloud》 (Brendan Gregg, 2020)
4. [Intel 64 and IA-32 Architectures Optimization Reference Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
5. [Linux Kernel Documentation - CPU Isolation](https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html)
6. [Google Benchmark](https://github.com/google/benchmark)
7. [Google Test](https://github.com/google/googletest)
