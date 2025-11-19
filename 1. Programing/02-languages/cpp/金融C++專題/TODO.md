# C++ 高頻交易系統 10 天學習計畫

> 專注於高性能、低延遲的實戰技能培養

---

## 進度總覽

| 天數     | 主題              | 狀態    | 筆記檔案                    |
| ------ | --------------- | ----- | ----------------------- |
| Day 1  | C++ 核心特性 Part 1 | ✅ 完成  | `Day01-C++核心特性Part1.md` |
| Day 2  | C++ 現代特性 Part 2 | ✅ 完成  | `Day02-C++現代特性Part2.md` |
| Day 3  | 編譯與優化           | ✅ 完成  | `Day03-編譯與優化.md`        |
| Day 4  | 多線程編程與同步        | ✅ 完成  | `Day04-多線程編程與同步.md`     |
| Day 5  | 無鎖程式設計與性能優化     | ✅ 完成  | `Day05-無鎖程式設計與性能優化.md`  |
| Day 6  | 網路程式設計基礎        | ✅ 完成  | `Day06-網路程式設計基礎.md`     |
| Day 7  | 網路進階與極致優化       | ✅ 完成  | `Day07-網路進階與極致優化.md`    |
| Day 8  | Linux 系統開發與調優   | ✅ 完成  | `Day08-Linux系統開發與調優.md` |
| Day 9  | 金融交易系統專題        | ✅ 完成  | `Day09-金融交易系統專題.md`     |
| Day 10 | 整合實戰項目          | ✅ 完成  | `Day10-整合實戰項目.md`       |

### 補充資源

- [x] 標準庫與第三方庫指南 - `標準庫與第三方庫指南.md`

---

## 學習目標



- 掌握現代 C++ 高性能特性

- 精通多執行緒與無鎖程式設計

- 熟練網路程式設計與延遲優化

- 理解 Linux 系統程式設計

- 能夠構建基本的高頻交易系統



---



## Day 1-2: 深入 C++ 語法



### Day 1: C++ 核心特性 (Part 1)



#### 1.1 智能指標

- `unique_ptr` vs `shared_ptr` vs `weak_ptr`

- 自定義刪除器

- `make_unique` / `make_shared` 的優勢

- **陷阱:** 引用計數的原子操作開銷

- 實作:資源池使用智能指標管理



#### 1.2 移動語義與右值引用

- 左值 vs 右值的本質

- 移動建構子與移動賦值運算子

- `std::move` 與 `std::forward`

- RVO/NRVO 優化機制

- 五法則 vs 零法則

- 實作:高性能 String 類別支援移動語義



#### 1.3 Lambda 表達式

- 捕獲機制:`[=]`, `[&]`, `[this]`, `[*this]`

- 初始化捕獲 (C++14)

- 泛型 lambda

- `std::function` vs 裸 lambda 的性能差異

- 實作:事件回調系統



#### 1.4 模板進階

- 函數模板與類別模板

- 模板特化:全特化與偏特化

- 可變參數模板與參數包展開

- SFINAE 與 `std::enable_if`

- 類型萃取 (type traits)

- 實作:泛型訊息佇列



**實作練習:**

- 支援移動語義的 Vector 容器

- Type-safe 的訊息佇列

- 使用 SFINAE 的條件編譯函數



---



### Day 2: C++ 現代特性 (Part 2)



#### 2.1 C++17 核心特性

- **結構化綁定:** `auto [a, b] = pair;`

- **`std::optional`:** 取代 nullptr 的錯誤處理

- **`std::variant`:** 類型安全的 union + `std::visit`

- **`std::string_view`:** 零拷貝字串視圖

- **if/switch 初始化語句**

- **Fold expressions:** 簡化可變參數模板



#### 2.2 C++20 核心特性

- **Concepts:** 約束模板參數

  - 標準 concepts: `std::integral`, `std::floating_point`

  - 自定義 concepts

- **Ranges:** 管道語法與惰性求值

  - `std::views::filter`, `std::views::transform`

- **Three-way comparison (`<=>`)**

- **`std::span`:** 非擁有的陣列視圖



#### 2.3 記憶體管理深入

- **RAII 原則:** 資源與生命週期綁定

- **自定義記憶體分配**

  - `operator new` / `operator delete` 重載

  - placement new

  - 自定義 allocator

- **記憶體池 (Memory Pool)**

  - 固定大小記憶體池設計

  - 空閒列表管理

  - 實作:簡單物件池

- **零拷貝技術**

  - `std::string_view` 與 `std::span`

  - placement new 在預分配緩衝區

- **記憶體布局與對齊**

  - Cache line (64 bytes)

  - False sharing 問題與解決

  - `alignas` 指定對齊



#### 2.4 STL 容器選擇

- **序列容器性能特性**

  - `vector`: 連續記憶體,動態擴展

  - `array`: 固定大小

  - `deque`: 分段連續

- **關聯容器**

  - `map` / `unordered_map` 選擇

  - 自定義雜湊函數

- **容器選擇策略**

  - 插入/刪除頻率

  - Cache-friendly 設計



**實作練習:**

- 使用 `std::optional` 改進錯誤處理

- 使用 `std::variant` + `std::visit` 實作訊息分發

- 實作固定大小記憶體池

- 重現並解決 false sharing 問題



---



## Day 3: 編譯與鏈接



#### 3.1 編譯過程

- 預處理 → 編譯 → 組譯 → 鏈接

- 使用 `gcc -E`, `gcc -S` 查看中間結果

- 符號解析與重定位



#### 3.2 編譯器優化

- **優化等級:** `-O0`, `-O2`, `-O3`, `-Ofast`

- **特定優化:**

  - `-march=native`: 針對本機 CPU

  - `-flto`: Link Time Optimization

- **PGO (Profile-Guided Optimization)**

  - 基於實際運行數據優化



#### 3.3 函數內聯與模板

- `inline` 關鍵字的作用

- 編譯器自動內聯決策

- 強制內聯:`__attribute__((always_inline))`

- 模板實例化與外部模板



#### 3.4 靜態庫與動態庫

- **靜態庫 (.a):** 無執行期依賴,可內聯優化

- **動態庫 (.so):** 共享記憶體,易更新

- 符號可見性控制

- **HFT 選擇:** 靜態鏈接提供更好性能



#### 3.5 編譯器擴展

- **Branch prediction hints:** `__builtin_expect`, `likely()`, `unlikely()`

- **屬性:** `[[nodiscard]]`, `[[maybe_unused]]`

- **內建函數:** `__builtin_prefetch`, `__builtin_popcount`



#### 3.6 建構系統

- **CMake 基礎**

  - `CMakeLists.txt` 結構

  - `add_executable`, `target_link_libraries`

  - 編譯選項設定

- **ccache:** 編譯快取加速



**實作練習:**

- 撰寫 CMakeLists.txt 建構專案

- 使用 `likely`/`unlikely` 優化分支

- 對比不同優化等級的性能

- 使用 LTO 並測量提升



---



## Day 4-5: 深入 C++ 平行計算



### Day 4: 多執行緒基礎與同步



#### 4.1 多執行緒基礎

- `std::thread` 建立與管理

- `join()` vs `detach()`

- 執行緒參數傳遞

- `thread_local` 局部儲存

- `std::jthread` (C++20): 自動 join



#### 4.2 互斥與同步

- **Mutex 系列**

  - `std::mutex`, `std::recursive_mutex`

  - `std::shared_mutex` (C++17): 讀寫鎖

- **RAII 鎖管理**

  - `std::lock_guard`, `std::unique_lock`

  - `std::scoped_lock` (C++17): 多鎖同時上鎖

- **避免死鎖**

  - 鎖定順序策略

  - `std::scoped_lock` 同時鎖定

- **條件變數**

  - `std::condition_variable` 使用

  - 虛假喚醒與 predicate

  - 生產者-消費者模式



#### 4.3 原子操作

- **`std::atomic<T>` 基礎**

  - 支援的原子類型

  - `is_lock_free()` 檢查

- **原子操作**

  - `load`, `store`, `exchange`

  - `compare_exchange_weak` / `compare_exchange_strong` (CAS)

  - `fetch_add`, `fetch_sub` 等算術運算

- **記憶體順序 (Memory Ordering)** ⭐ 重要

  - `memory_order_relaxed`: 無同步保證

  - `memory_order_acquire` / `memory_order_release`: 獲取-釋放語義

  - `memory_order_seq_cst`: 順序一致 (預設)

  - Acquire-Release 在 SPSC 佇列中的應用

- **原子操作應用**

  - 無鎖計數器

  - 自旋鎖實作



**實作練習:**

- 實作執行緒安全的單例模式

- 使用條件變數實作生產者-消費者佇列

- 實作自旋鎖並對比 `std::mutex`

- 使用 `std::atomic` 實作無鎖計數器



---



### Day 5: 無鎖程式設計與性能優化



#### 5.1 無鎖程式設計基礎

- **為什麼需要無鎖**

  - 鎖的開銷:上下文切換、優先級反轉

  - 可預測的延遲、無死鎖

- **CAS 深入**

  - `compare_exchange_weak` vs `compare_exchange_strong`

  - ABA 問題與解決:版本號/標記

- **無鎖演算法設計原則**

  - 進度保證:無鎖、無等待



#### 5.2 無鎖數據結構 ⭐ 核心

- **無鎖棧 (Lock-Free Stack)**

  - 基於 CAS 實作

  - ABA 問題解決

- **無鎖佇列**

  - **SPSC (Single Producer Single Consumer):** 最高性能

  - MPSC, MPMC 佇列

- **環形緩衝區 (Ring Buffer)** ⭐ HFT 常用

  - 固定大小設計

  - 原子操作管理讀寫指標

  - Memory ordering 選擇

  - 在市場數據緩衝中的應用



#### 5.3 記憶體順序實戰

- **Relaxed ordering:** 計數器、統計

- **Acquire-Release:** SPSC 佇列

- **Sequential Consistency:** 何時必須使用

- 性能對比測試



#### 5.4 Cache 與並發性能 ⭐ 關鍵

- **Cache 架構**

  - L1/L2/L3 Cache

  - Cache line 大小 (64 bytes)

  - Cache 一致性協議

- **False Sharing (偽共享)** ⭐ 常見陷阱

  - 問題:多執行緒修改同一 cache line 不同變數

  - 檢測:perf c2c

  - 解決:`alignas(64)` 對齊與填充

- **Cache-Friendly 程式設計**

  - 資料局部性

  - 熱資料與冷資料分離

  - Prefetching:`__builtin_prefetch`



#### 5.5 執行緒池設計

- 工作佇列與工作執行緒

- 任務竊取 (Work Stealing)

- 執行緒數量選擇

- 實作:簡單執行緒池



#### 5.6 CPU 親和性 ⭐ HFT 必備

- **為什麼需要執行緒綁定**

  - 減少 Context Switch 和 Cache Miss

  - NUMA 架構考量

- **Linux API**

  - `pthread_setaffinity_np`

  - `sched_setaffinity`

- **在 HFT 中的應用**

  - 關鍵執行緒綁定到專用核心

  - 與 `isolcpus` 配合



**實作練習:**

- 實作無鎖棧

- 實作 SPSC 無鎖佇列

- 實作環形緩衝區並對比 mutex 保護佇列

- 重現並解決 false sharing

- 實作簡單執行緒池

- 實作執行緒綁定並測量改善



---



## Day 6-7: 深入 C++ 網路編程



### Day 6: 網路程式設計基礎



#### 6.1 Socket 程式設計

- **基本概念**

  - TCP vs UDP

  - Socket API:`socket`, `bind`, `listen`, `accept`, `connect`

  - `send` / `recv`, `sendto` / `recvfrom`

- **TCP 連接**

  - 三次握手、四次揮手

  - TIME_WAIT 狀態



#### 6.2 高性能 I/O 模型

- **阻塞 vs 非阻塞 I/O**

  - 設定:`fcntl()` 與 `O_NONBLOCK`

- **I/O 多工**

  - `select()`: 限制 1024 個 fd

  - `poll()`: 無 fd 數量限制

  - **`epoll()` (Linux)** ⭐ 核心

    - `epoll_create`, `epoll_ctl`, `epoll_wait`

    - **邊緣觸發 (ET) vs 水平觸發 (LT)**

    - 在高並發場景優勢

- **非同步 I/O**

  - `io_uring` (Linux 5.1+) 簡介



#### 6.3 高性能網路技術 ⭐ 重要

- **epoll 深入**

  - ET vs LT 模式選擇

  - Reactor 模式實作

- **零拷貝 (Zero-Copy)**

  - 傳統 I/O 的多次拷貝

  - `sendfile()`: 檔案到 socket

  - `splice()`: 管道與 socket

  - `mmap()`: 記憶體映射

- **TCP 調優** ⭐ HFT 必備

  - **`TCP_NODELAY`:** 禁用 Nagle (降低延遲)

  - `SO_REUSEADDR`, `SO_REUSEPORT`

  - `SO_SNDBUF`, `SO_RCVBUF`: 緩衝區大小

  - 系統參數:`/proc/sys/net/ipv4/tcp_*`



#### 6.4 UDP 程式設計

- UDP 特性:無連接、不可靠、低延遲

- **多播 (Multicast)** ⭐ 市場數據常用

  - `IP_ADD_MEMBERSHIP` 加入多播組

  - 在市場數據饋送中的應用



#### 6.5 網路程式設計模式

- **Reactor 模式:** 事件驅動 + epoll

- 單執行緒 vs 多執行緒 Reactor

- 主從 Reactor: 一個主執行緒接受,多個工作執行緒處理



#### 6.6 序列化

- JSON vs Protocol Buffers vs FlatBuffers

- 自定義二進制協定設計

  - 固定/可變長度

  - 長度前綴、魔數、CRC



**實作練習:**

- 實作 epoll echo 伺服器

- 實作 UDP 多播發送與接收

- 對比阻塞/非阻塞/epoll 性能

- 使用 `sendfile()` 實作零拷貝傳輸

- 調優 TCP 參數並測量延遲

- 實作 Reactor 模式框架



---



### Day 7: 網路進階與極致優化



#### 7.1 io_uring - 現代非同步 I/O

- Linux 5.1+ 引入

- Submission Queue + Completion Queue

- Shared memory ring buffer

- **優勢:** 更低系統呼叫開銷

- 實作:io_uring echo 伺服器



#### 7.2 Kernel Bypass ⭐ 極致性能

- **DPDK (Data Plane Development Kit)**

  - 繞過內核協定棧

  - Poll Mode Driver

  - 微秒甚至次微秒級延遲

- **適用場景:** 高頻交易、超低延遲應用

- **限制:** 需專用網卡、開發複雜



#### 7.3 高精度時間與時間戳 ⭐ 關鍵

- **時間的重要性**

  - 交易系統時間精度要求

  - 延遲測量與事件順序

- **Linux 時間 API**

  - `clock_gettime()` 與 Clock 類型

  - `CLOCK_MONOTONIC` vs `CLOCK_REALTIME`

- **CPU 時間戳計數器 (TSC)**

  - **`RDTSC` / `RDTSCP` 指令** ⭐ 最高精度

  - TSC 轉換為實際時間

  - 陷阱:頻率變化、多核不同步

- **網路時間同步**

  - NTP:毫秒級

  - **PTP (IEEE 1588):** 微秒甚至奈秒級

  - 硬體時間戳



#### 7.4 網路延遲優化 ⭐ 核心主題

- **延遲來源**

  - 應用層、系統呼叫、內核、網卡、網路傳輸

- **應用層優化**

  - 減少記憶體分配與拷貝

  - 預分配緩衝區與物件池

  - 無鎖數據結構

- **內核優化**

  - `TCP_NODELAY`

  - Busy polling:`SO_BUSY_POLL`

  - 執行緒與 IRQ 親和性

- **硬體優化**

  - 減少網路跳數

  - 低延遲交換機、專用網路

- **延遲測量**

  - 端到端測量

  - 分段測量識別瓶頸

  - 百分位數:P50, P99, P99.9



#### 7.5 網路程式庫

- **Boost.Asio:** 跨平台非同步 I/O

- **libuv / libev / libevent**

- **選擇建議:** 純 epoll/io_uring 最高性能



**實作練習:**

- 使用 io_uring 實作高性能伺服器

- 使用 RDTSC 實作高精度計時器

- 測量 TCP_NODELAY 對延遲的影響

- 測量端到端延遲並識別瓶頸

- 實作 busy polling



---



## Day 8: 深入 Linux 系統開發



#### 8.1 行程管理

- 行程 vs 執行緒

- `fork()`, `exec()`, `wait()`

- **行程間通訊 (IPC)**

  - 管道、訊號

  - 共享記憶體:`shm_open`, `mmap`

  - Unix Domain Socket



#### 8.2 排程與優先級 ⭐ HFT 必備

- **即時排程策略**

  - **`SCHED_FIFO`:** 先進先出,靜態優先級 ⭐

  - `SCHED_RR`: 時間片輪轉

  - `SCHED_DEADLINE`

- **設定即時優先級**

  - `sched_setscheduler()`, `pthread_setschedparam()`

  - 需要 root 或 `CAP_SYS_NICE`

- **在 HFT 應用**

  - 關鍵執行緒使用 SCHED_FIFO

  - 設定高優先級確保低延遲



#### 8.3 CPU 親和性與隔離 ⭐ 關鍵

- **CPU Affinity**

  - `sched_setaffinity()`, `pthread_setaffinity_np()`

- **CPU 隔離**

  - **`isolcpus` 內核參數:** 從排程器隔離 CPU

  - `nohz_full`: 減少定時器中斷

  - 專用 CPU 給延遲敏感任務

- **NUMA**

  - 本地 vs 遠端記憶體存取

  - `numactl` 工具



#### 8.4 記憶體管理

- **虛擬記憶體:** 頁表、頁面錯誤

- **記憶體映射:** `mmap()` 應用

- **記憶體鎖定** ⭐ 即時系統必備

  - **`mlock()` / `mlockall()`:** 鎖定記憶體防止分頁

  - 避免頁面錯誤延遲

- **Huge Pages** ⭐ 性能優化

  - 2MB 或 1GB 頁面

  - 減少 TLB miss

  - 設定:`vm.nr_hugepages`, THP

  - `mmap()` 使用:MAP_HUGETLB



#### 8.5 檔案 I/O

- 標準 I/O vs 系統呼叫 I/O

- 檔案描述符操作

- **直接 I/O:** `O_DIRECT` 繞過快取



#### 8.6 系統監控與分析 ⭐ 重要

- **`/proc` 檔案系統**

  - `/proc/cpuinfo`, `/proc/meminfo`

  - `/proc/[pid]/stat`, `/proc/sys/`

- **性能工具**

  - `top` / `htop`

  - **`perf`:** 性能分析 ⭐

    - `perf stat`, `perf record`, `perf report`

    - 火焰圖分析

- **追蹤工具**

  - `strace`: 系統呼叫追蹤

  - `ltrace`: 庫函數追蹤



#### 8.7 系統調優

- **內核參數:** `sysctl`, `/etc/sysctl.conf`

- **資源限制:** `ulimit`, `/etc/security/limits.conf`

- **電源管理**

  - CPU 頻率:performance mode

  - 禁用 C-states 提高延遲穩定性

- **中斷處理:** IRQ affinity



**實作練習:**

- 設定執行緒為 SCHED_FIFO

- 實作 CPU 親和性設定

- 使用 `mmap()` 實作記憶體映射檔案

- 使用 `mlockall()` 鎖定記憶體

- 配置 Huge Pages 並測試

- 使用 `perf` 分析性能瓶頸

- 調優系統參數



---



## Day 9: 金融交易系統特定主題



#### 9.1 FIX 協定 ⭐ 金融標準

- **FIX 概述**

  - Financial Information eXchange

  - 標籤-值對格式

- **訊息結構**

  - Header, Body, Trailer

  - 校驗和計算

- **常見訊息**

  - Logon, Heartbeat

  - New Order Single (D)

  - Execution Report (8)

  - Order Cancel Request (F)

- **高性能解析**

  - 零拷貝:直接在緩衝區操作

  - 避免字串轉換

  - 快速整數解析

  - 標籤查找優化



#### 9.2 市場數據處理 ⭐ 核心

- **市場數據類型**

  - Level 1: BBO (Best Bid/Offer)

  - Level 2: 深度報價

  - Level 3: 完整訂單簿

- **市場數據協定**

  - Binary 協定:CME MDP3, NASDAQ ITCH

  - **UDP Multicast:** 低延遲廣播

- **訂單簿實作** ⭐ 關鍵數據結構

  - **資料結構設計**

    - 價格層:std::map 或 skip list

    - 訂單佇列

  - **操作:** Add, Modify, Delete, Execution

  - **性能考量**

    - Cache-friendly 記憶體佈局

    - 物件池避免動態分配

    - 增量更新

- **數據處理管道**

  - 接收 → 解析 → 正規化 → 訂單簿維護 → 策略 → 風控 → 訂單



#### 9.3 訂單管理系統 (OMS)

- OMS 職責:生命週期管理、路由

- **訂單狀態機**

  - 狀態:New, Pending, Partial Fill, Filled, Cancelled, Rejected

  - 狀態轉換規則

- **訂單類型**

  - Market, Limit, Stop, Stop-Limit

  - Time-in-Force: GTC, IOC, FOK



#### 9.4 風險管理 ⭐ 安全關鍵

- **前置風控 (Pre-Trade Risk)**

  - 訂單限制:數量、持倉、價格偏離

  - 頻率限制

  - 資金檢查

  - **低延遲要求:** 納秒級檢查

- **風控系統設計**

  - 規則引擎

  - 快速拒絕機制



#### 9.5 交易策略

- **策略類型**

  - 市場製造 (Market Making)

  - 統計套利

  - 趨勢追蹤

- **策略引擎**

  - 事件驅動架構

  - 訊號生成



#### 9.6 延遲預算 ⭐ 系統設計核心

- **端到端延遲目標:** < 100μs

- **各組件延遲分配**

  - 網路接收:10μs

  - 協定解析:5μs

  - 訂單簿更新:5μs

  - 策略計算:20μs

  - 風控:5μs

  - 訂單組裝:5μs

  - 網路發送:10μs

  - 緩衝:40μs

- **HFT 系統架構**

  - 多執行緒:市場數據、策略、訂單發送

  - 無鎖佇列通訊

  - 執行緒綁定與隔離

- **共位 (Co-location)**

  - 伺服器放置在交易所機房

  - 減少網路延遲



**實作練習:**

- 實作 FIX 訊息解析器

- 實作高性能訂單簿

- 實作訂單狀態機

- 實作前置風控系統

- 實作簡單市場製造策略

- 測量各組件延遲



---



## Day 10: 整合實戰 - 構建高頻交易系統



### 10.1 系統需求與架構



**功能需求:**

- 接收市場數據 (UDP multicast)

- 維護訂單簿

- 執行策略 (市場製造)

- 訂單管理與發送

- 風控檢查

- 性能監控與日誌



**性能需求:**

- 端到端延遲 < 100μs

- 吞吐量 > 100,000 msg/s

- 低抖動



**系統架構:**

- **多執行緒設計**

  - 市場數據接收執行緒

  - 訂單簿維護執行緒

  - 策略執行緒

  - 訂單發送執行緒

- **無鎖通訊:** SPSC 佇列

- **共享狀態最小化**



### 10.2 核心組件實作



#### 市場數據接收模組

- UDP socket 設定

- 多播組加入

- 協定解析 (簡化二進制)

- 零拷貝技術

- 時間戳記錄



#### 訂單簿模組

- 資料結構實作

- 增量更新處理

- BBO 快速查詢

- 記憶體池管理



#### 策略引擎模組

- 簡單市場製造策略

- 訊號生成

- 參數配置



#### 風控模組

- 前置檢查實作

- 快速拒絕路徑



#### 訂單管理模組

- 生命週期管理

- 訂單發送

- 回報處理



### 10.3 性能優化 ⭐ 關鍵



**記憶體優化:**

- 物件池實作

- 避免動態分配

- Cache line 對齊

- False sharing 修正



**執行緒優化:**

- CPU 親和性設定

- 即時排程 (SCHED_FIFO)

- 記憶體鎖定 (mlockall)



**網路優化:**

- TCP_NODELAY

- Socket 緩衝區調優

- Busy polling



**程式碼優化:**

- Branch prediction hints

- 內聯關鍵函數

- 編譯器優化:-O3, -march=native, -flto



### 10.4 性能測量與分析 ⭐ 重要



**延遲測量:**

- 使用 RDTSC 高精度計時

- 端到端:市場數據 → 訂單發出

- 分段:各組件處理時間

- 統計:平均、P50、P99、P99.9、最大



**吞吐量測量:**

- 每秒訊息數

- 飽和測試



**性能剖析:**

- 使用 perf 找熱點

- 火焰圖生成

- Cache miss 分析

- 優化瓶頸



**對比測試:**

- 優化前後對比

- 不同設計選擇對比



### 10.5 日誌與監控



**日誌系統:**

- 非同步日誌:避免阻塞

- 日誌等級

- 結構化日誌



**監控指標:**

- 延遲:即時與歷史

- 吞吐量

- 訂單統計

- 系統資源



### 10.6 測試



**單元測試:**

- 各組件獨立測試

- Google Test



**整合測試:**

- 端到端流程

- 模擬數據注入



**壓力測試:**

- 高頻率數據

- 極端條件



### 10.7 部署



**編譯與打包:**

- Release build: -O3, -march=native, -flto

- 靜態鏈接

- Strip 符號表



**系統配置:**

- Huge pages

- CPU isolation (isolcpus)

- 網路參數調優

- 資源限制



### 10.8 最佳實踐總結



**程式碼品質:**

- 程式碼審查

- 文件化

- 可維護性



**常見陷阱:**

- 過早優化

- 忽略正確性

- 鎖的濫用

- 記憶體洩漏



**持續改進:**

- 性能監控

- 定期剖析

- 技術債務管理



### 10.9 延伸學習



**進階主題:**

- FPGA 加速

- 更複雜策略



**開源專案:**

- QuickFIX: FIX 引擎

- LMAX Disruptor: 高性能佇列

- Folly: Facebook C++ 庫



**書籍:**

- "High-Frequency Trading" by Irene Aldridge

- "C++ High Performance" by Björn Andrist

- "Systems Performance" by Brendan Gregg



---



## Day 10 實作專案



### 主要專案:簡化的高頻交易系統



**具體任務:**

1. 設計系統架構圖

2. 實作市場數據接收與解析

3. 實作高性能訂單簿

4. 實作簡單市場製造策略

5. 實作風控檢查

6. 實作訂單管理與發送

7. 整合各模組,實現執行緒間通訊

8. 性能優化:記憶體池、執行緒綁定、編譯器優化

9. 測量端到端延遲與吞吐量

10. 使用 perf 分析並優化瓶頸

11. 撰寫測試與文件

12. 準備部署配置



---



## 學習方法建議



### 核心原則

1. **理論與實踐結合** - 每個主題都要寫程式碼驗證

2. **測量驅動** - 優化前後都要測量,數據說話

3. **漸進式優化** - 先保證正確,再追求性能

4. **閱讀原始碼** - 學習開源專案實作技巧



### 時間分配

- 每天 8-10 小時密集學習

- 理論 40% + 實作 60%

- 重點在 Day 5 (無鎖)、Day 7 (網路優化)、Day 9-10 (整合)



### 關鍵技能優先級

⭐⭐⭐ 必須精通:

- 移動語義與零拷貝

- 記憶體池與 false sharing

- 無鎖佇列與環形緩衝區

- Memory ordering (acquire-release)

- epoll + TCP_NODELAY

- RDTSC 高精度計時

- CPU 親和性與 SCHED_FIFO

- mlockall + Huge Pages

- 訂單簿實作

- 延遲測量與優化



⭐⭐ 重要理解:

- 智能指標性能

- 模板與 concepts

- 編譯器優化 (LTO, PGO)

- io_uring

- DPDK 概念

- FIX 協定

- 風控系統



⭐ 了解即可:

- Coroutines

- 部分 C++20 特性

- TLS/SSL

- 回測引擎



---



## 評估標準



### 完成 10 天後,你應該能夠:



**技術能力:**

- [ ] 獨立實作高性能無鎖 SPSC 佇列

- [ ] 實作完整的訂單簿並達到微秒級更新

- [ ] 使用 RDTSC 精確測量納秒級延遲

- [ ] 配置系統達到 P99 < 50μs 的延遲

- [ ] 使用 perf 定位並優化性能瓶頸

- [ ] 實作完整的市場數據 → 策略 → 訂單流程



**系統理解:**

- [ ] 理解 HFT 系統的延遲預算分配

- [ ] 知道何時使用鎖、何時使用無鎖

- [ ] 理解 false sharing 並能避免

- [ ] 理解 memory ordering 的實際應用

- [ ] 知道如何系統性優化延遲



**實戰能力:**

- [ ] 能閱讀並理解開源 HFT 專案程式碼

- [ ] 能在面試中討論系統設計與優化

- [ ] 能快速定位並解決性能問題



---



## 附錄:工具與資源



### 必備工具

- 編譯器:GCC 11+ 或 Clang 14+

- 建構:CMake 3.20+

- 性能分析:perf, valgrind

- 除錯:gdb, lldb

- 版本控制:git



### 推薦環境

- OS: Ubuntu 22.04 或 RHEL 8+

- CPU: Intel Xeon 或 AMD EPYC (支援 constant_tsc)

- 網卡:支援硬體時間戳的 10GbE+



### 參考資料

- C++ Reference: https://en.cppreference.com

- Compiler Explorer: https://godbolt.org

- Quick Bench: https://quick-bench.com

- Mechanical Sympathy Blog

- Brendan Gregg's Performance Blog



---



**祝學習順利!專注於實作與測量,10 天後你將具備進入 HFT 領域的核心技能。**
"" 
