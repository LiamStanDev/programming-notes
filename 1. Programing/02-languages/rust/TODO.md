# Rust 系統開發筆記 TODO

## 📊 進度總覽

基於 **Rust 1.90+ (2025)** 最新特性編寫,著重於系統開發、高性能與網路編程。

- ✅ **01-進階語言特性/** (5/5 已完成) - 2024-11-14
- ✅ **02-高性能編程/** (5/5 已完成) - 2024-11-14
- ✅ **03-併發與並行/** (5/5 已完成) - 2024-11-14
- ✅ **04-異步編程/** (6/6 已完成) - 之前完成
- ✅ **05-內存管理與不安全/** (6/6 已完成) - 2024-11-14
- ✅ **06-網路編程/** (6/6 已完成) - 之前完成
- ✅ **07-錯誤處理與除錯/** (5/5 已完成) - 2025-01-17
- ✅ **08-測試與品質保證/** (5/5 已完成) - 2025-01-17
- ✅ **09-常用高性能庫/** (5/5 已完成) - 2024-11-14
- ✅ **10-宏與元編程/** (6/6 已完成) - 2025-01-17
- ✅ **11-WebAssembly/** (3/3 已完成) - 2025-01-18
- ✅ **12-嵌入式開發/** (3/3 已完成) - 2025-01-17
- ✅ **13-實戰項目/** (5/5 已完成) - 2025-01-18

**總計**: 63/63 檔案完成 (100%) 🎉

---

## 🎯 學習前提

- ✅ 已完成 Rust 聖經 (The Rust Programming Language) 閱讀
- ✅ 熟悉所有權、借用、生命期基礎概念
- ✅ 了解基本的 Trait、泛型、錯誤處理
- 🎯 目標: 系統開發、高性能應用、網路編程實戰

---

## 🔄 待完成清單

### 01-進階語言特性/ (深化核心概念)

鞏固 Rust 聖經學到的知識,並深入語言進階特性。

- [ ] `01-錯誤處理與Result_Option模式.md`
  - **`Result<T, E>` 完整操作手冊** (基於舊筆記整合)
    - 查詢變體: `is_ok`, `is_err`, `is_ok_and`, `is_err_and`
    - 提取值: `unwrap`, `expect`, `unwrap_or`, `unwrap_or_else`, `unwrap_or_default`
    - 轉換: `ok()`, `err()`, `transpose()`
    - Ok 操作: `map`, `and_then`, `map_or`, `map_or_else`, `inspect`
    - Err 操作: `map_err`, `or`, `or_else`, `inspect_err`
    - 邏輯組合: `and`, `or`
  - **`Option<T>` 完整操作手冊**
    - 查詢: `is_some`, `is_none`, `is_some_and`
    - 提取: `unwrap`, `expect`, `unwrap_or`, `unwrap_or_else`, `unwrap_or_default`
    - 轉換: `ok_or`, `ok_or_else`, `transpose`
    - 映射: `map`, `map_or`, `map_or_else`, `and_then`, `filter`
    - 組合: `or`, `or_else`, `xor`, `zip`, `unzip`
  - `?` 運算子原理與 `Try` trait
  - 自定義錯誤類型 (實現 `std::error::Error`)
  - **thiserror** 與 **anyhow** 使用場景
  - 錯誤傳播與上下文添加 (`.context()`)
  - Rust 2024 Edition 錯誤處理新特性

- [ ] `02-生命期進階技巧.md`
  - 生命期省略規則 (Elision Rules)
  - Higher-Rank Trait Bounds (HRTB): `for<'a>`
  - `'static` 生命期深入理解
  - 生命期與閉包、Trait objects
  - Lifetime variance (協變、逆變、不變)
  - 常見生命期錯誤與解決方案

- [ ] `03-Trait系統深度解析.md`
  - Trait 進階語法: 關聯類型、默認實現、父 Trait
  - `impl Trait` vs `dyn Trait` (靜態 vs 動態分派)
  - Trait objects 的 vtable 機制與性能
  - Object Safety 規則
  - Blanket implementations
  - Marker traits: `Send`, `Sync`, `Copy`, `Sized`
  - 孤兒規則 (Orphan Rule) 與 newtype 模式
  - Trait 別名 (Rust 1.75+)

- [ ] `04-型別系統與零大小型別.md`
  - Zero-Sized Types (ZST) 優化
  - `PhantomData<T>` 使用場景
  - 型別狀態模式 (Typestate Pattern)
  - Newtype 模式與型別安全
  - `repr(Rust)` vs `repr(C)` vs `repr(transparent)`
  - Const generics 進階應用
  - GATs (Generic Associated Types, Rust 1.65+)

- [ ] `05-宏系統與元編程.md`
  - Declarative macros (`macro_rules!`)
  - Procedural macros 三種類型
    - Derive macros
    - Attribute macros
    - Function-like macros
  - 宏衛生 (Macro Hygiene)
  - 常用宏庫: `paste`, `quote`, `syn`
  - 編譯期計算與 `const fn`
  - Build scripts (`build.rs`)

---

### 02-零成本抽象與性能/ (效能優化)

深入編譯器優化原理,掌握高性能 Rust 開發技巧。

- [ ] `01-編譯器優化原理.md`
  - LLVM IR 生成過程
  - Inline 機制: `#[inline]`, `#[inline(always)]`, `#[inline(never)]`
  - 編譯器優化等級: `--release`, `opt-level`, `lto`
  - Link-Time Optimization (LTO)
  - Profile-Guided Optimization (PGO)
  - Thin LTO vs Fat LTO
  - `#[cold]`, `#[hot]` 屬性
  - 查看生成的 assembly: `cargo asm`

- [ ] `02-記憶體佈局與對齊.md`
  - 結構體記憶體佈局
  - Padding 與對齊 (Alignment)
  - `#[repr(C)]`, `#[repr(packed)]`, `#[repr(align(N))]`
  - Cache line 對齊 (64 bytes)
  - False sharing 問題與避免
  - 內存佈局可視化工具

- [ ] `03-迭代器零成本抽象.md`
  - Iterator trait 原理
  - Iterator fusion (迭代器融合)
  - 避免不必要的分配
  - `collect()` 優化技巧
  - 自定義 Iterator 實現
  - `std::iter` 工具函數
  - Rayon 並行迭代器

- [ ] `04-SIMD與向量化.md`
  - `std::simd` (Portable SIMD, stable in Rust 1.78+)
  - 手動 SIMD: `std::arch` (x86/ARM intrinsics)
  - Auto-vectorization 觸發條件
  - `#[target_feature]` 與 CPU 特性檢測
  - SIMD 數據對齊要求
  - 性能測試與對比

- [ ] `05-性能測量與分析.md`
  - **Criterion.rs**: 統計基準測試
  - **Divan**: 更快的 benchmark 框架 (Rust 1.75+)
  - `cargo bench` 使用技巧
  - Profiling 工具: `perf`, `flamegraph`, `samply`
  - `cargo-llvm-lines`: 分析編譯產物大小
  - `cachegrind` / `callgrind` 使用
  - Microbenchmark 陷阱與避免

---

### 03-併發編程/ (多線程)

掌握 Rust 的併發安全保證與高性能並發編程。

- [ ] `01-Rust併發模型.md`
  - `Send` 與 `Sync` trait 深度解析
  - Rust 記憶體模型 (C++11-like)
  - Happens-before 關係
  - Data races vs Race conditions
  - 併發安全的設計模式
  - Thread-local storage

- [ ] `02-原子操作與Memory_Ordering.md`
  - `std::sync::atomic` 原子類型
  - Memory ordering 詳解
    - `Relaxed`, `Acquire`, `Release`, `AcqRel`, `SeqCst`
  - Compare-And-Swap (CAS) 操作
  - Fetch-and-Add 與原子運算
  - ABA 問題與解決方案
  - Spinlock 實現
  - 性能對比與選擇策略

- [ ] `03-Lock-Free資料結構.md`
  - **crossbeam** 庫生態
    - `crossbeam-epoch`: Epoch-based GC
    - `crossbeam-channel`: MPMC channel
    - `crossbeam-queue`: Lock-free queue
  - Lock-free stack 實現
  - Lock-free queue 實現
  - Hazard pointers 原理
  - 記憶體回收策略
  - 實戰案例: 高性能消息隊列

- [ ] `04-執行緒同步機制.md`
  - `std::sync::Mutex<T>` 與 `RwLock<T>`
  - `Condvar` (條件變量)
  - `Barrier` 與 `Once`
  - **parking_lot** 高性能鎖
    - `parking_lot::Mutex` (更快的 Mutex)
    - `parking_lot::RwLock`
    - `parking_lot::Once`
  - Deadlock 檢測與避免
  - Lock contention 分析

- [ ] `05-並行計算框架_Rayon.md`
  - Work-stealing 調度器原理
  - Parallel iterators API
  - `par_iter()`, `par_chunks()`, `par_bridge()`
  - `join()` 與 `scope()`
  - Thread pool 配置
  - 與 async 混用策略
  - 實戰案例: 並行數據處理

---

### 04-異步編程/ (Async/Await 深度)

**Rust 最核心的部分**,基於 Tokio 1.x 與最新 async 生態。

- [ ] `01-Future與Poll機制.md`
  - `Future` trait 原理 (Rust 1.36+)
  - `Poll<T>` 狀態機
  - `Pin<T>` 與自引用結構
  - `Waker` 與 `Context`
  - Async function 的 desugaring
  - Generator 與 coroutine 關係
  - 手動實現 Future

- [ ] `02-Tokio運行時架構.md`
  - Tokio 1.x 架構 (work-stealing + 單執行緒)
  - `#[tokio::main]` 宏展開
  - Runtime 配置: `Builder`
  - `tokio::spawn` vs `tokio::task::spawn_blocking`
  - Current-thread runtime vs Multi-threaded runtime
  - Task 調度與優先級
  - Runtime metrics 監控

- [ ] `03-異步IO與事件驅動.md`
  - **mio**: Metal I/O 原理
  - epoll (Linux) / kqueue (BSD) / IOCP (Windows)
  - Reactor pattern 實現
  - `tokio::net`: `TcpListener`, `TcpStream`, `UdpSocket`
  - `AsyncRead` 與 `AsyncWrite` trait
  - Buffered I/O: `BufReader`, `BufWriter`
  - `io_uring` 支持 (tokio-uring)

- [ ] `04-異步同步原語.md`
  - `tokio::sync::Mutex` vs `std::sync::Mutex`
  - `tokio::sync::RwLock`
  - `tokio::sync::Semaphore`
  - `tokio::sync::Barrier`
  - `tokio::sync::Notify` 與 `watch` channel
  - MPSC channel: `mpsc::channel`, `mpsc::unbounded_channel`
  - Broadcast channel
  - Oneshot channel

- [ ] `05-Select與流程控制.md`
  - `tokio::select!` 宏原理
  - `tokio::join!` 與 `tokio::try_join!`
  - `tokio::time::timeout`
  - `tokio::time::interval` 與 `tokio::time::sleep`
  - Cancellation safety
  - `tokio::task::JoinSet` (Rust 1.65+)
  - Graceful shutdown 模式

- [ ] `06-異步性能優化.md`
  - 避免阻塞 runtime 的操作
  - `spawn_blocking` 使用時機
  - Task 粒度選擇
  - Channel 選型與性能對比
  - 異步鎖的開銷
  - `tokio-console` 性能分析
  - 常見反模式與解決方案

---

### 05-系統編程/ (Linux 系統調用)

深入 Linux 系統編程,掌握底層 API 使用。

- [ ] `01-系統調用與FFI.md`
  - **libc** crate: C 標準庫綁定
  - **nix** crate: 類型安全的 POSIX API
  - 直接系統調用: `syscall!` 宏
  - `ioctl` 操作
  - FFI 與 `extern "C"`
  - `bindgen` 自動綁定 C 庫
  - 零成本 FFI 技巧

- [ ] `02-進程間通信_IPC.md`
  - Pipe 與 Named pipe (FIFO)
  - Unix domain socket
  - Shared memory (`mmap` + `/dev/shm`)
  - Message queue (POSIX vs System V)
  - 性能對比與選擇策略
  - **shared_memory** crate
  - 實戰案例: 高性能 IPC 通信

- [ ] `03-信號處理.md`
  - **signal-hook** 與 **signal-hook-tokio**
  - `signalfd` 非同步信號處理
  - Signal safety 問題
  - 優雅關閉 (SIGTERM, SIGINT)
  - 信號與 async runtime 整合
  - Ctrl+C 處理最佳實踐

- [ ] `04-記憶體映射與檔案IO.md`
  - `mmap` / `munmap` 操作
  - **memmap2** crate
  - `memfd_create` 匿名記憶體映射
  - Direct I/O (O_DIRECT)
  - **io_uring** 深度剖析 (tokio-uring, glommio)
  - Zero-copy I/O 技巧
  - 性能測試與對比

- [ ] `05-系統資源管理.md`
  - CPU affinity: `core_affinity` crate
  - cgroup v2 控制
  - Resource limits (`setrlimit`)
  - Namespaces 基礎
  - Capabilities 操作
  - `/proc` 與 `/sys` 文件系統讀取
  - 系統監控指標收集

---

### 06-網路編程/ (高性能網路)

構建高性能、低延遲的網路應用。

- [ ] `01-TCP_UDP基礎與優化.md`
  - `std::net`: `TcpStream`, `TcpListener`, `UdpSocket`
  - Socket 選項: `SO_REUSEADDR`, `TCP_NODELAY`, `SO_RCVBUF`
  - 阻塞 vs 非阻塞模式
  - Keepalive 配置
  - TCP 性能調優參數
  - UDP multicast 與 broadcast
  - 實戰案例: 簡單 TCP echo server

- [ ] `02-Tokio異步網路編程.md`
  - `tokio::net::TcpListener` 深入
  - 連接管理與 backpressure
  - 優雅關閉與半關閉 (half-close)
  - TLS 支持: **tokio-rustls** (Rust-native TLS)
  - 連接池實現
  - 超時與重試策略
  - 實戰案例: 高併發 TCP 服務器

- [ ] `03-HTTP客戶端與伺服器.md`
  - **hyper 1.x**: 底層 HTTP 庫
    - Client API
    - Server API
    - HTTP/1.1 與 HTTP/2 支持
  - **reqwest**: 高階 HTTP 客戶端
    - 連接池與重用
    - 超時、重試、中間件
  - **axum**: 現代 Web 框架 (Tokio 官方)
    - Router 與 Handler
    - Extractors
    - Middleware
  - **actix-web**: 高性能 Web 框架
  - 框架選型對比

- [ ] `04-WebSocket與雙向通信.md`
  - **tokio-tungstenite**: WebSocket 實現
  - 握手與升級協議
  - 訊息幀處理 (Text/Binary/Ping/Pong)
  - Backpressure 處理
  - 心跳與重連
  - 實戰案例: WebSocket 聊天服務器

- [ ] `05-協議解析與序列化.md`
  - **nom**: Parser combinator 庫
  - **bytes**: 高效 buffer 管理
    - `Bytes`, `BytesMut`
    - Zero-copy slicing
  - 自定義二進制協議解析
  - 幀處理 (Framing)
  - 流式解析技巧
  - Length-delimited codec
  - 實戰案例: 實現簡單的 RPC 協議

- [ ] `06-網路性能優化.md`
  - Zero-copy 技術
  - Sendfile / splice (Linux)
  - Buffer pooling 策略
  - 連接池優化
  - TCP tuning 參數
  - 性能測試: `wrk`, `hey`, `oha`
  - 延遲與吞吐量權衡

---

### 07-記憶體管理進階/ (深度控制)

掌握 Unsafe Rust 與底層記憶體管理技巧。

- [ ] `01-Unsafe_Rust實戰.md`
  - Unsafe 五種操作
  - 原始指針: `*const T`, `*mut T`
  - Undefined Behavior (UB) 常見場景
  - Unsafe 邊界設計原則
  - `std::ptr` 工具函數
  - Miri: UB 檢測工具
  - 實戰案例: 實現侵入式鏈表

- [ ] `02-自定義記憶體分配器.md`
  - `GlobalAlloc` trait
  - **jemalloc**: `tikv-jemallocator`
  - **mimalloc**: `mimalloc` crate
  - **snmalloc**: 微軟分配器
  - Arena allocator 實現
  - Bump allocator
  - 分配器性能對比
  - 系統默認分配器替換

- [ ] `03-Smart_Pointer內部實現.md`
  - `Box<T>` 實現原理
  - `Rc<T>` 與 `Arc<T>` 源碼分析
  - `Weak<T>` 弱引用機制
  - Reference cycle 檢測與避免
  - `Cell<T>` 與 `RefCell<T>` 內部可變性
  - `Cow<T>`: Clone-on-Write
  - 自定義智能指針

- [ ] `04-記憶體洩漏與偵測.md`
  - RAII 與自動清理
  - `std::mem::forget` 與 `ManuallyDrop`
  - Reference cycle 導致的洩漏
  - **valgrind** 使用
  - AddressSanitizer (ASan)
  - LeakSanitizer (LSan)
  - `cargo-leak` 工具

- [ ] `05-Pin與自引用結構.md`
  - `Pin<T>` 深度解析
  - 自引用結構問題
  - `Unpin` trait
  - Pinning 規則與保證
  - `pin_project` crate
  - Future 與 Pin 的關係

---

### 08-工具鏈與生態/ (開發效率)

掌握 Rust 開發工具鏈與最佳實踐。

- [ ] `01-Cargo進階使用.md`
  - Workspace 多項目管理
  - Features 與條件編譯
  - Build scripts (`build.rs`)
  - `Cargo.toml` 完整配置
  - **cargo-make**: 任務自動化
  - **cargo-nextest**: 更快的測試運行器
  - **cargo-expand**: 宏展開查看
  - **cargo-deny**: 依賴審計

- [ ] `02-測試與屬性測試.md`
  - 單元測試與集成測試
  - `#[cfg(test)]` 與測試模組
  - **proptest**: 屬性測試 (Property-based testing)
  - **quickcheck**: 隨機測試
  - Mock 與 測試替身
  - 測試覆蓋率: **cargo-tarpaulin**
  - Snapshot testing

- [ ] `03-除錯技巧.md`
  - **rust-gdb** / **rust-lldb**
  - `RUST_BACKTRACE=1` 堆疊追蹤
  - `dbg!()` 宏使用
  - Panic hook 自定義
  - **color-backtrace** 美化輸出
  - Remote debugging
  - Core dump 分析

- [ ] `04-性能分析與追蹤.md`
  - **tracing** 框架
    - Spans 與 Events
    - Structured logging
    - Subscriber 與 Layer
  - **tracing-subscriber** 配置
  - **tokio-console**: Async runtime 監控
  - **perf** + **flamegraph**
  - **samply**: 現代性能分析器
  - **pprof** 集成

- [ ] `05-代碼品質與文檔.md`
  - **rustdoc** 文檔撰寫
  - `///` 文檔註釋與 Markdown
  - 文檔測試 (Doc tests)
  - **clippy**: Lint 工具
  - **rustfmt**: 代碼格式化
  - `.clippy.toml` 配置
  - API 設計準則
  - 版本管理與 SemVer

---

### 09-常用高性能庫/ (生態系統精選)

掌握 Rust 生態中的高性能常用庫。

- [ ] `01-序列化與反序列化.md`
  - **serde**: 序列化框架
    - `Serialize` 與 `Deserialize` trait
    - `#[derive]` 宏使用
    - 自定義序列化邏輯
  - **serde_json**: JSON 支持
  - **bincode**: 二進制序列化
  - **rmp** / **rmp-serde**: MessagePack
  - **postcard**: 嵌入式序列化
  - **capnp** / **flatbuffers**: 零拷貝序列化
  - 性能對比與選型

- [ ] `02-日誌與結構化追蹤.md`
  - **log** crate 與 facade 模式
  - **env_logger**: 簡單日誌實現
  - **tracing**: 結構化追蹤 (推薦)
    - `#[instrument]` 宏
    - Span 與 Event
  - **tracing-subscriber**: 訂閱器配置
  - **tracing-appender**: 文件輸出
  - 日誌分級與過濾
  - 與 OpenTelemetry 整合

- [ ] `03-資料庫客戶端.md`
  - **sqlx**: 異步 SQL 庫 (編譯期檢查)
    - PostgreSQL / MySQL / SQLite 支持
    - Query macro: `query!`, `query_as!`
    - 連接池管理
    - Migration 工具
  - **diesel**: ORM 框架
  - **tokio-postgres**: 原生 PostgreSQL 客戶端
  - **redis**: Redis 客戶端
    - 連接池與 cluster 支持
  - 性能對比

- [ ] `04-時間與日期處理.md`
  - `std::time`: `SystemTime`, `Instant`, `Duration`
  - **chrono**: 日期時間庫 (傳統選擇)
    - `DateTime`, `NaiveDateTime`
    - 時區處理
  - **time**: 現代替代方案 (Rust 1.70+ 推薦)
    - Type-safe 設計
    - 格式化與解析
  - **tokio::time**: 異步時間操作
    - `sleep`, `interval`, `timeout`
  - Unix timestamp 與轉換

- [ ] `05-密碼學與TLS.md`
  - **rustls**: Pure Rust TLS 實現
    - 與 tokio 整合
    - Certificate handling
  - **ring**: 密碼學原語
  - **sodiumoxide**: libsodium 綁定
  - **argon2**: 密碼雜湊
  - **sha2**, **blake3**: 雜湊函數
  - Random number generation: `rand` crate
  - 常見安全最佳實踐

- [ ] `06-實用工具庫集.md`
  - **itertools**: 迭代器擴展
  - **once_cell**: 懶初始化 (部分已穩定為 `std::sync::OnceLock`)
  - **lazy_static**: 靜態變量初始化
  - **parking_lot**: 高性能同步原語
  - **crossbeam**: 併發工具集
  - **thiserror** vs **anyhow**: 錯誤處理
  - **derive_more**: Derive 宏擴展
  - **regex**: 正則表達式

---

### 10-實戰項目/ (整合應用)

將所學知識整合為完整的系統項目。

- [ ] `01-高性能HTTP代理服務器.md`
  - 架構設計: Client ↔ Proxy ↔ Backend
  - 連接池管理
  - 請求轉發與 Header 處理
  - 負載均衡策略 (Round-robin, Least-connection)
  - 限流與熔斷
  - Metrics 與監控
  - 完整實現與測試

- [ ] `02-異步消息隊列系統.md`
  - 架構: Producer → Queue → Consumer
  - MPMC channel 實現
  - 持久化策略 (mmap / append-only log)
  - 消費者組與 offset 管理
  - At-least-once / At-most-once 語義
  - Backpressure 處理
  - 性能測試與優化

- [ ] `03-TCP長連接管理服務器.md`
  - 連接狀態機設計
  - 心跳檢測機制
  - 斷線重連策略
  - 會話管理 (Session)
  - 消息推送 (Server push)
  - 優雅關閉與資源清理
  - 實戰案例: 聊天服務器 / 遊戲服務器

- [ ] `04-低延遲數據處理管道.md`
  - Lock-free SPSC queue
  - Zero-copy buffer 傳遞
  - CPU affinity 與 NUMA 優化
  - Batch processing 技巧
  - Latency 測量與分析
  - 實戰案例: 市場數據處理 / 日誌收集

- [ ] `05-CLI工具開發.md`
  - 參數解析: **clap** (Derive API)
  - 錯誤處理: **anyhow** + **thiserror**
  - 終端輸出美化: **colored**, **indicatif**, **dialoguer**
  - 配置管理: **config**, **dotenvy**
  - 日誌記錄: **tracing**
  - 實戰案例: 完整的文件處理工具
  - 跨平台考慮與性能優化
  - 分發與打包策略

- [ ] `06-完整微服務架構.md`
  - gRPC 服務實現 (**tonic**)
  - 服務發現與註冊
  - 配置中心整合
  - 分散式追蹤 (OpenTelemetry)
  - Metrics 導出 (Prometheus)
  - 健康檢查 API
  - Docker 容器化部署
  - Kubernetes 部署 YAML

---

## 📌 寫作規範

### 格式要求
- **檔案格式**: Markdown (`.md`), UTF-8 編碼
- **語言**: 繁體中文 + 英文術語標註
- **命名**: 描述性名稱 + 數字前綴
- **換行**: Unix-style (LF)

### 內容要求
- **Rust 版本**: 基於 Rust 1.90+ (2025) 最新穩定特性
- **術語標註**: 首次出現標註英文,如 `零成本抽象 (Zero-Cost Abstraction)`
- **程式碼範例**: 完整可運行、附註釋、包含 Cargo.toml 依賴版本
- **性能數據**: Benchmark 結果、優化前後對比
- **常見陷阱**: 實戰中的 pitfalls 與解決方案
- **最佳實踐**: 社群慣例與 idioms
- **參考資料**: 每篇結尾必須標註來源

### 圖表繪製
- 使用 Mermaid 繪製流程圖、架構圖
- 字串內容使用雙引號 `""`

### 參考資料格式
```markdown
---

## 參考資料

1. [The Rust Programming Language](https://doc.rust-lang.org/book/)
2. [Rust Async Book](https://rust-lang.github.io/async-book/)
3. [Tokio Tutorial](https://tokio.rs/tokio/tutorial)
4. 《Programming Rust, 2nd Edition》 (O'Reilly, 2021)
```

---

## 🎯 推薦完成順序

基於學習曲線與實用性,建議按以下順序完成:

### 階段一: 核心強化 (優先)
1. **01-進階語言特性/** - 鞏固基礎
2. **04-異步編程/** - Rust 最獨特且重要的部分

### 階段二: 實戰擴展
3. **06-網路編程/** - 結合 async,實戰性強
4. **09-常用高性能庫/** - 快速擴充技能樹

### 階段三: 深度優化
5. **02-零成本抽象與性能/** - 性能調優
6. **03-併發編程/** - 多執行緒深度
7. **07-記憶體管理進階/** - Unsafe 與底層

### 階段四: 系統開發
8. **05-系統編程/** - Linux 系統調用
9. **08-工具鏈與生態/** - 開發效率提升

### 階段五: 整合應用
10. **10-實戰項目/** - 完整系統設計

---

## 📚 重要說明

### 與舊筆記的關係
- 舊筆記 (`bckp-20251114/`) 作為參考資料保留
- 新筆記會整合舊筆記精華,並大幅擴展深度
- **Result/Option** 操作手冊會整合到 `01-錯誤處理與Result_Option模式.md`

### 版本更新策略
- 隨 Rust 穩定版更新內容
- 標註特性穩定的版本號 (如 "Rust 1.75+")
- 優先使用穩定特性,nightly 特性需明確標註

---

## 🎉 完成狀態

所有計劃中的檔案已全部完成！這份筆記涵蓋了：
- 進階語言特性與型別系統
- 高性能編程與零成本抽象
- 併發與並行編程
- 異步編程 (Tokio)
- 內存管理與 Unsafe Rust
- 網路編程
- 錯誤處理與除錯
- 測試與品質保證
- 常用高性能庫
- 宏與元編程
- WebAssembly 開發
- 嵌入式開發
- 實戰項目 (包含 CLI 工具開發)

**接下來的學習方向**:
1. 深入實踐各個項目範例
2. 閱讀 Rust 源碼與優秀開源項目
3. 參與開源貢獻
4. 持續追蹤 Rust 新特性與生態發展

---

*最後更新: 2025-01-18*
*目標 Rust 版本: 1.90+ (2025)*
*狀態: 全部完成 ✅*
