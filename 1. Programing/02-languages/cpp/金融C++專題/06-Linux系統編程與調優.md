# Linux 系統編程與調優

> 本章涵蓋 Linux 系統編程、底層 I/O、進程管理、實時調度、內存管理、性能監控等,是構建穩定高性能交易系統的基礎。

---

## 目錄

> **HFT 學習優先級**: ⭐⭐⭐ 必看 | ⭐⭐ 建議 | ⭐ 有空再看

1. [系統呼叫概述](#1-系統呼叫概述) ⭐⭐
2. [文件描述符與底層 I/O](#2-文件描述符與底層-io) ⭐⭐
3. [進階文件操作](#3-進階文件操作) ⭐⭐⭐
4. [進程管理](#4-進程管理) ⭐
5. [實時調度](#5-實時調度) ⭐⭐⭐
6. [CPU 親和性與隔離](#6-cpu-親和性與隔離) ⭐⭐⭐
7. [內存管理](#7-內存管理) ⭐⭐⭐
8. [信號與計時器](#8-信號與計時器) ⭐⭐
9. [性能監控](#9-性能監控) ⭐⭐⭐
10. [系統調優](#10-系統調優) ⭐⭐⭐

---

## 1. 系統呼叫概述

### 1.1 用戶態 vs 內核態

操作系統為了安全和穩定性,將 CPU 的運行權限分為兩個等級:

1.  **用戶態 (User Space / User Mode)**:
    - 應用程序運行的地方
    - 權限受限,不能直接訪問硬件或內核數據結構
    - 只能訪問分配給該進程的內存空間
    - 執行特權指令 (如 `IN/OUT`, `CLI/STI`) 會觸發異常

2.  **內核態 (Kernel Space / Kernel Mode)**:
    - 操作系統核心運行的地方
    - 擁有最高權限 (Ring 0),可訪問所有硬件和內存
    - 負責管理進程、內存、文件系統、網路等

```
┌─────────────────────────────────────┐
│         User Space (Ring 3)         │
│  ┌─────┐  ┌─────┐  ┌─────┐         │
│  │App A│  │App B│  │App C│  ...    │
│  └──┬──┘  └──┬──┘  └──┬──┘         │
└─────┼────────┼────────┼─────────────┘
      │        │        │  System Call
      ▼        ▼        ▼
┌─────────────────────────────────────┐
│        Kernel Space (Ring 0)        │
│  ┌──────────────────────────────┐   │
│  │  Process Mgmt | Memory Mgmt  │   │
│  │  File System  | Network      │   │
│  └──────────────────────────────┘   │
│           │                         │
│           ▼                         │
│  ┌──────────────────────────────┐   │
│  │   Hardware (CPU, Disk, NIC)  │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
```

**系統調用 (System Call)**: 應用程序請求內核服務的唯一合法方式。當應用程序需要執行特權操作 (如讀寫文件、發送網絡包、分配內存) 時,必須通過系統調用從用戶態切換到內核態。

**上下文切換 (Context Switch)**: 從用戶態切換到內核態時,CPU 需要:
1. 保存用戶態的寄存器狀態
2. 切換到內核棧
3. 執行內核代碼
4. 恢復用戶態狀態並返回

這個過程稱為上下文切換,是系統調用開銷的主要來源。

### 1.2 系統呼叫的開銷

系統呼叫涉及上下文切換,開銷約 **100-1000 CPU cycles** (約 50ns-500ns)。

**開銷來源分析**:

| 階段 | 操作 | 開銷 |
|------|------|------|
| 1. 進入內核 | 執行 `syscall` 指令,切換到 Ring 0 | ~20-50 cycles |
| 2. 保存狀態 | 保存用戶態寄存器到內核棧 | ~30-50 cycles |
| 3. 參數驗證 | 檢查指針有效性、權限等 | ~20-100 cycles |
| 4. 執行操作 | 實際的內核操作 | 依操作而異 |
| 5. 恢復狀態 | 從內核棧恢復寄存器 | ~30-50 cycles |
| 6. 返回用戶態 | 執行 `sysret` 指令 | ~20-50 cycles |

**額外開銷**:
- **TLB (Translation Lookaside Buffer) 刷新**: 頁表切換可能導致 TLB 失效
- **CPU 流水線 (Pipeline) 清空**: 特權級切換會清空流水線
- **緩存 (Cache) 污染**: 內核代碼可能驅逐用戶態熱點數據

```cpp
#include <unistd.h>
#include <fcntl.h>
#include <chrono>
#include <iostream>

void measure_syscall_overhead() {
    char buf[1];
    int fd = open("/dev/zero", O_RDONLY);
    
    auto start = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < 100000; ++i) {
        read(fd, buf, 1);  // 每次都是系統呼叫
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);
    
    std::cout << "Average syscall overhead: " 
              << duration.count() / 100000 << " ns\n";
    
    close(fd);
}
```

**HFT 優化策略**:

1. **批量操作**: 使用 `readv()/writev()` 代替多次 `read()/write()`
2. **記憶體映射**: 使用 `mmap()` 避免 `read()/write()` 系統呼叫
3. **非阻塞 I/O**: 結合 `epoll` 減少無效的系統呼叫
4. **用戶態處理**: 盡量在用戶態完成計算,減少進入內核態

### 1.3 常用系統呼叫分類

| 分類 | 系統呼叫 | 用途 | HFT 優先級 |
|------|----------|------|------------|
| **文件基礎** | open, read, write, close | 基本文件操作 | ⭐⭐ |
| **文件進階** | pread, pwrite, lseek | 定位讀寫 | ⭐⭐ |
| **分散/聚集** | readv, writev | 多緩衝區 I/O | ⭐⭐⭐ |
| **記憶體映射** | mmap, munmap, msync | 零拷貝, 共享記憶體 | ⭐⭐⭐ |
| **同步控制** | fsync, fdatasync | 數據持久化 | ⭐⭐ |
| **FD 控制** | fcntl, dup2 | 非阻塞, 重定向 | ⭐⭐⭐ |
| **元數據** | stat, fstat | 文件資訊 | ⭐⭐ |
| **進程** | fork, exec, wait, clone | 進程管理 | ⭐⭐ |
| **信號** | sigaction, signalfd | 信號處理 | ⭐⭐ |
| **時間** | clock_gettime, timerfd | 計時與定時 | ⭐⭐⭐ |
| **網路** | socket, bind, listen, accept | 網路通信 | ⭐⭐⭐ |
| **I/O 多工** | epoll_create, epoll_ctl, epoll_wait | 高並發 I/O | ⭐⭐⭐ |

---

## 2. 文件描述符與底層 I/O

### 2.1 文件描述符 (File Descriptor)

**文件描述符 (File Descriptor, FD)** 是一個非負整數,是進程訪問文件、管道、Socket 等 I/O 資源的抽象句柄。

**核心概念**:

在 Unix/Linux 中,"一切皆文件" (Everything is a file)。不僅普通文件,還有:
- **管道 (Pipe)**: 進程間通信
- **Socket**: 網路通信
- **設備文件**: `/dev/null`, `/dev/zero`, `/dev/sda`
- **偽文件系統**: `/proc`, `/sys`

都通過統一的文件描述符接口來訪問。

**內核數據結構**:

```
進程 A                          內核
┌──────────────┐               ┌─────────────────────────┐
│ FD Table     │               │  Open File Table        │
│ ┌───┬─────┐  │               │  ┌────┬───────────────┐ │
│ │ 0 │  ──────┼───────────────┼──┤    │ stdin         │ │
│ ├───┼─────┤  │               │  ├────┼───────────────┤ │
│ │ 1 │  ──────┼───────────────┼──┤    │ stdout        │ │
│ ├───┼─────┤  │               │  ├────┼───────────────┤ │
│ │ 2 │  ──────┼───────────────┼──┤    │ stderr        │ │
│ ├───┼─────┤  │               │  ├────┼───────────────┤ │
│ │ 3 │  ──────┼───────────────┼──┤    │ /tmp/test.txt │ │
│ └───┴─────┘  │               │  │    │ offset: 100   │ │
└──────────────┘               │  │    │ flags: O_RDWR │ │
                               │  └────┴───────────────┘ │
                               └─────────────────────────┘
```

- **FD Table (每進程)**: 每個進程維護自己的 FD 表,將整數索引映射到內核的 Open File Table
- **Open File Table (全局)**: 內核維護,包含文件偏移量 (offset)、打開標誌 (flags)、指向 inode 的指針
- **inode Table**: 文件的元數據 (大小、權限、磁盤位置等)

**為什麼 FD 從 3 開始?**

因為 0, 1, 2 已被預留:
- **0 (STDIN_FILENO)**: 標準輸入
- **1 (STDOUT_FILENO)**: 標準輸出
- **2 (STDERR_FILENO)**: 標準錯誤

```cpp
#include <unistd.h>
#include <fcntl.h>
#include <iostream>

void fd_basics() {
    // 標準文件描述符
    // 0 = stdin  (標準輸入)
    // 1 = stdout (標準輸出)
    // 2 = stderr (標準錯誤)
    
    // 打開文件會分配新的 FD (通常是最小的可用整數)
    int fd = open("/tmp/test.txt", O_CREAT | O_RDWR, 0644);
    if (fd < 0) {
        perror("open failed");
        return;
    }
    
    std::cout << "Allocated FD: " << fd << "\n";  // 通常是 3
    
    close(fd);
}
```

### 2.2 open() 與 close()

**open() 系統呼叫** 是打開或創建文件的標準方式,返回一個文件描述符供後續操作使用。

**函數原型**:
```cpp
#include <fcntl.h>
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);  // 創建文件時使用
```

**打開標誌 (Flags) 詳解**:

| 標誌 | 說明 | HFT 應用 |
|------|------|----------|
| `O_RDONLY` | 只讀打開 | 讀取配置、歷史數據 |
| `O_WRONLY` | 只寫打開 | 寫入日誌 |
| `O_RDWR` | 讀寫打開 | 共享內存、環形緩衝區 |
| `O_CREAT` | 文件不存在則創建 | 創建日誌文件 |
| `O_TRUNC` | 截斷為零長度 | 清空舊日誌 |
| `O_APPEND` | 追加模式,寫入時自動定位到末尾 | 多進程寫同一日誌 |
| `O_NONBLOCK` | 非阻塞模式 | 避免 I/O 阻塞 |
| `O_DIRECT` | 直接 I/O,繞過頁緩存 | 避免緩存污染 |
| `O_SYNC` | 同步寫入,每次 write 都刷盤 | 確保數據持久化 |
| `O_CLOEXEC` | exec 時自動關閉 | 安全性 |

**創建模式 (Mode)**: 當使用 `O_CREAT` 時,需要指定文件權限,通常用八進制表示:
- `0644`: 所有者讀寫,其他人只讀 (常用於普通文件)
- `0755`: 所有者讀寫執行,其他人讀執行 (常用於可執行文件)
- `0600`: 只有所有者可讀寫 (敏感文件)

```cpp
#include <fcntl.h>
#include <unistd.h>
#include <iostream>
#include <cstring>

void open_flags_example() {
    // 常用打開標誌
    // O_RDONLY   - 只讀
    // O_WRONLY   - 只寫
    // O_RDWR     - 讀寫
    // O_CREAT    - 不存在則創建
    // O_TRUNC    - 截斷為零長度
    // O_APPEND   - 追加寫入
    // O_NONBLOCK - 非阻塞模式
    // O_DIRECT   - 直接 I/O (繞過緩存)
    // O_SYNC     - 同步寫入
    
    // 創建新文件 (如果存在則截斷)
    int fd = open("/tmp/test.txt", O_CREAT | O_RDWR | O_TRUNC, 0644);
    if (fd < 0) {
        perror("open");
        return;
    }
    
    // 寫入數據
    const char* data = "Hello, Linux!\n";
    write(fd, data, strlen(data));
    
    close(fd);
    
    // 以追加模式打開
    fd = open("/tmp/test.txt", O_WRONLY | O_APPEND);
    if (fd >= 0) {
        const char* more = "More data\n";
        write(fd, more, strlen(more));
        close(fd);
    }
}

// 錯誤處理最佳實踐
int safe_open(const char* path, int flags, mode_t mode) {
    int fd = open(path, flags, mode);
    if (fd < 0) {
        // errno 包含錯誤碼
        std::cerr << "Failed to open " << path << ": " 
                  << strerror(errno) << "\n";
    }
    return fd;
}
```

### 2.3 read() 與 write()

**read() 和 write()** 是最基本的 I/O 系統呼叫,直接與內核交互進行數據傳輸。

**函數原型**:
```cpp
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

**返回值說明**:
- **> 0**: 實際讀取/寫入的字節數 (可能少於請求的 count)
- **= 0**: 對於 read 表示 EOF (End of File);對於 write 通常不會發生
- **< 0**: 錯誤,具體錯誤碼存在 `errno` 中

**短讀/短寫 (Short Read/Write)**:

系統呼叫可能返回比請求更少的字節數,這是正常行為,原因包括:
- 讀取時到達文件末尾
- 從終端或網路 Socket 讀取
- 被信號中斷 (`EINTR`)
- 內核緩衝區不足

**常見錯誤碼 (errno)**:

| 錯誤碼 | 說明 | 處理方式 |
|--------|------|----------|
| `EINTR` | 被信號中斷 | 重試操作 |
| `EAGAIN` / `EWOULDBLOCK` | 非阻塞模式下無數據可讀 | 稍後重試或使用 epoll |
| `EBADF` | 無效的文件描述符 | 檢查 FD 是否已關閉 |
| `EFAULT` | 緩衝區地址無效 | 檢查指針 |
| `EIO` | I/O 錯誤 | 硬件或文件系統問題 |

```cpp
#include <unistd.h>
#include <fcntl.h>
#include <cerrno>
#include <iostream>

// 處理部分讀取 (Short Read)
ssize_t read_all(int fd, void* buf, size_t count) {
    char* ptr = static_cast<char*>(buf);
    size_t remaining = count;
    
    while (remaining > 0) {
        ssize_t n = read(fd, ptr, remaining);
        
        if (n < 0) {
            if (errno == EINTR) continue;  // 被信號中斷,重試
            return -1;  // 錯誤
        }
        
        if (n == 0) break;  // EOF
        
        ptr += n;
        remaining -= n;
    }
    
    return count - remaining;
}

// 處理部分寫入 (Short Write)
ssize_t write_all(int fd, const void* buf, size_t count) {
    const char* ptr = static_cast<const char*>(buf);
    size_t remaining = count;
    
    while (remaining > 0) {
        ssize_t n = write(fd, ptr, remaining);
        
        if (n < 0) {
            if (errno == EINTR) continue;  // 被信號中斷,重試
            return -1;  // 錯誤
        }
        
        ptr += n;
        remaining -= n;
    }
    
    return count;
}

void read_write_example() {
    int fd = open("/tmp/test.txt", O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) return;
    
    // 寫入
    const char* message = "Test message for read/write demo";
    write_all(fd, message, strlen(message));
    
    // 重置文件位置到開頭
    lseek(fd, 0, SEEK_SET);
    
    // 讀取
    char buffer[100];
    ssize_t n = read_all(fd, buffer, sizeof(buffer) - 1);
    if (n > 0) {
        buffer[n] = '\0';
        std::cout << "Read: " << buffer << "\n";
    }
    
    close(fd);
}
```

### 2.4 lseek() - 文件定位

**lseek()** 用於改變文件的當前讀寫位置 (偏移量)。

**函數原型**:
```cpp
#include <unistd.h>
off_t lseek(int fd, off_t offset, int whence);
```

**whence 參數**:
- `SEEK_SET`: 從文件開頭計算偏移 (offset 必須 >= 0)
- `SEEK_CUR`: 從當前位置計算偏移 (offset 可正可負)
- `SEEK_END`: 從文件末尾計算偏移 (offset 通常 <= 0)

**文件偏移量 (File Offset)**:

每個打開的文件都有一個當前偏移量,記錄下次 read/write 的起始位置。這個偏移量存在內核的 Open File Table 中,因此:
- 同一個 FD 的 read/write 會自動推進偏移量
- 通過 `dup()` 複製的 FD 共享偏移量
- 通過 `open()` 多次打開同一文件會有獨立的偏移量

**稀疏文件 (Sparse File)**:

當 lseek 超過文件末尾後寫入數據,中間的"空洞"不會佔用磁盤空間,讀取時返回零。這種文件稱為稀疏文件,常用於:
- 虛擬機磁盤映像
- 數據庫預分配空間
- 下載工具的臨時文件

```cpp
#include <unistd.h>
#include <fcntl.h>
#include <iostream>

void lseek_example() {
    int fd = open("/tmp/test.txt", O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) return;
    
    // 寫入數據
    write(fd, "0123456789", 10);
    
    // SEEK_SET: 從文件開頭計算
    lseek(fd, 0, SEEK_SET);
    char c;
    read(fd, &c, 1);
    std::cout << "Position 0: " << c << "\n";  // '0'
    
    // SEEK_CUR: 從當前位置計算
    lseek(fd, 4, SEEK_CUR);  // 跳過 4 個字節
    read(fd, &c, 1);
    std::cout << "Position 5: " << c << "\n";  // '5'
    
    // SEEK_END: 從文件末尾計算
    lseek(fd, -3, SEEK_END);
    read(fd, &c, 1);
    std::cout << "Position -3 from end: " << c << "\n";  // '7'
    
    // 獲取當前位置
    off_t pos = lseek(fd, 0, SEEK_CUR);
    std::cout << "Current position: " << pos << "\n";
    
    // 獲取文件大小
    off_t size = lseek(fd, 0, SEEK_END);
    std::cout << "File size: " << size << "\n";
    
    close(fd);
}

// 創建稀疏文件 (Sparse File)
void create_sparse_file() {
    int fd = open("/tmp/sparse.dat", O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) return;
    
    // 跳到 1GB 位置
    lseek(fd, 1024 * 1024 * 1024, SEEK_SET);
    
    // 寫入一個字節
    write(fd, "X", 1);
    
    // 文件邏輯大小是 1GB+1,但實際只佔用一個磁盤塊
    close(fd);
}
```

### 2.5 stat() 與 fstat() - 文件元數據

```cpp
#include <sys/stat.h>
#include <unistd.h>
#include <fcntl.h>
#include <iostream>
#include <ctime>

void stat_example(const char* path) {
    struct stat sb;
    
    // stat: 通過路徑獲取
    if (stat(path, &sb) < 0) {
        perror("stat");
        return;
    }
    
    std::cout << "File: " << path << "\n";
    std::cout << "Size: " << sb.st_size << " bytes\n";
    std::cout << "Blocks: " << sb.st_blocks << "\n";
    std::cout << "I-node: " << sb.st_ino << "\n";
    
    // 文件類型
    std::cout << "Type: ";
    if (S_ISREG(sb.st_mode)) std::cout << "regular file\n";
    else if (S_ISDIR(sb.st_mode)) std::cout << "directory\n";
    else if (S_ISLNK(sb.st_mode)) std::cout << "symbolic link\n";
    else if (S_ISFIFO(sb.st_mode)) std::cout << "FIFO/pipe\n";
    else if (S_ISSOCK(sb.st_mode)) std::cout << "socket\n";
    
    // 權限
    std::cout << "Mode: " << std::oct << (sb.st_mode & 0777) << std::dec << "\n";
    
    // 時間戳
    std::cout << "Last modified: " << ctime(&sb.st_mtime);
}

void fstat_example(int fd) {
    struct stat sb;
    
    // fstat: 通過文件描述符獲取
    if (fstat(fd, &sb) < 0) {
        perror("fstat");
        return;
    }
    
    std::cout << "File size: " << sb.st_size << " bytes\n";
}
```

### 2.6 fcntl() - 文件描述符控制

**fcntl() (File Control)** 是一個多功能系統呼叫,用於對文件描述符進行各種控制操作。

**函數原型**:
```cpp
#include <fcntl.h>
int fcntl(int fd, int cmd, ... /* arg */);
```

**常用命令 (cmd)**:

| 命令 | 說明 | 用途 |
|------|------|------|
| `F_GETFL` | 獲取文件狀態標誌 | 檢查是否非阻塞 |
| `F_SETFL` | 設置文件狀態標誌 | 設置非阻塞模式 |
| `F_DUPFD` | 複製文件描述符 | 類似 dup() |
| `F_GETFD` | 獲取 FD 標誌 | 檢查 close-on-exec |
| `F_SETFD` | 設置 FD 標誌 | 設置 close-on-exec |
| `F_SETLK` | 設置文件鎖 (非阻塞) | 進程間同步 |
| `F_SETLKW` | 設置文件鎖 (阻塞等待) | 進程間同步 |
| `F_GETLK` | 獲取文件鎖信息 | 檢測鎖狀態 |

**文件鎖 (File Locking)**:

Linux 提供兩種文件鎖:
1. **建議鎖 (Advisory Lock)**: 需要所有進程配合,不強制執行
2. **強制鎖 (Mandatory Lock)**: 內核強制執行,但很少使用

fcntl 提供的是建議鎖,支持:
- **共享鎖 (F_RDLCK)**: 多個進程可同時持有,用於讀取
- **排他鎖 (F_WRLCK)**: 只有一個進程可持有,用於寫入
- **解鎖 (F_UNLCK)**: 釋放鎖

```cpp
#include <fcntl.h>
#include <unistd.h>
#include <iostream>

// 設置非阻塞模式
bool set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags < 0) {
        perror("fcntl F_GETFL");
        return false;
    }
    
    if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) < 0) {
        perror("fcntl F_SETFL");
        return false;
    }
    
    return true;
}

// 複製文件描述符
void dup_example() {
    int fd = open("/tmp/test.txt", O_RDWR | O_CREAT, 0644);
    if (fd < 0) return;
    
    // F_DUPFD: 複製 FD,返回 >= 指定值的最小可用 FD
    int fd2 = fcntl(fd, F_DUPFD, 10);
    std::cout << "Original FD: " << fd << ", Duplicated FD: " << fd2 << "\n";
    
    // 兩個 FD 指向同一個文件,共享偏移量
    write(fd, "Hello", 5);
    write(fd2, " World", 6);  // 會追加在 "Hello" 後面
    
    close(fd);
    close(fd2);
}

// 文件鎖 (Advisory Lock)
bool lock_file(int fd, bool exclusive) {
    struct flock fl;
    fl.l_type = exclusive ? F_WRLCK : F_RDLCK;
    fl.l_whence = SEEK_SET;
    fl.l_start = 0;
    fl.l_len = 0;  // 0 表示鎖定整個文件
    
    if (fcntl(fd, F_SETLK, &fl) < 0) {
        perror("fcntl lock");
        return false;
    }
    
    return true;
}

bool unlock_file(int fd) {
    struct flock fl;
    fl.l_type = F_UNLCK;
    fl.l_whence = SEEK_SET;
    fl.l_start = 0;
    fl.l_len = 0;
    
    return fcntl(fd, F_SETLK, &fl) == 0;
}
```

### 2.7 dup() 與 dup2() - 重定向

```cpp
#include <unistd.h>
#include <fcntl.h>
#include <iostream>

// 重定向 stdout 到文件
void redirect_stdout_example() {
    // 保存原始 stdout
    int saved_stdout = dup(STDOUT_FILENO);
    
    // 打開日誌文件
    int log_fd = open("/tmp/output.log", O_CREAT | O_WRONLY | O_TRUNC, 0644);
    
    // 將 stdout (fd 1) 重定向到日誌文件
    dup2(log_fd, STDOUT_FILENO);
    close(log_fd);
    
    // 現在 printf/cout 會寫入文件
    std::cout << "This goes to file\n";
    printf("This also goes to file\n");
    
    // 恢復原始 stdout
    dup2(saved_stdout, STDOUT_FILENO);
    close(saved_stdout);
    
    std::cout << "This goes to terminal\n";
}
```

---

## 3. 進階文件操作

本節介紹進階的文件 I/O 技術,這些技術對於 HFT 系統的性能優化至關重要。

### 3.1 pread() 與 pwrite() - 原子定位讀寫

**pread()/pwrite()** 結合了 lseek 和 read/write 的功能,在指定偏移量處讀寫,但不改變文件的當前位置。

**函數原型**:
```cpp
#include <unistd.h>
ssize_t pread(int fd, void *buf, size_t count, off_t offset);
ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);
```

**原子性 (Atomicity)**:

傳統的 `lseek() + read()/write()` 組合不是原子操作,在多線程環境下可能發生競態條件:

```
線程 A                    線程 B
lseek(fd, 100, SEEK_SET)
                         lseek(fd, 200, SEEK_SET)
read(fd, buf, 10)        // 實際讀取位置 200,而非預期的 100!
```

而 `pread()/pwrite()` 是原子操作,不會被其他線程干擾。

**HFT 應用場景**:
- 多線程寫入同一日誌文件的不同區域
- 隨機訪問大型數據文件
- 實現無鎖的環形緩衝區

```cpp
#include <unistd.h>
#include <fcntl.h>
#include <iostream>
#include <cstring>

void pread_pwrite_example() {
    int fd = open("/tmp/test.txt", O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) return;
    
    // 初始化文件
    const char* init = "0123456789";
    write(fd, init, 10);
    
    // pwrite: 在指定位置寫入,不改變當前位置
    const char* data = "ABC";
    pwrite(fd, data, 3, 3);  // 在偏移 3 處寫入 "ABC"
    // 文件內容: "012ABC6789"
    
    // pread: 在指定位置讀取,不改變當前位置
    char buf[4];
    pread(fd, buf, 3, 3);
    buf[3] = '\0';
    std::cout << "Read at offset 3: " << buf << "\n";  // "ABC"
    
    // 當前位置仍在文件末尾
    off_t pos = lseek(fd, 0, SEEK_CUR);
    std::cout << "Current position: " << pos << "\n";  // 10
    
    close(fd);
}

// HFT 應用: 多線程安全的日誌寫入
class ThreadSafeLogger {
public:
    ThreadSafeLogger(const char* path) {
        fd_ = open(path, O_WRONLY | O_CREAT | O_APPEND, 0644);
    }
    
    ~ThreadSafeLogger() {
        if (fd_ >= 0) close(fd_);
    }
    
    // 多線程可以同時調用,使用 pwrite 避免競態
    void log(const char* message, size_t len, off_t offset) {
        pwrite(fd_, message, len, offset);
    }
    
private:
    int fd_;
};
```

### 3.2 readv() 與 writev() - 分散/聚集 I/O

**分散/聚集 I/O (Scatter-Gather I/O)** 是減少系統呼叫次數的關鍵技術,對 HFT 性能優化至關重要。

**函數原型**:
```cpp
#include <sys/uio.h>
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);

struct iovec {
    void  *iov_base;    // 緩衝區起始地址
    size_t iov_len;     // 緩衝區長度
};
```

**核心概念**:

- **分散讀取 (Scatter Read)**: 從文件讀取連續數據,分散到多個不連續的緩衝區
- **聚集寫入 (Gather Write)**: 將多個不連續的緩衝區數據,聚集後連續寫入文件

```
傳統方式 (3 次系統呼叫):           readv/writev (1 次系統呼叫):
┌─────┐                           ┌─────┐
│ buf1│ ← write()                 │ buf1│ ─┐
└─────┘                           └─────┘  │
┌─────┐                           ┌─────┐  ├─→ writev() ─→ 文件
│ buf2│ ← write()                 │ buf2│ ─┤
└─────┘                           └─────┘  │
┌─────┐                           ┌─────┐  │
│ buf3│ ← write()                 │ buf3│ ─┘
└─────┘                           └─────┘
```

**性能優勢**:

| 方式 | 系統呼叫次數 | 上下文切換 | 延遲 |
|------|-------------|------------|------|
| 多次 write | N | N 次 | 高 |
| 緩衝區拷貝 + 單次 write | 1 | 1 次 | 中 (需要拷貝) |
| writev | 1 | 1 次 | **最低** |

**HFT 典型應用**:
- 發送網路協議消息 (Header + Body)
- 寫入帶時間戳的日誌 (Timestamp + Message)
- 批量發送多個訂單

```cpp
#include <sys/uio.h>
#include <unistd.h>
#include <fcntl.h>
#include <iostream>
#include <cstring>

void writev_example() {
    int fd = open("/tmp/test.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) return;
    
    // 準備多個緩衝區
    const char* header = "=== Header ===\n";
    const char* body = "This is the body content.\n";
    const char* footer = "=== Footer ===\n";
    
    // 設置 iovec 結構
    struct iovec iov[3];
    iov[0].iov_base = const_cast<char*>(header);
    iov[0].iov_len = strlen(header);
    iov[1].iov_base = const_cast<char*>(body);
    iov[1].iov_len = strlen(body);
    iov[2].iov_base = const_cast<char*>(footer);
    iov[2].iov_len = strlen(footer);
    
    // 一次系統呼叫寫入所有緩衝區
    ssize_t n = writev(fd, iov, 3);
    std::cout << "Wrote " << n << " bytes\n";
    
    close(fd);
}

void readv_example() {
    int fd = open("/tmp/test.txt", O_RDONLY);
    if (fd < 0) return;
    
    char buf1[16], buf2[32], buf3[16];
    
    struct iovec iov[3];
    iov[0].iov_base = buf1;
    iov[0].iov_len = sizeof(buf1) - 1;
    iov[1].iov_base = buf2;
    iov[1].iov_len = sizeof(buf2) - 1;
    iov[2].iov_base = buf3;
    iov[2].iov_len = sizeof(buf3) - 1;
    
    // 一次系統呼叫讀取到多個緩衝區
    ssize_t n = readv(fd, iov, 3);
    
    if (n > 0) {
        // 添加字符串結束符
        size_t remaining = n;
        for (int i = 0; i < 3 && remaining > 0; ++i) {
            size_t chunk = std::min(remaining, iov[i].iov_len);
            static_cast<char*>(iov[i].iov_base)[chunk] = '\0';
            remaining -= chunk;
        }
        
        std::cout << "Buffer 1: " << buf1 << "\n";
        std::cout << "Buffer 2: " << buf2 << "\n";
        std::cout << "Buffer 3: " << buf3 << "\n";
    }
    
    close(fd);
}
```

### 3.3 HFT 應用: 協議消息的高效發送

```cpp
#include <sys/uio.h>
#include <sys/socket.h>
#include <cstdint>
#include <cstring>

// 典型的交易協議消息結構
struct MessageHeader {
    uint32_t length;
    uint16_t type;
    uint16_t sequence;
};

struct OrderMessage {
    uint64_t order_id;
    uint32_t symbol_id;
    double price;
    uint32_t quantity;
    char side;  // 'B' or 'S'
};

// 使用 writev 一次發送完整消息
ssize_t send_order(int socket_fd, const OrderMessage& order, uint16_t seq) {
    // 準備頭部
    MessageHeader header;
    header.length = sizeof(MessageHeader) + sizeof(OrderMessage);
    header.type = 1;  // ORDER_NEW
    header.sequence = seq;
    
    // 設置 iovec
    struct iovec iov[2];
    iov[0].iov_base = &header;
    iov[0].iov_len = sizeof(header);
    iov[1].iov_base = const_cast<OrderMessage*>(&order);
    iov[1].iov_len = sizeof(order);
    
    // 一次系統呼叫發送
    return writev(socket_fd, iov, 2);
}

// 對比: 多次 write 的低效方式
ssize_t send_order_inefficient(int socket_fd, const OrderMessage& order, uint16_t seq) {
    MessageHeader header;
    header.length = sizeof(MessageHeader) + sizeof(OrderMessage);
    header.type = 1;
    header.sequence = seq;
    
    // 兩次系統呼叫 - 效率低!
    write(socket_fd, &header, sizeof(header));
    return write(socket_fd, &order, sizeof(order));
}
```

### 3.4 preadv() 與 pwritev()

結合定位與分散/聚集 I/O 的優勢 (Linux 2.6.30+)。

```cpp
#include <sys/uio.h>
#include <unistd.h>
#include <fcntl.h>

void pwritev_example() {
    int fd = open("/tmp/test.txt", O_RDWR | O_CREAT, 0644);
    if (fd < 0) return;
    
    const char* data1 = "First";
    const char* data2 = "Second";
    
    struct iovec iov[2];
    iov[0].iov_base = const_cast<char*>(data1);
    iov[0].iov_len = 5;
    iov[1].iov_base = const_cast<char*>(data2);
    iov[1].iov_len = 6;
    
    // 在偏移 100 處寫入,不改變文件位置
    pwritev(fd, iov, 2, 100);
    
    close(fd);
}
```

### 3.5 fsync() 與 fdatasync() - 數據持久化

**數據持久化 (Data Durability)** 確保寫入的數據真正保存到持久存儲 (如磁盤),而不是僅停留在易失性緩衝區中。

**Linux I/O 緩衝層次**:

```
應用程序
    │ write()
    ▼
┌─────────────────┐
│ 用戶空間緩衝區  │  (如 stdio 的緩衝區)
│ (User Buffer)   │
└────────┬────────┘
         │ 系統呼叫
         ▼
┌─────────────────┐
│ 頁緩存          │  (內核維護,內存中)
│ (Page Cache)    │  ← write() 返回後數據在這裡
└────────┬────────┘
         │ fsync() / fdatasync()
         ▼
┌─────────────────┐
│ 磁盤控制器緩存  │  (硬件緩存)
│ (Disk Cache)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 磁盤盤片        │  ← 真正持久化
│ (Disk Platter)  │
└─────────────────┘
```

**fsync vs fdatasync**:

| 函數 | 同步內容 | 性能 | 使用場景 |
|------|----------|------|----------|
| `fsync()` | 數據 + 元數據 (大小、修改時間等) | 較慢 | 需要完整一致性 |
| `fdatasync()` | 僅數據 | 較快 | 只關心數據內容 |

**元數據 (Metadata)** 包括:
- 文件大小
- 最後修改時間 (mtime)
- 最後訪問時間 (atime)
- 權限、所有者等

```cpp
#include <unistd.h>
#include <fcntl.h>
#include <iostream>

void sync_example() {
    int fd = open("/tmp/important.dat", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) return;
    
    const char* data = "Critical trading data";
    write(fd, data, strlen(data));
    
    // fsync: 同步數據和元數據到磁盤
    // 確保數據真正寫入磁盤,而不是停留在內核緩衝區
    if (fsync(fd) < 0) {
        perror("fsync");
    }
    
    // fdatasync: 只同步數據,不同步元數據 (更快)
    // 適用於只關心數據內容,不關心修改時間等元數據的場景
    write(fd, data, strlen(data));
    if (fdatasync(fd) < 0) {
        perror("fdatasync");
    }
    
    close(fd);
}

// HFT 日誌策略
class HFTLogger {
public:
    enum SyncPolicy {
        NO_SYNC,      // 最快,但可能丟失數據
        SYNC_DATA,    // 中等,保證數據持久化
        SYNC_ALL      // 最慢,保證數據+元數據持久化
    };
    
    void write_log(const char* data, size_t len, SyncPolicy policy) {
        write(fd_, data, len);
        
        switch (policy) {
            case SYNC_DATA:
                fdatasync(fd_);
                break;
            case SYNC_ALL:
                fsync(fd_);
                break;
            case NO_SYNC:
            default:
                break;
        }
    }
    
private:
    int fd_;
};
```

### 3.6 posix_fadvise() - 緩存控制提示

**posix_fadvise()** 向內核提供文件訪問模式的提示,讓內核優化頁緩存 (Page Cache) 策略。

**函數原型**:
```cpp
#include <fcntl.h>
int posix_fadvise(int fd, off_t offset, off_t len, int advice);
```

**頁緩存 (Page Cache)**:

Linux 會將讀取的文件數據緩存在內存中,稱為頁緩存。這通常能提高性能,但在某些場景下可能有害:
- 緩存了不會再次訪問的數據 (浪費內存)
- 驅逐了真正需要的熱點數據 (緩存污染)

**建議類型 (advice)**:

| 建議 | 說明 | HFT 應用 |
|------|------|----------|
| `POSIX_FADV_NORMAL` | 默認行為 | - |
| `POSIX_FADV_SEQUENTIAL` | 順序訪問,內核積極預讀 | 讀取歷史數據 |
| `POSIX_FADV_RANDOM` | 隨機訪問,關閉預讀 | 隨機查詢 |
| `POSIX_FADV_WILLNEED` | 即將訪問,提前載入緩存 | 預熱關鍵數據 |
| `POSIX_FADV_DONTNEED` | 不再需要,可以釋放緩存 | 處理完歷史數據後 |
| `POSIX_FADV_NOREUSE` | 只訪問一次 | 一次性處理 |

**為什麼 HFT 要關心緩存控制?**

1. **避免緩存污染**: 處理大型歷史數據時,如果不釋放緩存,可能驅逐實時交易路徑的熱點數據
2. **減少延遲抖動**: 預讀可能導致不可預測的磁盤 I/O
3. **控制內存使用**: 防止頁緩存佔用過多內存

```cpp
#include <fcntl.h>
#include <unistd.h>
#include <iostream>

void fadvise_example() {
    int fd = open("/tmp/large_file.dat", O_RDONLY);
    if (fd < 0) return;
    
    // POSIX_FADV_SEQUENTIAL: 順序讀取,內核會積極預讀
    posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);
    
    // POSIX_FADV_RANDOM: 隨機讀取,關閉預讀
    // posix_fadvise(fd, 0, 0, POSIX_FADV_RANDOM);
    
    // POSIX_FADV_WILLNEED: 即將訪問,提前載入到緩存
    // posix_fadvise(fd, 0, 1024*1024, POSIX_FADV_WILLNEED);
    
    // POSIX_FADV_DONTNEED: 不再需要,釋放緩存
    // 讀取完成後調用,避免污染 page cache
    // posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);
    
    // 讀取文件...
    char buf[4096];
    while (read(fd, buf, sizeof(buf)) > 0) {
        // 處理數據
    }
    
    // 釋放緩存
    posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);
    
    close(fd);
}

// HFT 應用: 處理大型歷史數據文件
void process_historical_data(const char* path) {
    int fd = open(path, O_RDONLY);
    if (fd < 0) return;
    
    // 提示內核這是順序讀取
    posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);
    
    // 讀取並處理...
    
    // 處理完成後釋放緩存,避免影響實時數據
    posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);
    
    close(fd);
}
```

### 3.7 truncate() 與 ftruncate() - 文件截斷/擴展

```cpp
#include <unistd.h>
#include <fcntl.h>
#include <iostream>

void truncate_example() {
    const char* path = "/tmp/test.txt";
    
    // 創建並寫入數據
    int fd = open(path, O_RDWR | O_CREAT | O_TRUNC, 0644);
    write(fd, "0123456789", 10);
    
    // ftruncate: 通過 FD 截斷
    ftruncate(fd, 5);  // 保留前 5 字節: "01234"
    
    close(fd);
    
    // truncate: 通過路徑截斷
    truncate(path, 3);  // 保留前 3 字節: "012"
    
    // 擴展文件 (會用零填充)
    truncate(path, 100);  // 擴展到 100 字節
}

// HFT 應用: 預分配日誌文件空間
void preallocate_log_file(const char* path, size_t size) {
    int fd = open(path, O_RDWR | O_CREAT, 0644);
    if (fd < 0) return;
    
    // 預分配空間,避免運行時擴展
    ftruncate(fd, size);
    
    // 配合 mmap 使用更高效
    // void* addr = mmap(nullptr, size, PROT_READ | PROT_WRITE, 
    //                   MAP_SHARED, fd, 0);
    
    close(fd);
}
```

---

## 4. 進程管理

### 4.1 概念解析: 進程 vs 線程

**進程 (Process)** 和 **線程 (Thread)** 是操作系統執行任務的基本單位。

| 特性         | 進程 (Process)             | 線程 (Thread)             |
| :----------- | :------------------------- | :------------------------ |
| **定義**     | 資源分配的最小單位 (工廠)  | CPU 調度的最小單位 (工人) |
| **內存**     | 獨立內存空間 (互不干擾)    | 共享進程內存 (方便通信)   |
| **開銷**     | 大 (創建/切換慢)           | 小 (創建/切換快)          |
| **通信**     | 困難 (IPC: 管道, 共享內存) | 容易 (直接讀寫變量)       |
| **穩定性**   | 高 (一個崩潰不影響其他)    | 低 (一個崩潰導致進程崩潰) |
| **HFT 應用** | 策略隔離, 風險控制         | 處理高頻數據, 訂單發送    |

### 4.2 fork 與 exec

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

### 4.3 進程間通信 (IPC) - 管道

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

### 4.4 共享內存 (Shared Memory)

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

## 5. 實時調度

### 5.1 概念解析: 什麼是實時 (Real-time)?

在 HFT 中,**實時 (Real-time)** 並不單純指"快",而是指**確定性 (Determinism)** 和**可預測性 (Predictability)**。

**延遲 vs 確定性**:

```
非實時系統延遲分布:            實時系統延遲分布:
    │  *                           │
頻率│ ***                       頻率│    ***
    │*****                         │   *****
    │******                        │  *******
    └────────────────→             └────────────────→
       1μs    100ms                   1μs  10μs
       
   平均 10μs,但可能突然卡頓         保證 99.99% < 10μs
```

- **非實時**: 平均延遲可能很低,但偶爾會有巨大的延遲尖峰 (Latency Spike),原因包括:
  - 垃圾回收 (GC)
  - 頁面錯誤 (Page Fault)
  - 被其他進程搶佔
  - 系統中斷處理

- **實時**: 保證在規定時間內 (例如 10μs) 必須完成響應,最壞情況延遲 (Worst-case Latency) 是有界的。

**Linux 實時調度**:

我們使用 **SCHED_FIFO** 或 **SCHED_RR** 調度策略來告訴操作系統:"這個線程非常重要,除非它自己主動讓出 CPU,否則絕對不要打斷它!"

**軟實時 vs 硬實時**:
- **軟實時 (Soft Real-time)**: 偶爾超過截止時間是可以接受的 (如視頻播放)
- **硬實時 (Hard Real-time)**: 絕對不能超過截止時間 (如汽車 ABS)
- **HFT 屬於軟實時**: 追求極低的尾部延遲 (P99, P99.9),但不是生死攸關

### 5.2 調度策略

Linux 支持多種調度策略,分為普通調度和實時調度兩類。

**調度策略詳解**:

| 策略 | 類型 | 優先級範圍 | 特點 | HFT 適用性 |
|------|------|------------|------|------------|
| `SCHED_OTHER` | 普通 | 0 (nice: -20~19) | 默認,CFS 公平調度 | 不適合 |
| `SCHED_BATCH` | 普通 | 0 | 批處理任務,允許更長時間片 | 不適合 |
| `SCHED_IDLE` | 普通 | 0 | 只在系統空閒時運行 | 不適合 |
| `SCHED_FIFO` | 實時 | 1-99 | 先進先出,不被搶佔 | **推薦** |
| `SCHED_RR` | 實時 | 1-99 | 輪轉,有時間片 | 可用 |
| `SCHED_DEADLINE` | 實時 | - | 基於截止時間調度 | 進階 |

**SCHED_FIFO vs SCHED_RR**:

- **SCHED_FIFO**: 
  - 一旦獲得 CPU,持續運行直到主動讓出 (yield/sleep/阻塞)
  - 同優先級的任務按 FIFO 順序
  - **HFT 首選**: 避免不必要的上下文切換

- **SCHED_RR** (Round Robin):
  - 有時間片限制,時間片用完後讓給同優先級的其他任務
  - 防止單個任務獨佔 CPU
  - 適合多個同優先級的實時任務

**優先級選擇建議**:

```
99: 系統關鍵任務 (不建議使用)
90: 市場數據接收
85: 策略引擎
80: 訂單發送
75: 風控檢查
50: 日誌記錄
 1: 低優先級實時任務
```

### 5.3 設置實時優先級

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

### 5.4 查看調度信息

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

## 6. CPU 親和性與隔離

### 6.1 設置 CPU 親和性

**CPU 親和性 (CPU Affinity)** 指定進程或線程只能在特定的 CPU 核心上運行。

**為什麼需要 CPU 親和性?**

1. **緩存親和性 (Cache Affinity)**: 
   - 線程的數據會緩存在其運行的 CPU 的 L1/L2 緩存中
   - 如果線程遷移到其他 CPU,緩存失效,需要重新載入
   - 綁定到固定 CPU 可以提高緩存命中率

2. **NUMA 架構優化**:
   - 在 NUMA (Non-Uniform Memory Access) 系統中,CPU 訪問本地內存比遠端內存快
   - 綁定線程到特定 CPU 可以確保使用本地內存

3. **減少上下文切換**:
   - 與 CPU 隔離配合使用,避免被其他進程打斷

4. **可預測性**:
   - 固定的 CPU 分配使性能更可預測

**API 函數**:

| 函數 | 作用 | 粒度 |
|------|------|------|
| `sched_setaffinity()` | 設置進程親和性 | 進程 |
| `sched_getaffinity()` | 獲取進程親和性 | 進程 |
| `pthread_setaffinity_np()` | 設置線程親和性 | 線程 |
| `pthread_getaffinity_np()` | 獲取線程親和性 | 線程 |

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

### 6.2 CPU 隔離

**CPU 隔離 (CPU Isolation)** 將某些 CPU 核心從系統調度器中移除,使其只運行指定的任務。

**隔離 vs 親和性**:

- **親和性**: 告訴調度器"我想在這個 CPU 上運行",但其他進程也可能被調度到這個 CPU
- **隔離**: 告訴內核"這個 CPU 不參與通用調度",只有明確綁定的進程才能使用

**內核啟動參數**:

| 參數 | 作用 |
|------|------|
| `isolcpus=2,3` | 將 CPU 2,3 從調度器移除 |
| `nohz_full=2,3` | 在 CPU 2,3 上禁用定時器中斷 (減少抖動) |
| `rcu_nocbs=2,3` | 將 RCU 回調移到其他 CPU |
| `irqaffinity=0,1` | 將中斷限制在 CPU 0,1 |

**典型 HFT 配置**:

```
CPU 0: 系統服務、中斷處理
CPU 1: 非關鍵任務
CPU 2: 市場數據接收 (隔離)
CPU 3: 策略引擎 (隔離)
CPU 4: 訂單發送 (隔離)
CPU 5-7: 其他關鍵任務 (隔離)
```

```bash
# 查看 CPU 信息
lscpu

# 隔離 CPU 2-3 (在 GRUB 配置中)
# 編輯 /etc/default/grub
GRUB_CMDLINE_LINUX="isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3"

# 更新 GRUB
sudo update-grub
sudo reboot

# 驗證隔離
cat /sys/devices/system/cpu/isolated
```

### 6.3 HFT 線程綁定策略

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

## 7. 內存管理

### 7.1 概念解析: 虛擬內存與缺頁中斷

- **虛擬內存**: 程序看到的內存地址是虛擬的,操作系統負責將其映射到物理內存。
- **缺頁中斷 (Page Fault)**: 當程序訪問一個還沒映射到物理內存的虛擬地址時,CPU 會觸發中斷,操作系統暫停程序,去分配物理內存。這會產生巨大的延遲 (微秒級甚至毫秒級)。

**HFT 解決方案**: 使用 `mlock` 鎖定內存,並預先訪問一遍 (Pre-faulting),確保所有內存都在物理內存中,永遠不會觸發缺頁中斷。

### 7.2 鎖定內存 (mlock)

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

### 7.3 Huge Pages

**Huge Pages (大頁)** 是一種使用比默認 4KB 更大頁面的內存管理機制。

**為什麼需要 Huge Pages?**

現代系統使用虛擬內存,每次內存訪問都需要通過頁表將虛擬地址轉換為物理地址。為了加速這個過程,CPU 有一個緩存叫 TLB (Translation Lookaside Buffer)。

**TLB (轉譯後備緩衝區)**:
- 緩存最近使用的頁表項
- 容量有限,通常只有幾百到幾千個條目
- TLB 未命中 (TLB Miss) 會導致昂貴的頁表遍歷

**Huge Pages 的優勢**:

| 頁大小 | 可尋址範圍 (1000 TLB 條目) | TLB 效率 |
|--------|---------------------------|----------|
| 4KB (默認) | 4MB | 低 |
| 2MB (Huge Page) | 2GB | **高** |
| 1GB (Gigantic Page) | 1TB | 極高 |

**HFT 應用場景**:
- 大型數據結構 (Order Book, 緩存)
- 共享內存
- 頻繁訪問的熱點數據

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

### 7.4 內存預分配

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

## 8. 信號與計時器

**信號 (Signal)** 是 Unix/Linux 中進程間通信和異常處理的基本機制。**計時器 (Timer)** 則是實現定時任務和精確延遲測量的關鍵工具。

### 8.1 信號處理基礎

**信號 (Signal)** 是一種軟體中斷,用於通知進程發生了某個事件。

**信號的來源**:
- **硬體異常**: 除零錯誤 (SIGFPE)、非法內存訪問 (SIGSEGV)
- **終端操作**: Ctrl+C (SIGINT)、Ctrl+\ (SIGQUIT)
- **系統調用**: `kill()`, `raise()`, `alarm()`
- **軟體條件**: 管道斷開 (SIGPIPE)、子進程終止 (SIGCHLD)

**常見信號**:

| 信號 | 編號 | 默認行為 | 說明 |
|------|------|----------|------|
| `SIGINT` | 2 | 終止 | Ctrl+C,用戶中斷 |
| `SIGTERM` | 15 | 終止 | 請求終止 (可捕獲) |
| `SIGKILL` | 9 | 終止 | 強制終止 (不可捕獲) |
| `SIGSEGV` | 11 | Core Dump | 段錯誤,非法內存訪問 |
| `SIGPIPE` | 13 | 終止 | 寫入已關閉的管道/Socket |
| `SIGCHLD` | 17 | 忽略 | 子進程狀態改變 |
| `SIGALRM` | 14 | 終止 | 定時器到期 |
| `SIGUSR1/2` | 10/12 | 終止 | 用戶自定義信號 |

**信號處理方式**:
1. **默認行為**: 終止、忽略、停止、繼續
2. **忽略**: `SIG_IGN`
3. **自定義處理函數**: 捕獲並處理

**HFT 中的信號處理原則**:
- 信號處理函數要盡量簡短,只設置標誌
- 避免在處理函數中調用非異步信號安全 (async-signal-safe) 的函數
- 使用 `signalfd()` 將信號轉為文件描述符,整合到事件循環

```cpp
#include <signal.h>
#include <unistd.h>
#include <iostream>
#include <atomic>

// 使用原子變量作為標誌
std::atomic<bool> should_exit{false};

// 信號處理函數 (應盡量簡單)
void signal_handler(int signum) {
    should_exit.store(true, std::memory_order_relaxed);
}

void signal_basics() {
    // 設置信號處理器 (舊式,不推薦)
    // signal(SIGINT, signal_handler);
    
    // 使用 sigaction (推薦)
    struct sigaction sa;
    sa.sa_handler = signal_handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    
    sigaction(SIGINT, &sa, nullptr);   // Ctrl+C
    sigaction(SIGTERM, &sa, nullptr);  // kill
    
    std::cout << "Press Ctrl+C to exit...\n";
    
    while (!should_exit.load(std::memory_order_relaxed)) {
        // 主循環
        sleep(1);
    }
    
    std::cout << "Exiting gracefully\n";
}
```

### 8.2 signalfd() - 將信號轉為文件描述符

**signalfd()** 將信號轉換為可讀取的文件描述符,使信號處理可以整合到 `epoll` 事件循環中。

**為什麼需要 signalfd?**

傳統信號處理有諸多問題:
1. **異步性**: 信號可能在任何時刻中斷程序執行
2. **重入問題**: 信號處理函數中只能調用異步信號安全的函數
3. **競態條件**: 信號與主程序之間的數據共享需要特殊處理

signalfd 將信號轉為同步的 I/O 事件,解決了這些問題:
- 信號通過 `read()` 同步讀取
- 可以與 `epoll` 一起監控
- 在主事件循環中統一處理

**函數原型**:
```cpp
#include <sys/signalfd.h>
int signalfd(int fd, const sigset_t *mask, int flags);
```

**使用步驟**:
1. 使用 `sigprocmask()` 阻塞要處理的信號
2. 創建 signalfd
3. 將 signalfd 加入 epoll
4. 在事件循環中讀取信號信息

```cpp
#include <sys/signalfd.h>
#include <sys/epoll.h>
#include <signal.h>
#include <unistd.h>
#include <iostream>

void signalfd_example() {
    // 阻塞要處理的信號
    sigset_t mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIGINT);
    sigaddset(&mask, SIGTERM);
    sigprocmask(SIG_BLOCK, &mask, nullptr);
    
    // 創建 signalfd
    int sfd = signalfd(-1, &mask, SFD_NONBLOCK);
    if (sfd < 0) {
        perror("signalfd");
        return;
    }
    
    // 可以加入 epoll 監控
    int epfd = epoll_create1(0);
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = sfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, sfd, &ev);
    
    std::cout << "Waiting for signals (Ctrl+C or kill)...\n";
    
    struct epoll_event events[10];
    bool running = true;
    
    while (running) {
        int n = epoll_wait(epfd, events, 10, 1000);
        
        for (int i = 0; i < n; ++i) {
            if (events[i].data.fd == sfd) {
                struct signalfd_siginfo fdsi;
                read(sfd, &fdsi, sizeof(fdsi));
                
                if (fdsi.ssi_signo == SIGINT) {
                    std::cout << "Received SIGINT\n";
                    running = false;
                } else if (fdsi.ssi_signo == SIGTERM) {
                    std::cout << "Received SIGTERM\n";
                    running = false;
                }
            }
        }
    }
    
    close(sfd);
    close(epfd);
}
```

### 8.3 clock_gettime() - 高精度時間

**clock_gettime()** 提供納秒級精度的時間獲取,是 HFT 系統中延遲測量的標準方式。

**函數原型**:
```cpp
#include <time.h>
int clock_gettime(clockid_t clk_id, struct timespec *tp);

struct timespec {
    time_t tv_sec;   // 秒
    long   tv_nsec;  // 納秒 (0-999999999)
};
```

**時鐘類型 (clk_id)**:

| 時鐘 | 說明 | HFT 用途 |
|------|------|----------|
| `CLOCK_REALTIME` | 系統實時時間 (Wall Clock) | 時間戳記錄、日誌 |
| `CLOCK_MONOTONIC` | 單調遞增時間,不受系統時間調整影響 | **延遲測量 (推薦)** |
| `CLOCK_MONOTONIC_RAW` | 不受 NTP 調整的單調時間 | 精確性能分析 |
| `CLOCK_PROCESS_CPUTIME_ID` | 進程 CPU 時間 | CPU 使用分析 |
| `CLOCK_THREAD_CPUTIME_ID` | 線程 CPU 時間 | 線程性能分析 |

**CLOCK_REALTIME vs CLOCK_MONOTONIC**:

- **CLOCK_REALTIME**: 
  - 表示真實世界的時間 (1970-01-01 以來的秒數)
  - 可能因 NTP 同步或手動調整而跳變
  - 用於記錄事件發生的絕對時間

- **CLOCK_MONOTONIC**:
  - 從某個未指定的起點開始單調遞增
  - 不會跳變,只會單調增加
  - **測量時間間隔必須使用此時鐘**

```cpp
#include <time.h>
#include <iostream>
#include <cstdint>

// 獲取納秒級時間戳
uint64_t get_timestamp_ns() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000000000ULL + ts.tv_nsec;
}

// 獲取實時時間
uint64_t get_realtime_ns() {
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts);
    return ts.tv_sec * 1000000000ULL + ts.tv_nsec;
}

void clock_gettime_example() {
    // CLOCK_REALTIME: 系統實時時間 (可能被調整)
    // CLOCK_MONOTONIC: 單調遞增時間 (適合測量時間間隔)
    // CLOCK_MONOTONIC_RAW: 不受 NTP 調整的單調時間
    
    struct timespec start, end;
    
    clock_gettime(CLOCK_MONOTONIC, &start);
    
    // 執行一些操作
    volatile int x = 0;
    for (int i = 0; i < 1000000; ++i) {
        x += i;
    }
    
    clock_gettime(CLOCK_MONOTONIC, &end);
    
    // 計算時間差
    long elapsed_ns = (end.tv_sec - start.tv_sec) * 1000000000L 
                    + (end.tv_nsec - start.tv_nsec);
    
    std::cout << "Elapsed: " << elapsed_ns << " ns\n";
    std::cout << "Elapsed: " << elapsed_ns / 1000.0 << " us\n";
}

// 時鐘解析度
void print_clock_resolution() {
    struct timespec res;
    
    clock_getres(CLOCK_REALTIME, &res);
    std::cout << "CLOCK_REALTIME resolution: " 
              << res.tv_nsec << " ns\n";
    
    clock_getres(CLOCK_MONOTONIC, &res);
    std::cout << "CLOCK_MONOTONIC resolution: " 
              << res.tv_nsec << " ns\n";
}
```

### 8.4 timerfd - 定時器文件描述符

**timerfd** 將定時器封裝為文件描述符,可以與 `epoll` 整合,實現高效的定時任務。

**傳統定時器的問題**:

- `sleep()/usleep()`: 阻塞整個線程
- `alarm()`: 只能設置一個定時器,精度為秒
- `setitimer()`: 通過信號通知,有異步處理的所有問題

**timerfd 的優勢**:

1. 可以創建多個獨立的定時器
2. 納秒級精度
3. 與 epoll 整合,實現統一的事件處理
4. 同步讀取,無競態條件

**函數原型**:
```cpp
#include <sys/timerfd.h>
int timerfd_create(int clockid, int flags);
int timerfd_settime(int fd, int flags, 
                    const struct itimerspec *new_value,
                    struct itimerspec *old_value);

struct itimerspec {
    struct timespec it_interval;  // 週期 (0 表示一次性)
    struct timespec it_value;     // 首次觸發延遲
};
```

**工作機制**:
1. 定時器到期時,FD 變為可讀
2. `read()` 返回到期次數 (8 字節 uint64_t)
3. 如果多次到期未讀取,次數會累加

```cpp
#include <sys/timerfd.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <iostream>
#include <cstdint>

void timerfd_example() {
    // 創建定時器
    int tfd = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK);
    if (tfd < 0) {
        perror("timerfd_create");
        return;
    }
    
    // 設置定時器: 首次 1 秒後觸發,之後每 500ms 觸發
    struct itimerspec its;
    its.it_value.tv_sec = 1;      // 首次延遲
    its.it_value.tv_nsec = 0;
    its.it_interval.tv_sec = 0;   // 週期
    its.it_interval.tv_nsec = 500000000;  // 500ms
    
    timerfd_settime(tfd, 0, &its, nullptr);
    
    // 與 epoll 整合
    int epfd = epoll_create1(0);
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = tfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, tfd, &ev);
    
    std::cout << "Timer started...\n";
    
    struct epoll_event events[10];
    int count = 0;
    
    while (count < 5) {
        int n = epoll_wait(epfd, events, 10, -1);
        
        for (int i = 0; i < n; ++i) {
            if (events[i].data.fd == tfd) {
                uint64_t expirations;
                read(tfd, &expirations, sizeof(expirations));
                
                std::cout << "Timer expired! Count: " << ++count 
                          << ", Expirations: " << expirations << "\n";
            }
        }
    }
    
    close(tfd);
    close(epfd);
}

// HFT 應用: 定期執行任務
class PeriodicTask {
public:
    PeriodicTask(int interval_ms) {
        tfd_ = timerfd_create(CLOCK_MONOTONIC, 0);
        
        struct itimerspec its;
        its.it_value.tv_sec = interval_ms / 1000;
        its.it_value.tv_nsec = (interval_ms % 1000) * 1000000;
        its.it_interval = its.it_value;
        
        timerfd_settime(tfd_, 0, &its, nullptr);
    }
    
    ~PeriodicTask() {
        if (tfd_ >= 0) close(tfd_);
    }
    
    void wait() {
        uint64_t expirations;
        read(tfd_, &expirations, sizeof(expirations));
    }
    
    int fd() const { return tfd_; }
    
private:
    int tfd_;
};
```

### 8.5 RDTSC - CPU 時間戳計數器

**RDTSC (Read Time-Stamp Counter)** 直接讀取 CPU 的硬件計數器,是最低延遲的計時方式。

**什麼是 TSC?**

**TSC (Time-Stamp Counter)** 是 x86/x64 CPU 中的一個 64 位寄存器,在每個 CPU 時鐘週期遞增。例如,3 GHz 的 CPU 每秒遞增 3×10^9 次。

**優勢**:
- **極低開銷**: 只需幾個 CPU 週期 (~20 cycles),相比 `clock_gettime()` 的 ~100 cycles
- **高精度**: 週期級精度,約 0.3ns@3GHz
- **無系統調用**: 完全在用戶態執行

**注意事項**:

1. **頻率校準**: TSC 計數的是週期數,需要知道 CPU 頻率才能轉換為時間
2. **多核同步**: 現代 CPU (Invariant TSC) 保證所有核心的 TSC 同步
3. **頻率變化**: 需要 Constant TSC 或 Invariant TSC 支持,否則會因省電模式而變化
4. **指令重排**: 使用 `rdtscp` 或配合內存屏障確保測量準確

**RDTSC vs RDTSCP**:

| 指令 | 特性 | 用途 |
|------|------|------|
| `rdtsc` | 可能被亂序執行 | 一般計時 |
| `rdtscp` | 序列化,等待之前的指令完成 | **精確延遲測量** |

**檢查 TSC 特性**:
```bash
# 檢查 CPU 是否支持 constant_tsc 和 nonstop_tsc
grep -E 'constant_tsc|nonstop_tsc' /proc/cpuinfo
```

```cpp
#include <x86intrin.h>
#include <iostream>
#include <cstdint>

// 讀取 TSC
inline uint64_t rdtsc() {
    return __rdtsc();
}

// 帶序列化的 RDTSC (更精確)
inline uint64_t rdtscp() {
    unsigned int aux;
    return __rdtscp(&aux);
}

// 獲取 TSC 頻率 (需要校準)
double calibrate_tsc_frequency() {
    struct timespec start, end;
    
    clock_gettime(CLOCK_MONOTONIC, &start);
    uint64_t tsc_start = rdtsc();
    
    // 等待一段時間
    struct timespec delay = {0, 100000000};  // 100ms
    nanosleep(&delay, nullptr);
    
    clock_gettime(CLOCK_MONOTONIC, &end);
    uint64_t tsc_end = rdtsc();
    
    // 計算頻率
    long elapsed_ns = (end.tv_sec - start.tv_sec) * 1000000000L 
                    + (end.tv_nsec - start.tv_nsec);
    uint64_t tsc_diff = tsc_end - tsc_start;
    
    return (double)tsc_diff / elapsed_ns;  // cycles per ns
}

void rdtsc_example() {
    double cycles_per_ns = calibrate_tsc_frequency();
    std::cout << "TSC frequency: " << cycles_per_ns << " cycles/ns\n";
    std::cout << "TSC frequency: " << cycles_per_ns * 1000 << " MHz\n";
    
    // 測量操作延遲
    uint64_t start = rdtscp();
    
    // 執行操作
    volatile int x = 0;
    for (int i = 0; i < 1000; ++i) {
        x += i;
    }
    
    uint64_t end = rdtscp();
    uint64_t cycles = end - start;
    double ns = cycles / cycles_per_ns;
    
    std::cout << "Elapsed: " << cycles << " cycles\n";
    std::cout << "Elapsed: " << ns << " ns\n";
}

// HFT 延遲測量類
class LatencyTimer {
public:
    LatencyTimer() : cycles_per_ns_(calibrate_tsc_frequency()) {}
    
    void start() {
        start_ = rdtscp();
    }
    
    uint64_t stop_cycles() {
        return rdtscp() - start_;
    }
    
    double stop_ns() {
        return stop_cycles() / cycles_per_ns_;
    }
    
private:
    double cycles_per_ns_;
    uint64_t start_;
};
```

---

## 9. 性能監控

### 9.1 perf 工具

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

### 9.2 /proc 文件系統

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

### 9.3 系統調用追蹤

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

## 10. 系統調優

### 10.1 中斷親和性 (IRQ Affinity)

```bash
# 查看網卡中斷
cat /proc/interrupts | grep eth0

# 設置中斷親和性到 CPU 0
echo 1 | sudo tee /proc/irq/<IRQ_NUMBER>/smp_affinity

# 使用 irqbalance 自動平衡
sudo systemctl start irqbalance
```

### 10.2 關閉不必要的服務

```bash
# 查看運行的服務
systemctl list-units --type=service --state=running

# 關閉不必要的服務
sudo systemctl stop bluetooth
sudo systemctl disable bluetooth

# 關閉圖形界面 (服務器)
sudo systemctl set-default multi-user.target
```

### 10.3 內核參數調優

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

### 10.4 HFT 系統完整調優腳本

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

1. **系統呼叫概述**: 用戶態/內核態、開銷分析、分類速查
2. **文件描述符與底層 I/O**: open、read、write、fcntl、stat
3. **進階文件操作**: pread/pwrite、readv/writev、fsync、fadvise
4. **進程管理**: fork、exec、IPC (管道、共享內存)
5. **實時調度**: SCHED_FIFO、優先級設置
6. **CPU 親和性**: 線程綁定、CPU 隔離
7. **內存管理**: mlock、Huge Pages、內存池
8. **信號與計時器**: signalfd、clock_gettime、timerfd、RDTSC
9. **性能監控**: perf、/proc、strace
10. **系統調優**: 中斷親和性、內核參數

### 系統呼叫速查表

| 分類 | 系統呼叫 | 用途 | HFT 優先級 |
|------|----------|------|------------|
| 文件基礎 | open, read, write, close | 基本文件操作 | ⭐⭐ |
| 文件進階 | pread, pwrite, lseek | 定位讀寫 | ⭐⭐ |
| 分散/聚集 | readv, writev | 多緩衝區 I/O | ⭐⭐⭐ |
| 記憶體映射 | mmap, munmap, msync | 零拷貝, 共享記憶體 | ⭐⭐⭐ |
| 同步控制 | fsync, fdatasync | 數據持久化 | ⭐⭐ |
| FD 控制 | fcntl, dup2 | 非阻塞, 重定向 | ⭐⭐⭐ |
| 元數據 | stat, fstat | 文件資訊 | ⭐⭐ |
| 進程 | fork, exec, wait | 進程管理 | ⭐⭐ |
| 信號 | sigaction, signalfd | 信號處理 | ⭐⭐ |
| 時間 | clock_gettime, timerfd | 計時與定時 | ⭐⭐⭐ |

### HFT 系統調優檢查清單

- [ ] 使用 `readv()/writev()` 減少系統呼叫
- [ ] 使用 `mmap()` 實現零拷貝
- [ ] 隔離 CPU 核心 (isolcpus)
- [ ] 設置實時調度策略 (SCHED_FIFO)
- [ ] 綁定關鍵線程到獨立 CPU
- [ ] 鎖定內存 (mlockall)
- [ ] 配置 Huge Pages
- [ ] 調整網路參數
- [ ] 關閉不必要的服務
- [ ] 設置中斷親和性
- [ ] 關閉 CPU 頻率調節
- [ ] 使用 RDTSC 測量延遲

### 性能提升總結

| 優化技術 | 延遲改善 | 穩定性 | 複雜度 | HFT 推薦 |
| -------- | -------- | ------ | ------ | -------- |
| readv/writev | 20-50% | 高 | 低 | 必須 |
| CPU 隔離 | 30-50% | 高 | 中等 | 必須 |
| 實時調度 | 20-40% | 高 | 低 | 必須 |
| mlock | 10-20% | 高 | 低 | 必須 |
| Huge Pages | 5-15% | 中等 | 中等 | 推薦 |
| 中斷親和性 | 10-30% | 中等 | 中等 | 推薦 |
| RDTSC 計時 | 精度提升 | 高 | 低 | 推薦 |

### HFT 最佳實踐

1. **減少系統呼叫**: 使用 `readv()/writev()` 批量操作
2. **避免阻塞**: 使用 `O_NONBLOCK` + `epoll`
3. **零拷貝**: `mmap()` + `sendfile()`
4. **精準計時**: `clock_gettime(CLOCK_MONOTONIC)` 或 `RDTSC`
5. **預分配**: `ftruncate()` + `mmap()` 避免運行時分配
6. **緩存控制**: `posix_fadvise()` 優化 page cache
7. **信號處理**: 使用 `signalfd()` 整合到事件循環

---

## 參考資料 (References)

1. [The Linux Programming Interface](http://man7.org/tlpi/) - Michael Kerrisk
2. [Advanced Programming in the UNIX Environment](https://www.apuebook.com/) - Stevens & Rago
3. [Real-Time Linux Wiki](https://rt.wiki.kernel.org/)
4. [perf Documentation](https://perf.wiki.kernel.org/)
5. [Linux Performance](http://www.brendangregg.com/linuxperf.html) - Brendan Gregg
6. [SCHED_DEADLINE Documentation](https://www.kernel.org/doc/Documentation/scheduler/sched-deadline.txt)
7. [Linux System Call Table](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)
