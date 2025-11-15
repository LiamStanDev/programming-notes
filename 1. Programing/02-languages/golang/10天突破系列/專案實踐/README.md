# 專案實踐

本目錄包含三個實戰專案，用於鞏固 10 天特訓計劃中學到的知識。

## 專案列表

### 1. [project-01-restful-api](./project-01-restful-api/)
**技術棧**: Fiber + sqlx + JWT

使用 Fiber 和 sqlx 實現生產級 RESTful CRUD 服務，包含：
- Repository/Service/Handler 分層架構
- JWT 認證中間件
- 請求驗證與錯誤處理
- 數據庫遷移
- Docker 容器化

---

### 2. [project-02-concurrent-service](./project-02-concurrent-service/)
**技術棧**: Goroutine + Channel + Context + Worker Pool

實現基於 Worker Pool 的併發服務，包含：
- Context 超時控制
- Fan-out/Fan-in 數據處理
- errgroup 錯誤收集
- 優雅關閉

---

### 3. [project-03-database-service](./project-03-database-service/)
**技術棧**: gRPC + sqlx + Protobuf

實現數據庫微服務，包含：
- gRPC 接口定義
- Protobuf 數據模型
- sqlx 高性能數據訪問
- 連接池管理

---

## 學習建議

1. **按順序完成**：從 project-01 開始，逐步增加複雜度
2. **動手實踐**：不要只閱讀代碼，親自實現每個功能
3. **擴展功能**：完成基礎功能後，嘗試添加新特性
4. **性能測試**：使用 benchmarks 測試關鍵路徑
5. **代碼審查**：對照最佳實踐檢查自己的實現

---

待完整實現...
