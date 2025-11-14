# 第十一章：監控與日誌

## 1. 監控架構與原理

### 1.1 Metrics Server

- **用途**：收集 Kubernetes 叢集中各節點與 Pod 的資源使用狀態（CPU、記憶體）。
- **原理**：Metrics Server 透過 Kubelet API 取得即時資源數據，供 HPA（自動水平擴展）等功能使用。
- **架構圖**：
  ```
  [Kubelet] → [Metrics Server] → [API Server] → [用戶/控制器]
  ```

### 1.2 Prometheus

- **用途**：時序數據監控與告警，支援多種 exporter。
- **原理**：Prometheus 以 pull 方式定期從各 exporter 抓取 metrics，並儲存於本地時序資料庫。
- **架構圖**：
  ```
  [Node Exporter/Pod Exporter] ← Prometheus ← Alertmanager
  ```

### 1.3 Grafana

- **用途**：可視化監控數據，支援多種資料來源（如 Prometheus）。
- **原理**：Grafana 連接 Prometheus，透過查詢語言（PromQL）繪製儀表板。

---

## 2. Logging 與 Tracing 設計

### 2.1 Logging：EFK/ELK Stack

- **EFK/ELK 組成**：
  - **Elasticsearch**：全文檢索與儲存日誌。
  - **Fluentd/Logstash**：日誌收集與轉換。
  - **Kibana**：日誌查詢與視覺化。
- **流程**：
  ```
  [Pod/Node 日誌] → [Fluentd/Logstash] → [Elasticsearch] → [Kibana]
  ```

### 2.2 Tracing：Jaeger

- **用途**：分散式追蹤，分析請求在微服務間的流向與延遲。
- **原理**：應用程式嵌入 Jaeger SDK，將 trace 資料送至 Jaeger Collector，最後儲存並可視化。

---

## 3. 部署 YAML/指令範例與常用查詢

### 3.1 Metrics Server 部署

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

- **查詢節點/Pod 資源用量**：
  ```bash
  kubectl top node
  kubectl top pod -A
  ```

---

### 3.2 Prometheus & Grafana 部署（Helm）

```bash
# 安裝 Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus

# 安裝 Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana
```

- **Prometheus 常用查詢（PromQL）**：
  - 查詢 CPU 使用率：
    ```
    sum(rate(container_cpu_usage_seconds_total{image!=""}[5m])) by (pod)
    ```
  - 查詢記憶體用量：
    ```
    sum(container_memory_usage_bytes{image!=""}) by (pod)
    ```

---

### 3.3 EFK/ELK Stack 部署（簡易範例）

```bash
# Elasticsearch
kubectl apply -f https://download.elastic.co/downloads/eck/2.11.1/all-in-one.yaml

# Fluentd DaemonSet 範例
kubectl apply -f https://raw.githubusercontent.com/fluent/fluentd-kubernetes-daemonset/master/fluentd-daemonset-elasticsearch-rbac.yaml

# Kibana
kubectl apply -f https://raw.githubusercontent.com/elastic/kibana/master/deploy/kubernetes/kibana.yaml
```

- **Kibana 查詢語法（KQL）**：
  - 查詢特定 Pod 日誌：
    ```
    kubernetes.pod_name:"my-app-pod"
    ```
  - 查詢錯誤日誌：
    ```
    log.level:"error"
    ```

---

### 3.4 Jaeger 部署（All-in-One）

```bash
kubectl create namespace observability
kubectl apply -n observability -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.54.0/jaeger-operator.yaml
```

- **Jaeger 查詢**：
  - 於 UI 依服務、trace ID、時間範圍查詢請求流向與延遲。

---

## 4. 監控與日誌管理的實務建議與最佳實踐

- **資源監控**：
  - 定期檢查 Metrics Server、Prometheus 資料正確性。
  - 設定告警（Alertmanager）以即時發現異常。
- **日誌管理**：
  - 日誌格式統一（JSON 格式佳），便於解析。
  - 設定日誌保留週期與索引管理，避免儲存爆量。
- **Tracing**：
  - 於關鍵服務導入 tracing，分析瓶頸與異常。
- **安全性**：
  - 限制監控與日誌系統的存取權限。
  - 加密敏感日誌內容。
- **高可用性**：
  - 監控與日誌服務建議多副本部署，避免單點故障。
- **自動化**：
  - 使用 Helm、Operator 等工具自動化部署與升級。
- **效能優化**：
  - 適度調整 metrics 與日誌收集頻率，平衡效能與精細度。
- **定期檢討**：
  - 定期檢查監控指標、日誌查詢規則與告警條件，確保符合實際需求。

---

本章涵蓋 Kubernetes 監控與日誌的主流方案、部署方式與實務建議，協助讀者建立穩健的維運基礎。