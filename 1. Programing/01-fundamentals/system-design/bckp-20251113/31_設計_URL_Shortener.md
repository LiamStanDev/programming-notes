# URL Shortener 系統設計

## 1. 需求分析與理論解釋

### 1.1 需求分析
- 將長網址轉換為短網址，方便分享與管理。
- 支援高併發短網址查詢與跳轉。
- 確保短網址唯一性與不可預測性。
- 支援統計分析（如點擊次數、來源追蹤）。
- 可擴展性與高可用性。

### 1.2 核心挑戰
- **唯一性**：短網址需全域唯一，避免碰撞。
- **高可用性**：系統需能承受大量查詢與寫入流量。
- **一致性**：分散式部署下，短網址生成與查詢需保持一致。
- **延展性**：系統需能隨需求水平擴展。
- **安全性**：防止惡意攻擊（如暴力破解、濫用）。

### 1.3 常見設計模式
- **Base62/Hash 編碼**：將自增 ID 或隨機值轉為短字串。
- **資料庫分片**：分散儲存短網址對應關係。
- **快取（Cache）**：提升查詢效能，減少資料庫壓力。
- **API Gateway**：統一流量入口，便於管理與擴展。
- **CDN**：加速靜態內容與跳轉頁面。

---

## 2. 架構圖

```mermaid
flowchart TD
    A[流量入口<br>API Gateway/Load Balancer] --> B[短網址生成服務]
    A --> C[短網址查詢服務]
    B --> D[唯一ID生成器<br>(如Snowflake/DB自增)]
    B --> E[資料儲存<br>主資料庫]
    C --> F[快取層<br>Redis/Memcached]
    C --> E
    E <--> G[統計分析/日誌]
    F -. cache miss .-> E
```

- **流量入口**：API Gateway 或 Load Balancer，統一管理請求。
- **短網址生成服務**：負責產生唯一短碼，寫入資料庫。
- **短網址查詢服務**：根據短碼查詢原始網址，並執行跳轉。
- **唯一ID生成器**：確保短碼唯一（可用 Snowflake、DB 自增、UUID）。
- **資料儲存**：主資料庫（如 MySQL、PostgreSQL、NoSQL）。
- **快取層**：提升查詢效能，減少資料庫壓力。
- **統計分析/日誌**：記錄點擊、來源等資訊。

---

## 3. API/資料流設計與真實世界範例

### 3.1 API 設計

#### 3.1.1 產生短網址 API

- **Endpoint**：`POST /api/shorten`
- **Request**
  ```json
  {
    "long_url": "https://example.com/very/long/url"
  }
  ```
- **Response**
  ```json
  {
    "short_url": "https://sho.rt/abc123"
  }
  ```

#### 3.1.2 查詢短網址 API（跳轉）

- **Endpoint**：`GET /{short_code}`
- **流程**：
  1. 查詢快取（Redis）是否有對應原始網址。
  2. 若無，查詢資料庫並回寫快取。
  3. 執行 301/302 跳轉。

### 3.2 資料表設計（以 MySQL 為例）

```sql
CREATE TABLE short_urls (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    click_count BIGINT DEFAULT 0
);
```

### 3.3 Redis 快取配置

- **Key**：`short:{short_code}`
- **Value**：`long_url`
- **TTL**：可依流量與熱度設定（如 1 天）

### 3.4 資料流範例

1. 使用者呼叫 `/api/shorten`，服務產生唯一短碼並寫入資料庫。
2. 使用者訪問短網址 `/abc123`，API 先查 Redis，命中則直接跳轉，未命中則查資料庫並回寫快取。
3. 點擊行為可異步寫入統計分析系統。

---

## 4. 架構師實務建議與 Trade-off 分析

### 4.1 唯一性
- **方案**：自增 ID、分散式 ID（Snowflake）、隨機碼（Hash+碰撞檢查）。
- **Trade-off**：自增 ID 易於實作但有預測性，分散式 ID 複雜但可水平擴展，隨機碼需處理碰撞。

### 4.2 可用性
- **方案**：多活部署、快取、資料庫主從。
- **Trade-off**：多活需解決資料同步與一致性，快取提升效能但需考慮失效策略。

### 4.3 擴展性
- **方案**：服務無狀態化、資料庫分片、快取分散。
- **Trade-off**：分片需設計分片鍵與路由邏輯，快取分散需考慮一致性哈希。

### 4.4 成本
- **方案**：依流量選擇雲端服務或自建，快取與資料庫資源彈性擴充。
- **Trade-off**：雲端成本隨流量浮動，自建需考慮維運與擴容。

### 4.5 複雜度
- **方案**：初期可單體部署，流量成長後逐步微服務化。
- **Trade-off**：單體架構易於維護但擴展有限，微服務架構彈性高但開發與維運複雜。

---

## 5. 結論

URL Shortener 系統設計需兼顧唯一性、高可用、可擴展與成本控制。建議初期以簡單架構快速上線，隨流量成長逐步引入快取、分片與分散式 ID 等進階設計，並持續監控與優化系統效能與穩定性。