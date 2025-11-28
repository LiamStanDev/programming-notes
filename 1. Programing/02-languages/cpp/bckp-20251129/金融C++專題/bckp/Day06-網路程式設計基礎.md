# Day 6: 網路程式設計基礎

## 學習目標

本課程深入探討 C++ 網路編程的核心技術,涵蓋 Socket API、高性能 I/O 模型、零拷貝技術與 TCP 調優。這些技術是構建低延遲網路應用(尤其是高頻交易系統)的基礎。

**核心主題:**
- Socket 編程基礎 (TCP/UDP)
- 高性能 I/O 模型 (select/poll/epoll)
- 零拷貝技術 (sendfile/splice/mmap)
- TCP 調優與延遲優化
- UDP 多播 (Multicast)
- 網路編程設計模式 (Reactor)

---

## 6.1 Socket 程式設計基礎

### 6.1.1 TCP 客戶端

```cpp
#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>

int main() {
    // 1. 創建 socket
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock == -1) {
        perror("socket creation failed");
        return 1;
    }
    
    // 2. 設置服務器地址
    sockaddr_in server_addr{};
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(8080);
    inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);
    
    // 3. 連接服務器
    if (connect(sock, (sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
        perror("connect failed");
        close(sock);
        return 1;
    }
    
    // 4. 發送數據
    const char* message = "Hello, Server!";
    send(sock, message, strlen(message), 0);
    
    // 5. 接收響應
    char buffer[1024] = {0};
    ssize_t n = recv(sock, buffer, sizeof(buffer) - 1, 0);
    if (n > 0) {
        std::cout << "Server response: " << buffer << "\n";
    }
    
    // 6. 關閉連接
    close(sock);
    return 0;
}
```

### 6.1.2 TCP 服務器

```cpp
#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>

int main() {
    // 1. 創建 socket
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd == -1) {
        perror("socket creation failed");
        return 1;
    }
    
    // 2. 設置 socket 選項 (允許地址重用)
    int opt = 1;
    if (setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) == -1) {
        perror("setsockopt failed");
        close(server_fd);
        return 1;
    }
    
    // 3. 綁定地址
    sockaddr_in address{};
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(8080);
    
    if (bind(server_fd, (sockaddr*)&address, sizeof(address)) == -1) {
        perror("bind failed");
        close(server_fd);
        return 1;
    }
    
    // 4. 監聽連接
    if (listen(server_fd, 10) == -1) {
        perror("listen failed");
        close(server_fd);
        return 1;
    }
    
    std::cout << "Server listening on port 8080...\n";
    
    // 5. 接受客戶端連接
    sockaddr_in client_addr{};
    socklen_t client_len = sizeof(client_addr);
    int client_fd = accept(server_fd, (sockaddr*)&client_addr, &client_len);
    if (client_fd == -1) {
        perror("accept failed");
        close(server_fd);
        return 1;
    }
    
    std::cout << "Client connected\n";
    
    // 6. 接收並回顯數據
    char buffer[1024] = {0};
    ssize_t n = recv(client_fd, buffer, sizeof(buffer) - 1, 0);
    if (n > 0) {
        std::cout << "Received: " << buffer << "\n";
        send(client_fd, buffer, n, 0);  // 回顯
    }
    
    // 7. 關閉連接
    close(client_fd);
    close(server_fd);
    return 0;
}
```

### 6.1.3 TCP 連接狀態

```
客戶端                    服務器
  |                          |
  |  --- SYN (seq=x) --->    |  (1) 客戶端發起連接
  |                          |
  |  <-- SYN-ACK -------     |  (2) 服務器響應
  |     (seq=y, ack=x+1)     |
  |                          |
  |  --- ACK (ack=y+1) --->  |  (3) 客戶端確認
  |                          |
  |  ===== 連接建立 =====    |
  |                          |
  |  <----- 數據傳輸 ----->  |
  |                          |
  |  --- FIN (seq=m) --->    |  (4) 客戶端關閉
  |                          |
  |  <-- ACK (ack=m+1) ---   |  (5) 服務器確認
  |                          |
  |  <-- FIN (seq=n) ----    |  (6) 服務器關閉
  |                          |
  |  --- ACK (ack=n+1) --->  |  (7) 客戶端確認
  |                          |
  | (TIME_WAIT 2MSL)         |
  |                          |
  |  ===== 連接關閉 =====    |
```

**TIME_WAIT 狀態:**
- 持續時間: 2 * MSL (Maximum Segment Lifetime, 通常 60s)
- 目的: 確保遠端收到最後的 ACK,防止舊連接干擾新連接
- HFT 影響: 可能限制高頻重連,需調整 `SO_REUSEADDR`

---

## 6.2 非阻塞 I/O 與多路復用

### 6.2.1 阻塞 vs 非阻塞 I/O

```cpp
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <iostream>

// 設置非阻塞模式
int set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1) return -1;
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

// 非阻塞讀取示例
void nonblocking_read_example(int fd) {
    char buffer[1024];
    
    while (true) {
        ssize_t n = read(fd, buffer, sizeof(buffer));
        
        if (n > 0) {
            // 成功讀取數據
            std::cout << "Read " << n << " bytes\n";
        } else if (n == 0) {
            // 連接關閉
            std::cout << "Connection closed\n";
            break;
        } else {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 暫無數據,稍後重試
                std::cout << "No data available, retry later\n";
                break;
            } else {
                // 真實錯誤
                perror("read error");
                break;
            }
        }
    }
}
```

### 6.2.2 select()

```cpp
#include <sys/select.h>
#include <iostream>
#include <vector>

void select_example(const std::vector<int>& fds) {
    fd_set readfds;
    
    while (true) {
        FD_ZERO(&readfds);
        int max_fd = -1;
        
        // 添加所有 fd
        for (int fd : fds) {
            FD_SET(fd, &readfds);
            if (fd > max_fd) max_fd = fd;
        }
        
        // 等待事件 (超時 1 秒)
        timeval timeout{1, 0};
        int ret = select(max_fd + 1, &readfds, nullptr, nullptr, &timeout);
        
        if (ret == -1) {
            perror("select");
            break;
        } else if (ret == 0) {
            std::cout << "Timeout\n";
            continue;
        }
        
        // 檢查哪些 fd 就緒
        for (int fd : fds) {
            if (FD_ISSET(fd, &readfds)) {
                std::cout << "FD " << fd << " is ready\n";
                // 處理該 fd
            }
        }
    }
}
```

**select 限制:**
- 最多監控 1024 個文件描述符 (FD_SETSIZE)
- 每次調用需重新設置 fd_set
- 線性掃描所有 fd 檢查就緒狀態,O(n) 複雜度

### 6.2.3 poll()

```cpp
#include <poll.h>
#include <iostream>
#include <vector>

void poll_example(const std::vector<int>& fds) {
    std::vector<pollfd> pollfds;
    
    for (int fd : fds) {
        pollfd pfd{};
        pfd.fd = fd;
        pfd.events = POLLIN;  // 監控可讀事件
        pollfds.push_back(pfd);
    }
    
    while (true) {
        int ret = poll(pollfds.data(), pollfds.size(), 1000);  // 超時 1 秒
        
        if (ret == -1) {
            perror("poll");
            break;
        } else if (ret == 0) {
            std::cout << "Timeout\n";
            continue;
        }
        
        // 檢查事件
        for (size_t i = 0; i < pollfds.size(); ++i) {
            if (pollfds[i].revents & POLLIN) {
                std::cout << "FD " << pollfds[i].fd << " has data\n";
                // 處理該 fd
            }
            if (pollfds[i].revents & (POLLERR | POLLHUP)) {
                std::cout << "FD " << pollfds[i].fd << " error/hangup\n";
            }
        }
    }
}
```

**poll 改進:**
- 無 1024 fd 限制
- 不需每次重建 fd 集合
- 但仍需 O(n) 掃描

### 6.2.4 epoll - Linux 高性能 I/O

```cpp
#include <sys/epoll.h>
#include <unistd.h>
#include <iostream>
#include <vector>
#include <cstring>

class EpollServer {
private:
    int epoll_fd_;
    int listen_fd_;
    
    static constexpr int MAX_EVENTS = 64;
    
    int set_nonblocking(int fd) {
        int flags = fcntl(fd, F_GETFL, 0);
        if (flags == -1) return -1;
        return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
    }
    
    void add_fd(int fd, uint32_t events) {
        epoll_event ev{};
        ev.events = events;
        ev.data.fd = fd;
        
        if (epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, fd, &ev) == -1) {
            perror("epoll_ctl: add");
        }
    }
    
    void handle_accept() {
        sockaddr_in client_addr{};
        socklen_t len = sizeof(client_addr);
        
        int client_fd = accept(listen_fd_, (sockaddr*)&client_addr, &len);
        if (client_fd == -1) {
            perror("accept");
            return;
        }
        
        std::cout << "New connection: " << client_fd << "\n";
        
        set_nonblocking(client_fd);
        add_fd(client_fd, EPOLLIN | EPOLLET);  // 邊緣觸發
    }
    
    void handle_read(int fd) {
        char buffer[1024];
        
        while (true) {
            ssize_t n = read(fd, buffer, sizeof(buffer));
            
            if (n > 0) {
                std::cout << "Read " << n << " bytes from FD " << fd << "\n";
                // 回顯數據
                write(fd, buffer, n);
            } else if (n == 0) {
                std::cout << "Connection closed: " << fd << "\n";
                epoll_ctl(epoll_fd_, EPOLL_CTL_DEL, fd, nullptr);
                close(fd);
                break;
            } else {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    // ET 模式下已讀完所有數據
                    break;
                } else {
                    perror("read");
                    epoll_ctl(epoll_fd_, EPOLL_CTL_DEL, fd, nullptr);
                    close(fd);
                    break;
                }
            }
        }
    }
    
public:
    EpollServer(int port) {
        // 創建 listen socket
        listen_fd_ = socket(AF_INET, SOCK_STREAM, 0);
        if (listen_fd_ == -1) {
            throw std::runtime_error("socket creation failed");
        }
        
        int opt = 1;
        setsockopt(listen_fd_, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
        
        sockaddr_in addr{};
        addr.sin_family = AF_INET;
        addr.sin_addr.s_addr = INADDR_ANY;
        addr.sin_port = htons(port);
        
        if (bind(listen_fd_, (sockaddr*)&addr, sizeof(addr)) == -1) {
            throw std::runtime_error("bind failed");
        }
        
        if (listen(listen_fd_, 128) == -1) {
            throw std::runtime_error("listen failed");
        }
        
        set_nonblocking(listen_fd_);
        
        // 創建 epoll 實例
        epoll_fd_ = epoll_create1(0);
        if (epoll_fd_ == -1) {
            throw std::runtime_error("epoll_create1 failed");
        }
        
        // 添加 listen_fd 到 epoll
        add_fd(listen_fd_, EPOLLIN | EPOLLET);
        
        std::cout << "Server started on port " << port << "\n";
    }
    
    ~EpollServer() {
        close(epoll_fd_);
        close(listen_fd_);
    }
    
    void run() {
        epoll_event events[MAX_EVENTS];
        
        while (true) {
            int nfds = epoll_wait(epoll_fd_, events, MAX_EVENTS, -1);
            
            if (nfds == -1) {
                perror("epoll_wait");
                break;
            }
            
            for (int i = 0; i < nfds; ++i) {
                if (events[i].data.fd == listen_fd_) {
                    // 新連接
                    handle_accept();
                } else {
                    // 數據就緒
                    handle_read(events[i].data.fd);
                }
            }
        }
    }
};

int main() {
    try {
        EpollServer server(8080);
        server.run();
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << "\n";
        return 1;
    }
    return 0;
}
```

### 6.2.5 epoll 邊緣觸發 (ET) vs 水平觸發 (LT)

```cpp
// 水平觸發 (Level-Triggered, 默認)
// - 只要 fd 可讀,epoll_wait 就會返回該 fd
// - 適合: 每次只讀取部分數據的場景
epoll_event ev{};
ev.events = EPOLLIN;  // LT 模式
ev.data.fd = fd;

// 邊緣觸發 (Edge-Triggered)
// - 僅在狀態變化時通知一次
// - 必須一次性讀完所有數據
// - 性能更高,但編程難度更大
ev.events = EPOLLIN | EPOLLET;  // ET 模式
```

**ET 模式要點:**
1. 必須使用非阻塞 I/O
2. 讀/寫必須循環直到返回 `EAGAIN`
3. 處理 `EAGAIN` 後才能再次等待事件

**性能對比:**

| 特性 | select | poll | epoll (LT) | epoll (ET) |
|------|--------|------|------------|------------|
| fd 數量限制 | 1024 | 無限制 | 無限制 | 無限制 |
| 時間複雜度 | O(n) | O(n) | O(活躍連接數) | O(活躍連接數) |
| 跨平台 | ✅ | ✅ | Linux only | Linux only |
| HFT 推薦 | ❌ | ❌ | ✅ | ✅✅ |

---

## 6.3 零拷貝技術

### 6.3.1 傳統 I/O 的拷貝問題

```
傳統讀取文件並發送到 socket:

read(file_fd, buffer, size);
write(socket_fd, buffer, size);

拷貝次數: 4 次
1. DMA 拷貝: 磁盤 -> 內核緩衝區
2. CPU 拷貝: 內核緩衝區 -> 用戶空間 buffer
3. CPU 拷貝: 用戶空間 buffer -> socket 緩衝區
4. DMA 拷貝: socket 緩衝區 -> 網卡

上下文切換: 4 次 (read 系統調用 2 次, write 系統調用 2 次)
```

### 6.3.2 sendfile() - 文件到 Socket

```cpp
#include <sys/sendfile.h>
#include <fcntl.h>
#include <unistd.h>
#include <iostream>

void sendfile_example(int socket_fd, const char* filepath) {
    int file_fd = open(filepath, O_RDONLY);
    if (file_fd == -1) {
        perror("open");
        return;
    }
    
    // 獲取文件大小
    off_t offset = 0;
    struct stat st;
    fstat(file_fd, &st);
    
    // 零拷貝發送
    ssize_t sent = sendfile(socket_fd, file_fd, &offset, st.st_size);
    
    if (sent == -1) {
        perror("sendfile");
    } else {
        std::cout << "Sent " << sent << " bytes\n";
    }
    
    close(file_fd);
}
```

**sendfile 優勢:**
- 拷貝次數: 2 次 (僅 DMA 拷貝)
- 上下文切換: 2 次 (1 次系統調用)
- 適用: 靜態文件服務器、CDN

### 6.3.3 splice() - 管道與 Socket

```cpp
#include <fcntl.h>
#include <unistd.h>
#include <iostream>

void splice_example(int input_fd, int output_fd) {
    // 創建管道
    int pipefd[2];
    if (pipe(pipefd) == -1) {
        perror("pipe");
        return;
    }
    
    const size_t BUFFER_SIZE = 65536;
    
    while (true) {
        // 從 input_fd 讀到管道
        ssize_t n = splice(input_fd, nullptr, pipefd[1], nullptr, 
                           BUFFER_SIZE, SPLICE_F_MOVE);
        if (n <= 0) break;
        
        // 從管道寫到 output_fd
        ssize_t written = splice(pipefd[0], nullptr, output_fd, nullptr, 
                                  n, SPLICE_F_MOVE);
        if (written != n) break;
    }
    
    close(pipefd[0]);
    close(pipefd[1]);
}
```

### 6.3.4 mmap() - 記憶體映射

```cpp
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>
#include <iostream>
#include <cstring>

void mmap_example(const char* filepath) {
    int fd = open(filepath, O_RDWR);
    if (fd == -1) {
        perror("open");
        return;
    }
    
    // 獲取文件大小
    struct stat st;
    fstat(fd, &st);
    size_t size = st.st_size;
    
    // 映射文件到內存
    void* addr = mmap(nullptr, size, PROT_READ | PROT_WRITE, 
                      MAP_SHARED, fd, 0);
    
    if (addr == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return;
    }
    
    // 直接操作內存 (修改會自動同步到文件)
    char* data = static_cast<char*>(addr);
    std::cout << "First 100 bytes:\n" 
              << std::string(data, std::min(size, size_t(100))) << "\n";
    
    // 修改數據
    if (size > 0) {
        data[0] = 'X';
    }
    
    // 取消映射
    munmap(addr, size);
    close(fd);
}
```

**mmap 應用場景:**
- 大文件隨機訪問
- 進程間共享內存
- 數據庫文件訪問

---

## 6.4 TCP 調優

### 6.4.1 TCP_NODELAY - 禁用 Nagle 算法

```cpp
#include <netinet/tcp.h>
#include <sys/socket.h>

void enable_tcp_nodelay(int socket_fd) {
    int flag = 1;
    if (setsockopt(socket_fd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag)) == -1) {
        perror("TCP_NODELAY");
    } else {
        std::cout << "TCP_NODELAY enabled\n";
    }
}
```

**Nagle 算法:**
- 目的: 減少小包數量,提高網路效率
- 行為: 緩存小數據包,等待 ACK 或積累到 MSS 再發送
- 延遲影響: 可能增加數十毫秒延遲

**HFT 必須禁用:** 低延遲優先於帶寬效率

### 6.4.2 Socket 緩衝區調整

```cpp
void tune_socket_buffers(int socket_fd) {
    // 發送緩衝區
    int sndbuf = 1024 * 1024;  // 1MB
    if (setsockopt(socket_fd, SOL_SOCKET, SO_SNDBUF, &sndbuf, sizeof(sndbuf)) == -1) {
        perror("SO_SNDBUF");
    }
    
    // 接收緩衝區
    int rcvbuf = 1024 * 1024;  // 1MB
    if (setsockopt(socket_fd, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf)) == -1) {
        perror("SO_RCVBUF");
    }
    
    std::cout << "Buffer sizes set to 1MB\n";
}
```

### 6.4.3 地址重用

```cpp
void enable_address_reuse(int socket_fd) {
    // SO_REUSEADDR: 允許綁定到 TIME_WAIT 狀態的地址
    int reuse_addr = 1;
    if (setsockopt(socket_fd, SOL_SOCKET, SO_REUSEADDR, &reuse_addr, sizeof(reuse_addr)) == -1) {
        perror("SO_REUSEADDR");
    }
    
    // SO_REUSEPORT: 允許多個進程綁定同一端口 (Linux 3.9+)
    int reuse_port = 1;
    if (setsockopt(socket_fd, SOL_SOCKET, SO_REUSEPORT, &reuse_port, sizeof(reuse_port)) == -1) {
        perror("SO_REUSEPORT");
    }
    
    std::cout << "Address reuse enabled\n";
}
```

### 6.4.4 系統級 TCP 參數調優

```bash
# 查看當前設置
sysctl -a | grep tcp

# 調整 TCP 緩衝區大小 (讀, 默認, 寫 in bytes)
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"

# 縮短 TIME_WAIT 超時時間
sudo sysctl -w net.ipv4.tcp_fin_timeout=15

# 啟用 TCP 快速回收 TIME_WAIT socket (謹慎使用)
sudo sysctl -w net.ipv4.tcp_tw_reuse=1

# 增加 backlog 大小
sudo sysctl -w net.core.somaxconn=4096

# 持久化配置
echo "net.ipv4.tcp_rmem = 4096 87380 16777216" | sudo tee -a /etc/sysctl.conf
```

---

## 6.5 UDP 程式設計與多播

### 6.5.1 UDP 基本操作

```cpp
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <cstring>

// UDP 發送端
void udp_sender() {
    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    
    sockaddr_in dest_addr{};
    dest_addr.sin_family = AF_INET;
    dest_addr.sin_port = htons(9999);
    inet_pton(AF_INET, "127.0.0.1", &dest_addr.sin_addr);
    
    const char* message = "UDP message";
    sendto(sock, message, strlen(message), 0, 
           (sockaddr*)&dest_addr, sizeof(dest_addr));
    
    close(sock);
}

// UDP 接收端
void udp_receiver() {
    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(9999);
    
    bind(sock, (sockaddr*)&addr, sizeof(addr));
    
    char buffer[1024];
    sockaddr_in sender_addr{};
    socklen_t sender_len = sizeof(sender_addr);
    
    ssize_t n = recvfrom(sock, buffer, sizeof(buffer) - 1, 0,
                         (sockaddr*)&sender_addr, &sender_len);
    
    if (n > 0) {
        buffer[n] = '\0';
        std::cout << "Received: " << buffer << "\n";
    }
    
    close(sock);
}
```

### 6.5.2 UDP 多播 (Multicast)

```cpp
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <cstring>

// 多播發送端
class MulticastSender {
private:
    int sock_;
    sockaddr_in group_addr_;
    
public:
    MulticastSender(const char* group_ip, int port) {
        sock_ = socket(AF_INET, SOCK_DGRAM, 0);
        
        // 設置 TTL (Time To Live)
        int ttl = 64;
        setsockopt(sock_, IPPROTO_IP, IP_MULTICAST_TTL, &ttl, sizeof(ttl));
        
        group_addr_.sin_family = AF_INET;
        group_addr_.sin_port = htons(port);
        inet_pton(AF_INET, group_ip, &group_addr_.sin_addr);
        
        std::cout << "Multicast sender ready\n";
    }
    
    void send(const char* message) {
        sendto(sock_, message, strlen(message), 0,
               (sockaddr*)&group_addr_, sizeof(group_addr_));
        std::cout << "Sent: " << message << "\n";
    }
    
    ~MulticastSender() {
        close(sock_);
    }
};

// 多播接收端
class MulticastReceiver {
private:
    int sock_;
    
public:
    MulticastReceiver(const char* group_ip, int port) {
        sock_ = socket(AF_INET, SOCK_DGRAM, 0);
        
        // 允許多個接收者
        int reuse = 1;
        setsockopt(sock_, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));
        
        // 綁定到任意地址
        sockaddr_in addr{};
        addr.sin_family = AF_INET;
        addr.sin_addr.s_addr = INADDR_ANY;
        addr.sin_port = htons(port);
        bind(sock_, (sockaddr*)&addr, sizeof(addr));
        
        // 加入多播組
        ip_mreq mreq{};
        inet_pton(AF_INET, group_ip, &mreq.imr_multiaddr);
        mreq.imr_interface.s_addr = INADDR_ANY;
        
        if (setsockopt(sock_, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq)) == -1) {
            perror("IP_ADD_MEMBERSHIP");
        } else {
            std::cout << "Joined multicast group " << group_ip << ":" << port << "\n";
        }
    }
    
    void receive() {
        char buffer[1024];
        
        while (true) {
            ssize_t n = recvfrom(sock_, buffer, sizeof(buffer) - 1, 0, nullptr, nullptr);
            
            if (n > 0) {
                buffer[n] = '\0';
                std::cout << "Received: " << buffer << "\n";
            }
        }
    }
    
    ~MulticastReceiver() {
        close(sock_);
    }
};

// 使用示例
int main() {
    const char* GROUP_IP = "239.255.0.1";  // 多播地址範圍: 224.0.0.0 - 239.255.255.255
    const int PORT = 12345;
    
    // 接收端
    MulticastReceiver receiver(GROUP_IP, PORT);
    
    // 發送端 (在另一個進程中運行)
    // MulticastSender sender(GROUP_IP, PORT);
    // sender.send("Market data update");
    
    receiver.receive();
    
    return 0;
}
```

**HFT 中的多播應用:**
- 市場數據饋送 (Market Data Feed)
- 交易所同時向所有參與者廣播報價
- 低延遲 (單向傳輸,無 TCP 握手)
- 高帶寬效率 (一次發送,多方接收)

---

## 6.6 Reactor 模式

### 6.6.1 Reactor 模式架構

```
┌─────────────────────────────────────────┐
│           Event Loop (Reactor)          │
│                                         │
│  while (true) {                        │
│    events = epoll_wait()               │
│    for each event:                     │
│      dispatch_handler(event)           │
│  }                                     │
└─────────────────────────────────────────┘
         ▲                    │
         │                    ▼
    ┌────────┐         ┌──────────────┐
    │ Events │         │   Handlers   │
    │        │         │              │
    │ Accept │         │ AcceptHandler│
    │  Read  │◄────────┤  ReadHandler │
    │  Write │         │ WriteHandler │
    └────────┘         └──────────────┘
```

### 6.6.2 簡單 Reactor 實現

```cpp
#include <sys/epoll.h>
#include <unistd.h>
#include <functional>
#include <map>
#include <iostream>

class Reactor {
private:
    int epoll_fd_;
    bool running_ = true;
    
    using Handler = std::function<void(int fd, uint32_t events)>;
    std::map<int, Handler> handlers_;
    
    static constexpr int MAX_EVENTS = 64;
    
public:
    Reactor() {
        epoll_fd_ = epoll_create1(0);
        if (epoll_fd_ == -1) {
            throw std::runtime_error("epoll_create1 failed");
        }
    }
    
    ~Reactor() {
        close(epoll_fd_);
    }
    
    void add_handler(int fd, uint32_t events, Handler handler) {
        epoll_event ev{};
        ev.events = events;
        ev.data.fd = fd;
        
        if (epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, fd, &ev) == -1) {
            perror("epoll_ctl: add");
            return;
        }
        
        handlers_[fd] = std::move(handler);
    }
    
    void remove_handler(int fd) {
        epoll_ctl(epoll_fd_, EPOLL_CTL_DEL, fd, nullptr);
        handlers_.erase(fd);
        close(fd);
    }
    
    void run() {
        epoll_event events[MAX_EVENTS];
        
        while (running_) {
            int nfds = epoll_wait(epoll_fd_, events, MAX_EVENTS, -1);
            
            if (nfds == -1) {
                if (errno == EINTR) continue;
                perror("epoll_wait");
                break;
            }
            
            for (int i = 0; i < nfds; ++i) {
                int fd = events[i].data.fd;
                auto it = handlers_.find(fd);
                
                if (it != handlers_.end()) {
                    it->second(fd, events[i].events);
                }
            }
        }
    }
    
    void stop() {
        running_ = false;
    }
};

// 使用示例
int main() {
    Reactor reactor;
    
    // 創建並添加 listen socket
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    // ... (bind, listen, set_nonblocking)
    
    reactor.add_handler(listen_fd, EPOLLIN, [&](int fd, uint32_t events) {
        // Accept handler
        int client_fd = accept(fd, nullptr, nullptr);
        std::cout << "New connection: " << client_fd << "\n";
        
        reactor.add_handler(client_fd, EPOLLIN, [&](int cfd, uint32_t ev) {
            // Read handler
            char buffer[1024];
            ssize_t n = read(cfd, buffer, sizeof(buffer));
            
            if (n <= 0) {
                reactor.remove_handler(cfd);
            } else {
                write(cfd, buffer, n);  // Echo
            }
        });
    });
    
    reactor.run();
    return 0;
}
```

---

## 實作練習

### 練習 1: epoll Echo 服務器

**需求:** 實現支持多客戶端的 echo 服務器,使用 epoll ET 模式。

```cpp
// 完整實現見前文 EpollServer 類
// 測試:
// 1. 編譯: g++ -std=c++17 -O2 echo_server.cpp -o server
// 2. 運行: ./server
// 3. 客戶端: telnet localhost 8080
```

### 練習 2: UDP 多播市場數據模擬

**需求:** 模擬市場數據發送與接收,使用 UDP 多播。

```cpp
// 發送端: 每秒發送一次模擬報價
#include <thread>
#include <chrono>
#include <sstream>

struct MarketData {
    char symbol[8];
    double bid;
    double ask;
    uint64_t timestamp;
};

void market_data_publisher() {
    MulticastSender sender("239.255.0.1", 12345);
    
    MarketData data;
    std::strcpy(data.symbol, "AAPL");
    
    while (true) {
        data.bid = 150.0 + (rand() % 100) / 100.0;
        data.ask = data.bid + 0.05;
        data.timestamp = std::chrono::system_clock::now().time_since_epoch().count();
        
        sender.send(reinterpret_cast<const char*>(&data));
        
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
}
```

### 練習 3: 零拷貝文件傳輸對比

**需求:** 對比傳統 read/write 與 sendfile 的性能。

```cpp
#include <chrono>
#include <sys/sendfile.h>

// 傳統方式
double traditional_copy(const char* src, int dest_fd) {
    int src_fd = open(src, O_RDONLY);
    char buffer[4096];
    
    auto start = std::chrono::high_resolution_clock::now();
    
    while (true) {
        ssize_t n = read(src_fd, buffer, sizeof(buffer));
        if (n <= 0) break;
        write(dest_fd, buffer, n);
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    close(src_fd);
    
    return std::chrono::duration<double, std::milli>(end - start).count();
}

// 零拷貝方式
double zerocopy_copy(const char* src, int dest_fd) {
    int src_fd = open(src, O_RDONLY);
    struct stat st;
    fstat(src_fd, &st);
    
    auto start = std::chrono::high_resolution_clock::now();
    
    off_t offset = 0;
    sendfile(dest_fd, src_fd, &offset, st.st_size);
    
    auto end = std::chrono::high_resolution_clock::now();
    close(src_fd);
    
    return std::chrono::duration<double, std::milli>(end - start).count();
}

// 測試
int main() {
    const char* test_file = "/tmp/test_file.bin";
    
    // 創建測試文件 (100MB)
    int fd = open(test_file, O_CREAT | O_WRONLY, 0644);
    char buffer[1024] = {0};
    for (int i = 0; i < 100 * 1024; ++i) {
        write(fd, buffer, sizeof(buffer));
    }
    close(fd);
    
    int dest = open("/dev/null", O_WRONLY);
    
    double t1 = traditional_copy(test_file, dest);
    std::cout << "Traditional: " << t1 << " ms\n";
    
    double t2 = zerocopy_copy(test_file, dest);
    std::cout << "Zero-copy: " << t2 << " ms\n";
    std::cout << "Speedup: " << t1 / t2 << "x\n";
    
    close(dest);
    return 0;
}
```

### 練習 4: TCP_NODELAY 延遲測試

**需求:** 測量啟用/禁用 TCP_NODELAY 對小包延遲的影響。

```cpp
#include <netinet/tcp.h>
#include <chrono>

void latency_test(int socket_fd, bool nodelay) {
    int flag = nodelay ? 1 : 0;
    setsockopt(socket_fd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));
    
    const int ITERATIONS = 1000;
    const char* message = "PING";
    char buffer[16];
    
    auto start = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < ITERATIONS; ++i) {
        send(socket_fd, message, 4, 0);
        recv(socket_fd, buffer, sizeof(buffer), 0);
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration<double, std::micro>(end - start);
    
    std::cout << (nodelay ? "TCP_NODELAY enabled: " : "TCP_NODELAY disabled: ")
              << duration.count() / ITERATIONS << " μs/roundtrip\n";
}
```

### 練習 5: 連接池實作

**需求:** 實作高效的 TCP 連接池,支持連接重用。

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <queue>
#include <mutex>
#include <condition_variable>

class ConnectionPool {
private:
    std::string host_;
    uint16_t port_;
    size_t maxSize_;
    
    std::queue<int> available_;
    std::mutex mutex_;
    std::condition_variable cv_;
    size_t totalConnections_ = 0;
    
    int createConnection() {
        int sock = socket(AF_INET, SOCK_STREAM, 0);
        if (sock < 0) return -1;
        
        // TCP 優化
        int nodelay = 1;
        setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &nodelay, sizeof(nodelay));
        
        sockaddr_in addr{};
        addr.sin_family = AF_INET;
        addr.sin_port = htons(port_);
        inet_pton(AF_INET, host_.c_str(), &addr.sin_addr);
        
        if (connect(sock, (sockaddr*)&addr, sizeof(addr)) < 0) {
            close(sock);
            return -1;
        }
        
        return sock;
    }
    
public:
    ConnectionPool(const std::string& host, uint16_t port, size_t maxSize)
        : host_(host), port_(port), maxSize_(maxSize) {}
    
    ~ConnectionPool() {
        std::lock_guard<std::mutex> lock(mutex_);
        while (!available_.empty()) {
            close(available_.front());
            available_.pop();
        }
    }
    
    // 獲取連接
    int acquire() {
        std::unique_lock<std::mutex> lock(mutex_);
        
        // 有可用連接
        if (!available_.empty()) {
            int conn = available_.front();
            available_.pop();
            return conn;
        }
        
        // 創建新連接
        if (totalConnections_ < maxSize_) {
            int conn = createConnection();
            if (conn >= 0) {
                ++totalConnections_;
                return conn;
            }
        }
        
        // 等待可用連接
        cv_.wait(lock, [this] { return !available_.empty(); });
        int conn = available_.front();
        available_.pop();
        return conn;
    }
    
    // 釋放連接
    void release(int conn) {
        std::lock_guard<std::mutex> lock(mutex_);
        available_.push(conn);
        cv_.notify_one();
    }
    
    // RAII 包裝
    class Connection {
        ConnectionPool& pool_;
        int fd_;
    public:
        Connection(ConnectionPool& pool) : pool_(pool), fd_(pool.acquire()) {}
        ~Connection() { pool_.release(fd_); }
        int fd() const { return fd_; }
        operator int() const { return fd_; }
    };
};

// 使用示例
int main() {
    ConnectionPool pool("127.0.0.1", 8080, 10);
    
    // 並發請求
    std::vector<std::thread> threads;
    for (int i = 0; i < 20; ++i) {
        threads.emplace_back([&pool, i] {
            ConnectionPool::Connection conn(pool);
            
            // 發送請求
            std::string request = "Request " + std::to_string(i);
            send(conn, request.c_str(), request.size(), 0);
            
            // 接收響應
            char buffer[1024];
            recv(conn, buffer, sizeof(buffer), 0);
        });
    }
    
    for (auto& t : threads) t.join();
}
```

### 練習 6: 高性能 HTTP 解析器

**需求:** 實作零拷貝的 HTTP 請求解析器。

```cpp
#include <string_view>
#include <unordered_map>

class HTTPParser {
public:
    struct Request {
        std::string_view method;
        std::string_view uri;
        std::string_view version;
        std::unordered_map<std::string_view, std::string_view> headers;
        std::string_view body;
    };
    
    enum class State {
        RequestLine,
        Headers,
        Body,
        Complete,
        Error
    };
    
private:
    State state_ = State::RequestLine;
    const char* data_ = nullptr;
    size_t size_ = 0;
    size_t pos_ = 0;
    Request request_;
    size_t contentLength_ = 0;
    
    std::string_view readLine() {
        size_t start = pos_;
        while (pos_ < size_ - 1) {
            if (data_[pos_] == '\r' && data_[pos_ + 1] == '\n') {
                std::string_view line(data_ + start, pos_ - start);
                pos_ += 2;
                return line;
            }
            ++pos_;
        }
        return {};
    }
    
    bool parseRequestLine(std::string_view line) {
        // "GET /path HTTP/1.1"
        size_t methodEnd = line.find(' ');
        if (methodEnd == std::string_view::npos) return false;
        
        request_.method = line.substr(0, methodEnd);
        
        size_t uriEnd = line.find(' ', methodEnd + 1);
        if (uriEnd == std::string_view::npos) return false;
        
        request_.uri = line.substr(methodEnd + 1, uriEnd - methodEnd - 1);
        request_.version = line.substr(uriEnd + 1);
        
        return true;
    }
    
    bool parseHeader(std::string_view line) {
        // "Content-Length: 123"
        size_t colonPos = line.find(':');
        if (colonPos == std::string_view::npos) return false;
        
        std::string_view name = line.substr(0, colonPos);
        std::string_view value = line.substr(colonPos + 1);
        
        // 跳過前導空格
        while (!value.empty() && value[0] == ' ') {
            value.remove_prefix(1);
        }
        
        request_.headers[name] = value;
        
        // 檢查 Content-Length
        if (name == "Content-Length") {
            contentLength_ = 0;
            for (char c : value) {
                if (c >= '0' && c <= '9') {
                    contentLength_ = contentLength_ * 10 + (c - '0');
                }
            }
        }
        
        return true;
    }
    
public:
    State parse(const char* data, size_t size) {
        data_ = data;
        size_ = size;
        pos_ = 0;
        
        while (pos_ < size_) {
            switch (state_) {
            case State::RequestLine: {
                auto line = readLine();
                if (line.empty()) return state_;
                
                if (!parseRequestLine(line)) {
                    state_ = State::Error;
                    return state_;
                }
                state_ = State::Headers;
                break;
            }
            
            case State::Headers: {
                auto line = readLine();
                if (line.empty()) return state_;
                
                if (line.size() == 0) {
                    // 空行,headers 結束
                    if (contentLength_ > 0) {
                        state_ = State::Body;
                    } else {
                        state_ = State::Complete;
                    }
                } else {
                    parseHeader(line);
                }
                break;
            }
            
            case State::Body: {
                size_t remaining = size_ - pos_;
                if (remaining >= contentLength_) {
                    request_.body = std::string_view(data_ + pos_, contentLength_);
                    pos_ += contentLength_;
                    state_ = State::Complete;
                } else {
                    return state_;  // 需要更多數據
                }
                break;
            }
            
            default:
                return state_;
            }
        }
        
        return state_;
    }
    
    const Request& getRequest() const { return request_; }
    
    void reset() {
        state_ = State::RequestLine;
        request_ = {};
        contentLength_ = 0;
    }
};

// 測試
int main() {
    const char* request = 
        "GET /api/data HTTP/1.1\r\n"
        "Host: localhost\r\n"
        "Content-Length: 13\r\n"
        "\r\n"
        "Hello, World!";
    
    HTTPParser parser;
    auto state = parser.parse(request, strlen(request));
    
    if (state == HTTPParser::State::Complete) {
        auto& req = parser.getRequest();
        std::cout << "Method: " << req.method << "\n";
        std::cout << "URI: " << req.uri << "\n";
        std::cout << "Version: " << req.version << "\n";
        std::cout << "Body: " << req.body << "\n";
    }
}
```

### 練習 7: 實作 SO_BUSY_POLL 低延遲接收

**需求:** 使用 busy polling 降低 socket 接收延遲。

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <chrono>
#include <vector>
#include <algorithm>

class LowLatencyReceiver {
    int socket_;
    
public:
    bool init(uint16_t port, int busyPollUs = 50) {
        socket_ = socket(AF_INET, SOCK_DGRAM, 0);
        if (socket_ < 0) return false;
        
        // 啟用 SO_BUSY_POLL
        if (setsockopt(socket_, SOL_SOCKET, SO_BUSY_POLL, 
                       &busyPollUs, sizeof(busyPollUs)) < 0) {
            perror("SO_BUSY_POLL");
            // 繼續,可能不支持
        }
        
        // 增大接收緩衝區
        int bufSize = 4 * 1024 * 1024;
        setsockopt(socket_, SOL_SOCKET, SO_RCVBUF, &bufSize, sizeof(bufSize));
        
        // 綁定
        sockaddr_in addr{};
        addr.sin_family = AF_INET;
        addr.sin_addr.s_addr = INADDR_ANY;
        addr.sin_port = htons(port);
        
        if (bind(socket_, (sockaddr*)&addr, sizeof(addr)) < 0) {
            return false;
        }
        
        return true;
    }
    
    ~LowLatencyReceiver() {
        if (socket_ >= 0) close(socket_);
    }
    
    // 測量接收延遲
    void measureLatency(int iterations) {
        char buffer[1500];
        std::vector<double> latencies;
        latencies.reserve(iterations);
        
        for (int i = 0; i < iterations; ++i) {
            // 假設發送方包含時間戳
            ssize_t n = recv(socket_, buffer, sizeof(buffer), 0);
            auto recvTime = std::chrono::high_resolution_clock::now();
            
            if (n > sizeof(uint64_t)) {
                uint64_t sendTime;
                std::memcpy(&sendTime, buffer, sizeof(sendTime));
                
                auto sendTimePoint = std::chrono::time_point<
                    std::chrono::high_resolution_clock,
                    std::chrono::nanoseconds>(
                        std::chrono::nanoseconds(sendTime));
                
                auto latency = std::chrono::duration<double, std::micro>(
                    recvTime - sendTimePoint).count();
                
                latencies.push_back(latency);
            }
        }
        
        // 統計
        std::sort(latencies.begin(), latencies.end());
        size_t n = latencies.size();
        
        std::cout << "Latency Statistics (μs):\n"
                  << "  Min: " << latencies[0] << "\n"
                  << "  P50: " << latencies[n/2] << "\n"
                  << "  P99: " << latencies[n*99/100] << "\n"
                  << "  Max: " << latencies[n-1] << "\n";
    }
};

// 發送端 (需要另外運行)
void sender(const char* destIp, uint16_t port, int count) {
    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    inet_pton(AF_INET, destIp, &addr.sin_addr);
    
    for (int i = 0; i < count; ++i) {
        char buffer[64];
        auto now = std::chrono::high_resolution_clock::now();
        uint64_t timestamp = now.time_since_epoch().count();
        
        std::memcpy(buffer, &timestamp, sizeof(timestamp));
        sendto(sock, buffer, sizeof(buffer), 0, 
               (sockaddr*)&addr, sizeof(addr));
        
        usleep(1000);  // 1ms 間隔
    }
    
    close(sock);
}

int main() {
    LowLatencyReceiver receiver;
    if (receiver.init(9999, 50)) {
        std::cout << "Receiver started with SO_BUSY_POLL=50μs\n";
        receiver.measureLatency(1000);
    }
}
```

---

## 關鍵要點總結

### I/O 模型選擇

| 場景 | 推薦模型 | 理由 |
|------|----------|------|
| 少量連接 | 阻塞 I/O | 簡單 |
| 中等連接 (<1000) | poll/select | 跨平台 |
| 大量連接 (>1000) | epoll (Linux) | 高效 |
| HFT 系統 | epoll ET | 最低延遲 |

### TCP 調優清單 (HFT)

```cpp
// 1. 禁用 Nagle
int nodelay = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &nodelay, sizeof(nodelay));

// 2. 設置大緩衝區
int bufsize = 1024 * 1024;
setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &bufsize, sizeof(bufsize));
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &bufsize, sizeof(bufsize));

// 3. 地址重用
int reuse = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

// 4. 非阻塞模式
fcntl(fd, F_SETFL, fcntl(fd, F_GETFL) | O_NONBLOCK);
```

### 零拷貝技術應用

| 技術 | 適用場景 | 性能提升 |
|------|----------|----------|
| `sendfile` | 文件 -> socket | 2-3x |
| `splice` | socket <-> pipe | 2x |
| `mmap` | 大文件隨機訪問 | 避免 read/write |

### UDP vs TCP (HFT)

| 特性 | UDP | TCP |
|------|-----|-----|
| 可靠性 | ❌ | ✅ |
| 延遲 | 更低 | 較高 |
| 順序保證 | ❌ | ✅ |
| 市場數據接收 | ✅ (多播) | ❌ |
| 訂單發送 | ❌ | ✅ |

---

## 參考資料 (References)

1. W. Richard Stevens, *UNIX Network Programming, Volume 1: The Sockets Networking API*, 3rd Edition (2003)
2. Linux man pages:
   - [epoll(7)](https://man7.org/linux/man-pages/man7/epoll.7.html)
   - [tcp(7)](https://man7.org/linux/man-pages/man7/tcp.7.html)
   - [sendfile(2)](https://man7.org/linux/man-pages/man2/sendfile.2.html)
   - [splice(2)](https://man7.org/linux/man-pages/man2/splice.2.html)
3. [The C10K Problem](http://www.kegel.com/c10k.html) by Dan Kegel
4. [Epoll is fundamentally broken](https://idea.popcount.org/2017-02-20-epoll-is-fundamentally-broken-12/) (深入理解 epoll 限制)
5. [TCP Tuning Guide](https://fasterdata.es.net/network-tuning/tcp-tuning/) by ESnet
6. [Multicast Howto](https://tldp.org/HOWTO/Multicast-HOWTO.html)
7. Brendan Gregg, [Linux Performance](https://www.brendangregg.com/linuxperf.html)
