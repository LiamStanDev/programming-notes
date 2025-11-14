# Chapter 01：基礎入門

## 1. Kubernetes 定義與核心理念

**Kubernetes**（簡稱 K8s）是一套開源的容器編排平台，主要用於自動化部署、擴展及管理容器化應用。其核心理念包括：

- **去中心化管理**：透過 API Server 統一管理資源，實現自動化與彈性調度。
- **聲明式配置**：以 YAML 檔案描述期望狀態，Kubernetes 負責維持系統達到該狀態。
- **自我修復**：自動偵測異常並重啟失效容器，確保服務高可用。
- **彈性擴展**：根據負載自動調整應用副本數量。
- **基礎設施抽象化**：屏蔽底層硬體差異，統一管理多雲或本地資源。

---

## 2. 主要功能與應用場景

### 主要功能

- **自動部署與滾動更新**：支援零停機部署與版本回滾。
- **服務發現與負載平衡**：自動分配流量至健康的 Pod。
- **資源監控與自動調整**：根據資源使用情況自動擴縮容。
- **密鑰與組態管理**：安全地管理敏感資訊與應用設定。
- **儲存編排**：動態掛載本地或雲端儲存資源。

### 應用場景

- **微服務架構**：管理大量獨立服務，實現彈性擴展與自動修復。
- **DevOps 持續交付**：自動化部署流程，提升開發與運維效率。
- **多雲/混合雲管理**：統一管理多個雲端或本地資源。
- **大數據與 AI 工作負載**：彈性調度資源，支援批次與即時運算。

---

## 3. 基本運作流程

1. **撰寫 YAML 配置檔**：描述應用（如 Deployment、Service）。
2. **提交至 Kubernetes API Server**：透過 `kubectl apply` 指令。
3. **調度與部署**：Scheduler 根據資源狀態分配 Pod 至節點。
4. **監控與自我修復**：Controller 持續監控，異常時自動重建。
5. **服務存取**：Service 負責流量導向，實現服務發現與負載平衡。

---

## 4. 官方文件風格的 YAML/指令範例

### Deployment YAML 範例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
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
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### Service YAML 範例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

### 常用 kubectl 指令

```bash
# 建立資源
kubectl apply -f deployment.yaml

# 查詢 Pod 狀態
kubectl get pods

# 查看服務
kubectl get svc

# 進入 Pod 進行除錯
kubectl exec -it <pod-name> -- /bin/sh

# 刪除資源
kubectl delete -f deployment.yaml
```

---

## 5. 專業工程師的實務建議與最佳實踐

- **資源限制與請求**：務必為 Pod 設定 `resources.requests` 與 `resources.limits`，避免資源爭用。
- **使用 Namespace 隔離環境**：將開發、測試、正式環境分開管理，提升安全性與可維護性。
- **版本控管 YAML 檔案**：將所有配置檔納入 Git 版本控管，確保可追溯與一致性。
- **監控與日誌整合**：結合 Prometheus、Grafana、ELK 等工具，強化監控與故障排查能力。
- **自動化部署流程**：建議結合 CI/CD 工具（如 GitLab CI、ArgoCD）實現自動化部署。
- **定期備份與測試還原**：定期備份重要資源與資料，並驗證還原流程。
- **最小權限原則**：Role-Based Access Control (RBAC) 僅授權必要權限，降低風險。
- **避免單點故障**：Master 與 etcd 建議多副本部署，提升高可用性。

---

> 本章節涵蓋 Kubernetes 基礎知識，建議搭配官方文件與實際操作練習，加深理解。