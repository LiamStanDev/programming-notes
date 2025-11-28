# Linux 網路編程基礎 (Linux Network Programming Fundamentals)

## 概述

Linux 網路編程是 HFT 系統的核心技能。本章涵蓋 Socket API、TCP/UDP 協議、非阻塞 I/O 與網路除錯工具。

## Socket 基礎

### TCP Socket 建立與連線

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <cstring>

// TCP Client
int create_tcp_client(const char* host, uint16_t port) {
    // 1. 建立 socket
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        throw std::runtime_error("socket() failed");
    }
    
    // 2. 設定伺服器位址
    sockaddr_in server_addr{};
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    
    if (inet_pton(AF_INET, host, &server_addr.sin_addr) <= 0) {
        close(sockfd);
        throw std::runtime_error("inet_pton() failed");
    }
    
    // 3. 連線
    if (connect(sockfd, (sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        close(sockfd);
        throw std::runtime_error("connect() failed");
    }
    
    return sockfd;
}

// TCP Server
int create_tcp_server(uint16_t port, int backlog = 128) {
    // 1. 建立 socket
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        throw std::runtime_error("socket() failed");
    }
    
    // 2. 設定 socket 選項
    int optval = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof(optval));
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEPORT, &optval, sizeof(optval));
    
    // 3. 綁定位址
    sockaddr_in server_addr{};
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(port);
    
    if (bind(sockfd, (sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        close(sockfd);
        throw std::runtime_error("bind() failed");
    }
    
    // 4. 監聽
    if (listen(sockfd, backlog) < 0) {
        close(sockfd);
        throw std::runtime_error("listen() failed");
    }
    
    return sockfd;
}

// Accept 連線
int accept_connection(int server_sockfd) {
    sockaddr_in client_addr{};
    socklen_t addr_len = sizeof(client_addr);
    
    int client_sockfd = accept(server_sockfd, (sockaddr*)&client_addr, &addr_len);
    if (client_sockfd < 0) {
        throw std::runtime_error("accept() failed");
    }
    
    // 取得客戶端資訊
    char client_ip[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &client_addr.sin_addr, client_ip, sizeof(client_ip));
    printf("Accepted connection from %s:%d\n", client_ip, ntohs(client_addr.sin_port));
    
    return client_sockfd;
}
```

### UDP Socket

```cpp
// UDP Server
int create_udp_server(uint16_t port) {
    int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd < 0) {
        throw std::runtime_error("socket() failed");
    }
    
    int optval = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof(optval));
    
    sockaddr_in server_addr{};
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(port);
    
    if (bind(sockfd, (sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        close(sockfd);
        throw std::runtime_error("bind() failed");
    }
    
    return sockfd;
}

// UDP 接收
ssize_t udp_receive(int sockfd, void* buffer, size_t len, sockaddr_in* src_addr) {
    socklen_t addr_len = sizeof(*src_addr);
    
    ssize_t n = recvfrom(sockfd, buffer, len, 0, 
                         (sockaddr*)src_addr, &addr_len);
    
    if (n < 0) {
        throw std::runtime_error("recvfrom() failed");
    }
    
    return n;
}

// UDP 發送
ssize_t udp_send(int sockfd, const void* buffer, size_t len, 
                 const sockaddr_in* dest_addr) {
    ssize_t n = sendto(sockfd, buffer, len, 0,
                      (sockaddr*)dest_addr, sizeof(*dest_addr));
    
    if (n < 0) {
        throw std::runtime_error("sendto() failed");
    }
    
    return n;
}
```

### Multicast (組播) 接收

```cpp
#include <arpa/inet.h>

// HFT 市場資料常用 Multicast 傳輸
int join_multicast(const char* group_ip, uint16_t port, const char* iface_ip) {
    int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd < 0) {
        throw std::runtime_error("socket() failed");
    }
    
    // 允許多個程序綁定同一埠
    int optval = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof(optval));
    
    // 綁定到組播埠
    sockaddr_in local_addr{};
    local_addr.sin_family = AF_INET;
    local_addr.sin_addr.s_addr = INADDR_ANY;
    local_addr.sin_port = htons(port);
    
    if (bind(sockfd, (sockaddr*)&local_addr, sizeof(local_addr)) < 0) {
        close(sockfd);
        throw std::runtime_error("bind() failed");
    }
    
    // 加入組播群組
    ip_mreq mreq{};
    inet_pton(AF_INET, group_ip, &mreq.imr_multiaddr);
    inet_pton(AF_INET, iface_ip, &mreq.imr_interface);
    
    if (setsockopt(sockfd, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq)) < 0) {
        close(sockfd);
        throw std::runtime_error("setsockopt(IP_ADD_MEMBERSHIP) failed");
    }
    
    return sockfd;
}
```

## Socket 選項調整

### TCP 效能優化

```cpp
#include <netinet/tcp.h>

void optimize_tcp_socket(int sockfd) {
    int optval;
    
    // 1. TCP_NODELAY: 禁用 Nagle 演算法 (HFT 必須)
    optval = 1;
    setsockopt(sockfd, IPPROTO_TCP, TCP_NODELAY, &optval, sizeof(optval));
    
    // 2. 增大接收/發送緩衝區
    optval = 2 * 1024 * 1024;  // 2 MB
    setsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &optval, sizeof(optval));
    setsockopt(sockfd, SOL_SOCKET, SO_SNDBUF, &optval, sizeof(optval));
    
    // 3. TCP_QUICKACK: 快速 ACK (降低延遲)
    optval = 1;
    setsockopt(sockfd, IPPROTO_TCP, TCP_QUICKACK, &optval, sizeof(optval));
    
    // 4. SO_KEEPALIVE: 保持連線檢測
    optval = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_KEEPALIVE, &optval, sizeof(optval));
    
    // 調整 keepalive 參數
    optval = 60;  // 60 秒開始探測
    setsockopt(sockfd, IPPROTO_TCP, TCP_KEEPIDLE, &optval, sizeof(optval));
    
    optval = 10;  // 探測間隔 10 秒
    setsockopt(sockfd, IPPROTO_TCP, TCP_KEEPINTVL, &optval, sizeof(optval));
    
    optval = 3;   // 探測次數 3 次
    setsockopt(sockfd, IPPROTO_TCP, TCP_KEEPCNT, &optval, sizeof(optval));
}
```

### UDP 效能優化

```cpp
void optimize_udp_socket(int sockfd) {
    int optval;
    
    // 1. 增大接收緩衝區 (避免丟包)
    optval = 16 * 1024 * 1024;  // 16 MB
    setsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &optval, sizeof(optval));
    
    // 2. 設定接收超時 (可選)
    timeval tv{};
    tv.tv_sec = 1;
    tv.tv_usec = 0;
    setsockopt(sockfd, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));
    
    // 3. 設定 TOS (Type of Service) - 最低延遲
    optval = IPTOS_LOWDELAY;
    setsockopt(sockfd, IPPROTO_IP, IP_TOS, &optval, sizeof(optval));
}
```

## 非阻塞 I/O

### 設定非阻塞模式

```cpp
#include <fcntl.h>

void set_nonblocking(int sockfd) {
    int flags = fcntl(sockfd, F_GETFL, 0);
    if (flags < 0) {
        throw std::runtime_error("fcntl(F_GETFL) failed");
    }
    
    if (fcntl(sockfd, F_SETFL, flags | O_NONBLOCK) < 0) {
        throw std::runtime_error("fcntl(F_SETFL) failed");
    }
}

// 非阻塞接收
ssize_t nonblocking_recv(int sockfd, void* buffer, size_t len) {
    ssize_t n = recv(sockfd, buffer, len, 0);
    
    if (n < 0) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            // 無資料可讀
            return 0;
        }
        throw std::runtime_error("recv() failed");
    }
    
    return n;
}

// 非阻塞發送
ssize_t nonblocking_send(int sockfd, const void* buffer, size_t len) {
    ssize_t n = send(sockfd, buffer, len, MSG_DONTWAIT);
    
    if (n < 0) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            // 發送緩衝區滿
            return 0;
        }
        throw std::runtime_error("send() failed");
    }
    
    return n;
}
```

## select / poll / epoll

### select 範例

```cpp
#include <sys/select.h>

// select: 監控多個檔案描述符
void select_example(const std::vector<int>& sockfds) {
    fd_set readfds;
    int max_fd = 0;
    
    while (true) {
        // 設定監控集合
        FD_ZERO(&readfds);
        for (int fd : sockfds) {
            FD_SET(fd, &readfds);
            max_fd = std::max(max_fd, fd);
        }
        
        // 設定超時
        timeval tv{};
        tv.tv_sec = 1;
        tv.tv_usec = 0;
        
        // 等待事件
        int ret = select(max_fd + 1, &readfds, nullptr, nullptr, &tv);
        
        if (ret < 0) {
            throw std::runtime_error("select() failed");
        } else if (ret == 0) {
            // 超時
            continue;
        }
        
        // 檢查哪些 fd 可讀
        for (int fd : sockfds) {
            if (FD_ISSET(fd, &readfds)) {
                handle_readable(fd);
            }
        }
    }
}

// select 限制: 最多監控 FD_SETSIZE (通常 1024) 個 fd
```

### poll 範例

```cpp
#include <poll.h>

void poll_example(const std::vector<int>& sockfds) {
    std::vector<pollfd> fds;
    
    for (int fd : sockfds) {
        pollfd pfd{};
        pfd.fd = fd;
        pfd.events = POLLIN;  // 監控可讀事件
        fds.push_back(pfd);
    }
    
    while (true) {
        int ret = poll(fds.data(), fds.size(), 1000);  // 1 秒超時
        
        if (ret < 0) {
            throw std::runtime_error("poll() failed");
        } else if (ret == 0) {
            continue;  // 超時
        }
        
        for (auto& pfd : fds) {
            if (pfd.revents & POLLIN) {
                handle_readable(pfd.fd);
            }
            if (pfd.revents & POLLERR) {
                handle_error(pfd.fd);
            }
            if (pfd.revents & POLLHUP) {
                handle_disconnect(pfd.fd);
            }
        }
    }
}
```

### epoll 範例 (推薦)

```cpp
#include <sys/epoll.h>

class EpollReactor {
    int epoll_fd_;
    
public:
    EpollReactor() {
        epoll_fd_ = epoll_create1(0);
        if (epoll_fd_ < 0) {
            throw std::runtime_error("epoll_create1() failed");
        }
    }
    
    ~EpollReactor() {
        close(epoll_fd_);
    }
    
    void add_fd(int fd, uint32_t events = EPOLLIN) {
        epoll_event ev{};
        ev.events = events | EPOLLET;  // Edge-Triggered
        ev.data.fd = fd;
        
        if (epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, fd, &ev) < 0) {
            throw std::runtime_error("epoll_ctl(ADD) failed");
        }
    }
    
    void modify_fd(int fd, uint32_t events) {
        epoll_event ev{};
        ev.events = events | EPOLLET;
        ev.data.fd = fd;
        
        if (epoll_ctl(epoll_fd_, EPOLL_CTL_MOD, fd, &ev) < 0) {
            throw std::runtime_error("epoll_ctl(MOD) failed");
        }
    }
    
    void remove_fd(int fd) {
        if (epoll_ctl(epoll_fd_, EPOLL_CTL_DEL, fd, nullptr) < 0) {
            throw std::runtime_error("epoll_ctl(DEL) failed");
        }
    }
    
    void event_loop() {
        constexpr int MAX_EVENTS = 1024;
        epoll_event events[MAX_EVENTS];
        
        while (true) {
            int n = epoll_wait(epoll_fd_, events, MAX_EVENTS, -1);
            
            if (n < 0) {
                if (errno == EINTR) continue;  // 被信號中斷
                throw std::runtime_error("epoll_wait() failed");
            }
            
            for (int i = 0; i < n; ++i) {
                int fd = events[i].data.fd;
                uint32_t ev = events[i].events;
                
                if (ev & EPOLLIN) {
                    handle_readable(fd);
                }
                if (ev & EPOLLOUT) {
                    handle_writable(fd);
                }
                if (ev & EPOLLERR || ev & EPOLLHUP) {
                    handle_error(fd);
                }
            }
        }
    }
};
```

## HFT 應用: 市場資料接收器

### UDP Multicast 市場資料接收

```cpp
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>

struct MarketDataPacket {
    uint32_t sequence;
    uint32_t symbol_id;
    double price;
    int64_t volume;
    uint64_t timestamp;
} __attribute__((packed));

class MarketDataReceiver {
    int sockfd_;
    int epoll_fd_;
    
    // 缺口檢測
    std::unordered_map<uint32_t, uint32_t> expected_seq_;
    
public:
    MarketDataReceiver(const char* group_ip, uint16_t port, const char* iface_ip) {
        sockfd_ = join_multicast(group_ip, port, iface_ip);
        
        // 優化 UDP socket
        optimize_udp_socket(sockfd_);
        
        // 設定非阻塞
        set_nonblocking(sockfd_);
        
        // 建立 epoll
        epoll_fd_ = epoll_create1(0);
        if (epoll_fd_ < 0) {
            throw std::runtime_error("epoll_create1() failed");
        }
        
        // 加入 epoll 監控
        epoll_event ev{};
        ev.events = EPOLLIN | EPOLLET;
        ev.data.fd = sockfd_;
        epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, sockfd_, &ev);
    }
    
    ~MarketDataReceiver() {
        close(sockfd_);
        close(epoll_fd_);
    }
    
    void run() {
        epoll_event events[1];
        char buffer[65536];
        
        while (true) {
            int n = epoll_wait(epoll_fd_, events, 1, -1);
            
            if (n > 0 && events[0].events & EPOLLIN) {
                // Edge-Triggered: 需讀取所有資料
                while (true) {
                    ssize_t bytes = recv(sockfd_, buffer, sizeof(buffer), 0);
                    
                    if (bytes < 0) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) {
                            break;  // 無更多資料
                        }
                        throw std::runtime_error("recv() failed");
                    }
                    
                    if (bytes == 0) {
                        break;
                    }
                    
                    process_packet(buffer, bytes);
                }
            }
        }
    }
    
private:
    void process_packet(const char* data, size_t len) {
        if (len < sizeof(MarketDataPacket)) {
            return;  // 資料不完整
        }
        
        const auto* pkt = reinterpret_cast<const MarketDataPacket*>(data);
        
        // 檢測缺口
        auto& expected = expected_seq_[pkt->symbol_id];
        if (expected != 0 && pkt->sequence != expected) {
            handle_gap(pkt->symbol_id, expected, pkt->sequence);
        }
        expected = pkt->sequence + 1;
        
        // 處理市場資料
        update_order_book(pkt->symbol_id, pkt->price, pkt->volume);
    }
    
    void handle_gap(uint32_t symbol_id, uint32_t expected, uint32_t received) {
        std::cerr << "Gap detected for symbol " << symbol_id 
                  << ": expected " << expected 
                  << ", received " << received << "\n";
        
        // HFT 系統需實作重傳請求機制
        request_retransmission(symbol_id, expected, received - 1);
    }
};
```

## 零拷貝技術 (Zero-Copy)

### sendfile() - 檔案到 Socket

```cpp
#include <sys/sendfile.h>

// 高效傳輸檔案 (避免 user-space 拷貝)
ssize_t send_file_zerocopy(int out_sockfd, int in_filefd, off_t offset, size_t count) {
    return sendfile(out_sockfd, in_filefd, &offset, count);
}
```

### splice() - Socket 到 Socket

```cpp
#include <fcntl.h>

// 零拷貝轉發資料 (proxy)
ssize_t splice_sockets(int in_fd, int out_fd, size_t len) {
    return splice(in_fd, nullptr, out_fd, nullptr, len, SPLICE_F_MOVE);
}
```

### MSG_ZEROCOPY (Kernel 4.14+)

```cpp
// 發送大量資料時避免拷貝
ssize_t send_zerocopy(int sockfd, const void* buffer, size_t len) {
    return send(sockfd, buffer, len, MSG_ZEROCOPY);
}

// 需配合 epoll 監控 EPOLLERR 以確認傳輸完成
```

## 網路除錯工具

### netstat - 檢視連線狀態

```bash
# 檢視所有 TCP 連線
netstat -antp

# 檢視監聽埠
netstat -lntp

# 統計連線數
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

### ss - netstat 替代品 (更快)

```bash
# 檢視 TCP 連線
ss -antp

# 檢視 socket 緩衝區使用
ss -tm

# 檢視特定埠
ss -lntp 'sport = :8080'
```

### tcpdump - 封包擷取

```bash
# 擷取特定埠的 TCP 封包
sudo tcpdump -i eth0 tcp port 8080 -w capture.pcap

# 擷取 UDP multicast
sudo tcpdump -i eth0 udp dst 239.1.1.1 and port 9000

# 即時檢視封包內容
sudo tcpdump -i eth0 -X tcp port 8080
```

### strace - 追蹤系統調用

```bash
# 追蹤程式的網路系統調用
strace -e trace=network ./my_program

# 追蹤特定 PID
strace -p 1234 -e trace=network
```

## 檢查清單

- [ ] TCP 連線禁用 Nagle 演算法 (`TCP_NODELAY`)
- [ ] 增大 Socket 緩衝區 (`SO_RCVBUF`, `SO_SNDBUF`)
- [ ] UDP 接收端設定大緩衝區避免丟包
- [ ] 使用非阻塞 I/O + epoll (避免 select/poll)
- [ ] epoll 使用 Edge-Triggered 模式提升效能
- [ ] Multicast 接收設定正確的網路介面
- [ ] 實作缺口檢測與重傳機制 (UDP)
- [ ] 使用零拷貝技術 (sendfile, splice, MSG_ZEROCOPY)
- [ ] 使用 tcpdump 驗證網路封包格式
- [ ] 使用 ss 監控 Socket 緩衝區使用情況

---

## 參考資料 (References)

1. Stevens, W. Richard, et al. *Unix Network Programming, Volume 1* (2003)
2. [Linux Socket Programming](https://man7.org/linux/man-pages/man7/socket.7.html)
3. [epoll(7) - Linux Manual](https://man7.org/linux/man-pages/man7/epoll.7.html)
4. [TCP(7) - Linux Manual](https://man7.org/linux/man-pages/man7/tcp.7.html)
5. [io_uring and networking](https://kernel.dk/io_uring.pdf)
6. Kerrisk, Michael. *The Linux Programming Interface* (2010)
