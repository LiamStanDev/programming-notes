# C++ 學習筆記 TODO

## 📊 進度總覽

- ✅ **01-核心語言特性/** (4/4 完成)
- ✅ **02-高性能編程/** (5/5 完成)
- ✅ **03-併發與並行/** (5/5 完成)
- ✅ **04-網路編程/** (5/5 完成)
- ⏳ **05-系統編程/** (0/5 待完成)
- ⏳ **06-工具鏈與調試/** (0/5 待完成)
- ⏳ **07-標準庫與第三方庫/** (0/5 待完成)
- ⏳ **08-實戰模式/** (0/5 待完成)

**總計**: 19/34 檔案完成 (55.9%)

---

## 🔄 待完成清單

### 05-系統編程/ (System Programming)

此部分延續網路編程,專注於 Linux 系統層面的高效能技術,對 HFT 系統至關重要。

- [ ] `01-系統調用與性能.md`
  - 系統調用基礎與開銷分析
  - vDSO (Virtual Dynamic Shared Object) 優化
  - 常用系統調用性能比較 (read/write/mmap/io_uring)
  - 系統調用追蹤與分析 (strace, perf)

- [ ] `02-進程間通信_IPC.md`
  - IPC 機制對比 (Pipe, Message Queue, Shared Memory, Socket)
  - POSIX vs System V IPC
  - 共享記憶體 (Shared Memory) 實現與最佳實踐
  - Memory-Mapped Files (mmap)
  - HFT 場景: Feed Handler 與 Trading Engine 通信

- [ ] `03-CPU親和性與調度.md`
  - CPU Affinity (sched_setaffinity)
  - NUMA 架構與最佳化
  - Real-time Scheduling (SCHED_FIFO, SCHED_RR)
  - cgroup 與資源隔離
  - HFT 場景: 核心綁定與延遲優化

- [ ] `04-信號處理與異常安全.md`
  - Signal Handling 機制
  - async-signal-safe 函數
  - signalfd 與現代信號處理
  - Core Dump 分析
  - 生產環境異常處理策略

- [ ] `05-Linux特定優化技術.md`
  - Huge Pages (2MB/1GB Pages)
  - Transparent Huge Pages (THP) 利弊
  - CPU Isolation (isolcpus)
  - IRQ Affinity 與中斷優化
  - /proc 與 /sys 調校參數
  - HFT 系統調校 Checklist

---

### 06-工具鏈與調試/ (Toolchain & Debugging)

開發高效能 C++ 系統必備的工具與技巧。

- [ ] `01-CMake構建系統.md`
  - CMake 基礎語法與專案結構
  - 現代 CMake 最佳實踐 (target-based)
  - 第三方庫整合 (find_package, FetchContent)
  - 編譯選項與優化配置
  - 跨平台構建技巧

- [ ] `02-編譯器標誌與優化.md`
  - GCC vs Clang 優化差異
  - 優化等級 (-O0/-O1/-O2/-O3/-Ofast)
  - LTO (Link-Time Optimization)
  - PGO (Profile-Guided Optimization)
  - 編譯器內建函數 (Built-ins)
  - 編譯選項對延遲的影響

- [ ] `03-性能分析工具.md`
  - perf: CPU profiling, cache analysis
  - valgrind: cachegrind, callgrind
  - gdb: 調試技巧與腳本
  - 火焰圖 (Flame Graphs)
  - Intel VTune, AMD uProf
  - HFT 場景: 微秒級性能瓶頸定位

- [ ] `04-記憶體與並發檢測.md`
  - AddressSanitizer (ASan): 記憶體錯誤檢測
  - ThreadSanitizer (TSan): 數據競爭檢測
  - UndefinedBehaviorSanitizer (UBSan)
  - LeakSanitizer (LSan)
  - Helgrind, DRD (Valgrind 工具)
  - Sanitizer 對效能的影響

- [ ] `05-基準測試框架.md`
  - Google Benchmark 使用指南
  - 基準測試設計原則
  - 結果分析與統計顯著性
  - 微基準測試陷阱
  - Catch2, doctest 單元測試
  - CI/CD 整合性能回歸測試

---

### 07-標準庫與第三方庫/ (Standard & Third-Party Libraries)

深入理解 STL 性能特性,並掌握高效能第三方庫。

- [ ] `01-STL容器性能特性.md`
  - 各容器時間複雜度與記憶體佈局
  - vector vs deque vs list 選擇策略
  - unordered_map/set 實現與性能
  - flat_map/flat_set (C++23)
  - 容器適配器 (stack, queue, priority_queue)
  - 小物件優化 (SSO, Small Buffer Optimization)

- [ ] `02-自定義記憶體分配器.md`
  - Allocator 概念與介面
  - std::allocator vs 自定義分配器
  - Memory Pool 實現
  - Arena Allocator
  - PMR (Polymorphic Memory Resources, C++17)
  - HFT 場景: 減少記憶體碎片與分配延遲

- [ ] `03-Boost庫精選.md`
  - Boost.Asio: 異步網路框架
  - Boost.Lockfree: Lock-free 容器
  - Boost.Intrusive: 侵入式容器
  - Boost.Circular_Buffer
  - Boost.Interprocess: 共享記憶體
  - 何時選擇 Boost vs STL

- [ ] `04-高性能第三方庫.md`
  - Abseil (Google): 基礎庫
  - Folly (Meta): 高性能組件
  - fmt: 格式化庫 (C++20 std::format 基礎)
  - spdlog: 高性能日誌
  - simdjson: SIMD JSON 解析
  - HFT 常用庫選型

- [ ] `05-序列化庫.md`
  - Protocol Buffers (protobuf)
  - FlatBuffers: 零拷貝序列化
  - Cap'n Proto
  - MessagePack
  - SBE (Simple Binary Encoding): 低延遲金融協議
  - 性能對比與選擇策略

---

### 08-實戰模式/ (Practical Patterns)

將前述技術整合為完整的 HFT 系統設計模式。

- [ ] `01-低延遲設計模式.md`
  - 消除動態記憶體分配
  - 物件池 (Object Pool) 模式
  - Disruptor 模式 (LMAX Architecture)
  - Lock-free 架構設計
  - 消息傳遞 vs 共享狀態
  - 延遲預算分配 (Latency Budget)

- [ ] `02-市場數據處理管線.md`
  - Market Data Feed Handler 架構
  - 多播接收與重組
  - 數據規範化 (Normalization)
  - Order Book 維護策略
  - 快照與增量更新
  - 完整實作範例

- [ ] `03-交易系統架構.md`
  - Trading Engine 設計
  - Order Management System (OMS)
  - Risk Management 整合
  - Pre-trade 與 Post-trade Checks
  - FIX Protocol 處理
  - 狀態機設計模式

- [ ] `04-錯誤處理與可靠性.md`
  - 錯誤處理策略 (exceptions vs error codes vs std::expected)
  - Circuit Breaker 模式
  - Graceful Degradation
  - 災難恢復 (Disaster Recovery)
  - 日誌與監控最佳實踐
  - 生產環境故障案例分析

- [ ] `05-部署與運維.md`
  - 編譯與發布流程
  - 系統調校 Checklist
  - 熱更新策略
  - 效能監控指標
  - 容量規劃
  - 生產環境最佳實踐

---

## 📌 備註

- **目標受眾**: 具備 C# 與 Rust 經驗,專注於高頻交易系統開發
- **內容定位**: HFT 導向的實戰技術,強調極致性能與低延遲
- **寫作風格**: 
  - 繁體中文 + 英文術語標註
  - 大量程式碼範例
  - 每篇包含 HFT 實際應用場景
  - 每篇附完整參考資料

## 🎯 下一步建議

**優先完成順序**:
1. **05-系統編程/** - 延續網路編程,對 HFT 系統至關重要
2. **06-工具鏈與調試/** - 提升開發與性能分析能力
3. **07-標準庫與第三方庫/** - 擴充技術工具箱
4. **08-實戰模式/** - 整合所有知識為完整系統

---

*最後更新: 2025-11-14*
