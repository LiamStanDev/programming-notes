# Metrics、Logs、Tracing：可觀測性的三大支柱

## 一、理論解釋與設計模式

### 1. Metrics（指標）

**定義**：Metrics 是系統在運行時收集的數值型資料，通常以時間序列方式儲存。常見如 CPU 使用率、記憶體消耗、請求數量、延遲等。

**設計模式**：
- Pull-based（如 Prometheus）：監控系統定期向目標拉取指標。
- Push-based（如 StatsD）：應用主動推送指標到收集端。
- 聚合與降維：對大量原始指標進行聚合（如平均值、最大值）以減少儲存壓力。
- 分類與標籤（Labeling）：為指標加上多維度標籤，方便後續查詢與分析。

---

### 2. Logs（日誌）

**定義**：Logs 是系統或應用在運行過程中產生的文字紀錄，記錄事件、錯誤、狀態變化等資訊，通常具備時間戳與結構化（如 JSON）或非結構化格式。

**設計模式**：
- 結構化日誌：以 JSON、Key-Value 等格式記錄，便於機器解析與查詢。
- 日誌收集與集中化：利用 Log Agent（如 Filebeat、Fluentd）將日誌集中傳送至日誌平台（如 ELK）。
- 日誌輪替與儲存策略：定期壓縮、刪除或歸檔舊日誌，控制儲存成本。
- 關鍵字與模式搜尋：支援全文檢索與複雜查詢。

---

### 3. Tracing（追蹤）

**定義**：Tracing 追蹤單一請求在分散式系統中的流動路徑，記錄每個服務節點的延遲、錯誤與上下游關聯，常用於分析瓶頸與根因。

**設計模式**：
- 分散式追蹤協議（如 OpenTracing、OpenTelemetry）：統一追蹤資料格式與傳遞方式。
- Trace ID 傳遞：每個請求分配唯一 ID，跨服務傳遞以串聯完整路徑。
- 取樣（Sampling）：僅記錄部分請求以降低效能負擔。
- 可視化：以圖形化方式呈現請求路徑與延遲分布。

---

## 二、架構圖解

```mermaid
flowchart TD
    subgraph 應用服務
        A[應用程式]
        B[API Gateway]
        C[資料庫]
    end
    subgraph Metrics
        D[Metrics Exporter]
        E[Prometheus]
        F[Grafana]
    end
    subgraph Logs
        G[Log Agent]
        H[Log Storage (Elasticsearch)]
        I[Kibana]
    end
    subgraph Tracing
        J[Tracing SDK]
        K[Jaeger Collector]
        L[Jaeger UI]
    end

    A -- 指標暴露 --> D
    D -- 拉取 --> E
    E -- 查詢 --> F

    A -- 日誌輸出 --> G
    G -- 傳送 --> H
    H -- 查詢 --> I

    A -- Trace 資料 --> J
    J -- 傳送 --> K
    K -- 查詢 --> L

    B -- 呼叫 --> A
    A -- 查詢 --> C
```

---

## 三、真實世界範例

### 1. Metrics：Prometheus + Grafana

- **Prometheus**：負責定期拉取各服務的指標（如 HTTP 請求數、延遲），並以時間序列資料儲存。
- **Grafana**：連接 Prometheus，提供指標查詢與儀表板視覺化。

### 2. Logs：ELK Stack（Elasticsearch, Logstash, Kibana）

- **Filebeat/Fluentd**：收集應用日誌並傳送至 Logstash。
- **Logstash**：處理、轉換日誌資料後送入 Elasticsearch。
- **Elasticsearch**：儲存與索引日誌資料，支援高效查詢。
- **Kibana**：提供日誌搜尋、視覺化與告警功能。

### 3. Tracing：Jaeger

- **Tracing SDK**：嵌入應用程式，於每個請求產生 Trace ID 並記錄各節點資訊。
- **Jaeger Collector**：集中收集追蹤資料。
- **Jaeger UI**：可視化請求路徑、延遲分布與瓶頸分析。

---

## 四、架構師實務建議與 Trade-off 分析

### 1. Metrics

- **優點**：低儲存成本、易於聚合與長期趨勢分析，適合監控系統健康狀態。
- **缺點**：難以追蹤單一請求細節，對於突發性問題不易定位。
- **建議**：指標設計應聚焦於關鍵業務與系統資源，避免過度細分導致維護困難。

### 2. Logs

- **優點**：細節豐富，適合問題追蹤與審計。
- **缺點**：儲存與查詢成本高，需良好結構化與索引設計。
- **建議**：推行結構化日誌，並設計合理的日誌等級與輪替策略，避免雜訊與儲存爆炸。

### 3. Tracing

- **優點**：可視化單一請求全鏈路，有助於瓶頸分析與根因追蹤。
- **缺點**：實作成本高，對系統效能有一定影響，資料量大時需取樣。
- **建議**：優先在關鍵服務與跨服務調用場景導入，並結合 Metrics、Logs 形成完整可觀測性。

### 4. 綜合 Trade-off

- **整合性**：三者互補，缺一不可。Metrics 適合監控、Logs 適合追蹤細節、Tracing 適合全鏈路分析。
- **資源分配**：根據業務需求與預算，合理分配儲存與運算資源。
- **自動化與告警**：結合指標與日誌自動化告警，提升異常響應速度。
- **開源 vs 商用**：開源方案（如 Prometheus、ELK、Jaeger）彈性高但需自行維運，商用方案（如 Datadog、New Relic）則提供一站式服務但成本較高。

---

## 五、結語

Metrics、Logs、Tracing 是現代分散式系統可觀測性的三大支柱。架構師應根據系統特性與業務需求，靈活選擇與整合各種工具與設計模式，打造高效、可維運的監控體系。