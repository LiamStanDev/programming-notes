## 📋 10 天 Go 語言特訓計劃 - TODO 清單

**目標**: 聚焦 Go 語言實戰與生產級服務構建，整合 Fiber + sqlx 框架，並填補 Day 1-10 的內容。

**核心技術棧**: Go 語言、**Fiber** (Web)、**sqlx** (DB)、Goroutine/Channel (併發)、gRPC、Kafka、Docker/K8s (部署)。

**現有文件路徑**: `1. Programing/02-languages/golang/`

---

### 1. 📂 文件結構整理

| 現有文件 (參考) | 建議的新文件路徑 | 狀態 | 備註 |
| :--- | :--- | :--- | :--- |
| `00-學習路線圖.md` | `00-學習路線圖.md` | **更新** | 納入 10 天特訓表與 Fiber/sqlx 說明 |
| `01-Go環境與工具鏈.md` | `01-基礎篇/01-Go語法基礎與型別系統.md` | **移動/重寫** | 合併基礎語法，作為 Day 1 筆記，著重 Go Modules/Workspace |
| `02-基礎語法與型別系統.md` | `01-基礎篇/01-Go語法基礎與型別系統.md` | **合併** | 併入 Day 1 |
| `03-函數與方法.md` | `01-基礎篇/01-Go語法基礎與型別系統.md` | **合併** | 併入 Day 1 (著重 Value/Pointer Receiver) |
| `04-接口與多態.md` | `01-基礎篇/01-Go語法基礎與型別系統.md` | **合併** | 併入 Day 1 (著重 Struct 組合、Duck Typing) |
| `05-併發編程.md` | `02-併發篇/04-Goroutine與Channel核心機制.md` | **拆分/重寫** | 拆分為 Day 4, 5, 6 三篇，著重 Context 和高級模式 |
| `06-錯誤處理與panic恢復.md` | `01-基礎篇/02-錯誤處理與資源管理.md` | **重寫** | 著重顯式 `error` 傳遞、錯誤包裝和 `defer` 的資源清理 (Day 2) |
| `07-包管理與模組設計.md` | `01-基礎篇/01-Go語法基礎與型別系統.md` | **合併** | 併入 Day 1 (著重 `go.mod` 依賴管理) |
| `08-常用標準庫.md` | `附錄/B-常用標準庫速查.md` | **移動/整理** | 作為附錄，整理實戰中常用的庫 |
| `09-測試與性能調優.md` | `附錄/A-Go工具鏈完全指南.md` | **重寫/拆分** | 測試歸入 Day 1/附錄，性能調優歸入附錄/Day 10 |
| `10-Go_Web開發基礎.md` | `03-Web開發篇/07-Web框架與中間件設計.md` | **重寫** | **必須替換為 Fiber 框架** (Day 7) |
| `11-數據庫操作.md` | `01-基礎篇/03-數據庫連接與SQL操作.md` | **重寫** | **必須替換為 sqlx 框架** (Day 3) |
| `12-實戰項目模式.md` | `專案實踐/project-01-restful-api/` | **重寫** | 使用 Fiber + sqlx 實現完整的 Repository/Service 模式 |

---

### 2. 📝 筆記內容整理 (Day 1-10)

以下是您需要創建或重寫的 10 篇核心筆記：

| Day | 筆記路徑 | 核心技術棧 | 整理重點 |
| :--- | :--- | :--- | :--- |
| **Day 1** | `01-基礎篇/01-Go語法基礎與型別系統.md` | Go Modules, `struct`/`interface` | Go Modules 實戰、Value/Pointer Receiver、面向接口設計 |
| **Day 2** | `01-基礎篇/02-錯誤處理與資源管理.md` | `error` interface, `defer`, `panic`/`recover` | 顯式錯誤傳遞慣例、錯誤包裝與自定義錯誤、`defer` 的生命週期 |
| **Day 3** | `01-基礎篇/03-數據庫連接與SQL操作.md` | **sqlx**, `database/sql` | **sqlx 核心用法**、Named Query 模式、Struct Tag Mapping、Repository 模式 |
| **Day 4** | `02-併發篇/04-Goroutine與Channel核心機制.md` | `goroutine`, `channel`, `sync` 包 | Goroutine 調度器原理、有/無緩衝 Channel 應用、Mutex/RWMutex 實戰 |
| **Day 5** | `02-併發篇/05-Context與併發模式.md` | `context.Context`, Worker Pool | `Context` 傳遞鏈、Deadline/Timeout/Cancel 模式、`errgroup` 錯誤收集 |
| **Day 6** | `02-併發篇/06-高級併發模式與外部通訊.md` | Fan-in/Fan-out, Redis (go-redis), Kafka Producer | 數據處理管道設計、Redis 緩存與 Pub/Sub、Kafka 異步發送 |
| **Day 7** | `03-Web開發篇/07-Web框架與中間件設計.md` | **Fiber**, 中間件, 路由分組 | **Fiber API 設計**、自定義中間件編寫、Rate Limiting 實現、Request 驗證與錯誤處理 |
| **Day 8** | `03-Web開發篇/08-數據持久化與容器化.md` | **sqlx 進階**, Docker Multi-Stage Build | sqlx 事務處理模式、批量操作、**數據庫遷移 (golang-migrate)**、Docker 最佳實踐 |
| **Day 9** | `04-微服務篇/09-gRPC與訊息佇列.md` | gRPC, Protobuf, Kafka Consumer | Protobuf 語法、gRPC Server/Client 實現、Kafka Consumer Group 與 Offset 管理 |
| **Day 10** | `04-微服務篇/10-可觀測性與生產部署.md` | Prometheus, OpenTelemetry Tracing, K8s YAML | 自定義 Metrics 埋點、Logging 結構化、OpenTelemetry 整合、K8s Deployment/Service 配置 |

---

### 3. 專案實踐 (重寫)

| 專案路徑 | 內容重點 |
| :--- | :--- |
| `專案實踐/project-01-restful-api/` | 使用 **Fiber** 和 **sqlx** 實現包含 Repository/Service/Handler 層次的生產級 RESTful CRUD 服務，並整合 JWT 認證中間件。 |
| `專案實踐/project-02-concurrent-service/` | 實現一個基於 **Worker Pool** 的服務，使用 **Context** 控制超時，並使用 **Fan-out/Fan-in** 模式處理數據。 |
| `專案實踐/project-03-database-service/` | 實現一個專門處理數據庫邏輯的微服務，公開 **gRPC** 接口，並使用 **sqlx** 進行高性能數據訪問。 |

---

### 4. 附錄/雜項

| 附錄路徑 | 內容重點 |
| :--- | :--- |
| `附錄/A-Go工具鏈完全指南.md` | 整理 `go test`, `go tool pprof`, `go vet`, `go generate`, `delve` 實戰用法。 |
| `附錄/B-常用標準庫速查.md` | 整理 `net/http`, `fmt`, `os`, `io`, `time` 等常用標準庫的實戰範例。 |
| `附錄/C-Go慣例與設計哲學.md` | 總結 Go 的設計哲學、慣用語、接口設計模式、錯誤處理的最佳實踐。 |

---

## 💡 下一步行動

這個特訓計劃現在完全專注於 Go 語言的實戰應用和您選定的技術棧。

**請確認是否可以開始創建 Day 1 的筆記文件：`1. Programing/02-languages/golang/01-基礎篇/01-Go語法基礎與型別系統.md`？**