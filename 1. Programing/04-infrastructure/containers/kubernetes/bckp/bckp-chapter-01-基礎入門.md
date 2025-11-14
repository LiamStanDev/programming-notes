# Kubernetes 基礎入門

## 1. Kubernetes 定義與發展背景

Kubernetes（簡稱 K8s）是一個開源的容器編排平台，由 Google 於 2014 年捐贈給 CNCF（Cloud Native Computing Foundation）。它自動化應用程式容器的部署、擴展與管理，解決傳統部署流程中效率低落、彈性不足等問題。Kubernetes 的設計靈感來自 Google 內部的 Borg 系統，並迅速成為雲原生應用的事實標準。

---

## 2. 核心概念

### 容器（Container）
輕量級、可攜帶的執行環境，將應用程式及其依賴封裝在一起，確保在不同環境下都能一致運作。常見技術如 Docker。

### 叢集（Cluster）
由多個節點（Node）組成的集合，Kubernetes 透過叢集管理所有資源與工作負載。

### 節點（Node）
叢集中的一台伺服器，分為「主節點（Master）」與「工作節點（Worker）」。每個節點都可運行多個 Pod。

### Pod
Kubernetes 最小的可部署單元，通常包含一個或多個容器，這些容器共享網路與儲存資源。

### Service
一種抽象，定義一組 Pod 的存取方式，實現服務發現與負載均衡。

### 編排（Orchestration）
自動化管理多個容器的部署、擴展、監控與維護。Kubernetes 提供資源調度、負載均衡、故障自動修復等功能。

---

## 3. Kubernetes 的用途與優勢

- **自動化部署與擴展**：自動化應用部署、升級與擴容，減少人為操作錯誤。
- **高可用性與自我修復**：Pod 異常時自動重啟或替換，確保服務穩定。
- **資源最佳化**：動態分配硬體資源，提高運算效率。
- **跨雲支援**：可在公有雲、私有雲或混合雲環境中運行，具高度彈性。

---

## 4. 適用場景與限制

### 適用場景
- 微服務架構應用
- 需彈性擴展的 Web 服務
- DevOps 持續整合/部署（CI/CD）流程
- 多租戶或多環境隔離需求

### 限制
- 學習曲線較陡，初期建置與維運需投入較多資源
- 小型專案或單一應用可能過於複雜
- 需搭配容器映像檔管理、網路與儲存等基礎設施

---

## 5. 學習建議、常見誤區與資源

### 學習建議
- **官方文件**：[Kubernetes 官方文件（繁體中文）](https://kubernetes.io/zh-tw/docs/)
- **互動教學**：[Katacoda Kubernetes Playground](https://www.katacoda.com/courses/kubernetes)
- **社群資源**：CNCF、Kubernetes Taiwan 社群、YouTube 教學頻道
- **實作練習**：建議從 Minikube 或 Kind 等本地環境開始，逐步熟悉核心概念與指令操作

### 常見誤區
- 誤以為 Kubernetes 能自動處理所有基礎設施（如網路、儲存），實際仍需額外配置
- 忽略資源限制（requests/limits），導致資源爭用或服務不穩
- 將所有應用直接搬上 Kubernetes，未評估是否適合容器化
- 過度依賴預設設定，忽略安全性與權限控管

---

## 6. 官方文件風格 YAML/指令範例

### 建立 Pod 的 YAML 範例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

### 建立 Deployment 的 YAML 範例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

### 使用 kubectl 建立資源

```bash
kubectl apply -f nginx-demo.yaml
kubectl apply -f nginx-deployment.yaml
```

### 查詢 Pod 狀態

```bash
kubectl get pods
```

### 查詢 Deployment 狀態

```bash
kubectl get deployments
```

### 刪除 Pod

```bash
kubectl delete pod nginx-demo
```

---

## 7. 專業工程師實務建議與最佳實踐

- **資源定義要明確**：建議為每個 Pod/Deployment 設定資源限制（requests/limits），避免資源爭用。
- **版本控管 YAML 檔案**：將所有 Kubernetes 配置檔納入版本控制（如 Git），方便追蹤與回溯。
- **善用命名空間（Namespace）**：區分不同環境（如 dev、staging、prod），提升管理效率與安全性。
- **監控與日誌**：部署 Prometheus、Grafana 等監控工具，並妥善收集日誌，便於問題追蹤。
- **自動化部署流程**：結合 CI/CD 工具（如 GitHub Actions、Jenkins）自動化部署，減少人為錯誤。
- **定期備份與測試還原**：確保關鍵資料與配置有備份機制，並定期演練還原流程。
- **權限最小化原則**：設定 RBAC，僅授權必要權限，提升安全性。
- **定期檢查映像檔安全性**：使用可信來源映像檔，並定期掃描漏洞。

---

## 8. 小結

Kubernetes 是現代雲原生架構的核心技術，雖然學習門檻較高，但能大幅提升應用部署與維運效率。建議初學者循序漸進，從基礎指令與本地環境實作開始，逐步深入理解其運作原理與最佳實踐。遇到問題時，善用官方文件與社群資源，保持持續學習的心態。
