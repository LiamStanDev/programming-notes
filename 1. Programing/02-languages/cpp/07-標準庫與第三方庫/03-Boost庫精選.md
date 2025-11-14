# Boost 庫精選

Boost 提供許多高品質的 C++ 庫,特別適合高效能系統開發。

---

## 1. Boost.Asio - 異步網路

### 1.1 基礎 TCP 伺服器

```cpp
#include <boost/asio.hpp>

using boost::asio::ip::tcp;

class TcpServer {
    boost::asio::io_context io_context_;
    tcp::acceptor acceptor_;
    
public:
    TcpServer(uint16_t port)
        : acceptor_(io_context_, tcp::endpoint(tcp::v4(), port))
    {
        start_accept();
    }
    
    void run() {
        io_context_.run();
    }
    
private:
    void start_accept() {
        auto socket = std::make_shared<tcp::socket>(io_context_);
        
        acceptor_.async_accept(*socket, [this, socket](boost::system::error_code ec) {
            if (!ec) {
                handle_client(socket);
            }
            start_accept();
        });
    }
    
    void handle_client(std::shared_ptr<tcp::socket> socket) {
        auto buffer = std::make_shared<std::array<char, 1024>>();
        
        socket->async_read_some(
            boost::asio::buffer(*buffer),
            [socket, buffer](boost::system::error_code ec, size_t bytes) {
                if (!ec) {
                    // 處理數據
                }
            }
        );
    }
};
```

---

## 2. Boost.Lockfree

### 2.1 Lock-Free Queue

```cpp
#include <boost/lockfree/queue.hpp>

boost::lockfree::queue<MarketTick, boost::lockfree::capacity<4096>> queue;

// 生產者
void producer() {
    MarketTick tick = get_tick();
    queue.push(tick);  // lock-free!
}

// 消費者
void consumer() {
    MarketTick tick;
    if (queue.pop(tick)) {
        process_tick(tick);
    }
}
```

---

## 參考資料

1. **Boost 文件**
   - [Boost.Asio](https://www.boost.org/doc/libs/release/doc/html/boost_asio.html)
   - [Boost.Lockfree](https://www.boost.org/doc/libs/release/doc/html/lockfree.html)
