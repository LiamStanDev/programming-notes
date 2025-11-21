# Linux 系統編程與調優

> 本章涵蓋 Linux 系統編程、進程管理、實時調度、內存管理、性能監控等,是構建穩定高性能交易系統的基礎。

---

## 目錄

> **HFT 學習優先級**: ⭐⭐⭐ 必看 | ⭐⭐ 建議 | ⭐ 有空再看

1. [進程管理](#1-進程管理) ⭐
2. [實時調度](#2-實時調度) ⭐⭐⭐
3. [CPU 親和性與隔離](#3-cpu-親和性與隔離) ⭐⭐⭐
4. [內存管理](#4-內存管理) ⭐⭐⭐
5. [性能監控](#5-性能監控) ⭐⭐⭐
6. [系統調優](#6-系統調優) ⭐⭐⭐

---

## 1. 進程管理

### 1.0 概念解析: 進程 vs 線程

**進程 (Process)** 和 **線程 (Thread)** 是操作系統執行任務的基本單位。

| 特性         | 進程 (Process)             | 線程 (Thread)             |
| :----------- | :------------------------- | :------------------------ |
| **定義**     | 資源分配的最小單位 (工廠)  | CPU 調度的最小單位 (工人) |
| **內存**     | 獨立內存空間 (互不干擾)    | 共享進程內存 (方便通信)   |
| **開銷**     | 大 (創建/切換慢)           | 小 (創建/切換快)          |
| **通信**     | 困難 (IPC: 管道, 共享內存) | 容易 (直接讀寫變量)       |
| **穩定性**   | 高 (一個崩潰不影響其他)    | 低 (一個崩潰導致進程崩潰) |
| **HFT 應用** | 策略隔離, 風險控制         | 處理高頻數據, 訂單發送    |

### 1.1 概念解析: 用戶態 vs 內核態

操作系統為了安全,將權限分為兩個等級:

1.  **用戶態 (User Space)**: 應用程序運行的地方。權限受限,不能直接訪問硬件。
2.  **內核態 (Kernel Space)**: 操作系統核心運行的地方。擁有最高權限,可訪問硬件 (網卡, 硬盤)。

**系統調用 (System Call)**: 當應用程序需要訪問硬件(如讀寫文件, 發送網絡包)時,必須從用戶態切換到內核態,這叫系統調用。
**HFT 的目標**: 盡量減少系統調用,因為上下文切換 (Context Switch) 非常耗時 (約 1-2 微秒)。

### 1.2 fork 與 exec

```cpp
#include <unistd.h>
#include <sys/wait.h>
#include <iostream>

void fork_example() {
    pid_t pid = fork();

    if (pid < 0) {
        std::cerr << "Fork failed\n";
        return;
    }

    if (pid == 0) {
        // 子進程
        std::cout << "Child process, PID: " << getpid() << "\n";

        // 執行新程序
        execl("/bin/ls", "ls", "-l", nullptr);

        // 如果 exec 失敗才會執行到這裡
        std::cerr << "exec failed\n";
        exit(1);
    } else {
        // 父進程
        std::cout << "Parent process, PID: " << getpid() << "\n";
        std::cout << "Child PID: " << pid << "\n";

        // 等待子進程結束
        int status;
        waitpid(pid, &status, 0);

        if (WIFEXITED(status)) {
            std::cout << "Child exited with status: " << WEXITSTATUS(status) << "\n";
        }
    }
}
```

### 1.2 進程間通信 (IPC) - 管道

```cpp
#include <unistd.h>
#include <cstring>
#include <iostream>

void pipe_example() {
    int pipefd[2];

    if (pipe(pipefd) < 0) {
        std::cerr << "Pipe failed\n";
        return;
    }

    pid_t pid = fork();

    if (pid == 0) {
        // 子進程 - 讀取
        close(pipefd[1]);  // 關閉寫端

        char buffer[100];
        ssize_t n = read(pipefd[0], buffer, sizeof(buffer) - 1);
        if (n > 0) {
            buffer[n] = '\0';
            std::cout << "Child received: " << buffer << "\n";
        }

        close(pipefd[0]);
        exit(0);
    } else {
        // 父進程 - 寫入
        close(pipefd[0]);  // 關閉讀端

        const char* message = "Hello from parent!";
        write(pipefd[1], message, std::strlen(message));

        close(pipefd[1]);
        wait(nullptr);
    }
}
```

### 1.3 共享內存 (Shared Memory)

```cpp
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <cstring>
#include <iostream>

struct SharedData {
    int counter;
    char message[256];
};

void shared_memory_writer() {
    const char* name = "/my_shm";

    // 創建共享內存
    int shm_fd = shm_open(name, O_CREAT | O_RDWR, 0666);
    if (shm_fd < 0) {
        std::cerr << "shm_open failed\n";
        return;
    }

    // 設置大小
    ftruncate(shm_fd, sizeof(SharedData));

    // 映射到內存
    void* ptr = mmap(nullptr, sizeof(SharedData), PROT_READ | PROT_WRITE,
                     MAP_SHARED, shm_fd, 0);
    if (ptr == MAP_FAILED) {
        std::cerr << "mmap failed\n";
        return;
    }

    SharedData* data = static_cast<SharedData*>(ptr);
    data->counter = 42;
    std::strcpy(data->message, "Hello from writer!");

    std::cout << "Data written to shared memory\n";

    munmap(ptr, sizeof(SharedData));
    close(shm_fd);
}

void shared_memory_reader() {
    const char* name = "/my_shm";

    // 打開共享內存
    int shm_fd = shm_open(name, O_RDONLY, 0666);
    if (shm_fd < 0) {
        std::cerr << "shm_open failed\n";
        return;
    }

    // 映射到內存
    void* ptr = mmap(nullptr, sizeof(SharedData), PROT_READ,
                     MAP_SHARED, shm_fd, 0);
    if (ptr == MAP_FAILED) {
        std::cerr << "mmap failed\n";
        return;
    }

    SharedData* data = static_cast<SharedData*>(ptr);
    std::cout << "Counter: " << data->counter << "\n";
    std::cout << "Message: " << data->message << "\n";

    munmap(ptr, sizeof(SharedData));
    close(shm_fd);

    // 清理
    shm_unlink(name);
}
```

---

## 2. 實時調度

### 2.0 概念解析: 什麼是實時 (Real-time)?

在 HFT 中,**實時**並不單純指"快",而是指**確定性 (Determinism)**。

- **非實時**: 通常很快,但偶爾會卡頓 100ms (例如 GC, 系統調度)。
- **實時**: 保證在規定時間內 (例如 10us) 必須完成響應,絕不允許不可預測的延遲。

我們使用 **SCHED_FIFO** 調度策略來告訴操作系統: "這個線程非常重要,除非它自己主動讓出 CPU,否則絕對不要打斷它!"

### 2.1 調度策略

Linux 支持多種調度策略:

- **SCHED_OTHER**: 默認策略,時間片輪轉
- **SCHED_FIFO**: 實時先進先出
- **SCHED_RR**: 實時輪轉
- **SCHED_DEADLINE**: 截止時間調度 (Deadline Scheduling)

### 2.2 設置實時優先級

```cpp
#include <sched.h>
#include <pthread.h>
#include <iostream>

bool set_realtime_priority(int priority) {
    struct sched_param param;
    param.sched_priority = priority;  // 1-99

    if (sched_setscheduler(0, SCHED_FIFO, &param) < 0) {
        std::cerr << "Failed to set SCHED_FIFO\n";
        return false;
    }

    std::cout << "Set SCHED_FIFO with priority " << priority << "\n";
    return true;
}

void realtime_thread_example() {
    std::thread t([]() {
        // 設置線程為實時優先級
        struct sched_param param;
        param.sched_priority = 80;

        pthread_t thread = pthread_self();
        if (pthread_setschedparam(thread, SCHED_FIFO, &param) == 0) {
            std::cout << "Thread set to SCHED_FIFO priority 80\n";
        }

        // 執行關鍵任務
        while (true) {
            // ...
        }
    });

    t.detach();
}
```

### 2.3 查看調度信息

```cpp
#include <sched.h>
#include <iostream>

void print_scheduling_info() {
    int policy = sched_getscheduler(0);

    const char* policy_name;
    switch (policy) {
        case SCHED_OTHER: policy_name = "SCHED_OTHER"; break;
        case SCHED_FIFO: policy_name = "SCHED_FIFO"; break;
        case SCHED_RR: policy_name = "SCHED_RR"; break;
        default: policy_name = "UNKNOWN"; break;
    }

    std::cout << "Current policy: " << policy_name << "\n";

    struct sched_param param;
    sched_getparam(0, &param);
    std::cout << "Priority: " << param.sched_priority << "\n";

    // 獲取優先級範圍
    int min_prio = sched_get_priority_min(SCHED_FIFO);
    int max_prio = sched_get_priority_max(SCHED_FIFO);
    std::cout << "SCHED_FIFO priority range: " << min_prio << "-" << max_prio << "\n";
}
```

---

## 3. CPU 親和性與隔離

### 3.1 設置 CPU 親和性

```cpp
#include <sched.h>
#include <pthread.h>
#include <iostream>

bool set_cpu_affinity(int cpu_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);

    if (sched_setaffinity(0, sizeof(cpu_set_t), &cpuset) < 0) {
        std::cerr << "Failed to set CPU affinity\n";
        return false;
    }

    std::cout << "Pinned to CPU " << cpu_id << "\n";
    return true;
}

bool set_thread_affinity(pthread_t thread, int cpu_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);

    if (pthread_setaffinity_np(thread, sizeof(cpu_set_t), &cpuset) != 0) {
        std::cerr << "Failed to set thread affinity\n";
        return false;
    }

    std::cout << "Thread pinned to CPU " << cpu_id << "\n";
    return true;
}

void affinity_example() {
    std::thread t([]() {
        std::cout << "Thread running\n";
        // 執行任務...
    });

    // 綁定到 CPU 2
    set_thread_affinity(t.native_handle(), 2);

    t.join();
}
```

### 3.2 CPU 隔離

```bash
# 查看 CPU 信息
lscpu

# 隔離 CPU 2-3 (在 GRUB 配置中)
# 編輯 /etc/default/grub
GRUB_CMDLINE_LINUX="isolcpus=2,3"

# 更新 GRUB
sudo update-grub
sudo reboot

# 驗證隔離
cat /sys/devices/system/cpu/isolated
```

### 3.3 HFT 線程綁定策略

```cpp
#include <thread>
#include <vector>

class HFTSystem {
public:
    void start() {
        // 市場數據接收線程 - CPU 2
        market_data_thread_ = std::thread([this]() {
            set_thread_affinity(pthread_self(), 2);
            set_realtime_priority(90);
            receive_market_data();
        });

        // 策略引擎線程 - CPU 3
        strategy_thread_ = std::thread([this]() {
            set_thread_affinity(pthread_self(), 3);
            set_realtime_priority(85);
            run_strategy();
        });

        // 訂單發送線程 - CPU 4
        order_thread_ = std::thread([this]() {
            set_thread_affinity(pthread_self(), 4);
            set_realtime_priority(80);
            send_orders();
        });
    }

private:
    std::thread market_data_thread_;
    std::thread strategy_thread_;
    std::thread order_thread_;

    void receive_market_data() { /* ... */ }
    void run_strategy() { /* ... */ }
    void send_orders() { /* ... */ }
};
```

---

## 4. 內存管理

### 4.0 概念解析: 虛擬內存與缺頁中斷

- **虛擬內存**: 程序看到的內存地址是虛擬的,操作系統負責將其映射到物理內存。
- **缺頁中斷 (Page Fault)**: 當程序訪問一個還沒映射到物理內存的虛擬地址時,CPU 會觸發中斷,操作系統暫停程序,去分配物理內存。這會產生巨大的延遲 (微秒級甚至毫秒級)。

**HFT 解決方案**: 使用 `mlock` 鎖定內存,並預先訪問一遍 (Pre-faulting),確保所有內存都在物理內存中,永遠不會觸發缺頁中斷。

### 4.1 鎖定內存 (mlock)

```cpp
#include <sys/mman.h>
#include <iostream>

bool lock_memory() {
    // 鎖定當前和未來的所有內存頁
    if (mlockall(MCL_CURRENT | MCL_FUTURE) < 0) {
        std::cerr << "mlockall failed\n";
        return false;
    }

    std::cout << "Memory locked\n";
    return true;
}

void unlock_memory() {
    munlockall();
    std::cout << "Memory unlocked\n";
}

void mlock_example() {
    lock_memory();

    // 分配內存 (不會被 swap 出去)
    const size_t size = 1024 * 1024;  // 1MB
    char* buffer = new char[size];

    // 使用內存...

    delete[] buffer;
    unlock_memory();
}
```

### 4.2 Huge Pages

```bash
# 查看 Huge Pages 配置
cat /proc/meminfo | grep Huge

# 配置 Huge Pages (2MB 頁)
sudo sysctl -w vm.nr_hugepages=1024

# 永久配置
echo "vm.nr_hugepages=1024" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

```cpp
#include <sys/mman.h>
#include <iostream>

void* allocate_huge_pages(size_t size) {
    void* addr = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);

    if (addr == MAP_FAILED) {
        std::cerr << "Failed to allocate huge pages\n";
        return nullptr;
    }

    std::cout << "Allocated " << size << " bytes using huge pages\n";
    return addr;
}

void hugepage_example() {
    const size_t size = 2 * 1024 * 1024;  // 2MB
    void* buffer = allocate_huge_pages(size);

    if (buffer) {
        // 使用內存...
        munmap(buffer, size);
    }
}
```

### 4.3 內存預分配

```cpp
#include <vector>
#include <iostream>

template<typename T>
class PreallocatedPool {
public:
    PreallocatedPool(size_t capacity) {
        pool_.reserve(capacity);
        for (size_t i = 0; i < capacity; ++i) {
            pool_.emplace_back();
        }
        std::cout << "Preallocated " << capacity << " objects\n";
    }

    T* acquire() {
        if (pool_.empty()) return nullptr;

        T* obj = &pool_.back();
        pool_.pop_back();
        return obj;
    }

    void release(T* obj) {
        pool_.push_back(*obj);
    }

private:
    std::vector<T> pool_;
};

struct Order {
    int id;
    double price;
    int quantity;
};

void pool_example() {
    PreallocatedPool<Order> pool(10000);

    // 快速獲取對象 (無動態分配)
    Order* order = pool.acquire();
    if (order) {
        order->id = 1;
        order->price = 100.0;
        order->quantity = 10;

        // 使用訂單...

        pool.release(order);
    }
}
```

---

## 5. 性能監控

### 5.1 perf 工具

```bash
# 記錄性能數據
sudo perf record -g ./trading_system

# 查看報告
sudo perf report

# CPU 週期分析
sudo perf stat ./trading_system

# 實時監控
sudo perf top

# 緩存未命中分析
sudo perf stat -e cache-misses,cache-references ./trading_system
```

### 5.2 /proc 文件系統

```cpp
#include <fstream>
#include <iostream>
#include <string>

void read_proc_stat() {
    std::ifstream stat_file("/proc/self/stat");

    std::string comm;
    char state;
    int ppid, pgrp, session, tty_nr, tpgid;
    unsigned long flags, minflt, cminflt, majflt, cmajflt;
    unsigned long utime, stime;
    long cutime, cstime, priority, nice, num_threads;

    stat_file >> comm >> comm >> state >> ppid >> pgrp >> session >> tty_nr
              >> tpgid >> flags >> minflt >> cminflt >> majflt >> cmajflt
              >> utime >> stime >> cutime >> cstime >> priority >> nice
              >> num_threads;

    std::cout << "Process state: " << state << "\n";
    std::cout << "User time: " << utime << "\n";
    std::cout << "System time: " << stime << "\n";
    std::cout << "Priority: " << priority << "\n";
    std::cout << "Nice: " << nice << "\n";
    std::cout << "Threads: " << num_threads << "\n";
}

void read_proc_status() {
    std::ifstream status_file("/proc/self/status");
    std::string line;

    while (std::getline(status_file, line)) {
        if (line.find("VmRSS:") == 0 || line.find("VmSize:") == 0) {
            std::cout << line << "\n";
        }
    }
}
```

### 5.3 系統調用追蹤

```bash
# 追蹤系統調用
strace ./trading_system

# 統計系統調用
strace -c ./trading_system

# 追蹤特定系統調用
strace -e open,read,write ./trading_system

# 追蹤運行中的進程
sudo strace -p <PID>
```

---

## 6. 系統調優

### 6.1 中斷親和性 (IRQ Affinity)

```bash
# 查看網卡中斷
cat /proc/interrupts | grep eth0

# 設置中斷親和性到 CPU 0
echo 1 | sudo tee /proc/irq/<IRQ_NUMBER>/smp_affinity

# 使用 irqbalance 自動平衡
sudo systemctl start irqbalance
```

### 6.2 關閉不必要的服務

```bash
# 查看運行的服務
systemctl list-units --type=service --state=running

# 關閉不必要的服務
sudo systemctl stop bluetooth
sudo systemctl disable bluetooth

# 關閉圖形界面 (服務器)
sudo systemctl set-default multi-user.target
```

### 6.3 內核參數調優

```bash
# 編輯 /etc/sysctl.conf

# 網路優化
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864

# 減少 swap
vm.swappiness = 1

# 文件描述符限制
fs.file-max = 2097152

# 應用配置
sudo sysctl -p
```

### 6.4 HFT 系統完整調優腳本

```bash
#!/bin/bash

# HFT 系統調優腳本

# 1. CPU 隔離 (需要重啟)
# 編輯 /etc/default/grub
# GRUB_CMDLINE_LINUX="isolcpus=2-7 nohz_full=2-7 rcu_nocbs=2-7"

# 2. 實時內核 (可選)
# sudo apt install linux-image-rt-amd64

# 3. 關閉 CPU 頻率調節
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance | sudo tee $cpu
done

# 4. 關閉透明大頁
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled

# 5. 配置 Huge Pages
echo 1024 | sudo tee /proc/sys/vm/nr_hugepages

# 6. 網路調優
sudo sysctl -w net.core.rmem_max=134217728
sudo sysctl -w net.core.wmem_max=134217728
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 67108864"
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 67108864"

# 7. 減少 swap
sudo sysctl -w vm.swappiness=1

# 8. 文件描述符
sudo sysctl -w fs.file-max=2097152

echo "System tuned for HFT"
```

---

## 總結

本章涵蓋了 Linux 系統編程與調優的核心技術:

1. **進程管理**: fork、exec、IPC
2. **實時調度**: SCHED_FIFO、優先級設置
3. **CPU 親和性**: 線程綁定、CPU 隔離
4. **內存管理**: mlock、Huge Pages、內存池
5. **性能監控**: perf、/proc、strace
6. **系統調優**: 中斷親和性、內核參數

**HFT 系統調優檢查清單:**

- [ ] 隔離 CPU 核心
- [ ] 設置實時調度策略
- [ ] 綁定關鍵線程到獨立 CPU
- [ ] 鎖定內存 (mlockall)
- [ ] 配置 Huge Pages
- [ ] 調整網路參數
- [ ] 關閉不必要的服務
- [ ] 設置中斷親和性
- [ ] 關閉 CPU 頻率調節
- [ ] 測量並優化延遲

**性能提升總結:**

| 優化技術   | 延遲改善 | 穩定性 | 複雜度 | HFT 推薦 |
| ---------- | -------- | ------ | ------ | -------- |
| CPU 隔離   | 30-50%   | 高     | 中等   | 必須     |
| 實時調度   | 20-40%   | 高     | 低     | 必須     |
| mlock      | 10-20%   | 高     | 低     | 必須     |
| Huge Pages | 5-15%    | 中等   | 中等   | 推薦     |
| 中斷親和性 | 10-30%   | 中等   | 中等   | 推薦     |

---

## 參考資料 (References)

1. [Linux Programming Interface](http://man7.org/tlpi/) - Michael Kerrisk
2. [Real-Time Linux Wiki](https://rt.wiki.kernel.org/)
3. [perf Documentation](https://perf.wiki.kernel.org/)
4. [Linux Performance](http://www.brendangregg.com/linuxperf.html) - Brendan Gregg
5. [SCHED_DEADLINE Documentation](https://www.kernel.org/doc/Documentation/scheduler/sched-deadline.txt)
