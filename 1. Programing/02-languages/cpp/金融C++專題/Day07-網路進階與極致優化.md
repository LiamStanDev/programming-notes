# Day 7: 網路進階與極致優化

## 學習目標

本課程深入探討極致網路性能優化技術,涵蓋現代異步 I/O (io_uring)、內核旁路 (DPDK)、高精度時間測量 (RDTSC) 與系統性延遲優化方法。這些是構建微秒級低延遲系統的關鍵技術。

**核心主題:**
- io_uring 現代異步 I/O
- DPDK 內核旁路技術
- 高精度時間測量 (RDTSC, PTP)
- 網路延遲優化全棧方法
- 延遲測量與分析

---

## 7.1 io_uring - 現代異步 I/O

### 7.1.1 io_uring 架構

```
傳統異步 I/O (AIO) 問題:
- 僅支持直接 I/O (O_DIRECT)
- 每個操作需要一次系統調用
- 性能受限

io_uring 設計 (Linux 5.1+):

用戶空間                內核空間
┌──────────────┐       ┌──────────────┐
│ Submission   │──────>│   I/O 引擎   │
│   Queue      │       │              │
│    (SQ)      │       │  處理請求    │
└──────────────┘       └──────────────┘
                              │
                              ▼
┌──────────────┐       ┌──────────────┐
│ Completion   │<──────│   結果返回   │
│   Queue      │       │              │
│    (CQ)      │       └──────────────┘
└──────────────┘

優勢:
1. 共享內存環形緩衝區 (無拷貝)
2. 批量提交請求 (減少系統調用)
3. 支持所有 I/O 類型
4. 輪詢模式 (IORING_SETUP_IOPOLL)
```

### 7.1.2 io_uring 基本使用

```cpp
#include <liburing.h>
#include <iostream>
#include <cstring>

class IOUringEchoServer {
private:
    io_uring ring_;
    int listen_fd_;
    
    static constexpr int QUEUE_DEPTH = 256;
    static constexpr int BUFFER_SIZE = 4096;
    
    enum class OpType {
        ACCEPT,
        READ,
        WRITE
    };
    
    struct RequestData {
        OpType type;
        int fd;
        char buffer[BUFFER_SIZE];
        size_t len;
    };
    
public:
    IOUringEchoServer(int port) {
        // 初始化 io_uring
        if (io_uring_queue_init(QUEUE_DEPTH, &ring_, 0) < 0) {
            throw std::runtime_error("io_uring_queue_init failed");
        }
        
        // 創建並配置 listen socket
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
        
        std::cout << "io_uring server started on port " << port << "\n";
    }
    
    ~IOUringEchoServer() {
        io_uring_queue_exit(&ring_);
        close(listen_fd_);
    }
    
    void submit_accept() {
        auto* req_data = new RequestData{OpType::ACCEPT, listen_fd_, {}, 0};
        
        io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
        io_uring_prep_accept(sqe, listen_fd_, nullptr, nullptr, 0);
        io_uring_sqe_set_data(sqe, req_data);
    }
    
    void submit_read(int fd) {
        auto* req_data = new RequestData{OpType::READ, fd, {}, 0};
        
        io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
        io_uring_prep_read(sqe, fd, req_data->buffer, BUFFER_SIZE, 0);
        io_uring_sqe_set_data(sqe, req_data);
    }
    
    void submit_write(int fd, const char* data, size_t len) {
        auto* req_data = new RequestData{OpType::WRITE, fd, {}, len};
        std::memcpy(req_data->buffer, data, len);
        
        io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
        io_uring_prep_write(sqe, fd, req_data->buffer, len, 0);
        io_uring_sqe_set_data(sqe, req_data);
    }
    
    void run() {
        // 提交初始 accept
        submit_accept();
        io_uring_submit(&ring_);
        
        io_uring_cqe* cqe;
        
        while (true) {
            // 等待完成事件
            int ret = io_uring_wait_cqe(&ring_, &cqe);
            if (ret < 0) {
                std::cerr << "io_uring_wait_cqe error: " << strerror(-ret) << "\n";
                break;
            }
            
            auto* req_data = static_cast<RequestData*>(io_uring_cqe_get_data(cqe));
            
            if (req_data->type == OpType::ACCEPT) {
                if (cqe->res >= 0) {
                    int client_fd = cqe->res;
                    std::cout << "New connection: " << client_fd << "\n";
                    
                    // 提交讀取請求
                    submit_read(client_fd);
                    
                    // 繼續接受新連接
                    submit_accept();
                    io_uring_submit(&ring_);
                }
            } else if (req_data->type == OpType::READ) {
                if (cqe->res > 0) {
                    std::cout << "Read " << cqe->res << " bytes from FD " 
                              << req_data->fd << "\n";
                    
                    // 回顯數據
                    submit_write(req_data->fd, req_data->buffer, cqe->res);
                    io_uring_submit(&ring_);
                } else {
                    // 連接關閉或錯誤
                    std::cout << "Connection closed: " << req_data->fd << "\n";
                    close(req_data->fd);
                }
            } else if (req_data->type == OpType::WRITE) {
                if (cqe->res > 0) {
                    // 寫入成功,繼續讀取
                    submit_read(req_data->fd);
                    io_uring_submit(&ring_);
                }
            }
            
            delete req_data;
            io_uring_cqe_seen(&ring_, cqe);
        }
    }
};

int main() {
    try {
        IOUringEchoServer server(8080);
        server.run();
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << "\n";
        return 1;
    }
    return 0;
}
```

### 7.1.3 io_uring 高級特性

```cpp
#include <liburing.h>

// 輪詢模式 (減少中斷延遲)
void setup_polling_mode() {
    io_uring ring;
    io_uring_params params{};
    
    // IOPOLL: 輪詢完成隊列 (需要支持的驅動)
    params.flags = IORING_SETUP_IOPOLL;
    
    // SQPOLL: 內核線程輪詢提交隊列 (進一步減少系統調用)
    params.flags |= IORING_SETUP_SQPOLL;
    params.sq_thread_idle = 2000;  // 線程空閒 2 秒後休眠
    
    io_uring_queue_init_params(256, &ring, &params);
    
    // 使用時無需 io_uring_submit (SQPOLL 自動處理)
    // 僅需填充 SQE 即可
}

// 批量操作
void batch_operations(io_uring& ring, const std::vector<int>& fds) {
    char buffers[10][4096];
    
    // 批量提交多個讀取請求
    for (size_t i = 0; i < fds.size(); ++i) {
        io_uring_sqe* sqe = io_uring_get_sqe(&ring);
        io_uring_prep_read(sqe, fds[i], buffers[i], sizeof(buffers[i]), 0);
    }
    
    // 一次系統調用提交所有請求
    io_uring_submit(&ring);
}

// 鏈接操作 (一個完成後自動執行下一個)
void linked_operations(io_uring& ring, int fd) {
    char buffer[4096];
    
    // 讀取操作
    io_uring_sqe* sqe1 = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe1, fd, buffer, sizeof(buffer), 0);
    sqe1->flags |= IOSQE_IO_LINK;  // 標記為鏈接
    
    // 寫入操作 (僅在讀取成功後執行)
    io_uring_sqe* sqe2 = io_uring_get_sqe(&ring);
    io_uring_prep_write(sqe2, fd, buffer, sizeof(buffer), 0);
    
    io_uring_submit(&ring);
}
```

**io_uring vs epoll 性能對比:**

| 特性 | epoll | io_uring | 優勢 |
|------|-------|----------|------|
| 系統調用次數 | 每個 I/O 1-2 次 | 批量 1 次 | io_uring |
| 內存拷貝 | 有 | 零拷貝 | io_uring |
| 延遲 | 微秒級 | 亞微秒級 | io_uring |
| CPU 使用 | 中斷驅動 | 可輪詢 | io_uring (SQPOLL) |

---

## 7.2 DPDK - 內核旁路技術

### 7.2.1 DPDK 核心概念

```
傳統網路棧:
應用程序 -> 系統調用 -> 內核網路棧 -> 驅動 -> 網卡

延遲來源:
- 系統調用開銷
- 上下文切換
- 內核處理 (協議棧、防火牆等)
- 中斷處理

DPDK (Data Plane Development Kit):
應用程序 (用戶空間) <-> 直接訪問 <-> 網卡

工作原理:
1. 網卡直接映射到用戶空間 (mmap)
2. PMD (Poll Mode Driver): 輪詢而非中斷
3. 大頁內存 (Huge Pages): 減少 TLB miss
4. CPU 親和性: 綁定核心避免遷移
5. 無鎖隊列: 多核協作
```

### 7.2.2 DPDK 基本示例

```cpp
#include <rte_eal.h>
#include <rte_ethdev.h>
#include <rte_mbuf.h>
#include <iostream>

#define RX_RING_SIZE 1024
#define TX_RING_SIZE 1024
#define NUM_MBUFS 8191
#define MBUF_CACHE_SIZE 250
#define BURST_SIZE 32

class DPDKApp {
private:
    uint16_t port_id_ = 0;
    rte_mempool* mbuf_pool_ = nullptr;
    
    static rte_eth_conf port_conf_;
    
public:
    int init(int argc, char** argv) {
        // 初始化 EAL (Environment Abstraction Layer)
        int ret = rte_eal_init(argc, argv);
        if (ret < 0) {
            std::cerr << "EAL initialization failed\n";
            return -1;
        }
        
        // 創建 mbuf 內存池
        mbuf_pool_ = rte_pktmbuf_pool_create("MBUF_POOL", NUM_MBUFS,
            MBUF_CACHE_SIZE, 0, RTE_MBUF_DEFAULT_BUF_SIZE, rte_socket_id());
        
        if (mbuf_pool_ == nullptr) {
            std::cerr << "Cannot create mbuf pool\n";
            return -1;
        }
        
        // 配置端口
        if (configure_port() < 0) {
            return -1;
        }
        
        return 0;
    }
    
    int configure_port() {
        rte_eth_dev_info dev_info;
        rte_eth_dev_info_get(port_id_, &dev_info);
        
        // 配置設備
        int ret = rte_eth_dev_configure(port_id_, 1, 1, &port_conf_);
        if (ret < 0) {
            std::cerr << "Cannot configure device\n";
            return ret;
        }
        
        // 設置 RX 隊列
        ret = rte_eth_rx_queue_setup(port_id_, 0, RX_RING_SIZE,
            rte_eth_dev_socket_id(port_id_), nullptr, mbuf_pool_);
        if (ret < 0) {
            return ret;
        }
        
        // 設置 TX 隊列
        ret = rte_eth_tx_queue_setup(port_id_, 0, TX_RING_SIZE,
            rte_eth_dev_socket_id(port_id_), nullptr);
        if (ret < 0) {
            return ret;
        }
        
        // 啟動設備
        ret = rte_eth_dev_start(port_id_);
        if (ret < 0) {
            return ret;
        }
        
        // 啟用混雜模式
        rte_eth_promiscuous_enable(port_id_);
        
        std::cout << "Port " << port_id_ << " configured\n";
        return 0;
    }
    
    void run() {
        std::cout << "Starting packet processing...\n";
        
        rte_mbuf* bufs[BURST_SIZE];
        
        while (true) {
            // 批量接收數據包
            uint16_t nb_rx = rte_eth_rx_burst(port_id_, 0, bufs, BURST_SIZE);
            
            if (nb_rx == 0) {
                continue;
            }
            
            // 處理數據包
            for (uint16_t i = 0; i < nb_rx; ++i) {
                process_packet(bufs[i]);
            }
            
            // 批量發送 (echo)
            uint16_t nb_tx = rte_eth_tx_burst(port_id_, 0, bufs, nb_rx);
            
            // 釋放未發送的包
            if (unlikely(nb_tx < nb_rx)) {
                for (uint16_t i = nb_tx; i < nb_rx; ++i) {
                    rte_pktmbuf_free(bufs[i]);
                }
            }
        }
    }
    
private:
    void process_packet(rte_mbuf* pkt) {
        // 數據包處理邏輯
        // 可訪問: rte_pktmbuf_mtod(pkt, void*) 獲取數據指針
    }
};

rte_eth_conf DPDKApp::port_conf_ = {
    .rxmode = {
        .max_rx_pkt_len = RTE_ETHER_MAX_LEN,
    },
};

int main(int argc, char** argv) {
    DPDKApp app;
    
    if (app.init(argc, argv) < 0) {
        return -1;
    }
    
    app.run();
    return 0;
}
```

### 7.2.3 DPDK 優化技術

```cpp
// 1. CPU 親和性
void set_cpu_affinity(int core_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(core_id, &cpuset);
    
    pthread_t thread = pthread_self();
    pthread_setaffinity_np(thread, sizeof(cpuset), &cpuset);
}

// 2. 無鎖環形隊列
#include <rte_ring.h>

rte_ring* create_lockfree_ring(const char* name, unsigned count) {
    return rte_ring_create(name, count, rte_socket_id(),
                           RING_F_SP_ENQ | RING_F_SC_DEQ);
    // SP_ENQ: Single Producer
    // SC_DEQ: Single Consumer
}

// 3. 批量操作
void batch_processing_example(rte_ring* ring) {
    void* objs[32];
    
    // 批量入隊
    unsigned enqueued = rte_ring_enqueue_bulk(ring, objs, 32, nullptr);
    
    // 批量出隊
    unsigned dequeued = rte_ring_dequeue_bulk(ring, objs, 32, nullptr);
}

// 4. 預取 (Prefetch)
void prefetch_example(rte_mbuf** pkts, int count) {
    for (int i = 0; i < count; ++i) {
        // 預取下一個包
        if (i + 1 < count) {
            rte_prefetch0(rte_pktmbuf_mtod(pkts[i + 1], void*));
        }
        
        // 處理當前包
        process_packet(pkts[i]);
    }
}
```

**DPDK 性能數據:**
- 單核處理: 10-40 Mpps (百萬包/秒)
- 延遲: < 1 微秒
- 適用場景: 高頻交易、DPI、負載均衡

**限制:**
- 需要專用網卡
- 獨占網卡 (其他程序無法使用)
- 開發複雜度高
- 需要大頁內存支持

---

## 7.3 高精度時間測量

### 7.3.1 clock_gettime()

```cpp
#include <time.h>
#include <iostream>

void clock_gettime_example() {
    timespec ts;
    
    // CLOCK_REALTIME: 系統時間 (可調整,可能倒退)
    clock_gettime(CLOCK_REALTIME, &ts);
    std::cout << "REALTIME: " << ts.tv_sec << "s " << ts.tv_nsec << "ns\n";
    
    // CLOCK_MONOTONIC: 單調時間 (不可調整,不會倒退)
    clock_gettime(CLOCK_MONOTONIC, &ts);
    std::cout << "MONOTONIC: " << ts.tv_sec << "s " << ts.tv_nsec << "ns\n";
    
    // CLOCK_MONOTONIC_RAW: 原始單調時間 (不受 NTP 影響)
    clock_gettime(CLOCK_MONOTONIC_RAW, &ts);
    std::cout << "MONOTONIC_RAW: " << ts.tv_sec << "s " << ts.tv_nsec << "ns\n";
}

// 高精度計時器
class HighResTimer {
private:
    timespec start_;
    
public:
    void start() {
        clock_gettime(CLOCK_MONOTONIC, &start_);
    }
    
    double elapsed_ns() const {
        timespec end;
        clock_gettime(CLOCK_MONOTONIC, &end);
        
        return (end.tv_sec - start_.tv_sec) * 1e9 +
               (end.tv_nsec - start_.tv_nsec);
    }
    
    double elapsed_us() const {
        return elapsed_ns() / 1000.0;
    }
};

// 使用示例
void benchmark_function() {
    HighResTimer timer;
    timer.start();
    
    // 執行操作
    volatile int sum = 0;
    for (int i = 0; i < 1000000; ++i) {
        sum += i;
    }
    
    std::cout << "Elapsed: " << timer.elapsed_us() << " μs\n";
}
```

### 7.3.2 RDTSC - CPU 時間戳計數器

```cpp
#include <x86intrin.h>
#include <iostream>

// 讀取 TSC
inline uint64_t rdtsc() {
    return __rdtsc();
}

// 讀取 TSC (帶序列化)
inline uint64_t rdtscp() {
    unsigned int aux;
    return __rdtscp(&aux);
}

class TSCTimer {
private:
    uint64_t start_tsc_;
    double tsc_ghz_;  // TSC 頻率 (GHz)
    
    double calibrate_tsc() {
        const int ITERATIONS = 10;
        double total_ghz = 0;
        
        for (int i = 0; i < ITERATIONS; ++i) {
            timespec ts_start, ts_end;
            uint64_t tsc_start, tsc_end;
            
            clock_gettime(CLOCK_MONOTONIC, &ts_start);
            tsc_start = rdtsc();
            
            // 睡眠 10ms
            timespec sleep_time{0, 10000000};
            nanosleep(&sleep_time, nullptr);
            
            tsc_end = rdtsc();
            clock_gettime(CLOCK_MONOTONIC, &ts_end);
            
            double elapsed_ns = (ts_end.tv_sec - ts_start.tv_sec) * 1e9 +
                                (ts_end.tv_nsec - ts_start.tv_nsec);
            double elapsed_cycles = tsc_end - tsc_start;
            
            total_ghz += elapsed_cycles / elapsed_ns;
        }
        
        return total_ghz / ITERATIONS;
    }
    
public:
    TSCTimer() {
        tsc_ghz_ = calibrate_tsc();
        std::cout << "TSC frequency: " << tsc_ghz_ << " GHz\n";
    }
    
    void start() {
        start_tsc_ = rdtscp();  // 使用 rdtscp 確保序列化
    }
    
    double elapsed_ns() const {
        uint64_t end_tsc = rdtscp();
        uint64_t cycles = end_tsc - start_tsc_;
        return cycles / tsc_ghz_;
    }
    
    double elapsed_cycles() const {
        return rdtscp() - start_tsc_;
    }
};

// 使用示例
void ultra_low_latency_benchmark() {
    TSCTimer timer;
    timer.start();
    
    // 極短操作
    volatile int x = 42;
    x = x * 2;
    
    std::cout << "Operation took " << timer.elapsed_ns() << " ns ("
              << timer.elapsed_cycles() << " cycles)\n";
}
```

**RDTSC 注意事項:**
1. **頻率變化**: 現代 CPU 支持 `constant_tsc` (恆定頻率)
2. **多核同步**: 需要 `synchronized TSC` (大多數現代 CPU 支持)
3. **序列化**: 使用 `rdtscp` 或 `cpuid; rdtsc` 避免亂序執行
4. **校準**: 需定期校準 TSC 頻率

**檢查 TSC 特性:**
```bash
# 檢查是否支持恆定 TSC
cat /proc/cpuinfo | grep constant_tsc

# 檢查當前 CPU 頻率
cat /proc/cpuinfo | grep "cpu MHz"
```

### 7.3.3 PTP (Precision Time Protocol)

```cpp
#include <linux/ptp_clock.h>
#include <sys/ioctl.h>
#include <fcntl.h>

class PTPClock {
private:
    int fd_;
    
public:
    PTPClock(const char* device = "/dev/ptp0") {
        fd_ = open(device, O_RDONLY);
        if (fd_ == -1) {
            throw std::runtime_error("Cannot open PTP device");
        }
    }
    
    ~PTPClock() {
        close(fd_);
    }
    
    timespec get_time() {
        ptp_sys_offset offset{};
        offset.n_samples = 1;
        
        if (ioctl(fd_, PTP_SYS_OFFSET, &offset) == -1) {
            throw std::runtime_error("PTP_SYS_OFFSET failed");
        }
        
        timespec ts;
        ts.tv_sec = offset.ts[1].sec;
        ts.tv_nsec = offset.ts[1].nsec;
        return ts;
    }
    
    void get_capabilities() {
        ptp_clock_caps caps{};
        
        if (ioctl(fd_, PTP_CLOCK_GETCAPS, &caps) == -1) {
            throw std::runtime_error("PTP_CLOCK_GETCAPS failed");
        }
        
        std::cout << "PTP Clock Capabilities:\n"
                  << "  Max adjustment: " << caps.max_adj << " ppb\n"
                  << "  Alarms: " << caps.n_alarm << "\n"
                  << "  External timestamps: " << caps.n_ext_ts << "\n"
                  << "  Programmable pins: " << caps.n_pins << "\n"
                  << "  PPS: " << caps.pps << "\n";
    }
};

// 使用硬體時間戳
void hardware_timestamp_example(int socket_fd) {
    // 啟用 TX 時間戳
    int flags = SOF_TIMESTAMPING_TX_HARDWARE |
                SOF_TIMESTAMPING_RAW_HARDWARE;
    
    if (setsockopt(socket_fd, SOL_SOCKET, SO_TIMESTAMPING, &flags, sizeof(flags)) == -1) {
        perror("SO_TIMESTAMPING");
    }
    
    // 發送數據後,可從錯誤隊列讀取硬體時間戳
    char ctrl[CMSG_SPACE(sizeof(timespec))];
    msghdr msg{};
    msg.msg_control = ctrl;
    msg.msg_controllen = sizeof(ctrl);
    
    if (recvmsg(socket_fd, &msg, MSG_ERRQUEUE) > 0) {
        for (cmsghdr* cmsg = CMSG_FIRSTHDR(&msg); cmsg; cmsg = CMSG_NXTHDR(&msg, cmsg)) {
            if (cmsg->cmsg_level == SOL_SOCKET && cmsg->cmsg_type == SCM_TIMESTAMPING) {
                timespec* ts = (timespec*)CMSG_DATA(cmsg);
                std::cout << "HW timestamp: " << ts[2].tv_sec << "s " 
                          << ts[2].tv_nsec << "ns\n";
            }
        }
    }
}
```

**時間同步精度對比:**

| 協議 | 精度 | 需求 | HFT 適用 |
|------|------|------|----------|
| NTP | 毫秒級 | 網路連接 | ❌ |
| PTP (軟體) | 微秒級 | 支持的網卡 | ⚠️ |
| PTP (硬體) | 納秒級 | 專用網卡 + 交換機 | ✅ |
| GPS | 納秒級 | GPS 接收器 | ✅ |

---

## 7.4 網路延遲優化

### 7.4.1 延遲來源分析

```
端到端延遲組成:

┌─────────────────────────────────────────────────────┐
│ 應用層         │ 處理邏輯、數據序列化            │
│ (1-50μs)       │ 優化: 無鎖結構、零拷貝          │
├─────────────────────────────────────────────────────┤
│ 系統調用       │ send(), recv()                 │
│ (0.5-2μs)      │ 優化: io_uring 批量操作         │
├─────────────────────────────────────────────────────┤
│ 內核網路棧     │ TCP/IP 協議處理                │
│ (2-10μs)       │ 優化: TCP_NODELAY, 參數調優     │
├─────────────────────────────────────────────────────┤
│ 網卡驅動       │ DMA 傳輸、中斷處理             │
│ (1-5μs)        │ 優化: DPDK (繞過內核)          │
├─────────────────────────────────────────────────────┤
│ 網路傳輸       │ 交換機、路由器、線路            │
│ (10-1000μs)    │ 優化: Co-location, 專線        │
└─────────────────────────────────────────────────────┘

總延遲 (典型 HFT): 20-100μs (單向)
```

### 7.4.2 應用層優化

```cpp
#include <atomic>
#include <cstring>

// 1. 對象池 (避免動態分配)
template<typename T, size_t N>
class ObjectPool {
private:
    T objects_[N];
    std::atomic<size_t> next_free_{0};
    
public:
    T* acquire() {
        size_t idx = next_free_.fetch_add(1, std::memory_order_relaxed);
        if (idx >= N) {
            return nullptr;  // 池耗盡
        }
        return &objects_[idx];
    }
    
    void release(T* obj) {
        // 簡化版: 實際需要更複雜的回收機制
        (void)obj;
    }
};

// 2. 零拷貝消息序列化
struct __attribute__((packed)) MarketDataMessage {
    uint64_t timestamp;
    char symbol[8];
    double bid;
    double ask;
    uint32_t bid_size;
    uint32_t ask_size;
};

void send_market_data_zerocopy(int socket_fd, const MarketDataMessage& msg) {
    // 直接發送結構體 (無序列化開銷)
    send(socket_fd, &msg, sizeof(msg), MSG_DONTWAIT);
}

// 3. 無鎖單生產者單消費者隊列 (SPSC)
template<typename T, size_t SIZE>
class SPSCQueue {
private:
    alignas(64) std::atomic<size_t> head_{0};
    alignas(64) std::atomic<size_t> tail_{0};
    T data_[SIZE];
    
public:
    bool try_push(const T& value) {
        size_t head = head_.load(std::memory_order_relaxed);
        size_t next = (head + 1) % SIZE;
        
        if (next == tail_.load(std::memory_order_acquire)) {
            return false;  // 隊列滿
        }
        
        data_[head] = value;
        head_.store(next, std::memory_order_release);
        return true;
    }
    
    bool try_pop(T& value) {
        size_t tail = tail_.load(std::memory_order_relaxed);
        
        if (tail == head_.load(std::memory_order_acquire)) {
            return false;  // 隊列空
        }
        
        value = data_[tail];
        tail_.store((tail + 1) % SIZE, std::memory_order_release);
        return true;
    }
};

// 4. 預分配緩衝區
class PreallocatedBuffers {
private:
    static constexpr size_t BUFFER_SIZE = 4096;
    static constexpr size_t NUM_BUFFERS = 1024;
    
    char buffers_[NUM_BUFFERS][BUFFER_SIZE];
    std::atomic<size_t> next_{0};
    
public:
    char* get_buffer() {
        size_t idx = next_.fetch_add(1, std::memory_order_relaxed) % NUM_BUFFERS;
        return buffers_[idx];
    }
};
```

### 7.4.3 內核優化

```cpp
#include <sys/socket.h>
#include <netinet/tcp.h>

void kernel_optimizations(int socket_fd) {
    // 1. TCP_NODELAY (禁用 Nagle)
    int nodelay = 1;
    setsockopt(socket_fd, IPPROTO_TCP, TCP_NODELAY, &nodelay, sizeof(nodelay));
    
    // 2. TCP_QUICKACK (禁用延遲 ACK)
    int quickack = 1;
    setsockopt(socket_fd, IPPROTO_TCP, TCP_QUICKACK, &quickack, sizeof(quickack));
    
    // 3. SO_BUSY_POLL (忙輪詢)
    int busy_poll_us = 50;  // 50 微秒
    setsockopt(socket_fd, SOL_SOCKET, SO_BUSY_POLL, &busy_poll_us, sizeof(busy_poll_us));
    
    // 4. 大緩衝區
    int bufsize = 4 * 1024 * 1024;  // 4MB
    setsockopt(socket_fd, SOL_SOCKET, SO_SNDBUF, &bufsize, sizeof(bufsize));
    setsockopt(socket_fd, SOL_SOCKET, SO_RCVBUF, &bufsize, sizeof(bufsize));
    
    // 5. SO_PRIORITY (設置優先級)
    int priority = 6;  // 高優先級
    setsockopt(socket_fd, SOL_SOCKET, SO_PRIORITY, &priority, sizeof(priority));
}

// 系統參數調優 (需要 root 權限)
void system_tuning() {
    system("sysctl -w net.core.busy_poll=50");
    system("sysctl -w net.core.busy_read=50");
    system("sysctl -w net.ipv4.tcp_low_latency=1");
    
    // IRQ 親和性 (將網卡中斷綁定到特定 CPU)
    // echo "2" > /proc/irq/<IRQ_NUM>/smp_affinity_list
}
```

### 7.4.4 延遲測量

```cpp
#include <vector>
#include <algorithm>

class LatencyStats {
private:
    std::vector<double> samples_;
    
public:
    void add_sample(double latency_ns) {
        samples_.push_back(latency_ns);
    }
    
    void print_stats() {
        if (samples_.empty()) return;
        
        std::sort(samples_.begin(), samples_.end());
        
        size_t n = samples_.size();
        double sum = 0;
        for (double s : samples_) sum += s;
        
        std::cout << "Latency Statistics (ns):\n"
                  << "  Count: " << n << "\n"
                  << "  Mean: " << sum / n << "\n"
                  << "  Min: " << samples_[0] << "\n"
                  << "  P50: " << samples_[n * 50 / 100] << "\n"
                  << "  P95: " << samples_[n * 95 / 100] << "\n"
                  << "  P99: " << samples_[n * 99 / 100] << "\n"
                  << "  P99.9: " << samples_[n * 999 / 1000] << "\n"
                  << "  Max: " << samples_[n - 1] << "\n";
    }
    
    void print_histogram() {
        const int BUCKETS = 10;
        std::vector<int> hist(BUCKETS, 0);
        
        double min = samples_[0];
        double max = samples_.back();
        double range = max - min;
        
        for (double s : samples_) {
            int bucket = std::min(BUCKETS - 1, 
                                  static_cast<int>((s - min) / range * BUCKETS));
            hist[bucket]++;
        }
        
        std::cout << "\nHistogram:\n";
        for (int i = 0; i < BUCKETS; ++i) {
            double lower = min + i * range / BUCKETS;
            double upper = min + (i + 1) * range / BUCKETS;
            std::cout << "[" << lower << " - " << upper << "): "
                      << std::string(hist[i] / 10, '*') << " (" << hist[i] << ")\n";
        }
    }
};

// 端到端延遲測量
void measure_roundtrip_latency(int socket_fd, int iterations) {
    LatencyStats stats;
    TSCTimer timer;
    
    char send_buf[64] = "PING";
    char recv_buf[64];
    
    for (int i = 0; i < iterations; ++i) {
        timer.start();
        
        send(socket_fd, send_buf, 4, 0);
        recv(socket_fd, recv_buf, sizeof(recv_buf), 0);
        
        double latency = timer.elapsed_ns();
        stats.add_sample(latency);
        
        // 等待一段時間避免過載
        usleep(100);
    }
    
    stats.print_stats();
    stats.print_histogram();
}
```

---

## 實作練習

### 練習 1: io_uring Echo 服務器

**需求:** 實現完整的 io_uring echo 服務器,支持多客戶端。

```cpp
// 見前文 IOUringEchoServer 實現
// 編譯: g++ -std=c++17 -O3 io_uring_server.cpp -luring -o server
// 測試: telnet localhost 8080
```

### 練習 2: RDTSC 高精度計時器校準

**需求:** 實現 TSC 頻率自動校準,並測量納秒級操作。

```cpp
// 見前文 TSCTimer 實現
// 測試不同操作的週期數:
void benchmark_operations() {
    TSCTimer timer;
    
    // 測試整數加法
    timer.start();
    volatile int x = 0;
    for (int i = 0; i < 1000; ++i) x++;
    std::cout << "1000 additions: " << timer.elapsed_cycles() << " cycles\n";
    
    // 測試原子操作
    timer.start();
    std::atomic<int> y{0};
    for (int i = 0; i < 1000; ++i) y.fetch_add(1, std::memory_order_relaxed);
    std::cout << "1000 atomic adds: " << timer.elapsed_cycles() << " cycles\n";
    
    // 測試內存訪問
    timer.start();
    int arr[1000];
    for (int i = 0; i < 1000; ++i) arr[i] = i;
    std::cout << "1000 memory writes: " << timer.elapsed_cycles() << " cycles\n";
}
```

### 練習 3: TCP_NODELAY 影響測試

**需求:** 對比啟用/禁用 TCP_NODELAY 對小包延遲的影響。

```cpp
void nodelay_benchmark() {
    // 創建連接對
    int sv[2];
    socketpair(AF_UNIX, SOCK_STREAM, 0, sv);
    
    // 測試禁用 TCP_NODELAY (Nagle 啟用)
    std::cout << "Testing with Nagle enabled:\n";
    int nodelay = 0;
    setsockopt(sv[0], IPPROTO_TCP, TCP_NODELAY, &nodelay, sizeof(nodelay));
    measure_roundtrip_latency(sv[0], 1000);
    
    // 測試啟用 TCP_NODELAY (Nagle 禁用)
    std::cout << "\nTesting with Nagle disabled:\n";
    nodelay = 1;
    setsockopt(sv[0], IPPROTO_TCP, TCP_NODELAY, &nodelay, sizeof(nodelay));
    measure_roundtrip_latency(sv[0], 1000);
    
    close(sv[0]);
    close(sv[1]);
}
```

### 練習 4: 延遲瓶頸識別

**需求:** 分段測量網路延遲,識別瓶頸。

```cpp
struct LatencyBreakdown {
    double serialization_ns;
    double kernel_ns;
    double network_ns;
    double deserialization_ns;
};

LatencyBreakdown measure_breakdown(int socket_fd) {
    TSCTimer timer;
    LatencyBreakdown result{};
    
    MarketDataMessage msg{};
    msg.timestamp = 12345;
    std::strcpy(msg.symbol, "AAPL");
    msg.bid = 150.0;
    msg.ask = 150.05;
    
    // 1. 序列化 (本例中無需序列化,直接發送結構體)
    timer.start();
    // ... 序列化邏輯 ...
    result.serialization_ns = timer.elapsed_ns();
    
    // 2. 系統調用
    timer.start();
    send(socket_fd, &msg, sizeof(msg), 0);
    result.kernel_ns = timer.elapsed_ns();
    
    // 3. 網路傳輸 + 接收
    timer.start();
    MarketDataMessage recv_msg;
    recv(socket_fd, &recv_msg, sizeof(recv_msg), 0);
    result.network_ns = timer.elapsed_ns();
    
    // 4. 反序列化
    timer.start();
    // ... 反序列化邏輯 ...
    result.deserialization_ns = timer.elapsed_ns();
    
    return result;
}

void print_breakdown(const LatencyBreakdown& lb) {
    double total = lb.serialization_ns + lb.kernel_ns + 
                   lb.network_ns + lb.deserialization_ns;
    
    std::cout << "Latency Breakdown:\n"
              << "  Serialization: " << lb.serialization_ns 
              << " ns (" << (lb.serialization_ns / total * 100) << "%)\n"
              << "  Kernel: " << lb.kernel_ns 
              << " ns (" << (lb.kernel_ns / total * 100) << "%)\n"
              << "  Network: " << lb.network_ns 
              << " ns (" << (lb.network_ns / total * 100) << "%)\n"
              << "  Deserialization: " << lb.deserialization_ns 
              << " ns (" << (lb.deserialization_ns / total * 100) << "%)\n"
              << "  Total: " << total << " ns\n";
}
```

---

## 關鍵要點總結

### 技術選擇矩陣

| 技術 | 延遲降低 | 開發複雜度 | 硬體要求 | HFT 推薦 |
|------|----------|------------|----------|----------|
| epoll ET | 低 | 低 | 無 | ✅ 基礎 |
| io_uring | 中 | 中 | Linux 5.1+ | ✅ 進階 |
| DPDK | 極高 | 極高 | 專用網卡 | ✅ 極致 |
| RDTSC | N/A | 低 | 無 | ✅ 測量 |
| PTP | N/A | 中 | 支持網卡 | ✅ 同步 |

### 延遲優化檢查清單

**應用層:**
- [ ] 使用對象池避免動態分配
- [ ] 無鎖數據結構 (SPSC 隊列)
- [ ] 零拷貝消息序列化
- [ ] 預分配緩衝區

**系統調用:**
- [ ] 使用 io_uring 批量操作
- [ ] 減少系統調用次數

**內核:**
- [ ] 啟用 `TCP_NODELAY`
- [ ] 啟用 `SO_BUSY_POLL`
- [ ] 調整 TCP 緩衝區
- [ ] 設置 IRQ 親和性

**硬體:**
- [ ] DPDK 內核旁路 (極致場景)
- [ ] Co-location (共置機房)
- [ ] 專線網路
- [ ] 低延遲交換機

### 時間測量建議

| 場景 | 推薦工具 | 精度 |
|------|----------|------|
| 粗粒度性能分析 | `std::chrono` | 微秒 |
| 精確延遲測量 | `clock_gettime(CLOCK_MONOTONIC)` | 納秒 |
| 極短操作測量 | RDTSC | 週期 (sub-ns) |
| 全局時間同步 | PTP | 納秒 |

---

## 參考資料 (References)

1. Jens Axboe, [Efficient IO with io_uring](https://kernel.dk/io_uring.pdf) (2019)
2. [DPDK Documentation](https://doc.dpdk.org/)
3. Intel, [Using the RDTSC Instruction for Performance Monitoring](https://www.intel.com/content/dam/www/public/us/en/documents/white-papers/ia-32-ia-64-benchmark-code-execution-paper.pdf)
4. Linux man pages:
   - [io_uring(7)](https://man7.org/linux/man-pages/man7/io_uring.7.html)
   - [clock_gettime(2)](https://man7.org/linux/man-pages/man2/clock_gettime.2.html)
5. [PTP Linux](https://www.kernel.org/doc/html/latest/driver-api/ptp.html)
6. Brendan Gregg, [Systems Performance: Enterprise and the Cloud](https://www.brendangregg.com/systems-performance-2nd-edition-book.html), 2nd Edition (2020)
7. [Low Latency Performance Tuning](https://rigtorp.se/low-latency-guide/) by Erik Rigtorp
8. [Busy Polling Socket](https://www.kernel.org/doc/Documentation/networking/busy-poll.txt)
