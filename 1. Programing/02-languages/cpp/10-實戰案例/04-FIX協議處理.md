# FIX 協議處理

## 概述

FIX (Financial Information eXchange) 是金融交易行業的標準通訊協議。本文涵蓋 FIX 協議的解析、生成與會話管理。

### 協議特性

| 特性 | 說明 |
|-----|------|
| 格式 | 鍵值對 (tag=value) |
| 分隔符 | SOH (0x01) |
| 版本 | FIX 4.0 - FIX 5.0 |
| 會話 | 登入、心跳、登出 |

### 訊息結構

```
8=FIX.4.2|9=178|35=D|49=SENDER|56=TARGET|34=1|52=20240115-10:30:00|...10=123|
│       │     │    │         │         │    │                        │
Header  Length Type SenderID TargetID Seq  SendingTime              Checksum
```

---

## FIX 訊息解析

### 基礎解析器

```cpp
#include <string>
#include <unordered_map>
#include <string_view>

class FIXMessage {
public:
    using FieldMap = std::unordered_map<int, std::string>;
    
private:
    FieldMap fields_;
    std::string raw_message_;
    
public:
    static constexpr char SOH = '\x01';
    
    bool parse(std::string_view message) {
        raw_message_ = message;
        fields_.clear();
        
        size_t pos = 0;
        
        while (pos < message.size()) {
            size_t eq = message.find('=', pos);
            if (eq == std::string_view::npos) break;
            
            size_t soh = message.find(SOH, eq);
            if (soh == std::string_view::npos) {
                soh = message.size();
            }
            
            int tag = parse_int(message.substr(pos, eq - pos));
            std::string value(message.substr(eq + 1, soh - eq - 1));
            
            fields_[tag] = std::move(value);
            
            pos = soh + 1;
        }
        
        return !fields_.empty();
    }
    
    std::string get_field(int tag) const {
        auto it = fields_.find(tag);
        return (it != fields_.end()) ? it->second : "";
    }
    
    int get_int(int tag) const {
        return std::stoi(get_field(tag));
    }
    
    double get_double(int tag) const {
        return std::stod(get_field(tag));
    }
    
    bool has_field(int tag) const {
        return fields_.find(tag) != fields_.end();
    }
    
    const FieldMap& fields() const { return fields_; }
    
private:
    static int parse_int(std::string_view str) {
        int result = 0;
        for (char c : str) {
            if (c >= '0' && c <= '9') {
                result = result * 10 + (c - '0');
            }
        }
        return result;
    }
};

// FIX 標準 Tag 定義
namespace FIXTag {
    constexpr int BeginString = 8;
    constexpr int BodyLength = 9;
    constexpr int MsgType = 35;
    constexpr int SenderCompID = 49;
    constexpr int TargetCompID = 56;
    constexpr int MsgSeqNum = 34;
    constexpr int SendingTime = 52;
    constexpr int CheckSum = 10;
    
    // 訂單相關
    constexpr int ClOrdID = 11;
    constexpr int Symbol = 55;
    constexpr int Side = 54;
    constexpr int OrderQty = 38;
    constexpr int Price = 44;
    constexpr int OrdType = 40;
}

// 訊息類型
namespace FIXMsgType {
    constexpr const char* Logon = "A";
    constexpr const char* Logout = "5";
    constexpr const char* Heartbeat = "0";
    constexpr const char* TestRequest = "1";
    constexpr const char* ResendRequest = "2";
    constexpr const char* NewOrderSingle = "D";
    constexpr const char* OrderCancelRequest = "F";
    constexpr const char* ExecutionReport = "8";
}
```

### 高效能解析器

```cpp
class FastFIXParser {
    static constexpr size_t MAX_FIELDS = 256;
    
    struct Field {
        int tag;
        const char* value_start;
        size_t value_len;
    };
    
    Field fields_[MAX_FIELDS];
    size_t field_count_{0};
    
public:
    bool parse(const char* data, size_t len) {
        field_count_ = 0;
        
        const char* ptr = data;
        const char* end = data + len;
        
        while (ptr < end && field_count_ < MAX_FIELDS) {
            // 查找 '='
            const char* eq = ptr;
            while (eq < end && *eq != '=') ++eq;
            
            if (eq >= end) break;
            
            // 解析 tag
            int tag = 0;
            for (const char* t = ptr; t < eq; ++t) {
                tag = tag * 10 + (*t - '0');
            }
            
            // 查找 SOH
            const char* soh = eq + 1;
            while (soh < end && *soh != '\x01') ++soh;
            
            // 存儲欄位
            fields_[field_count_].tag = tag;
            fields_[field_count_].value_start = eq + 1;
            fields_[field_count_].value_len = soh - (eq + 1);
            
            ++field_count_;
            
            ptr = soh + 1;
        }
        
        return field_count_ > 0;
    }
    
    const char* get_field(int tag, size_t& len) const {
        for (size_t i = 0; i < field_count_; ++i) {
            if (fields_[i].tag == tag) {
                len = fields_[i].value_len;
                return fields_[i].value_start;
            }
        }
        
        len = 0;
        return nullptr;
    }
    
    int get_int(int tag) const {
        size_t len;
        const char* value = get_field(tag, len);
        
        if (!value) return 0;
        
        int result = 0;
        for (size_t i = 0; i < len; ++i) {
            result = result * 10 + (value[i] - '0');
        }
        
        return result;
    }
};
```

---

## FIX 訊息生成

### 訊息構建器

```cpp
#include <sstream>
#include <iomanip>
#include <chrono>

class FIXMessageBuilder {
    std::ostringstream body_;
    std::string sender_comp_id_;
    std::string target_comp_id_;
    int msg_seq_num_;
    
public:
    FIXMessageBuilder(std::string sender, std::string target, int seq_num)
        : sender_comp_id_(std::move(sender)),
          target_comp_id_(std::move(target)),
          msg_seq_num_(seq_num) {}
    
    FIXMessageBuilder& add_field(int tag, const std::string& value) {
        body_ << tag << '=' << value << FIXMessage::SOH;
        return *this;
    }
    
    FIXMessageBuilder& add_field(int tag, int value) {
        body_ << tag << '=' << value << FIXMessage::SOH;
        return *this;
    }
    
    FIXMessageBuilder& add_field(int tag, double value) {
        body_ << tag << '=' << std::fixed << std::setprecision(2) 
              << value << FIXMessage::SOH;
        return *this;
    }
    
    std::string build(const std::string& msg_type) {
        std::ostringstream header;
        
        // BeginString
        header << FIXTag::BeginString << "=FIX.4.2" << FIXMessage::SOH;
        
        // MsgType
        body_.str("");
        body_.clear();
        
        body_ << FIXTag::MsgType << '=' << msg_type << FIXMessage::SOH;
        body_ << FIXTag::SenderCompID << '=' << sender_comp_id_ << FIXMessage::SOH;
        body_ << FIXTag::TargetCompID << '=' << target_comp_id_ << FIXMessage::SOH;
        body_ << FIXTag::MsgSeqNum << '=' << msg_seq_num_ << FIXMessage::SOH;
        body_ << FIXTag::SendingTime << '=' << get_utc_timestamp() << FIXMessage::SOH;
        
        std::string body = body_.str();
        
        // BodyLength
        header << FIXTag::BodyLength << '=' << body.size() << FIXMessage::SOH;
        
        std::string message = header.str() + body;
        
        // CheckSum
        int checksum = calculate_checksum(message);
        
        std::ostringstream oss;
        oss << message << FIXTag::CheckSum << '=' 
            << std::setfill('0') << std::setw(3) << checksum << FIXMessage::SOH;
        
        return oss.str();
    }
    
private:
    static std::string get_utc_timestamp() {
        auto now = std::chrono::system_clock::now();
        auto time_t = std::chrono::system_clock::to_time_t(now);
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            now.time_since_epoch()) % 1000;
        
        std::ostringstream oss;
        oss << std::put_time(std::gmtime(&time_t), "%Y%m%d-%H:%M:%S");
        oss << '.' << std::setfill('0') << std::setw(3) << ms.count();
        
        return oss.str();
    }
    
    static int calculate_checksum(const std::string& message) {
        int sum = 0;
        for (char c : message) {
            sum += static_cast<unsigned char>(c);
        }
        return sum % 256;
    }
};

// 使用範例
std::string create_new_order() {
    FIXMessageBuilder builder("SENDER", "TARGET", 1);
    
    builder.add_field(FIXTag::ClOrdID, "ORD12345")
           .add_field(FIXTag::Symbol, "AAPL")
           .add_field(FIXTag::Side, "1")  // Buy
           .add_field(FIXTag::OrderQty, 100)
           .add_field(FIXTag::Price, 150.50)
           .add_field(FIXTag::OrdType, "2");  // Limit
    
    return builder.build(FIXMsgType::NewOrderSingle);
}
```

---

## FIX 會話管理

### 會話狀態機

```cpp
#include <atomic>
#include <mutex>

class FIXSession {
public:
    enum class State {
        DISCONNECTED,
        CONNECTING,
        LOGGED_IN,
        LOGGING_OUT
    };
    
private:
    State state_{State::DISCONNECTED};
    std::mutex mutex_;
    
    std::string sender_comp_id_;
    std::string target_comp_id_;
    
    std::atomic<int> outgoing_seq_num_{1};
    std::atomic<int> incoming_seq_num_{1};
    
    std::chrono::steady_clock::time_point last_heartbeat_;
    int heartbeat_interval_{30};  // 秒
    
public:
    FIXSession(std::string sender, std::string target)
        : sender_comp_id_(std::move(sender)),
          target_comp_id_(std::move(target)) {}
    
    std::string create_logon() {
        FIXMessageBuilder builder(sender_comp_id_, target_comp_id_, 
                                 outgoing_seq_num_.fetch_add(1));
        
        builder.add_field(98, "0")  // EncryptMethod (None)
               .add_field(108, heartbeat_interval_);  // HeartBtInt
        
        return builder.build(FIXMsgType::Logon);
    }
    
    std::string create_logout(const std::string& text = "") {
        FIXMessageBuilder builder(sender_comp_id_, target_comp_id_,
                                 outgoing_seq_num_.fetch_add(1));
        
        if (!text.empty()) {
            builder.add_field(58, text);  // Text
        }
        
        return builder.build(FIXMsgType::Logout);
    }
    
    std::string create_heartbeat(const std::string& test_req_id = "") {
        FIXMessageBuilder builder(sender_comp_id_, target_comp_id_,
                                 outgoing_seq_num_.fetch_add(1));
        
        if (!test_req_id.empty()) {
            builder.add_field(112, test_req_id);  // TestReqID
        }
        
        last_heartbeat_ = std::chrono::steady_clock::now();
        
        return builder.build(FIXMsgType::Heartbeat);
    }
    
    bool process_message(const FIXMessage& msg) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        // 驗證序列號
        int seq_num = msg.get_int(FIXTag::MsgSeqNum);
        int expected = incoming_seq_num_.load();
        
        if (seq_num < expected) {
            // 重複訊息
            return false;
        } else if (seq_num > expected) {
            // 序列號跳躍，需要重傳
            request_resend(expected, seq_num - 1);
            return false;
        }
        
        incoming_seq_num_.fetch_add(1);
        
        // 處理訊息類型
        std::string msg_type = msg.get_field(FIXTag::MsgType);
        
        if (msg_type == FIXMsgType::Logon) {
            handle_logon(msg);
        } else if (msg_type == FIXMsgType::Logout) {
            handle_logout(msg);
        } else if (msg_type == FIXMsgType::Heartbeat) {
            handle_heartbeat(msg);
        } else if (msg_type == FIXMsgType::TestRequest) {
            handle_test_request(msg);
        }
        
        return true;
    }
    
    bool needs_heartbeat() const {
        auto now = std::chrono::steady_clock::now();
        auto elapsed = std::chrono::duration_cast<std::chrono::seconds>(
            now - last_heartbeat_);
        
        return elapsed.count() >= heartbeat_interval_;
    }
    
    State state() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return state_;
    }
    
private:
    void handle_logon(const FIXMessage& msg) {
        state_ = State::LOGGED_IN;
        last_heartbeat_ = std::chrono::steady_clock::now();
    }
    
    void handle_logout(const FIXMessage& msg) {
        state_ = State::DISCONNECTED;
    }
    
    void handle_heartbeat(const FIXMessage& msg) {
        last_heartbeat_ = std::chrono::steady_clock::now();
    }
    
    void handle_test_request(const FIXMessage& msg) {
        std::string test_req_id = msg.get_field(112);
        // 發送心跳回應
        create_heartbeat(test_req_id);
    }
    
    void request_resend(int begin_seq, int end_seq) {
        FIXMessageBuilder builder(sender_comp_id_, target_comp_id_,
                                 outgoing_seq_num_.fetch_add(1));
        
        builder.add_field(7, begin_seq)   // BeginSeqNo
               .add_field(16, end_seq);    // EndSeqNo
        
        std::string msg = builder.build(FIXMsgType::ResendRequest);
        // 發送重傳請求...
    }
};
```

---

## 訂單管理

### FIX 訂單介面

```cpp
class FIXOrderInterface {
    FIXSession& session_;
    
    struct OrderState {
        std::string cl_ord_id;
        std::string symbol;
        char side;
        double price;
        uint64_t quantity;
        uint64_t filled_quantity{0};
        std::string status;  // NEW, FILLED, CANCELED, etc.
    };
    
    std::unordered_map<std::string, OrderState> orders_;
    std::mutex mutex_;
    
public:
    explicit FIXOrderInterface(FIXSession& session) : session_(session) {}
    
    std::string send_new_order(const std::string& symbol, char side,
                               double price, uint64_t quantity) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        std::string cl_ord_id = generate_cl_ord_id();
        
        FIXMessageBuilder builder("SENDER", "TARGET", 
                                 session_.outgoing_seq_num());
        
        builder.add_field(FIXTag::ClOrdID, cl_ord_id)
               .add_field(FIXTag::Symbol, symbol)
               .add_field(FIXTag::Side, std::string(1, side))
               .add_field(FIXTag::OrderQty, static_cast<int>(quantity))
               .add_field(FIXTag::Price, price)
               .add_field(FIXTag::OrdType, "2");  // Limit
        
        std::string msg = builder.build(FIXMsgType::NewOrderSingle);
        
        // 記錄訂單狀態
        OrderState state;
        state.cl_ord_id = cl_ord_id;
        state.symbol = symbol;
        state.side = side;
        state.price = price;
        state.quantity = quantity;
        state.status = "PENDING_NEW";
        
        orders_[cl_ord_id] = state;
        
        return msg;
    }
    
    std::string send_cancel_order(const std::string& orig_cl_ord_id) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        auto it = orders_.find(orig_cl_ord_id);
        if (it == orders_.end()) {
            return "";
        }
        
        std::string new_cl_ord_id = generate_cl_ord_id();
        
        FIXMessageBuilder builder("SENDER", "TARGET",
                                 session_.outgoing_seq_num());
        
        builder.add_field(FIXTag::ClOrdID, new_cl_ord_id)
               .add_field(41, orig_cl_ord_id)  // OrigClOrdID
               .add_field(FIXTag::Symbol, it->second.symbol)
               .add_field(FIXTag::Side, std::string(1, it->second.side));
        
        return builder.build(FIXMsgType::OrderCancelRequest);
    }
    
    void handle_execution_report(const FIXMessage& msg) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        std::string cl_ord_id = msg.get_field(FIXTag::ClOrdID);
        std::string exec_type = msg.get_field(150);  // ExecType
        std::string ord_status = msg.get_field(39);  // OrdStatus
        
        auto it = orders_.find(cl_ord_id);
        if (it == orders_.end()) {
            return;
        }
        
        OrderState& state = it->second;
        
        // 更新狀態
        state.status = ord_status;
        
        if (msg.has_field(14)) {  // CumQty
            state.filled_quantity = msg.get_int(14);
        }
        
        // 處理執行類型
        if (exec_type == "0") {
            // New
        } else if (exec_type == "1" || exec_type == "2") {
            // Partial Fill or Fill
        } else if (exec_type == "4") {
            // Canceled
        }
    }
    
private:
    static std::string generate_cl_ord_id() {
        static std::atomic<uint64_t> counter{1};
        
        uint64_t id = counter.fetch_add(1);
        
        std::ostringstream oss;
        oss << "ORD" << std::setfill('0') << std::setw(10) << id;
        
        return oss.str();
    }
};
```

---

## 完整 FIX 引擎

### 引擎實現

```cpp
#include <thread>
#include <queue>

class FIXEngine {
    FIXSession session_;
    FIXOrderInterface order_interface_;
    
    int socket_fd_{-1};
    std::thread receiver_thread_;
    std::thread heartbeat_thread_;
    
    std::atomic<bool> running_{false};
    
    std::queue<std::string> outgoing_queue_;
    std::mutex outgoing_mutex_;
    
public:
    FIXEngine(std::string sender, std::string target)
        : session_(std::move(sender), std::move(target)),
          order_interface_(session_) {}
    
    bool connect(const std::string& host, uint16_t port) {
        // 創建 TCP 連接
        socket_fd_ = socket(AF_INET, SOCK_STREAM, 0);
        if (socket_fd_ < 0) {
            return false;
        }
        
        sockaddr_in addr;
        addr.sin_family = AF_INET;
        addr.sin_port = htons(port);
        inet_pton(AF_INET, host.c_str(), &addr.sin_addr);
        
        if (connect(socket_fd_, (sockaddr*)&addr, sizeof(addr)) < 0) {
            close(socket_fd_);
            return false;
        }
        
        return true;
    }
    
    void start() {
        running_.store(true);
        
        // 發送登入訊息
        send_message(session_.create_logon());
        
        // 啟動接收執行緒
        receiver_thread_ = std::thread(&FIXEngine::receive_loop, this);
        
        // 啟動心跳執行緒
        heartbeat_thread_ = std::thread(&FIXEngine::heartbeat_loop, this);
    }
    
    void stop() {
        // 發送登出訊息
        send_message(session_.create_logout());
        
        running_.store(false);
        
        if (receiver_thread_.joinable()) {
            receiver_thread_.join();
        }
        
        if (heartbeat_thread_.joinable()) {
            heartbeat_thread_.join();
        }
        
        if (socket_fd_ >= 0) {
            close(socket_fd_);
        }
    }
    
    void send_order(const std::string& symbol, char side, 
                   double price, uint64_t quantity) {
        std::string msg = order_interface_.send_new_order(symbol, side, price, quantity);
        send_message(msg);
    }
    
private:
    void send_message(const std::string& message) {
        std::lock_guard<std::mutex> lock(outgoing_mutex_);
        
        ssize_t sent = send(socket_fd_, message.data(), message.size(), 0);
        
        if (sent < 0) {
            // 處理錯誤
        }
    }
    
    void receive_loop() {
        char buffer[65536];
        std::string incomplete;
        
        while (running_.load()) {
            ssize_t received = recv(socket_fd_, buffer, sizeof(buffer), 0);
            
            if (received <= 0) {
                break;
            }
            
            incomplete.append(buffer, received);
            
            // 解析訊息
            while (true) {
                size_t msg_end = incomplete.find("10=");
                if (msg_end == std::string::npos) {
                    break;
                }
                
                msg_end = incomplete.find('\x01', msg_end);
                if (msg_end == std::string::npos) {
                    break;
                }
                
                std::string raw_msg = incomplete.substr(0, msg_end + 1);
                incomplete.erase(0, msg_end + 1);
                
                FIXMessage msg;
                if (msg.parse(raw_msg)) {
                    process_message(msg);
                }
            }
        }
    }
    
    void heartbeat_loop() {
        while (running_.load()) {
            std::this_thread::sleep_for(std::chrono::seconds(10));
            
            if (session_.needs_heartbeat()) {
                send_message(session_.create_heartbeat());
            }
        }
    }
    
    void process_message(const FIXMessage& msg) {
        if (!session_.process_message(msg)) {
            return;
        }
        
        std::string msg_type = msg.get_field(FIXTag::MsgType);
        
        if (msg_type == FIXMsgType::ExecutionReport) {
            order_interface_.handle_execution_report(msg);
        }
    }
};
```

---

## 參考資料

1. [FIX Protocol Specification](https://www.fixtrading.org/standards/)
2. [QuickFIX/C++](https://www.quickfixengine.org/)
3. [FIX Protocol Session Layer](https://www.fixtrading.org/standards/fix-session-layer/)
4. [FIX Message Structure](https://www.onixs.biz/fix-dictionary.html)
5. [Building a FIX Engine](https://www.cmegroup.com/confluence/display/EPICSANDBOX/FIX+Best+Practices)
