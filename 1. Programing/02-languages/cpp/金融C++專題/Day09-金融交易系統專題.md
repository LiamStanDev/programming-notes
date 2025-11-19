# Day 9: 金融交易系統專題

> 本章深入金融交易系統的核心組件,涵蓋 FIX 協定、訂單簿、訂單管理、風控與交易策略。

---

## 9.1 FIX 協定 (Financial Information eXchange)

### 9.1.1 FIX 協定概述

FIX 協定是金融業的標準通訊協定,用於電子交易訊息傳遞。

**核心特點:**
- 標籤-值對 (Tag-Value Pair) 格式
- 純文字協定 (ASCII)
- 自描述訊息結構
- 版本: FIX 4.0, 4.2, 4.4, 5.0

**訊息格式:**
```
8=FIX.4.4|9=176|35=D|49=SENDER|56=TARGET|34=123|52=20231120-10:30:00.000|
11=ORDER001|21=1|55=AAPL|54=1|60=20231120-10:30:00.000|38=100|40=2|44=150.50|
10=128|
```

> `|` 為 SOH (0x01) 分隔符號

---

### 9.1.2 FIX 訊息結構

#### 訊息組成

```
+------------------+------------------+------------------+
|     Header       |      Body        |     Trailer      |
+------------------+------------------+------------------+
| 8, 9, 35, 49,    | 業務相關欄位     | 10 (Checksum)    |
| 56, 34, 52       |                  |                  |
+------------------+------------------+------------------+
```

**必要 Header 欄位:**

| Tag | 名稱 | 說明 |
|-----|------|------|
| 8 | BeginString | 協定版本 (FIX.4.4) |
| 9 | BodyLength | 訊息體長度 |
| 35 | MsgType | 訊息類型 |
| 49 | SenderCompID | 發送方 ID |
| 56 | TargetCompID | 接收方 ID |
| 34 | MsgSeqNum | 訊息序號 |
| 52 | SendingTime | 發送時間 |

**常見訊息類型 (Tag 35):**

| 值 | 類型 | 說明 |
|----|------|------|
| A | Logon | 登入 |
| 0 | Heartbeat | 心跳 |
| 1 | TestRequest | 測試請求 |
| 5 | Logout | 登出 |
| D | NewOrderSingle | 新訂單 |
| F | OrderCancelRequest | 取消訂單 |
| G | OrderCancelReplaceRequest | 修改訂單 |
| 8 | ExecutionReport | 執行回報 |

---

### 9.1.3 常見 FIX 訊息

#### New Order Single (35=D)

```cpp
// 新訂單關鍵欄位
struct NewOrderSingle {
    std::string clOrdId;      // 11: 客戶訂單ID
    char handlInst;           // 21: 處理指令 (1=自動)
    std::string symbol;       // 55: 標的
    char side;                // 54: 方向 (1=買, 2=賣)
    std::string transactTime; // 60: 交易時間
    int orderQty;             // 38: 數量
    char ordType;             // 40: 訂單類型 (1=市價, 2=限價)
    double price;             // 44: 價格 (限價單)
    char timeInForce;         // 59: 有效期 (0=Day, 1=GTC, 3=IOC, 4=FOK)
};
```

#### Execution Report (35=8)

```cpp
// 執行回報關鍵欄位
struct ExecutionReport {
    std::string orderId;      // 37: 交易所訂單ID
    std::string execId;       // 17: 執行ID
    char execType;            // 150: 執行類型
    char ordStatus;           // 39: 訂單狀態
    std::string symbol;       // 55: 標的
    char side;                // 54: 方向
    int leavesQty;            // 151: 剩餘數量
    int cumQty;               // 14: 累計成交量
    double avgPx;             // 6: 平均成交價
};

// 訂單狀態 (Tag 39)
// 0=New, 1=PartiallyFilled, 2=Filled, 4=Canceled, 8=Rejected
```

---

### 9.1.4 高性能 FIX 解析器

#### 設計原則

1. **零拷貝**: 直接在接收緩衝區操作
2. **避免字串轉換**: 使用 string_view
3. **快速整數解析**: 手寫解析避免 stoi
4. **標籤查找優化**: 使用查表或完美雜湊

#### 零拷貝解析實作

```cpp
#include <string_view>
#include <cstdint>
#include <array>

class FastFIXParser {
public:
    // 標籤-值對結構
    struct Field {
        int tag;
        std::string_view value;
    };
    
    // 解析結果
    static constexpr int MAX_FIELDS = 64;
    std::array<Field, MAX_FIELDS> fields_;
    int fieldCount_ = 0;
    
    // 解析整個訊息
    bool parse(const char* data, size_t len) {
        fieldCount_ = 0;
        const char* end = data + len;
        const char* pos = data;
        
        while (pos < end && fieldCount_ < MAX_FIELDS) {
            // 解析標籤
            int tag = 0;
            while (pos < end && *pos != '=') {
                tag = tag * 10 + (*pos - '0');
                ++pos;
            }
            if (pos >= end) return false;
            ++pos; // skip '='
            
            // 解析值
            const char* valueStart = pos;
            while (pos < end && *pos != '\x01') {
                ++pos;
            }
            
            fields_[fieldCount_++] = {
                tag, 
                std::string_view(valueStart, pos - valueStart)
            };
            
            if (pos < end) ++pos; // skip SOH
        }
        
        return true;
    }
    
    // 快速查找特定標籤
    std::string_view getField(int tag) const {
        for (int i = 0; i < fieldCount_; ++i) {
            if (fields_[i].tag == tag) {
                return fields_[i].value;
            }
        }
        return {};
    }
    
    // 快速整數解析
    static int64_t parseInteger(std::string_view sv) {
        int64_t result = 0;
        bool negative = false;
        const char* p = sv.data();
        const char* end = p + sv.size();
        
        if (p < end && *p == '-') {
            negative = true;
            ++p;
        }
        
        while (p < end) {
            result = result * 10 + (*p - '0');
            ++p;
        }
        
        return negative ? -result : result;
    }
    
    // 快速浮點解析 (定點數)
    static double parsePrice(std::string_view sv) {
        int64_t intPart = 0;
        int64_t fracPart = 0;
        int fracDigits = 0;
        bool inFrac = false;
        bool negative = false;
        
        const char* p = sv.data();
        const char* end = p + sv.size();
        
        if (p < end && *p == '-') {
            negative = true;
            ++p;
        }
        
        while (p < end) {
            if (*p == '.') {
                inFrac = true;
            } else if (inFrac) {
                fracPart = fracPart * 10 + (*p - '0');
                ++fracDigits;
            } else {
                intPart = intPart * 10 + (*p - '0');
            }
            ++p;
        }
        
        double result = intPart;
        if (fracDigits > 0) {
            static const double pow10[] = {
                1, 10, 100, 1000, 10000, 100000, 1000000
            };
            result += fracPart / pow10[fracDigits];
        }
        
        return negative ? -result : result;
    }
};
```

#### 使用查表優化標籤查找

```cpp
class IndexedFIXParser : public FastFIXParser {
public:
    // 常用標籤的直接索引
    static constexpr int TAG_INDEX_SIZE = 256;
    std::array<int, TAG_INDEX_SIZE> tagIndex_;
    
    IndexedFIXParser() {
        tagIndex_.fill(-1);
    }
    
    bool parse(const char* data, size_t len) {
        tagIndex_.fill(-1);
        bool result = FastFIXParser::parse(data, len);
        
        // 建立索引
        for (int i = 0; i < fieldCount_; ++i) {
            int tag = fields_[i].tag;
            if (tag < TAG_INDEX_SIZE) {
                tagIndex_[tag] = i;
            }
        }
        
        return result;
    }
    
    // O(1) 查找常用標籤
    std::string_view getFieldFast(int tag) const {
        if (tag < TAG_INDEX_SIZE && tagIndex_[tag] >= 0) {
            return fields_[tagIndex_[tag]].value;
        }
        return getField(tag);
    }
    
    // 直接取得訊息類型
    char getMsgType() const {
        auto sv = getFieldFast(35);
        return sv.empty() ? '\0' : sv[0];
    }
};
```

#### Checksum 計算

```cpp
// FIX checksum: 所有字元 (包含 SOH) 的和 mod 256
int calculateChecksum(const char* data, size_t len) {
    int sum = 0;
    for (size_t i = 0; i < len; ++i) {
        sum += static_cast<unsigned char>(data[i]);
    }
    return sum % 256;
}

// 格式化為 3 位數字串
std::string formatChecksum(int checksum) {
    char buf[4];
    snprintf(buf, sizeof(buf), "%03d", checksum);
    return buf;
}
```

---

### 9.1.5 FIX Session 管理

```cpp
class FIXSession {
public:
    enum class State {
        Disconnected,
        LogonSent,
        Active,
        LogoutSent
    };
    
private:
    State state_ = State::Disconnected;
    int inSeqNum_ = 1;   // 預期收到的序號
    int outSeqNum_ = 1;  // 下一個發送序號
    
    std::string senderCompId_;
    std::string targetCompId_;
    
public:
    // 處理接收訊息
    void onMessage(const IndexedFIXParser& parser) {
        // 檢查序號
        int seqNum = FastFIXParser::parseInteger(parser.getFieldFast(34));
        if (seqNum != inSeqNum_) {
            // 序號不連續,需要重傳請求
            handleSequenceGap(seqNum);
            return;
        }
        ++inSeqNum_;
        
        // 根據訊息類型處理
        char msgType = parser.getMsgType();
        switch (msgType) {
            case 'A': onLogon(parser); break;
            case '0': onHeartbeat(parser); break;
            case '1': onTestRequest(parser); break;
            case '5': onLogout(parser); break;
            case '8': onExecutionReport(parser); break;
            default: break;
        }
    }
    
    // 發送訊息
    void sendNewOrder(const NewOrderSingle& order) {
        // 組裝 FIX 訊息...
        // 使用 outSeqNum_++
    }
    
private:
    void onLogon(const IndexedFIXParser& parser) {
        state_ = State::Active;
    }
    
    void onHeartbeat(const IndexedFIXParser& parser) {
        // 重置心跳計時器
    }
    
    void onTestRequest(const IndexedFIXParser& parser) {
        // 回應 Heartbeat
    }
    
    void onLogout(const IndexedFIXParser& parser) {
        state_ = State::Disconnected;
    }
    
    void onExecutionReport(const IndexedFIXParser& parser) {
        // 處理執行回報
    }
    
    void handleSequenceGap(int receivedSeq) {
        // 發送 ResendRequest
    }
};
```

---

## 9.2 市場數據處理

### 9.2.1 市場數據類型

**Level 1 (BBO - Best Bid/Offer):**
- 最佳買價/賣價
- 最佳買量/賣量
- 最新成交價/量

**Level 2 (Market Depth):**
- 多個價格層級
- 每個價格的累計數量
- 通常 5-20 個層級

**Level 3 (Full Order Book):**
- 完整訂單資訊
- 每個訂單的 ID、數量、時間
- 可追蹤訂單變化

---

### 9.2.2 市場數據協定

#### 二進制協定

**CME MDP 3.0:**
```cpp
// CME Market Data Protocol 訊息結構
struct MDPMessageHeader {
    uint32_t msgSize;
    uint16_t blockLength;
    uint16_t templateId;
    uint16_t schemaId;
    uint16_t version;
} __attribute__((packed));
```

**NASDAQ ITCH:**
```cpp
// ITCH 訊息類型
struct ITCHAddOrder {
    char messageType;       // 'A'
    uint16_t stockLocate;
    uint16_t trackingNumber;
    uint64_t timestamp;     // 奈秒
    uint64_t orderRefNum;
    char buySell;
    uint32_t shares;
    char stock[8];
    uint32_t price;         // 4 位小數
} __attribute__((packed));
```

#### UDP Multicast 接收

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

class MulticastReceiver {
    int socket_;
    char buffer_[65536];
    
public:
    bool init(const char* multicastAddr, uint16_t port, 
              const char* interfaceAddr) {
        // 建立 UDP socket
        socket_ = socket(AF_INET, SOCK_DGRAM, 0);
        if (socket_ < 0) return false;
        
        // 允許位址重用
        int reuse = 1;
        setsockopt(socket_, SOL_SOCKET, SO_REUSEADDR, 
                   &reuse, sizeof(reuse));
        
        // 綁定位址
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
    
    // 接收訊息
    ssize_t receive() {
        return recv(socket_, buffer_, sizeof(buffer_), 0);
    }
    
    const char* data() const { return buffer_; }
};
```

---

### 9.2.3 訂單簿實作

訂單簿是 HFT 系統的核心數據結構。

#### 資料結構設計

```cpp
#include <map>
#include <list>
#include <unordered_map>

// 單個訂單
struct Order {
    uint64_t orderId;
    double price;
    int quantity;
    bool isBuy;
    uint64_t timestamp;
};

// 價格層級
struct PriceLevel {
    double price;
    int totalQuantity;
    std::list<Order*> orders;  // FIFO 佇列
};

// 訂單簿
class OrderBook {
public:
    using PriceLevels = std::map<double, PriceLevel>;
    
private:
    std::string symbol_;
    
    // 買方 (降序): 最高價在前
    PriceLevels bids_;
    // 賣方 (升序): 最低價在前
    PriceLevels asks_;
    
    // 訂單 ID 到訂單的映射 (快速查找)
    std::unordered_map<uint64_t, Order*> orderMap_;
    
public:
    explicit OrderBook(const std::string& symbol) : symbol_(symbol) {}
    
    // 新增訂單
    void addOrder(uint64_t orderId, double price, int qty, bool isBuy) {
        auto* order = new Order{orderId, price, qty, isBuy, 
                                getCurrentTimestamp()};
        orderMap_[orderId] = order;
        
        auto& levels = isBuy ? bids_ : asks_;
        auto& level = levels[price];
        
        if (level.orders.empty()) {
            level.price = price;
            level.totalQuantity = 0;
        }
        
        level.orders.push_back(order);
        level.totalQuantity += qty;
    }
    
    // 修改訂單
    void modifyOrder(uint64_t orderId, int newQty) {
        auto it = orderMap_.find(orderId);
        if (it == orderMap_.end()) return;
        
        Order* order = it->second;
        auto& levels = order->isBuy ? bids_ : asks_;
        auto levelIt = levels.find(order->price);
        
        if (levelIt != levels.end()) {
            levelIt->second.totalQuantity += (newQty - order->quantity);
            order->quantity = newQty;
        }
    }
    
    // 刪除訂單
    void deleteOrder(uint64_t orderId) {
        auto it = orderMap_.find(orderId);
        if (it == orderMap_.end()) return;
        
        Order* order = it->second;
        auto& levels = order->isBuy ? bids_ : asks_;
        auto levelIt = levels.find(order->price);
        
        if (levelIt != levels.end()) {
            auto& level = levelIt->second;
            level.totalQuantity -= order->quantity;
            
            // 從列表中移除
            level.orders.remove(order);
            
            // 如果價格層級空了,移除它
            if (level.orders.empty()) {
                levels.erase(levelIt);
            }
        }
        
        orderMap_.erase(it);
        delete order;
    }
    
    // 執行 (部分或全部成交)
    void executeOrder(uint64_t orderId, int executedQty) {
        auto it = orderMap_.find(orderId);
        if (it == orderMap_.end()) return;
        
        Order* order = it->second;
        order->quantity -= executedQty;
        
        auto& levels = order->isBuy ? bids_ : asks_;
        auto levelIt = levels.find(order->price);
        
        if (levelIt != levels.end()) {
            levelIt->second.totalQuantity -= executedQty;
        }
        
        if (order->quantity <= 0) {
            deleteOrder(orderId);
        }
    }
    
    // 取得 BBO
    struct BBO {
        double bidPrice = 0;
        int bidSize = 0;
        double askPrice = 0;
        int askSize = 0;
    };
    
    BBO getBBO() const {
        BBO bbo;
        
        if (!bids_.empty()) {
            auto it = bids_.rbegin();  // 最高價
            bbo.bidPrice = it->first;
            bbo.bidSize = it->second.totalQuantity;
        }
        
        if (!asks_.empty()) {
            auto it = asks_.begin();   // 最低價
            bbo.askPrice = it->first;
            bbo.askSize = it->second.totalQuantity;
        }
        
        return bbo;
    }
    
    // 取得深度數據
    std::vector<std::pair<double, int>> getDepth(bool isBuy, int levels) const {
        std::vector<std::pair<double, int>> result;
        const auto& book = isBuy ? bids_ : asks_;
        
        if (isBuy) {
            // 買方從高到低
            for (auto it = book.rbegin(); 
                 it != book.rend() && result.size() < levels; ++it) {
                result.emplace_back(it->first, it->second.totalQuantity);
            }
        } else {
            // 賣方從低到高
            for (auto it = book.begin(); 
                 it != book.end() && result.size() < levels; ++it) {
                result.emplace_back(it->first, it->second.totalQuantity);
            }
        }
        
        return result;
    }
    
private:
    uint64_t getCurrentTimestamp() {
        return std::chrono::duration_cast<std::chrono::nanoseconds>(
            std::chrono::high_resolution_clock::now().time_since_epoch()
        ).count();
    }
};
```

---

### 9.2.4 高性能訂單簿優化

#### 使用物件池避免動態分配

```cpp
#include <array>
#include <cstddef>

template<typename T, size_t PoolSize>
class ObjectPool {
    std::array<T, PoolSize> pool_;
    std::array<T*, PoolSize> freeList_;
    size_t freeCount_ = PoolSize;
    
public:
    ObjectPool() {
        for (size_t i = 0; i < PoolSize; ++i) {
            freeList_[i] = &pool_[i];
        }
    }
    
    T* allocate() {
        if (freeCount_ == 0) return nullptr;
        return freeList_[--freeCount_];
    }
    
    void deallocate(T* ptr) {
        freeList_[freeCount_++] = ptr;
    }
};

// 使用物件池的訂單簿
class PooledOrderBook {
    static constexpr size_t MAX_ORDERS = 100000;
    ObjectPool<Order, MAX_ORDERS> orderPool_;
    
    // ... 其他成員
    
public:
    void addOrder(uint64_t orderId, double price, int qty, bool isBuy) {
        Order* order = orderPool_.allocate();
        if (!order) return;  // 池滿
        
        *order = {orderId, price, qty, isBuy, getCurrentTimestamp()};
        // ... 加入訂單簿
    }
    
    void deleteOrder(uint64_t orderId) {
        // ... 從訂單簿移除
        orderPool_.deallocate(order);
    }
};
```

#### 使用平坦陣列提高 Cache 效能

```cpp
// 適合價格範圍有限的情況
class ArrayOrderBook {
    static constexpr int PRICE_LEVELS = 10000;
    static constexpr double TICK_SIZE = 0.01;
    static constexpr double BASE_PRICE = 100.0;
    
    // 直接用價格索引
    std::array<int, PRICE_LEVELS> bidQty_{};
    std::array<int, PRICE_LEVELS> askQty_{};
    
    int priceToIndex(double price) const {
        return static_cast<int>((price - BASE_PRICE) / TICK_SIZE);
    }
    
    double indexToPrice(int index) const {
        return BASE_PRICE + index * TICK_SIZE;
    }
    
public:
    void addBid(double price, int qty) {
        int idx = priceToIndex(price);
        if (idx >= 0 && idx < PRICE_LEVELS) {
            bidQty_[idx] += qty;
        }
    }
    
    void addAsk(double price, int qty) {
        int idx = priceToIndex(price);
        if (idx >= 0 && idx < PRICE_LEVELS) {
            askQty_[idx] += qty;
        }
    }
    
    // 快速 BBO 查找
    std::pair<double, int> getBestBid() const {
        for (int i = PRICE_LEVELS - 1; i >= 0; --i) {
            if (bidQty_[i] > 0) {
                return {indexToPrice(i), bidQty_[i]};
            }
        }
        return {0, 0};
    }
    
    std::pair<double, int> getBestAsk() const {
        for (int i = 0; i < PRICE_LEVELS; ++i) {
            if (askQty_[i] > 0) {
                return {indexToPrice(i), askQty_[i]};
            }
        }
        return {0, 0};
    }
};
```

#### 使用侵入式容器

```cpp
#include <boost/intrusive/list.hpp>
#include <boost/intrusive/set.hpp>

namespace bi = boost::intrusive;

// 侵入式訂單結構
struct IntrusiveOrder : 
    public bi::list_base_hook<>,   // 用於價格層級的訂單列表
    public bi::set_base_hook<>     // 用於全局訂單查找
{
    uint64_t orderId;
    double price;
    int quantity;
    bool isBuy;
};

// 比較器
struct OrderIdCompare {
    bool operator()(const IntrusiveOrder& a, const IntrusiveOrder& b) const {
        return a.orderId < b.orderId;
    }
};

class IntrusiveOrderBook {
    // 訂單按 ID 排序的集合 (快速查找)
    using OrderSet = bi::set<IntrusiveOrder, 
                             bi::compare<OrderIdCompare>>;
    
    // 價格層級中的訂單列表
    using OrderList = bi::list<IntrusiveOrder>;
    
    OrderSet allOrders_;
    // ... 價格層級管理
};
```

---

## 9.3 訂單管理系統 (OMS)

### 9.3.1 OMS 職責

- **訂單生命週期管理**: 追蹤訂單從建立到完成
- **路由決策**: 選擇最佳執行場所
- **訂單分割**: 大訂單分成小訂單
- **狀態同步**: 與交易所保持一致

---

### 9.3.2 訂單狀態機

```cpp
enum class OrderStatus {
    Created,        // 剛建立
    PendingNew,     // 已發送,等待確認
    New,            // 交易所已接受
    PartiallyFilled,// 部分成交
    Filled,         // 完全成交
    PendingCancel,  // 取消中
    Canceled,       // 已取消
    PendingReplace, // 修改中
    Rejected,       // 被拒絕
    Expired         // 已過期
};

class OrderStateMachine {
public:
    bool canTransition(OrderStatus from, OrderStatus to) const {
        // 定義有效的狀態轉換
        static const std::set<std::pair<OrderStatus, OrderStatus>> validTransitions = {
            {OrderStatus::Created, OrderStatus::PendingNew},
            {OrderStatus::PendingNew, OrderStatus::New},
            {OrderStatus::PendingNew, OrderStatus::Rejected},
            {OrderStatus::New, OrderStatus::PartiallyFilled},
            {OrderStatus::New, OrderStatus::Filled},
            {OrderStatus::New, OrderStatus::PendingCancel},
            {OrderStatus::New, OrderStatus::PendingReplace},
            {OrderStatus::New, OrderStatus::Expired},
            {OrderStatus::PartiallyFilled, OrderStatus::Filled},
            {OrderStatus::PartiallyFilled, OrderStatus::PendingCancel},
            {OrderStatus::PartiallyFilled, OrderStatus::PendingReplace},
            {OrderStatus::PendingCancel, OrderStatus::Canceled},
            {OrderStatus::PendingCancel, OrderStatus::Filled},  // 取消前成交
            {OrderStatus::PendingReplace, OrderStatus::New},    // 修改成功
            {OrderStatus::PendingReplace, OrderStatus::Rejected},
        };
        
        return validTransitions.count({from, to}) > 0;
    }
};
```

---

### 9.3.3 訂單管理實作

```cpp
#include <unordered_map>
#include <string>
#include <functional>

struct ManagedOrder {
    std::string clientOrderId;
    std::string exchangeOrderId;
    std::string symbol;
    
    OrderStatus status = OrderStatus::Created;
    char side;              // '1'=Buy, '2'=Sell
    char orderType;         // '1'=Market, '2'=Limit
    char timeInForce;       // '0'=Day, '3'=IOC, '4'=FOK
    
    int quantity;
    int filledQuantity = 0;
    int leavesQuantity;
    
    double price;
    double avgFillPrice = 0;
    
    uint64_t createTime;
    uint64_t lastUpdateTime;
};

class OrderManager {
public:
    using OrderCallback = std::function<void(const ManagedOrder&)>;
    
private:
    std::unordered_map<std::string, ManagedOrder> ordersByClientId_;
    std::unordered_map<std::string, std::string> exchangeToClientId_;
    
    OrderStateMachine stateMachine_;
    OrderCallback onOrderUpdate_;
    
public:
    void setOrderCallback(OrderCallback cb) {
        onOrderUpdate_ = std::move(cb);
    }
    
    // 建立新訂單
    std::string createOrder(const std::string& symbol, char side,
                            int qty, double price, char orderType,
                            char timeInForce) {
        ManagedOrder order;
        order.clientOrderId = generateClientOrderId();
        order.symbol = symbol;
        order.side = side;
        order.quantity = qty;
        order.leavesQuantity = qty;
        order.price = price;
        order.orderType = orderType;
        order.timeInForce = timeInForce;
        order.createTime = getCurrentTimestamp();
        
        ordersByClientId_[order.clientOrderId] = order;
        
        return order.clientOrderId;
    }
    
    // 處理執行回報
    void onExecutionReport(const std::string& clientOrderId,
                           const std::string& exchangeOrderId,
                           char execType, OrderStatus newStatus,
                           int filledQty, double fillPrice) {
        auto it = ordersByClientId_.find(clientOrderId);
        if (it == ordersByClientId_.end()) return;
        
        ManagedOrder& order = it->second;
        
        // 檢查狀態轉換是否有效
        if (!stateMachine_.canTransition(order.status, newStatus)) {
            // 記錄錯誤
            return;
        }
        
        // 更新訂單
        order.exchangeOrderId = exchangeOrderId;
        exchangeToClientId_[exchangeOrderId] = clientOrderId;
        
        if (filledQty > 0) {
            // 更新成交資訊
            double totalValue = order.avgFillPrice * order.filledQuantity 
                              + fillPrice * filledQty;
            order.filledQuantity += filledQty;
            order.leavesQuantity -= filledQty;
            order.avgFillPrice = totalValue / order.filledQuantity;
        }
        
        order.status = newStatus;
        order.lastUpdateTime = getCurrentTimestamp();
        
        // 回調通知
        if (onOrderUpdate_) {
            onOrderUpdate_(order);
        }
    }
    
    // 取消訂單
    bool cancelOrder(const std::string& clientOrderId) {
        auto it = ordersByClientId_.find(clientOrderId);
        if (it == ordersByClientId_.end()) return false;
        
        ManagedOrder& order = it->second;
        
        if (!stateMachine_.canTransition(order.status, 
                                         OrderStatus::PendingCancel)) {
            return false;
        }
        
        order.status = OrderStatus::PendingCancel;
        // 發送取消請求到交易所...
        
        return true;
    }
    
    // 查詢訂單
    const ManagedOrder* getOrder(const std::string& clientOrderId) const {
        auto it = ordersByClientId_.find(clientOrderId);
        return it != ordersByClientId_.end() ? &it->second : nullptr;
    }
    
private:
    std::string generateClientOrderId() {
        static uint64_t counter = 0;
        return "ORD" + std::to_string(++counter);
    }
    
    uint64_t getCurrentTimestamp() {
        return std::chrono::duration_cast<std::chrono::nanoseconds>(
            std::chrono::system_clock::now().time_since_epoch()
        ).count();
    }
};
```

---

## 9.4 風險管理

### 9.4.1 前置風控 (Pre-Trade Risk)

前置風控必須在訂單發送前完成,且延遲要求極低 (奈秒級)。

**風控檢查項目:**
- 訂單數量限制
- 價格合理性
- 持倉限制
- 資金/保證金檢查
- 頻率限制
- 每日損益限制

---

### 9.4.2 風控系統實作

```cpp
#include <atomic>
#include <chrono>

struct RiskLimits {
    int maxOrderQuantity = 10000;
    double maxPriceDeviation = 0.05;  // 5%
    int maxPosition = 100000;
    double maxNotional = 10000000;    // 1000萬
    int maxOrdersPerSecond = 100;
    double maxDailyLoss = 100000;
};

struct RiskState {
    std::atomic<int> position{0};
    std::atomic<double> notional{0};
    std::atomic<int> ordersInWindow{0};
    std::atomic<double> dailyPnL{0};
    std::atomic<uint64_t> windowStart{0};
};

class RiskManager {
public:
    enum class RiskResult {
        Passed,
        QuantityExceeded,
        PriceDeviation,
        PositionExceeded,
        NotionalExceeded,
        RateLimitExceeded,
        DailyLossExceeded
    };
    
private:
    RiskLimits limits_;
    RiskState state_;
    
public:
    void setLimits(const RiskLimits& limits) {
        limits_ = limits;
    }
    
    // 快速風控檢查
    RiskResult checkOrder(int quantity, double price, char side,
                          double referencePrice) {
        // 1. 數量檢查
        if (quantity > limits_.maxOrderQuantity) {
            return RiskResult::QuantityExceeded;
        }
        
        // 2. 價格偏離檢查
        double deviation = std::abs(price - referencePrice) / referencePrice;
        if (deviation > limits_.maxPriceDeviation) {
            return RiskResult::PriceDeviation;
        }
        
        // 3. 持倉檢查
        int positionDelta = (side == '1') ? quantity : -quantity;
        int newPosition = state_.position.load(std::memory_order_relaxed) 
                        + positionDelta;
        if (std::abs(newPosition) > limits_.maxPosition) {
            return RiskResult::PositionExceeded;
        }
        
        // 4. 名目金額檢查
        double orderNotional = quantity * price;
        double newNotional = state_.notional.load(std::memory_order_relaxed) 
                           + orderNotional;
        if (newNotional > limits_.maxNotional) {
            return RiskResult::NotionalExceeded;
        }
        
        // 5. 頻率限制
        if (!checkRateLimit()) {
            return RiskResult::RateLimitExceeded;
        }
        
        // 6. 每日損益檢查
        if (state_.dailyPnL.load(std::memory_order_relaxed) 
            < -limits_.maxDailyLoss) {
            return RiskResult::DailyLossExceeded;
        }
        
        return RiskResult::Passed;
    }
    
    // 更新持倉
    void updatePosition(int delta) {
        state_.position.fetch_add(delta, std::memory_order_relaxed);
    }
    
    // 更新損益
    void updatePnL(double delta) {
        // 使用 CAS 更新 double
        double current = state_.dailyPnL.load(std::memory_order_relaxed);
        while (!state_.dailyPnL.compare_exchange_weak(
            current, current + delta, std::memory_order_relaxed));
    }
    
private:
    bool checkRateLimit() {
        auto now = std::chrono::steady_clock::now().time_since_epoch().count();
        auto windowStart = state_.windowStart.load(std::memory_order_relaxed);
        
        // 1 秒窗口
        constexpr uint64_t WINDOW_NS = 1000000000;
        
        if (now - windowStart > WINDOW_NS) {
            // 新窗口
            state_.windowStart.store(now, std::memory_order_relaxed);
            state_.ordersInWindow.store(1, std::memory_order_relaxed);
            return true;
        }
        
        int orders = state_.ordersInWindow.fetch_add(
            1, std::memory_order_relaxed);
        return orders < limits_.maxOrdersPerSecond;
    }
};
```

---

### 9.4.3 無鎖風控優化

```cpp
// 更快的風控檢查 - 使用 branchless 技術
class FastRiskChecker {
    RiskLimits limits_;
    
public:
    // 批量檢查,利用 SIMD 指令
    bool checkBatch(const int* quantities, const double* prices,
                    int count, double refPrice) {
        bool allPassed = true;
        
        for (int i = 0; i < count; ++i) {
            // Branchless quantity check
            bool qtyOk = quantities[i] <= limits_.maxOrderQuantity;
            
            // Branchless price check
            double dev = std::abs(prices[i] - refPrice) / refPrice;
            bool priceOk = dev <= limits_.maxPriceDeviation;
            
            allPassed &= (qtyOk & priceOk);
        }
        
        return allPassed;
    }
    
    // 使用查表的快速拒絕
    static constexpr int QUANTITY_BUCKETS = 256;
    std::array<bool, QUANTITY_BUCKETS> quantityLookup_;
    
    void initLookup() {
        for (int i = 0; i < QUANTITY_BUCKETS; ++i) {
            quantityLookup_[i] = (i * 100 <= limits_.maxOrderQuantity);
        }
    }
    
    // O(1) 數量檢查 (量化到100的倍數)
    bool fastQuantityCheck(int quantity) const {
        int bucket = quantity / 100;
        if (bucket >= QUANTITY_BUCKETS) return false;
        return quantityLookup_[bucket];
    }
};
```

---

## 9.5 交易策略

### 9.5.1 市場製造策略 (Market Making)

市場製造者在買賣兩側持續報價,賺取買賣價差。

```cpp
class MarketMakingStrategy {
public:
    struct Parameters {
        double spreadBps = 10;        // 價差 (基點)
        int quoteSize = 100;          // 報價數量
        double positionLimit = 1000;  // 持倉限制
        double skewFactor = 0.1;      // 持倉偏移因子
    };
    
private:
    Parameters params_;
    std::string symbol_;
    int position_ = 0;
    
public:
    struct Quote {
        double bidPrice;
        int bidSize;
        double askPrice;
        int askSize;
    };
    
    Quote generateQuote(double midPrice) {
        Quote quote;
        
        // 基本價差
        double halfSpread = midPrice * params_.spreadBps / 10000 / 2;
        
        // 根據持倉調整 (持倉多則提高賣價降低買價)
        double skew = position_ * params_.skewFactor * halfSpread 
                    / params_.positionLimit;
        
        quote.bidPrice = midPrice - halfSpread - skew;
        quote.askPrice = midPrice + halfSpread - skew;
        
        // 調整報價數量
        quote.bidSize = params_.quoteSize;
        quote.askSize = params_.quoteSize;
        
        // 接近持倉限制時減少報價
        if (position_ > params_.positionLimit * 0.8) {
            quote.bidSize = 0;  // 停止買入
        }
        if (position_ < -params_.positionLimit * 0.8) {
            quote.askSize = 0;  // 停止賣出
        }
        
        return quote;
    }
    
    void onFill(int qty, char side) {
        if (side == '1') {
            position_ += qty;  // 買入
        } else {
            position_ -= qty;  // 賣出
        }
    }
};
```

---

### 9.5.2 策略引擎架構

```cpp
class StrategyEngine {
public:
    // 事件處理介面
    virtual void onMarketData(const OrderBook::BBO& bbo) = 0;
    virtual void onOrderUpdate(const ManagedOrder& order) = 0;
    virtual void onTimer() = 0;
    
    virtual ~StrategyEngine() = default;
};

class EventDrivenStrategy : public StrategyEngine {
    MarketMakingStrategy mm_;
    OrderManager& orderManager_;
    RiskManager& riskManager_;
    
    std::string currentBidOrderId_;
    std::string currentAskOrderId_;
    
public:
    EventDrivenStrategy(OrderManager& om, RiskManager& rm)
        : orderManager_(om), riskManager_(rm) {}
    
    void onMarketData(const OrderBook::BBO& bbo) override {
        // 計算中間價
        double midPrice = (bbo.bidPrice + bbo.askPrice) / 2;
        
        // 生成報價
        auto quote = mm_.generateQuote(midPrice);
        
        // 取消舊訂單
        if (!currentBidOrderId_.empty()) {
            orderManager_.cancelOrder(currentBidOrderId_);
        }
        if (!currentAskOrderId_.empty()) {
            orderManager_.cancelOrder(currentAskOrderId_);
        }
        
        // 風控檢查並發送新訂單
        if (quote.bidSize > 0) {
            auto result = riskManager_.checkOrder(
                quote.bidSize, quote.bidPrice, '1', midPrice);
            
            if (result == RiskManager::RiskResult::Passed) {
                currentBidOrderId_ = orderManager_.createOrder(
                    "AAPL", '1', quote.bidSize, quote.bidPrice, '2', '0');
            }
        }
        
        if (quote.askSize > 0) {
            auto result = riskManager_.checkOrder(
                quote.askSize, quote.askPrice, '2', midPrice);
            
            if (result == RiskManager::RiskResult::Passed) {
                currentAskOrderId_ = orderManager_.createOrder(
                    "AAPL", '2', quote.askSize, quote.askPrice, '2', '0');
            }
        }
    }
    
    void onOrderUpdate(const ManagedOrder& order) override {
        if (order.status == OrderStatus::PartiallyFilled ||
            order.status == OrderStatus::Filled) {
            mm_.onFill(order.filledQuantity, order.side);
            riskManager_.updatePosition(
                order.side == '1' ? order.filledQuantity 
                                  : -order.filledQuantity);
        }
    }
    
    void onTimer() override {
        // 定期健康檢查、統計等
    }
};
```

---

## 9.6 延遲預算與系統設計

### 9.6.1 延遲預算分配

**目標: 端到端延遲 < 100 微秒**

```
+------------+     +--------+     +----------+     +--------+     +------+     +-------+     +--------+
| 網路接收   | --> | 協定   | --> | 訂單簿   | --> | 策略   | --> | 風控 | --> | 訂單  | --> | 網路   |
| 10 μs      |     | 解析   |     | 更新     |     | 計算   |     | 5 μs |     | 組裝  |     | 發送   |
|            |     | 5 μs   |     | 5 μs     |     | 20 μs  |     |      |     | 5 μs  |     | 10 μs  |
+------------+     +--------+     +----------+     +--------+     +------+     +-------+     +--------+

緩衝: 40 μs
```

---

### 9.6.2 HFT 系統架構

```cpp
// 主要執行緒設計
class HFTSystem {
    // 執行緒1: 市場數據接收
    // - 綁定到專用 CPU
    // - 使用 SCHED_FIFO
    // - busy poll 接收 UDP
    std::thread marketDataThread_;
    
    // 執行緒2: 策略執行
    // - 從 MD 執行緒接收數據 (SPSC queue)
    // - 執行策略邏輯
    // - 發送訂單到發送執行緒
    std::thread strategyThread_;
    
    // 執行緒3: 訂單發送
    // - 接收策略的訂單請求
    // - FIX 編碼
    // - TCP 發送
    std::thread orderSendThread_;
    
    // 無鎖佇列
    SPSCQueue<MarketDataEvent> mdQueue_;
    SPSCQueue<OrderRequest> orderQueue_;
    
public:
    void start() {
        marketDataThread_ = std::thread([this] {
            setThreadAffinity(1);        // CPU 1
            setRealtimePriority(99);     // SCHED_FIFO
            mlockall(MCL_CURRENT | MCL_FUTURE);
            marketDataLoop();
        });
        
        strategyThread_ = std::thread([this] {
            setThreadAffinity(2);
            setRealtimePriority(98);
            mlockall(MCL_CURRENT | MCL_FUTURE);
            strategyLoop();
        });
        
        orderSendThread_ = std::thread([this] {
            setThreadAffinity(3);
            setRealtimePriority(97);
            mlockall(MCL_CURRENT | MCL_FUTURE);
            orderSendLoop();
        });
    }
    
private:
    void setThreadAffinity(int cpu) {
        cpu_set_t cpuset;
        CPU_ZERO(&cpuset);
        CPU_SET(cpu, &cpuset);
        pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);
    }
    
    void setRealtimePriority(int priority) {
        sched_param param;
        param.sched_priority = priority;
        pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);
    }
    
    void marketDataLoop();
    void strategyLoop();
    void orderSendLoop();
};
```

---

### 9.6.3 共位 (Co-location)

將伺服器放置在交易所機房內,減少網路延遲。

**優勢:**
- 網路延遲從毫秒降到微秒
- 更穩定的延遲
- 直接連接交易所基礎設施

**考量:**
- 高昂的租金
- 空間和功耗限制
- 需要遠端管理能力

---

## 實作練習

### 練習 1: FIX 解析器

實作高性能 FIX 訊息解析器:
- 零拷貝解析
- 快速標籤查找
- 支援 NewOrderSingle 和 ExecutionReport

### 練習 2: 訂單簿

實作完整的訂單簿:
- 支援 Add, Modify, Delete, Execute
- 使用物件池
- 微秒級更新性能

### 練習 3: 訂單狀態機

實作訂單管理系統:
- 完整的狀態轉換
- 訂單生命週期追蹤
- 處理執行回報

### 練習 4: 風控系統

實作前置風控:
- 數量、價格、持倉檢查
- 頻率限制
- 奈秒級檢查延遲

### 練習 5: 市場製造策略

實作簡單的市場製造策略:
- 雙邊報價
- 持倉調整
- 風控整合

### 練習 6: 延遲測量

測量各組件延遲:
- 使用 RDTSC
- 記錄 P50, P99, P99.9
- 識別瓶頸

---

## 參考資料 (References)

1. [FIX Protocol Specification](https://www.fixtrading.org/standards/)
2. [NASDAQ ITCH Protocol](https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHspecification.pdf)
3. [CME MDP 3.0](https://www.cmegroup.com/confluence/display/EPICSANDBOX/MDP+3.0)
4. 《High-Frequency Trading: A Practical Guide to Algorithmic Strategies》 (Irene Aldridge, 2013)
5. 《Trading and Exchanges: Market Microstructure for Practitioners》 (Larry Harris, 2002)
6. [QuickFIX Engine](https://quickfixengine.org/)
