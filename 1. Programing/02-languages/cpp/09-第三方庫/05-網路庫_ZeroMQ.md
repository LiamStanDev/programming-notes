# 網路庫 - ZeroMQ

## 概述

ZeroMQ (ØMQ) 是高效能異步訊息傳遞庫，提供類似 Socket 的 API，但抽象了底層網路複雜性。廣泛應用於分散式系統、高頻交易、實時數據處理等場景。

### 核心特點

1. **無 Broker 架構**: 點對點通訊，無中心節點
2. **多種傳輸層**: TCP, IPC, inproc, PGM (多播)
3. **智慧模式**: 自動重連、負載均衡、訊息佇列
4. **高效能**: 低延遲 (~30μs)、高吞吐 (~5M msg/s)
5. **語言無關**: 支援 40+ 程式語言

### 安裝與整合

```bash
# Ubuntu/Debian
sudo apt-get install libzmq3-dev

# 從源碼編譯
git clone https://github.com/zeromq/libzmq.git
cd libzmq
mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install
```

CMake 整合:

```cmake
find_package(PkgConfig REQUIRED)
pkg_check_modules(ZMQ REQUIRED libzmq)

add_executable(app main.cpp)
target_link_libraries(app ${ZMQ_LIBRARIES})
target_include_directories(app PRIVATE ${ZMQ_INCLUDE_DIRS})
```

---

## 基本概念

### Socket 類型

ZeroMQ 提供多種 Socket 模式:

| Socket 類型 | 用途 | 通訊模式 |
|------------|------|---------|
| `PAIR` | 1:1 獨佔連接 | 雙向 |
| `REQ/REP` | 請求-回應 | 同步 |
| `DEALER/ROUTER` | 異步請求-回應 | 異步 |
| `PUB/SUB` | 發布-訂閱 | 單向 |
| `PUSH/PULL` | 管線 (Pipeline) | 單向 |

### 傳輸協議

```cpp
// TCP: 跨機器通訊
zmq::context_t ctx;
zmq::socket_t sock(ctx, zmq::socket_type::pub);
sock.bind("tcp://*:5555");

// IPC: 同機器跨進程
sock.bind("ipc:///tmp/feeds.ipc");

// Inproc: 同進程內執行緒間
sock.bind("inproc://internal");

// PGM: 可靠多播 (需要特殊權限)
sock.bind("pgm://eth0;239.192.1.1:5555");
```

---

## REQ/REP 模式 - 請求回應

### 基本範例

伺服器端:

```cpp
#include <zmq.hpp>
#include <string>
#include <iostream>
#include <thread>
#include <chrono>

int main() {
    zmq::context_t context(1);
    zmq::socket_t socket(context, zmq::socket_type::rep);
    
    socket.bind("tcp://*:5555");
    std::cout << "Server listening on port 5555\n";
    
    while (true) {
        zmq::message_t request;
        
        // 接收請求
        socket.recv(request, zmq::recv_flags::none);
        
        std::string req_str(static_cast<char*>(request.data()), request.size());
        std::cout << "Received: " << req_str << "\n";
        
        // 模擬處理
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        
        // 發送回應
        std::string reply = "World";
        zmq::message_t response(reply.size());
        memcpy(response.data(), reply.data(), reply.size());
        
        socket.send(response, zmq::send_flags::none);
    }
    
    return 0;
}
```

客戶端:

```cpp
#include <zmq.hpp>
#include <string>
#include <iostream>

int main() {
    zmq::context_t context(1);
    zmq::socket_t socket(context, zmq::socket_type::req);
    
    socket.connect("tcp://localhost:5555");
    
    for (int i = 0; i < 10; ++i) {
        // 發送請求
        std::string request = "Hello";
        zmq::message_t req_msg(request.size());
        memcpy(req_msg.data(), request.data(), request.size());
        
        socket.send(req_msg, zmq::send_flags::none);
        
        // 接收回應
        zmq::message_t reply;
        socket.recv(reply, zmq::recv_flags::none);
        
        std::string reply_str(static_cast<char*>(reply.data()), reply.size());
        std::cout << "Reply " << i << ": " << reply_str << "\n";
    }
    
    return 0;
}
```

### 交易訂單系統

```cpp
#include <zmq.hpp>
#include <nlohmann/json.hpp>
#include <iostream>

using json = nlohmann::json;

// 訂單執行引擎
class OrderExecutionEngine {
    zmq::context_t context_;
    zmq::socket_t socket_;
    
public:
    OrderExecutionEngine() 
        : context_(1), 
          socket_(context_, zmq::socket_type::rep) {
        socket_.bind("tcp://*:5560");
    }
    
    void run() {
        while (true) {
            zmq::message_t request;
            socket_.recv(request, zmq::recv_flags::none);
            
            // 解析訂單請求
            std::string req_str(static_cast<char*>(request.data()), 
                               request.size());
            
            json order = json::parse(req_str);
            
            // 執行訂單
            json response = execute_order(order);
            
            // 發送回應
            std::string resp_str = response.dump();
            zmq::message_t reply(resp_str.size());
            memcpy(reply.data(), resp_str.data(), resp_str.size());
            
            socket_.send(reply, zmq::send_flags::none);
        }
    }
    
private:
    json execute_order(const json& order) {
        // 模擬訂單執行
        return {
            {"order_id", order["order_id"]},
            {"status", "FILLED"},
            {"filled_price", order["price"]},
            {"filled_qty", order["quantity"]}
        };
    }
};

// 交易客戶端
class TradingClient {
    zmq::context_t context_;
    zmq::socket_t socket_;
    
public:
    TradingClient()
        : context_(1),
          socket_(context_, zmq::socket_type::req) {
        socket_.connect("tcp://localhost:5560");
    }
    
    json send_order(const std::string& symbol, 
                    double price, 
                    uint64_t quantity) {
        json order = {
            {"order_id", generate_order_id()},
            {"symbol", symbol},
            {"price", price},
            {"quantity", quantity}
        };
        
        // 發送
        std::string order_str = order.dump();
        zmq::message_t request(order_str.size());
        memcpy(request.data(), order_str.data(), order_str.size());
        socket_.send(request, zmq::send_flags::none);
        
        // 接收
        zmq::message_t reply;
        socket_.recv(reply, zmq::recv_flags::none);
        
        std::string reply_str(static_cast<char*>(reply.data()), 
                             reply.size());
        return json::parse(reply_str);
    }
    
private:
    static uint64_t generate_order_id() {
        static uint64_t id = 1;
        return id++;
    }
};
```

---

## PUB/SUB 模式 - 發布訂閱

### 市場數據分發

發布者 (Market Data Feed):

```cpp
#include <zmq.hpp>
#include <string>
#include <sstream>
#include <thread>
#include <chrono>
#include <random>

class MarketDataPublisher {
    zmq::context_t context_;
    zmq::socket_t socket_;
    
public:
    MarketDataPublisher() 
        : context_(1),
          socket_(context_, zmq::socket_type::pub) {
        socket_.bind("tcp://*:5556");
        
        // 等待訂閱者連接
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
    
    void publish_tick(const std::string& symbol, 
                     double price, 
                     uint64_t volume) {
        // 主題: symbol
        std::ostringstream oss;
        oss << symbol << " " << price << " " << volume;
        
        std::string message = oss.str();
        
        zmq::message_t msg(message.size());
        memcpy(msg.data(), message.data(), message.size());
        
        socket_.send(msg, zmq::send_flags::none);
    }
    
    void run() {
        std::random_device rd;
        std::mt19937 gen(rd());
        std::uniform_real_distribution<> price_dist(100.0, 200.0);
        std::uniform_int_distribution<> vol_dist(100, 10000);
        
        std::vector<std::string> symbols = {"AAPL", "GOOGL", "MSFT", "TSLA"};
        
        while (true) {
            for (const auto& symbol : symbols) {
                double price = price_dist(gen);
                uint64_t volume = vol_dist(gen);
                
                publish_tick(symbol, price, volume);
            }
            
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    }
};

int main() {
    MarketDataPublisher publisher;
    publisher.run();
    return 0;
}
```

訂閱者 (Trading Strategy):

```cpp
#include <zmq.hpp>
#include <string>
#include <iostream>
#include <sstream>

class MarketDataSubscriber {
    zmq::context_t context_;
    zmq::socket_t socket_;
    
public:
    MarketDataSubscriber(const std::vector<std::string>& symbols)
        : context_(1),
          socket_(context_, zmq::socket_type::sub) {
        socket_.connect("tcp://localhost:5556");
        
        // 訂閱特定主題
        for (const auto& symbol : symbols) {
            socket_.set(zmq::sockopt::subscribe, symbol);
        }
    }
    
    void run() {
        while (true) {
            zmq::message_t message;
            socket_.recv(message, zmq::recv_flags::none);
            
            std::string msg_str(static_cast<char*>(message.data()), 
                               message.size());
            
            process_tick(msg_str);
        }
    }
    
private:
    void process_tick(const std::string& tick_data) {
        std::istringstream iss(tick_data);
        
        std::string symbol;
        double price;
        uint64_t volume;
        
        iss >> symbol >> price >> volume;
        
        std::cout << "Received: " << symbol 
                  << " @ " << price 
                  << " x " << volume << "\n";
        
        // 策略邏輯...
    }
};

int main() {
    // 只訂閱 AAPL 和 GOOGL
    MarketDataSubscriber subscriber({"AAPL", "GOOGL"});
    subscriber.run();
    return 0;
}
```

### 多級主題過濾

```cpp
#include <zmq.hpp>
#include <string>

class MultiTopicPublisher {
    zmq::context_t context_;
    zmq::socket_t socket_;
    
public:
    MultiTopicPublisher()
        : context_(1),
          socket_(context_, zmq::socket_type::pub) {
        socket_.bind("tcp://*:5557");
    }
    
    void publish(const std::string& exchange,
                const std::string& symbol,
                const std::string& data) {
        // 層級主題: "EXCHANGE.SYMBOL data"
        std::string message = exchange + "." + symbol + " " + data;
        
        zmq::message_t msg(message.size());
        memcpy(msg.data(), message.data(), message.size());
        
        socket_.send(msg, zmq::send_flags::none);
    }
};

class MultiTopicSubscriber {
    zmq::context_t context_;
    zmq::socket_t socket_;
    
public:
    MultiTopicSubscriber()
        : context_(1),
          socket_(context_, zmq::socket_type::sub) {
        socket_.connect("tcp://localhost:5557");
    }
    
    // 訂閱特定交易所的所有股票
    void subscribe_exchange(const std::string& exchange) {
        socket_.set(zmq::sockopt::subscribe, exchange);
    }
    
    // 訂閱特定交易所的特定股票
    void subscribe_symbol(const std::string& exchange, 
                         const std::string& symbol) {
        std::string topic = exchange + "." + symbol;
        socket_.set(zmq::sockopt::subscribe, topic);
    }
    
    void run() {
        while (true) {
            zmq::message_t message;
            socket_.recv(message, zmq::recv_flags::none);
            
            std::string msg_str(static_cast<char*>(message.data()), 
                               message.size());
            
            std::cout << "Received: " << msg_str << "\n";
        }
    }
};
```

---

## PUSH/PULL 模式 - 管線處理

### 分散式任務處理

任務生產者 (Ventilator):

```cpp
#include <zmq.hpp>
#include <string>
#include <random>
#include <iostream>

class TaskVentilator {
    zmq::context_t context_;
    zmq::socket_t sender_;
    zmq::socket_t sink_;
    
public:
    TaskVentilator()
        : context_(1),
          sender_(context_, zmq::socket_type::push),
          sink_(context_, zmq::socket_type::push) {
        
        sender_.bind("tcp://*:5557");  // 發送任務
        sink_.connect("tcp://localhost:5558");  // 通知 sink
    }
    
    void distribute_tasks(int num_tasks) {
        std::cout << "Distributing " << num_tasks << " tasks\n";
        
        // 通知 sink 開始計時
        zmq::message_t start_msg(1);
        sink_.send(start_msg, zmq::send_flags::none);
        
        std::random_device rd;
        std::mt19937 gen(rd());
        std::uniform_int_distribution<> dist(1, 100);
        
        for (int i = 0; i < num_tasks; ++i) {
            int workload = dist(gen);
            
            std::string task = std::to_string(workload);
            zmq::message_t message(task.size());
            memcpy(message.data(), task.data(), task.size());
            
            sender_.send(message, zmq::send_flags::none);
        }
    }
};

int main() {
    TaskVentilator ventilator;
    ventilator.distribute_tasks(100);
    return 0;
}
```

任務處理者 (Worker):

```cpp
#include <zmq.hpp>
#include <string>
#include <thread>
#include <chrono>
#include <iostream>

class TaskWorker {
    zmq::context_t context_;
    zmq::socket_t receiver_;
    zmq::socket_t sender_;
    
public:
    TaskWorker()
        : context_(1),
          receiver_(context_, zmq::socket_type::pull),
          sender_(context_, zmq::socket_type::push) {
        
        receiver_.connect("tcp://localhost:5557");
        sender_.connect("tcp://localhost:5558");
    }
    
    void run() {
        while (true) {
            zmq::message_t message;
            receiver_.recv(message, zmq::recv_flags::none);
            
            std::string task_str(static_cast<char*>(message.data()), 
                                message.size());
            int workload = std::stoi(task_str);
            
            // 模擬工作
            std::this_thread::sleep_for(std::chrono::milliseconds(workload));
            
            // 發送完成通知
            zmq::message_t done_msg(1);
            sender_.send(done_msg, zmq::send_flags::none);
            
            std::cout << "Completed task: " << workload << " ms\n";
        }
    }
};

int main() {
    TaskWorker worker;
    worker.run();
    return 0;
}
```

結果收集器 (Sink):

```cpp
#include <zmq.hpp>
#include <chrono>
#include <iostream>

class TaskSink {
    zmq::context_t context_;
    zmq::socket_t receiver_;
    
public:
    TaskSink()
        : context_(1),
          receiver_(context_, zmq::socket_type::pull) {
        receiver_.bind("tcp://*:5558");
    }
    
    void collect_results(int num_tasks) {
        // 等待開始信號
        zmq::message_t start_msg;
        receiver_.recv(start_msg, zmq::recv_flags::none);
        
        auto start_time = std::chrono::steady_clock::now();
        
        for (int i = 0; i < num_tasks; ++i) {
            zmq::message_t message;
            receiver_.recv(message, zmq::recv_flags::none);
            
            if ((i + 1) % 10 == 0) {
                std::cout << "Completed: " << (i + 1) << " tasks\n";
            }
        }
        
        auto end_time = std::chrono::steady_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(
            end_time - start_time);
        
        std::cout << "Total elapsed time: " << duration.count() << " ms\n";
    }
};

int main() {
    TaskSink sink;
    sink.collect_results(100);
    return 0;
}
```

---

## DEALER/ROUTER 模式 - 異步路由

### 負載均衡伺服器

```cpp
#include <zmq.hpp>
#include <string>
#include <vector>
#include <iostream>

class LoadBalancer {
    zmq::context_t context_;
    zmq::socket_t frontend_;  // 客戶端連接
    zmq::socket_t backend_;   // Worker 連接
    
public:
    LoadBalancer()
        : context_(1),
          frontend_(context_, zmq::socket_type::router),
          backend_(context_, zmq::socket_type::router) {
        
        frontend_.bind("tcp://*:5559");  // 客戶端
        backend_.bind("tcp://*:5560");   // Workers
    }
    
    void run() {
        std::vector<std::string> available_workers;
        
        zmq::pollitem_t items[] = {
            {backend_, 0, ZMQ_POLLIN, 0},
            {frontend_, 0, ZMQ_POLLIN, 0}
        };
        
        while (true) {
            zmq::poll(items, available_workers.empty() ? 1 : 2, 
                     std::chrono::milliseconds(-1));
            
            // 處理 worker 回應
            if (items[0].revents & ZMQ_POLLIN) {
                std::vector<zmq::message_t> frames;
                
                // 接收所有 frame
                while (true) {
                    zmq::message_t frame;
                    backend_.recv(frame, zmq::recv_flags::none);
                    
                    bool more = frame.more();
                    frames.push_back(std::move(frame));
                    
                    if (!more) break;
                }
                
                // Worker ID
                std::string worker_id(
                    static_cast<char*>(frames[0].data()), 
                    frames[0].size()
                );
                available_workers.push_back(worker_id);
                
                // 如果不是 READY 訊息，轉發給客戶端
                if (frames.size() > 2) {
                    for (size_t i = 2; i < frames.size(); ++i) {
                        frontend_.send(frames[i], 
                            i < frames.size() - 1 ? zmq::send_flags::sndmore : zmq::send_flags::none);
                    }
                }
            }
            
            // 處理客戶端請求
            if (items[1].revents & ZMQ_POLLIN && !available_workers.empty()) {
                std::vector<zmq::message_t> frames;
                
                while (true) {
                    zmq::message_t frame;
                    frontend_.recv(frame, zmq::recv_flags::none);
                    
                    bool more = frame.more();
                    frames.push_back(std::move(frame));
                    
                    if (!more) break;
                }
                
                // 分配給可用 worker
                std::string worker_id = available_workers.back();
                available_workers.pop_back();
                
                // 發送: Worker ID | empty | Client ID | empty | Data
                zmq::message_t worker_frame(worker_id.size());
                memcpy(worker_frame.data(), worker_id.data(), worker_id.size());
                backend_.send(worker_frame, zmq::send_flags::sndmore);
                
                zmq::message_t empty;
                backend_.send(empty, zmq::send_flags::sndmore);
                
                for (size_t i = 0; i < frames.size(); ++i) {
                    backend_.send(frames[i], 
                        i < frames.size() - 1 ? zmq::send_flags::sndmore : zmq::send_flags::none);
                }
            }
        }
    }
};
```

### 異步 Worker

```cpp
#include <zmq.hpp>
#include <string>
#include <thread>
#include <chrono>

class AsyncWorker {
    zmq::context_t context_;
    zmq::socket_t socket_;
    std::string id_;
    
public:
    AsyncWorker(const std::string& id)
        : context_(1),
          socket_(context_, zmq::socket_type::dealer),
          id_(id) {
        
        socket_.set(zmq::sockopt::routing_id, id);
        socket_.connect("tcp://localhost:5560");
        
        // 發送 READY 訊息
        send_ready();
    }
    
    void run() {
        while (true) {
            std::vector<zmq::message_t> frames;
            
            // 接收: empty | Client ID | empty | Data
            while (true) {
                zmq::message_t frame;
                socket_.recv(frame, zmq::recv_flags::none);
                
                bool more = frame.more();
                frames.push_back(std::move(frame));
                
                if (!more) break;
            }
            
            // 處理請求
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            
            // 發送回應
            for (size_t i = 0; i < frames.size(); ++i) {
                socket_.send(frames[i],
                    i < frames.size() - 1 ? zmq::send_flags::sndmore : zmq::send_flags::none);
            }
        }
    }
    
private:
    void send_ready() {
        zmq::message_t empty;
        socket_.send(empty, zmq::send_flags::sndmore);
        
        std::string ready = "READY";
        zmq::message_t ready_msg(ready.size());
        memcpy(ready_msg.data(), ready.data(), ready.size());
        socket_.send(ready_msg, zmq::send_flags::none);
    }
};
```

---

## 高級特性

### 零拷貝 (Zero-Copy)

```cpp
#include <zmq.hpp>
#include <cstring>

void zero_copy_send(zmq::socket_t& socket, const char* data, size_t size) {
    // 使用自定義釋放函數
    auto free_fn = [](void* data, void* hint) {
        delete[] static_cast<char*>(data);
    };
    
    // 數據所有權轉移給 ZeroMQ
    zmq::message_t message(const_cast<char*>(data), size, free_fn, nullptr);
    socket.send(message, zmq::send_flags::none);
}

// 大型數據傳輸
void send_large_data(zmq::socket_t& socket) {
    const size_t size = 10 * 1024 * 1024;  // 10 MB
    char* data = new char[size];
    
    // 填充數據...
    memset(data, 'A', size);
    
    // 零拷貝發送
    zero_copy_send(socket, data, size);
    // data 已轉移所有權，不要再使用
}
```

### 多部分訊息 (Multipart Messages)

```cpp
#include <zmq.hpp>
#include <string>

void send_multipart(zmq::socket_t& socket) {
    // 發送多部分訊息
    
    // 第一部分: Header
    std::string header = "HEADER";
    zmq::message_t header_msg(header.size());
    memcpy(header_msg.data(), header.data(), header.size());
    socket.send(header_msg, zmq::send_flags::sndmore);
    
    // 第二部分: Body
    std::string body = "BODY";
    zmq::message_t body_msg(body.size());
    memcpy(body_msg.data(), body.data(), body.size());
    socket.send(body_msg, zmq::send_flags::sndmore);
    
    // 最後一部分: Footer
    std::string footer = "FOOTER";
    zmq::message_t footer_msg(footer.size());
    memcpy(footer_msg.data(), footer.data(), footer.size());
    socket.send(footer_msg, zmq::send_flags::none);  // 最後一個不設置 sndmore
}

void recv_multipart(zmq::socket_t& socket) {
    std::vector<std::string> parts;
    
    while (true) {
        zmq::message_t message;
        socket.recv(message, zmq::recv_flags::none);
        
        std::string part(static_cast<char*>(message.data()), message.size());
        parts.push_back(part);
        
        if (!message.more()) break;  // 沒有更多部分
    }
    
    std::cout << "Received " << parts.size() << " parts\n";
}
```

### 心跳與超時檢測

```cpp
#include <zmq.hpp>
#include <chrono>

class HeartbeatClient {
    zmq::context_t context_;
    zmq::socket_t socket_;
    
    static constexpr int HEARTBEAT_INTERVAL_MS = 1000;
    static constexpr int HEARTBEAT_TIMEOUT_MS = 3000;
    
public:
    HeartbeatClient()
        : context_(1),
          socket_(context_, zmq::socket_type::dealer) {
        socket_.connect("tcp://localhost:5561");
    }
    
    void run() {
        auto last_heartbeat = std::chrono::steady_clock::now();
        
        zmq::pollitem_t items[] = {
            {socket_, 0, ZMQ_POLLIN, 0}
        };
        
        while (true) {
            zmq::poll(items, 1, std::chrono::milliseconds(HEARTBEAT_INTERVAL_MS));
            
            auto now = std::chrono::steady_clock::now();
            
            if (items[0].revents & ZMQ_POLLIN) {
                zmq::message_t message;
                socket_.recv(message, zmq::recv_flags::none);
                
                last_heartbeat = now;
                
                // 處理訊息...
            }
            
            // 檢查超時
            auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(
                now - last_heartbeat);
            
            if (elapsed.count() > HEARTBEAT_TIMEOUT_MS) {
                std::cerr << "Server timeout, reconnecting...\n";
                reconnect();
                last_heartbeat = now;
            }
            
            // 發送心跳
            send_heartbeat();
        }
    }
    
private:
    void send_heartbeat() {
        std::string hb = "HEARTBEAT";
        zmq::message_t message(hb.size());
        memcpy(message.data(), hb.data(), hb.size());
        socket_.send(message, zmq::send_flags::dontwait);
    }
    
    void reconnect() {
        socket_.close();
        socket_ = zmq::socket_t(context_, zmq::socket_type::dealer);
        socket_.connect("tcp://localhost:5561");
    }
};
```

---

## 效能優化

### 批次處理

```cpp
#include <zmq.hpp>
#include <vector>

class BatchPublisher {
    zmq::context_t context_;
    zmq::socket_t socket_;
    
    std::vector<std::string> buffer_;
    static constexpr size_t BATCH_SIZE = 100;
    
public:
    BatchPublisher()
        : context_(1),
          socket_(context_, zmq::socket_type::pub) {
        socket_.bind("tcp://*:5562");
        buffer_.reserve(BATCH_SIZE);
    }
    
    void publish(const std::string& message) {
        buffer_.push_back(message);
        
        if (buffer_.size() >= BATCH_SIZE) {
            flush();
        }
    }
    
    void flush() {
        if (buffer_.empty()) return;
        
        // 發送批次
        for (size_t i = 0; i < buffer_.size(); ++i) {
            zmq::message_t msg(buffer_[i].size());
            memcpy(msg.data(), buffer_[i].data(), buffer_[i].size());
            
            socket_.send(msg, zmq::send_flags::dontwait);
        }
        
        buffer_.clear();
    }
};
```

### 高水位標記 (HWM)

```cpp
#include <zmq.hpp>

void configure_hwm() {
    zmq::context_t context(1);
    zmq::socket_t socket(context, zmq::socket_type::pub);
    
    // 發送高水位標記: 1000 條訊息
    socket.set(zmq::sockopt::sndhwm, 1000);
    
    // 接收高水位標記
    socket.set(zmq::sockopt::rcvhwm, 1000);
    
    // 當達到 HWM 時的行為
    // PUB: 丟棄新訊息
    // PUSH: 阻塞
    // REQ/REP: 阻塞
    
    socket.bind("tcp://*:5563");
}
```

### IO 執行緒配置

```cpp
#include <zmq.hpp>

int main() {
    // 創建 4 個 IO 執行緒的 context
    zmq::context_t context(4);
    
    // 設置最大 sockets 數量
    context.set(zmq::ctxopt::max_sockets, 1024);
    
    // 設置 IO 執行緒親和性
    context.set(zmq::ctxopt::io_threads, 4);
    
    zmq::socket_t socket(context, zmq::socket_type::pub);
    socket.bind("tcp://*:5564");
    
    // 使用 context...
    
    return 0;
}
```

---

## 延遲測試

### 點對點延遲

```cpp
#include <zmq.hpp>
#include <chrono>
#include <vector>
#include <algorithm>
#include <iostream>

void latency_test_server() {
    zmq::context_t context(1);
    zmq::socket_t socket(context, zmq::socket_type::rep);
    socket.bind("tcp://*:5565");
    
    while (true) {
        zmq::message_t request;
        socket.recv(request, zmq::recv_flags::none);
        
        // 立即回覆
        socket.send(request, zmq::send_flags::none);
    }
}

void latency_test_client() {
    zmq::context_t context(1);
    zmq::socket_t socket(context, zmq::socket_type::req);
    socket.connect("tcp://localhost:5565");
    
    std::vector<uint64_t> latencies;
    latencies.reserve(10000);
    
    zmq::message_t message(8);
    
    for (int i = 0; i < 10000; ++i) {
        auto start = std::chrono::steady_clock::now();
        
        socket.send(message, zmq::send_flags::none);
        socket.recv(message, zmq::recv_flags::none);
        
        auto end = std::chrono::steady_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(
            end - start);
        
        latencies.push_back(duration.count());
    }
    
    // 統計
    std::sort(latencies.begin(), latencies.end());
    
    std::cout << "Latency (round-trip):\n";
    std::cout << "  Min:  " << latencies.front() / 1000.0 << " us\n";
    std::cout << "  P50:  " << latencies[latencies.size() * 50 / 100] / 1000.0 << " us\n";
    std::cout << "  P99:  " << latencies[latencies.size() * 99 / 100] / 1000.0 << " us\n";
    std::cout << "  Max:  " << latencies.back() / 1000.0 << " us\n";
}

// 典型結果 (localhost):
// Min:  25 us
// P50:  30 us
// P99:  65 us
// Max:  150 us
```

### 吞吐量測試

```cpp
#include <zmq.hpp>
#include <chrono>
#include <iostream>

void throughput_test_sender() {
    zmq::context_t context(1);
    zmq::socket_t socket(context, zmq::socket_type::push);
    socket.bind("tcp://*:5566");
    
    const size_t message_count = 1000000;
    const size_t message_size = 64;
    
    zmq::message_t message(message_size);
    
    auto start = std::chrono::steady_clock::now();
    
    for (size_t i = 0; i < message_count; ++i) {
        socket.send(message, zmq::send_flags::none);
    }
    
    auto end = std::chrono::steady_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start);
    
    double throughput = (message_count * 1000.0) / duration.count();
    
    std::cout << "Sent " << message_count << " messages in " 
              << duration.count() << " ms\n";
    std::cout << "Throughput: " << throughput << " msg/s\n";
}

void throughput_test_receiver() {
    zmq::context_t context(1);
    zmq::socket_t socket(context, zmq::socket_type::pull);
    socket.connect("tcp://localhost:5566");
    
    const size_t message_count = 1000000;
    
    zmq::message_t message;
    
    // 第一條訊息開始計時
    socket.recv(message, zmq::recv_flags::none);
    auto start = std::chrono::steady_clock::now();
    
    for (size_t i = 1; i < message_count; ++i) {
        socket.recv(message, zmq::recv_flags::none);
    }
    
    auto end = std::chrono::steady_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start);
    
    double throughput = (message_count * 1000.0) / duration.count();
    
    std::cout << "Received " << message_count << " messages in " 
              << duration.count() << " ms\n";
    std::cout << "Throughput: " << throughput << " msg/s\n";
}

// 典型結果 (localhost, 64 byte messages):
// Throughput: ~5,000,000 msg/s
```

---

## 參考資料

1. [ZeroMQ Official Guide](https://zguide.zeromq.org/)
2. [ZeroMQ API Reference](http://api.zeromq.org/)
3. [cppzmq - C++ Binding](https://github.com/zeromq/cppzmq)
4. [ZeroMQ - The Guide (Book)](http://zguide.zeromq.org/page:all)
5. [ØMQ Patterns](https://zguide.zeromq.org/docs/chapter2/)
