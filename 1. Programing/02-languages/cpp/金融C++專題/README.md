# 金融 C++ 專題筆記 - 重組版

本目錄包含重新組織的 C++ 金融交易系統筆記,按主題分類,每篇筆記獨立自洽、完整可讀。

## ⭐ HFT 學習優先級

**所有筆記都已添加學習優先級標註**,幫助您高效安排學習順序:

- **⭐⭐⭐ 必看**: HFT 開發必須掌握的核心技術
- **⭐⭐ 建議**: 對 HFT 性能有重要影響的技術
- **⭐ 有空再看**: 有助於提升代碼質量的技術
- **無星**: 不太重要,可選學習

每份筆記的目錄都標註了各章節的優先級,建議按優先級順序學習。

## 已完成筆記

**01-C++核心特性.md** (35KB)

- 智能指針 (unique_ptr, shared_ptr, weak_ptr)
- **6 個實戰練習**: LinkedList, BST, Graph, OrderBook, LRU Cache, 多態系統
- 移動語義、Lambda 表達式、模板編程
- C++17/20 新特性 (結構化綁定、Concepts、Ranges)
- RAII、STL 容器與算法、第三方庫

**02-並發與性能優化.md** (~25KB)

- 多線程基礎 (std::thread, thread_local)
- 同步原語 (mutex, condition_variable)
- 原子操作與內存序 (std::atomic, CAS)
- 無鎖編程 (無鎖棧、SPSC 隊列)
- 緩存優化 (False Sharing、預取)
- 線程池、CPU 親和性、性能測量
- **異步編程與協程** (std::async, C++20 Coroutines, 異步模式)

**03-編譯與優化.md** (13KB)

- 編譯流程 (預處理、編譯、彙編、鏈接)
- 編譯器優化級別 (-O0 到 -O3/-Ofast)
- CPU 特定優化 (-march=native, SIMD)
- 鏈接時優化 (LTO)、性能引導優化 (PGO)
- 內聯優化、編譯器內建函數
- CMake 構建系統

**04-網路編程.md** (23KB)

- TCP 基礎 (客戶端、服務器、狀態轉換)
- 非阻塞 I/O
- I/O 多路復用 (select, poll, epoll)
- 零拷貝技術 (sendfile, mmap)
- TCP 調優 (TCP_NODELAY, 緩衝區)
- UDP 編程與組播
- io_uring 進階、延遲優化

**05-Linux 系統編程與調優.md** (16KB)

- 進程管理 (fork, exec, IPC)
- 實時調度 (SCHED_FIFO, 優先級)
- CPU 親和性與隔離
- 內存管理 (mlock, Huge Pages, 內存池)
- 性能監控 (perf, /proc, strace)
- 系統調優 (中斷親和性、內核參數)

**06-金融交易系統專題.md** (21KB)

- FIX 協議 (消息解析、構建)
- 市場數據處理 (Level 1/2, UDP 組播)
- 訂單簿實現 (基本版、高性能優化)
- 風險管理 (限額檢查、持倉管理)
- HFT 系統架構 (多線程設計)
- 完整系統實現 (SPSC 隊列、數據結構)

**07-常用庫與工具鏈.md** (30KB)

- **序列化**: FlatBuffers ⭐⭐⭐⭐⭐, Protocol Buffers, JSON (nlohmann/json, simdjson)
- **網路**: Boost.Asio, ZeroMQ ⭐⭐⭐⭐⭐
- **日誌**: spdlog ⭐⭐⭐⭐⭐ (異步日誌,百萬條/秒)
- **測試與基準**: Google Test, Google Benchmark
- **數據結構**: Abseil (flat_hash_map)
- **時間**: Howard Hinnant's date
- **配置**: CLI11, yaml-cpp
- **HFT 專用**: QuickFIX (FIX 協議)

## 特色亮點

### 智能指針實戰練習 (01-C++核心特性)

根據您的要求,特別加強了智能指針的實踐案例:

1. **LinkedList** - unique_ptr 實現單向鏈表
2. **Binary Search Tree** - unique_ptr 實現 BST
3. **Graph** - shared_ptr/weak_ptr 避免循環引用
4. **HFT OrderBook** - 真實場景的訂單簿
5. **LRU Cache** - 複雜數據結構管理
6. **多態系統** - 智能指針與繼承結合

每個練習都包含完整可編譯代碼、詳細註釋和使用示例。

### 符合 AGENTS.md 規範

- 繁體中文 + 英文術語標註
- 無 emoji
- 每篇筆記獨立自洽
- 包含參考資料
- 使用 Mermaid 圖表

## 推薦學習路徑

### 第 1-3 天: 基礎

- 01-C++ 核心特性 (重點練習智能指針案例)
- 03-編譯與優化基礎

### 第 4-7 天: 進階

- 02-並發與性能優化
- 04-網路編程
- 05-Linux 系統編程與調優

### 第 8-10 天: 專家

- 06-金融交易系統專題
- 整合實戰項目

## 原始文件映射

| 新文件          | 原始來源                          | 大小 |
| --------------- | --------------------------------- | ---- |
| 01-C++ 核心特性 | Day01, Day02, STL 容器, 標準庫    | 45KB |
| 02-並發與性能   | Day04, Day05 + **異步編程**       | 32KB |
| 03-編譯與優化   | Day03                             | 13KB |
| 04-網路編程     | Day06, Day07                      | 23KB |
| 05-系統編程     | Day08                             | 16KB |
| 06-金融交易系統 | Day09, Day10                      | 21KB |
| 07-常用庫工具鏈 | **新增** (FlatBuffers, spdlog 等) | 30KB |

**總計**: **7 個主題筆記**,共 **180KB** 的高質量技術內容

## 使用指南

每個筆記文件都是獨立自洽的,可以單獨閱讀和學習。文件之間有適當的交叉引用,但不依賴特定的閱讀順序。

建議通過實際編寫代碼來學習,特別是智能指針的練習案例,這將幫助您深入理解 C++ 的內存管理和 RAII 原則。
