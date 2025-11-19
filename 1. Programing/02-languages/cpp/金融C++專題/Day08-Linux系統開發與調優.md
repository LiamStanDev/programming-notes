# Day 8: Linux 系統開發與調優

## 學習目標

本課程深入探討 Linux 系統編程與性能調優技術,涵蓋進程管理、實時調度、CPU 親和性、內存優化與系統監控。這些技術是構建穩定、低延遲系統的基礎。

**核心主題:**
- 進程與線程管理
- 實時調度策略 (SCHED_FIFO)
- CPU 親和性與隔離
- 內存管理 (mlock, Huge Pages)
- 系統監控與性能分析 (perf)

---

## 8.1 進程管理

### 8.1.1 進程創建與控制

```cpp
#include <unistd.h>
#include <sys/wait.h>
#include <iostream>

void fork_example() {
    pid_t pid = fork();
    
    if (pid == -1) {
        perror("fork failed");
        return;
    } else if (pid == 0) {
        // 子進程
        std::cout << "Child process, PID: " << getpid() << "\n";
        std::cout << "Parent PID: " << getppid() << "\n";
        
        // 執行新程序
        execlp("/bin/ls", "ls", "-l", nullptr);
        
        // 如果 exec 失敗才會執行到這裡
        perror("execlp failed");
        exit(1);
    } else {
        // 父進程
        std::cout << "Parent process, PID: " << getpid() << "\n";
        std::cout << "Created child PID: " << pid << "\n";
        
        // 等待子進程結束
        int status;
        waitpid(pid, &status, 0);
        
        if (WIFEXITED(status)) {
            std::cout << "Child exited with status: " << WEXITSTATUS(status) << "\n";
        }
    }
}
```

### 8.1.2 進程間通信 (IPC)

```cpp
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>
#include <cstring>

// 共享內存 (POSIX)
class SharedMemory {
private:
    int fd_;
    void* ptr_;
    size_t size_;
    
public:
    SharedMemory(const char* name, size_t size) : size_(size) {
        // 創建/打開共享內存對象
        fd_ = shm_open(name, O_CREAT | O_RDWR, 0666);
        if (fd_ == -1) {
            throw std::runtime_error("shm_open failed");
        }
        
        // 設置大小
        if (ftruncate(fd_, size) == -1) {
            throw std::runtime_error("ftruncate failed");
        }
        
        // 映射到地址空間
        ptr_ = mmap(nullptr, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd_, 0);
        if (ptr_ == MAP_FAILED) {
            throw std::runtime_error("mmap failed");
        }
    }
    
    ~SharedMemory() {
        munmap(ptr_, size_);
        close(fd_);
    }
    
    void* get() { return ptr_; }
    
    static void unlink(const char* name) {
        shm_unlink(name);
    }
};

// 使用示例
void shared_memory_example() {
    const char* SHM_NAME = "/my_shm";
    
    SharedMemory shm(SHM_NAME, 4096);
    
    // 寫入數據
    char* data = static_cast<char*>(shm.get());
    std::strcpy(data, "Hello from shared memory!");
    
    std::cout << "Data written to shared memory\n";
    
    // 另一個進程可以打開相同名稱的共享內存讀取數據
    
    SharedMemory::unlink(SHM_NAME);
}

// Unix Domain Socket
#include <sys/socket.h>
#include <sys/un.h>

void unix_socket_server() {
    const char* SOCKET_PATH = "/tmp/my_socket";
    
    // 創建 socket
    int server_fd = socket(AF_UNIX, SOCK_STREAM, 0);
    
    // 綁定地址
    sockaddr_un addr{};
    addr.sun_family = AF_UNIX;
    std::strcpy(addr.sun_path, SOCKET_PATH);
    
    unlink(SOCKET_PATH);  // 刪除舊的 socket 文件
    
    bind(server_fd, (sockaddr*)&addr, sizeof(addr));
    listen(server_fd, 5);
    
    std::cout << "Unix socket server listening...\n";
    
    // 接受連接
    int client_fd = accept(server_fd, nullptr, nullptr);
    
    char buffer[256];
    ssize_t n = recv(client_fd, buffer, sizeof(buffer) - 1, 0);
    if (n > 0) {
        buffer[n] = '\0';
        std::cout << "Received: " << buffer << "\n";
    }
    
    close(client_fd);
    close(server_fd);
    unlink(SOCKET_PATH);
}
```

---

## 8.2 實時調度策略

### 8.2.1 調度策略類型

```cpp
#include <sched.h>
#include <pthread.h>
#include <iostream>
#include <cstring>

void print_scheduling_policy(int policy) {
    switch (policy) {
        case SCHED_OTHER: std::cout << "SCHED_OTHER (normal)\n"; break;
        case SCHED_FIFO:  std::cout << "SCHED_FIFO (realtime)\n"; break;
        case SCHED_RR:    std::cout << "SCHED_RR (realtime)\n"; break;
        case SCHED_BATCH: std::cout << "SCHED_BATCH\n"; break;
        case SCHED_IDLE:  std::cout << "SCHED_IDLE\n"; break;
        default:          std::cout << "Unknown\n";
    }
}

void get_current_policy() {
    int policy = sched_getscheduler(0);  // 0 = 當前進程
    std::cout << "Current policy: ";
    print_scheduling_policy(policy);
    
    sched_param param;
    sched_getparam(0, &param);
    std::cout << "Priority: " << param.sched_priority << "\n";
}
```

### 8.2.2 設置 SCHED_FIFO (先進先出實時調度)

```cpp
#include <sched.h>
#include <iostream>

bool set_realtime_priority(int priority) {
    // 檢查優先級範圍
    int min_priority = sched_get_priority_min(SCHED_FIFO);
    int max_priority = sched_get_priority_max(SCHED_FIFO);
    
    std::cout << "SCHED_FIFO priority range: " << min_priority 
              << " - " << max_priority << "\n";
    
    if (priority < min_priority || priority > max_priority) {
        std::cerr << "Invalid priority\n";
        return false;
    }
    
    // 設置調度策略
    sched_param param{};
    param.sched_priority = priority;
    
    if (sched_setscheduler(0, SCHED_FIFO, &param) == -1) {
        perror("sched_setscheduler failed (need root or CAP_SYS_NICE)");
        return false;
    }
    
    std::cout << "Set to SCHED_FIFO with priority " << priority << "\n";
    return true;
}

// 線程級別設置
void set_thread_realtime() {
    pthread_t thread = pthread_self();
    
    sched_param param{};
    param.sched_priority = 80;  // 高優先級
    
    if (pthread_setschedparam(thread, SCHED_FIFO, &param) != 0) {
        perror("pthread_setschedparam");
    } else {
        std::cout << "Thread set to SCHED_FIFO\n";
    }
}
```

### 8.2.3 實時調度示例

```cpp
#include <thread>
#include <chrono>
#include <atomic>

std::atomic<bool> running{true};

void critical_realtime_task() {
    // 設置為實時優先級
    pthread_t thread = pthread_self();
    sched_param param{};
    param.sched_priority = 90;
    pthread_setschedparam(thread, SCHED_FIFO, &param);
    
    std::cout << "Critical task running with SCHED_FIFO priority 90\n";
    
    while (running) {
        // 極低延遲處理
        // 例如: 處理市場數據、發送訂單
        
        std::this_thread::sleep_for(std::chrono::microseconds(100));
    }
}

void background_task() {
    // 保持普通調度
    std::cout << "Background task running with SCHED_OTHER\n";
    
    while (running) {
        // 日誌、統計等非關鍵任務
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

int main() {
    // 需要 root 權限或 CAP_SYS_NICE capability
    std::thread t1(critical_realtime_task);
    std::thread t2(background_task);
    
    std::this_thread::sleep_for(std::chrono::seconds(5));
    running = false;
    
    t1.join();
    t2.join();
    
    return 0;
}
```

**調度策略比較:**

| 策略 | 優先級範圍 | 搶占 | 時間片 | HFT 應用 |
|------|-----------|------|--------|----------|
| SCHED_OTHER | 0 (nice -20~19) | 是 | 動態 | 普通任務 |
| SCHED_FIFO | 1-99 | 是 | 無 (運行直到阻塞) | ✅ 關鍵路徑 |
| SCHED_RR | 1-99 | 是 | 固定 | 多個實時任務 |
| SCHED_BATCH | 0 | 是 | 長 | 批處理 |

---

## 8.3 CPU 親和性與隔離

### 8.3.1 設置 CPU 親和性

```cpp
#include <sched.h>
#include <pthread.h>
#include <iostream>

// 進程級別
bool set_cpu_affinity(int cpu_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    
    if (sched_setaffinity(0, sizeof(cpuset), &cpuset) == -1) {
        perror("sched_setaffinity");
        return false;
    }
    
    std::cout << "Process bound to CPU " << cpu_id << "\n";
    return true;
}

// 線程級別
bool set_thread_affinity(pthread_t thread, int cpu_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    
    if (pthread_setaffinity_np(thread, sizeof(cpuset), &cpuset) != 0) {
        perror("pthread_setaffinity_np");
        return false;
    }
    
    std::cout << "Thread bound to CPU " << cpu_id << "\n";
    return true;
}

// 綁定到多個 CPU
bool set_affinity_multiple(const std::vector<int>& cpu_ids) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    
    for (int cpu : cpu_ids) {
        CPU_SET(cpu, &cpuset);
    }
    
    if (sched_setaffinity(0, sizeof(cpuset), &cpuset) == -1) {
        perror("sched_setaffinity");
        return false;
    }
    
    std::cout << "Process bound to CPUs: ";
    for (int cpu : cpu_ids) {
        std::cout << cpu << " ";
    }
    std::cout << "\n";
    return true;
}

// 查詢當前親和性
void get_cpu_affinity() {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    
    if (sched_getaffinity(0, sizeof(cpuset), &cpuset) == -1) {
        perror("sched_getaffinity");
        return;
    }
    
    std::cout << "Process can run on CPUs: ";
    for (int i = 0; i < CPU_SETSIZE; ++i) {
        if (CPU_ISSET(i, &cpuset)) {
            std::cout << i << " ";
        }
    }
    std::cout << "\n";
}
```

### 8.3.2 HFT 多線程親和性設置

```cpp
#include <thread>
#include <vector>

enum class ThreadRole {
    MARKET_DATA,
    STRATEGY,
    ORDER_SENDING,
    RISK_CHECK,
    LOGGING
};

class HFTSystem {
private:
    std::thread market_data_thread_;
    std::thread strategy_thread_;
    std::thread order_thread_;
    std::thread risk_thread_;
    std::thread log_thread_;
    
public:
    void start() {
        // 市場數據接收: 綁定到 CPU 2,最高優先級
        market_data_thread_ = std::thread([this]() {
            pthread_t self = pthread_self();
            
            cpu_set_t cpuset;
            CPU_ZERO(&cpuset);
            CPU_SET(2, &cpuset);
            pthread_setaffinity_np(self, sizeof(cpuset), &cpuset);
            
            sched_param param{};
            param.sched_priority = 95;
            pthread_setschedparam(self, SCHED_FIFO, &param);
            
            std::cout << "Market data thread on CPU 2, priority 95\n";
            market_data_loop();
        });
        
        // 策略計算: 綁定到 CPU 3,高優先級
        strategy_thread_ = std::thread([this]() {
            pthread_t self = pthread_self();
            
            cpu_set_t cpuset;
            CPU_ZERO(&cpuset);
            CPU_SET(3, &cpuset);
            pthread_setaffinity_np(self, sizeof(cpuset), &cpuset);
            
            sched_param param{};
            param.sched_priority = 90;
            pthread_setschedparam(self, SCHED_FIFO, &param);
            
            std::cout << "Strategy thread on CPU 3, priority 90\n";
            strategy_loop();
        });
        
        // 訂單發送: 綁定到 CPU 4,高優先級
        order_thread_ = std::thread([this]() {
            pthread_t self = pthread_self();
            
            cpu_set_t cpuset;
            CPU_ZERO(&cpuset);
            CPU_SET(4, &cpuset);
            pthread_setaffinity_np(self, sizeof(cpuset), &cpuset);
            
            sched_param param{};
            param.sched_priority = 92;
            pthread_setschedparam(self, SCHED_FIFO, &param);
            
            std::cout << "Order sending thread on CPU 4, priority 92\n";
            order_loop();
        });
        
        // 風控: CPU 5,中等優先級
        risk_thread_ = std::thread([this]() {
            pthread_t self = pthread_self();
            
            cpu_set_t cpuset;
            CPU_ZERO(&cpuset);
            CPU_SET(5, &cpuset);
            pthread_setaffinity_np(self, sizeof(cpuset), &cpuset);
            
            sched_param param{};
            param.sched_priority = 80;
            pthread_setschedparam(self, SCHED_FIFO, &param);
            
            std::cout << "Risk check thread on CPU 5, priority 80\n";
            risk_loop();
        });
        
        // 日誌: CPU 6-7,普通優先級
        log_thread_ = std::thread([this]() {
            pthread_t self = pthread_self();
            
            cpu_set_t cpuset;
            CPU_ZERO(&cpuset);
            CPU_SET(6, &cpuset);
            CPU_SET(7, &cpuset);
            pthread_setaffinity_np(self, sizeof(cpuset), &cpuset);
            
            std::cout << "Logging thread on CPUs 6-7, normal priority\n";
            log_loop();
        });
    }
    
private:
    void market_data_loop() { /* ... */ }
    void strategy_loop() { /* ... */ }
    void order_loop() { /* ... */ }
    void risk_loop() { /* ... */ }
    void log_loop() { /* ... */ }
};
```

### 8.3.3 CPU 隔離 (isolcpus)

```bash
# /etc/default/grub 添加內核參數
GRUB_CMDLINE_LINUX_DEFAULT="isolcpus=2,3,4,5 nohz_full=2,3,4,5 rcu_nocbs=2,3,4,5"

# isolcpus: 從調度器隔離 CPU 2-5
# nohz_full: 減少定時器中斷
# rcu_nocbs: RCU 回調移到其他 CPU

# 更新 grub 並重啟
sudo update-grub
sudo reboot

# 驗證
cat /sys/devices/system/cpu/isolated
# 輸出: 2-5
```

**CPU 隔離效果:**
- 隔離的 CPU 不會運行普通進程
- 減少上下文切換
- 減少中斷干擾
- 延遲抖動降低 50-90%

---

## 8.4 內存管理

### 8.4.1 內存鎖定 (mlock)

```cpp
#include <sys/mman.h>
#include <iostream>

// 鎖定當前進程的所有內存頁
bool lock_all_memory() {
    if (mlockall(MCL_CURRENT | MCL_FUTURE) == -1) {
        perror("mlockall failed (need sufficient RAM & permissions)");
        return false;
    }
    
    std::cout << "All memory locked (no paging)\n";
    return true;
}

// 鎖定特定內存區域
bool lock_memory_region(void* addr, size_t len) {
    if (mlock(addr, len) == -1) {
        perror("mlock");
        return false;
    }
    
    std::cout << "Locked " << len << " bytes at " << addr << "\n";
    return true;
}

// 預分配棧空間 (避免運行時缺頁)
void preallocate_stack(size_t size = 8 * 1024 * 1024) {
    // 8MB 棧
    char dummy[size];
    memset(dummy, 0, size);  // 觸發實際分配
}

// HFT 進程初始化
void hft_memory_init() {
    // 1. 鎖定所有內存
    if (!lock_all_memory()) {
        std::cerr << "Failed to lock memory, performance may degrade\n";
    }
    
    // 2. 預分配棧
    preallocate_stack();
    
    // 3. 預熱堆 (分配後釋放,確保物理內存可用)
    constexpr size_t HEAP_SIZE = 1024 * 1024 * 1024;  // 1GB
    void* heap = malloc(HEAP_SIZE);
    if (heap) {
        memset(heap, 0, HEAP_SIZE);
        free(heap);
    }
    
    std::cout << "Memory initialized for low-latency operation\n";
}
```

**mlockall 效果:**
- 防止內存被交換到磁盤
- 避免缺頁異常 (page fault) 延遲
- 關鍵路徑延遲降低 10-100μs

### 8.4.2 Huge Pages (大頁內存)

```cpp
#include <sys/mman.h>
#include <iostream>

// 分配大頁內存
void* allocate_huge_pages(size_t size) {
    // 對齊到 2MB (默認大頁大小)
    const size_t HUGE_PAGE_SIZE = 2 * 1024 * 1024;
    size = (size + HUGE_PAGE_SIZE - 1) & ~(HUGE_PAGE_SIZE - 1);
    
    void* addr = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                      MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
    
    if (addr == MAP_FAILED) {
        perror("mmap with MAP_HUGETLB failed");
        return nullptr;
    }
    
    std::cout << "Allocated " << size / HUGE_PAGE_SIZE << " huge pages\n";
    return addr;
}

// 使用示例
void huge_pages_example() {
    size_t size = 1024 * 1024 * 1024;  // 1GB
    
    void* mem = allocate_huge_pages(size);
    if (mem) {
        // 使用大頁內存
        memset(mem, 0, size);
        
        // 釋放
        munmap(mem, size);
    }
}
```

**配置 Huge Pages:**

```bash
# 查看當前設置
cat /proc/meminfo | grep Huge
# HugePages_Total:       0
# HugePages_Free:        0
# Hugepagesize:       2048 kB

# 分配 1000 個 2MB 大頁 (共 2GB)
sudo sysctl -w vm.nr_hugepages=1000

# 持久化
echo "vm.nr_hugepages=1000" | sudo tee -a /etc/sysctl.conf

# 驗證
cat /proc/meminfo | grep HugePages_Total
# HugePages_Total:    1000
```

**Huge Pages 優勢:**
- 減少 TLB (Translation Lookaside Buffer) miss
- 頁表更小,降低內存開銷
- 大數據結構訪問性能提升 10-30%

### 8.4.3 NUMA 優化

```cpp
#include <numa.h>
#include <numaif.h>
#include <iostream>

void numa_info() {
    if (numa_available() == -1) {
        std::cout << "NUMA not available\n";
        return;
    }
    
    int num_nodes = numa_num_configured_nodes();
    std::cout << "NUMA nodes: " << num_nodes << "\n";
    
    for (int i = 0; i < num_nodes; ++i) {
        long long free_mem = numa_node_size64(i, nullptr);
        std::cout << "Node " << i << " memory: " 
                  << free_mem / (1024 * 1024) << " MB\n";
    }
}

// 綁定內存到特定 NUMA 節點
void* numa_alloc_on_node(size_t size, int node) {
    void* mem = numa_alloc_onnode(size, node);
    if (!mem) {
        std::cerr << "numa_alloc_onnode failed\n";
        return nullptr;
    }
    
    std::cout << "Allocated " << size << " bytes on NUMA node " << node << "\n";
    return mem;
}

// HFT 應用: 線程與內存同 NUMA 節點
void numa_aware_setup() {
    int cpu = sched_getcpu();  // 當前 CPU
    int node = numa_node_of_cpu(cpu);  // CPU 所在 NUMA 節點
    
    std::cout << "Running on CPU " << cpu << ", NUMA node " << node << "\n";
    
    // 在同一 NUMA 節點分配內存 (減少遠程訪問延遲)
    void* mem = numa_alloc_on_node(1024 * 1024, node);
    
    // 使用內存...
    
    numa_free(mem, 1024 * 1024);
}
```

---

## 8.5 系統監控與性能分析

### 8.5.1 perf 性能分析

```bash
# 統計程序性能
perf stat ./my_program
# Performance counter stats for './my_program':
#     1234.56 msec task-clock
#        1234 context-switches
#         123 cpu-migrations
#       12345 page-faults
#  1234567890 cycles
#   987654321 instructions
#         0.8 IPC (instructions per cycle)

# 記錄性能數據
perf record -g ./my_program

# 查看報告
perf report

# 實時監控
perf top

# CPU 緩存分析
perf stat -e cache-references,cache-misses,cycles,instructions ./my_program
# 123,456,789      cache-references
#  12,345,678      cache-misses              #   10.00% miss rate

# 分析特定函數
perf record -g --call-graph dwarf ./my_program
perf report -g 'graph,0.5,caller'

# 火焰圖生成
perf record -F 99 -g ./my_program
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

### 8.5.2 使用 /proc 文件系統

```cpp
#include <fstream>
#include <iostream>
#include <string>

// 讀取進程狀態
void read_proc_status(pid_t pid = 0) {
    if (pid == 0) pid = getpid();
    
    std::ifstream status("/proc/" + std::to_string(pid) + "/status");
    std::string line;
    
    while (std::getline(status, line)) {
        if (line.find("VmRSS:") == 0 || 
            line.find("VmSize:") == 0 ||
            line.find("Threads:") == 0 ||
            line.find("voluntary_ctxt_switches:") == 0 ||
            line.find("nonvoluntary_ctxt_switches:") == 0) {
            std::cout << line << "\n";
        }
    }
}

// 讀取 CPU 統計
void read_cpu_stats() {
    std::ifstream stat("/proc/stat");
    std::string line;
    
    std::getline(stat, line);  // 第一行是總體 CPU 統計
    std::cout << "CPU: " << line << "\n";
}

// 監控上下文切換
void monitor_context_switches() {
    std::ifstream stat("/proc/" + std::to_string(getpid()) + "/status");
    std::string line;
    
    while (std::getline(stat, line)) {
        if (line.find("voluntary_ctxt_switches:") == 0 ||
            line.find("nonvoluntary_ctxt_switches:") == 0) {
            std::cout << line << "\n";
        }
    }
}
```

### 8.5.3 性能計數器

```cpp
#include <linux/perf_event.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <cstring>
#include <iostream>

class PerfCounter {
private:
    int fd_;
    
    static long perf_event_open(perf_event_attr* attr, pid_t pid, 
                                 int cpu, int group_fd, unsigned long flags) {
        return syscall(__NR_perf_event_open, attr, pid, cpu, group_fd, flags);
    }
    
public:
    PerfCounter(uint32_t type, uint64_t config) {
        perf_event_attr attr{};
        attr.type = type;
        attr.size = sizeof(attr);
        attr.config = config;
        attr.disabled = 1;
        attr.exclude_kernel = 1;
        attr.exclude_hv = 1;
        
        fd_ = perf_event_open(&attr, 0, -1, -1, 0);
        if (fd_ == -1) {
            perror("perf_event_open");
        }
    }
    
    ~PerfCounter() {
        if (fd_ != -1) close(fd_);
    }
    
    void start() {
        ioctl(fd_, PERF_EVENT_IOC_RESET, 0);
        ioctl(fd_, PERF_EVENT_IOC_ENABLE, 0);
    }
    
    uint64_t read() {
        uint64_t count = 0;
        ::read(fd_, &count, sizeof(count));
        return count;
    }
    
    void stop() {
        ioctl(fd_, PERF_EVENT_IOC_DISABLE, 0);
    }
};

// 使用示例
void benchmark_with_counters() {
    PerfCounter cycles(PERF_TYPE_HARDWARE, PERF_COUNT_HW_CPU_CYCLES);
    PerfCounter instructions(PERF_TYPE_HARDWARE, PERF_COUNT_HW_INSTRUCTIONS);
    PerfCounter cache_misses(PERF_TYPE_HARDWARE, PERF_COUNT_HW_CACHE_MISSES);
    
    cycles.start();
    instructions.start();
    cache_misses.start();
    
    // 執行測試代碼
    volatile int sum = 0;
    for (int i = 0; i < 10000000; ++i) {
        sum += i;
    }
    
    cycles.stop();
    instructions.stop();
    cache_misses.stop();
    
    uint64_t c = cycles.read();
    uint64_t i = instructions.read();
    uint64_t m = cache_misses.read();
    
    std::cout << "Cycles: " << c << "\n"
              << "Instructions: " << i << "\n"
              << "IPC: " << static_cast<double>(i) / c << "\n"
              << "Cache misses: " << m << "\n";
}
```

---

## 8.6 系統調優

### 8.6.1 內核參數調優

```bash
# 網路相關
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"
sysctl -w net.core.somaxconn=4096
sysctl -w net.core.netdev_max_backlog=5000

# 內存相關
sysctl -w vm.swappiness=0              # 禁用交換
sysctl -w vm.overcommit_memory=1       # 允許內存超額分配
sysctl -w vm.nr_hugepages=1000         # 大頁
sysctl -w vm.hugetlb_shm_group=1001    # 允許組使用大頁

# CPU 相關
sysctl -w kernel.sched_rt_runtime_us=-1  # 禁用實時調度限制 (謹慎!)

# 文件系統
sysctl -w fs.file-max=2097152          # 最大打開文件數

# 持久化
cat >> /etc/sysctl.conf << EOF
net.core.rmem_max=16777216
net.core.wmem_max=16777216
vm.swappiness=0
vm.nr_hugepages=1000
EOF
```

### 8.6.2 資源限制

```bash
# 查看當前限制
ulimit -a

# 設置無限制 (當前 shell)
ulimit -n 65536    # 打開文件數
ulimit -l unlimited # 鎖定內存
ulimit -s 16384    # 棧大小 (KB)

# 持久化 /etc/security/limits.conf
cat >> /etc/security/limits.conf << EOF
*  soft  nofile  65536
*  hard  nofile  65536
*  soft  memlock unlimited
*  hard  memlock unlimited
EOF
```

### 8.6.3 電源管理調優

```bash
# 設置 CPU 為性能模式 (最高頻率)
for i in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance | sudo tee $i
done

# 禁用 C-states (減少喚醒延遲)
# /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="intel_idle.max_cstate=0 processor.max_cstate=0"

# 更新 grub
sudo update-grub
sudo reboot

# 禁用 turbo boost (穩定頻率,降低抖動)
echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo

# 驗證
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# performance
```

### 8.6.4 IRQ 親和性

```bash
# 查看網卡中斷
cat /proc/interrupts | grep eth0

# 假設 eth0 的 IRQ 為 125
# 綁定到 CPU 1
echo 2 | sudo tee /proc/irq/125/smp_affinity
# (2 的二進制 = 0010 = CPU 1)

# 或使用十六進制掩碼
echo 0x04 | sudo tee /proc/irq/125/smp_affinity  # CPU 2

# 使用 irqbalance 自動平衡 (生產環境)
sudo systemctl start irqbalance
```

---

## 實作練習

### 練習 1: 實時調度測試

**需求:** 對比普通調度與 SCHED_FIFO 的延遲抖動。

```cpp
#include <sched.h>
#include <vector>
#include <algorithm>

void latency_test(bool use_realtime) {
    if (use_realtime) {
        sched_param param{};
        param.sched_priority = 90;
        sched_setscheduler(0, SCHED_FIFO, &param);
        std::cout << "Using SCHED_FIFO\n";
    } else {
        std::cout << "Using SCHED_OTHER\n";
    }
    
    std::vector<double> latencies;
    TSCTimer timer;
    
    for (int i = 0; i < 10000; ++i) {
        timer.start();
        
        // 模擬短操作
        volatile int x = 0;
        for (int j = 0; j < 100; ++j) x++;
        
        latencies.push_back(timer.elapsed_ns());
        
        usleep(100);
    }
    
    std::sort(latencies.begin(), latencies.end());
    size_t n = latencies.size();
    
    std::cout << "P50: " << latencies[n/2] << " ns\n"
              << "P99: " << latencies[n*99/100] << " ns\n"
              << "P99.9: " << latencies[n*999/1000] << " ns\n"
              << "Max: " << latencies[n-1] << " ns\n";
}

int main() {
    std::cout << "== Normal scheduling ==\n";
    latency_test(false);
    
    std::cout << "\n== Realtime scheduling ==\n";
    latency_test(true);
    
    return 0;
}
```

### 練習 2: CPU 親和性性能測試

**需求:** 測量綁定 vs 不綁定 CPU 的緩存命中率。

```cpp
#include <sched.h>
#include <vector>

void cache_test(bool bind_cpu) {
    if (bind_cpu) {
        cpu_set_t cpuset;
        CPU_ZERO(&cpuset);
        CPU_SET(2, &cpuset);
        sched_setaffinity(0, sizeof(cpuset), &cpuset);
        std::cout << "Bound to CPU 2\n";
    } else {
        std::cout << "Not bound\n";
    }
    
    const size_t SIZE = 10 * 1024 * 1024;  // 10M int
    std::vector<int> data(SIZE);
    
    // 填充數據
    for (size_t i = 0; i < SIZE; ++i) {
        data[i] = i;
    }
    
    // 測試: 順序訪問 (cache-friendly)
    volatile long long sum = 0;
    auto start = std::chrono::high_resolution_clock::now();
    
    for (int iter = 0; iter < 10; ++iter) {
        for (size_t i = 0; i < SIZE; ++i) {
            sum += data[i];
        }
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "Time: " << duration.count() << " ms\n";
    std::cout << "Sum: " << sum << "\n";
}

int main() {
    std::cout << "== Without CPU binding ==\n";
    cache_test(false);
    
    std::cout << "\n== With CPU binding ==\n";
    cache_test(true);
    
    // 使用 perf 測量緩存命中率:
    // perf stat -e cache-references,cache-misses ./program
    
    return 0;
}
```

### 練習 3: mlockall 效果測試

**需求:** 對比鎖定 vs 未鎖定內存的缺頁異常次數。

```cpp
#include <sys/mman.h>
#include <vector>

void page_fault_test(bool use_mlock) {
    if (use_mlock) {
        if (mlockall(MCL_CURRENT | MCL_FUTURE) == 0) {
            std::cout << "Memory locked\n";
        } else {
            perror("mlockall failed");
        }
    }
    
    // 記錄初始缺頁異常數
    std::ifstream stat("/proc/self/stat");
    std::string dummy;
    long minflt_before, majflt_before;
    for (int i = 0; i < 9; ++i) stat >> dummy;
    stat >> minflt_before >> dummy >> majflt_before;
    stat.close();
    
    // 分配大量內存並訪問
    const size_t SIZE = 1024 * 1024 * 100;  // 100MB
    std::vector<char> data(SIZE);
    
    for (size_t i = 0; i < SIZE; i += 4096) {
        data[i] = 1;  // 觸發缺頁
    }
    
    // 記錄結束缺頁異常數
    stat.open("/proc/self/stat");
    long minflt_after, majflt_after;
    for (int i = 0; i < 9; ++i) stat >> dummy;
    stat >> minflt_after >> dummy >> majflt_after;
    
    std::cout << "Minor page faults: " << (minflt_after - minflt_before) << "\n";
    std::cout << "Major page faults: " << (majflt_after - majflt_before) << "\n";
    
    if (use_mlock) {
        munlockall();
    }
}

int main() {
    std::cout << "== Without mlockall ==\n";
    page_fault_test(false);
    
    std::cout << "\n== With mlockall ==\n";
    page_fault_test(true);
    
    return 0;
}
```

---

## 關鍵要點總結

### HFT 系統調優檢查清單

**進程與調度:**
- [ ] 設置 SCHED_FIFO 高優先級 (90-99)
- [ ] 綁定關鍵線程到專用 CPU
- [ ] 配置 isolcpus 隔離 CPU

**內存:**
- [ ] mlockall() 鎖定所有內存
- [ ] 配置 Huge Pages (2MB 或 1GB)
- [ ] 預分配堆和棧
- [ ] swappiness = 0

**網路:**
- [ ] TCP_NODELAY
- [ ] SO_BUSY_POLL
- [ ] IRQ 親和性
- [ ] 大緩衝區

**系統:**
- [ ] CPU 性能模式 (禁用節能)
- [ ] 禁用 C-states
- [ ] 調整資源限制 (ulimit)
- [ ] 內核參數優化 (sysctl)

### 性能分析工具

| 工具 | 用途 | 示例命令 |
|------|------|----------|
| perf | CPU 性能分析 | `perf stat -e cache-misses ./app` |
| top/htop | 實時監控 | `htop` |
| /proc | 系統狀態 | `cat /proc/cpuinfo` |
| strace | 系統調用追蹤 | `strace -c ./app` |
| numactl | NUMA 控制 | `numactl --cpunodebind=0 ./app` |

### 延遲優化效果預估

| 優化項目 | 延遲降低 | 實施難度 |
|----------|----------|----------|
| SCHED_FIFO | 10-50μs | 低 |
| CPU 綁定 | 5-20μs | 低 |
| mlockall | 10-100μs (避免 swap) | 低 |
| Huge Pages | 5-15% | 中 |
| CPU 隔離 | 20-100μs (抖動) | 中 |
| NUMA 優化 | 10-30% | 高 |

---

## 參考資料 (References)

1. Linux man pages:
   - [sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html)
   - [mlock(2)](https://man7.org/linux/man-pages/man2/mlock.2.html)
   - [numa(7)](https://man7.org/linux/man-pages/man7/numa.7.html)
2. [Linux Performance](https://www.brendangregg.com/linuxperf.html) by Brendan Gregg
3. [Red Hat Performance Tuning Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/index)
4. [Realtime Linux](https://wiki.linuxfoundation.org/realtime/start)
5. [NUMA Best Practices](https://queue.acm.org/detail.cfm?id=2513149)
6. [perf Examples](http://www.brendangregg.com/perf.html)
7. Robert Love, *Linux System Programming*, 2nd Edition (2013)
8. [Low Latency Tuning Guide](https://rigtorp.se/low-latency-guide/) by Erik Rigtorp
