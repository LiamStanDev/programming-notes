# Chapter 04：Kubernetes 基本物件

本章將介紹 Kubernetes 中最核心的三大基本物件：Pod、Service、Namespace。內容包含定義、用途、YAML 範例、常用 kubectl 指令、物件間關聯、實務設計建議與最佳實踐，適合逐章學習與快速查詢。

---

## 1. Pod

### 1.1 定義與用途

- **Pod** 是 Kubernetes 中最小的可部署單位，通常包含一個或多個緊密相關的容器（Container）。
- 同一個 Pod 內的容器共享網路（IP、Port）與儲存（Volume）。
- 適用於需要緊密協作的多容器應用（如 sidecar 模式）。

### 1.2 YAML 範例

```yaml
apiVersion: v1 # api 文檔版本
kind: Pod # 資源類別
metadata: # 描述 Pod 的數據
  name: nginx-demo # Pod 名稱
  labels: # (opt) 定義 Pod 標籤，爲 key-value
    app: nginx
    team: liam-dev
  namespace: default # 命名空間
spec: # 規格
  containers: # pod 內部容器
    - name: nginx # 容器名稱
      image: nginx:1.29.1 # 指定鏡像
      imagePullPolicy: IfNotPresent # 鏡像拉取策略: Never, Always, IfNotPreset
      command: # (opt)容器啟動命令
        - nginx
        - -g
        - "daemon off;"
      workingDir: /usr/share/nginx/html # (opt) 定義容器啟動後的工作目錄
      ports:
        - name: http # (opt) 指定端口名稱
          containerPort: 80 # 描述容器內部要暴露的 Port
          protocol: TCP # 端口協議: TCP, UDP
      env: # 容器環境變數列表
        - name: JVP_OPTS # 環境變數名
          value: "-Xms128m -Xmx128m"  # 對應環境變數值
      resources: # 定義容器資源
        requests:  # 最少需要資源
          cpu: 100m # 限制 cpu 最多使用 0.1 核心，1000m 表示 1 核心
          memory: 128Mi # 限制內存最多使用 128 MB
        limits: # 最多使用
          cpu: 200m
          memory: 256Mi
  restartPolicy: Always # 定義 pod 重啟策略: Always, OnFailure (錯誤退出才重啟), Never (永不重啟)
```

### 1.3 常用 kubectl 指令

- 建立 Pod：
  `kubectl apply -f pod.yaml`
- 查詢 Pod 狀態：
  `kubectl get pods`
- 查看 Pod 詳細資訊：
  `kubectl describe pod my-nginx`
- 刪除 Pod：
  `kubectl delete pod my-nginx`

---

## 2. Service

### 2.1 定義與用途

- **Service** 提供一組 Pod 的穩定網路存取入口，解決 Pod IP 會變動的問題。
- 常見類型：ClusterIP（預設）、NodePort、LoadBalancer、ExternalName。
- 透過 Label Selector 將流量導向對應 Pod。

### 2.2 YAML 範例

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

### 2.3 常用 kubectl 指令

- 建立 Service：
  `kubectl apply -f service.yaml`
- 查詢 Service 狀態：
  `kubectl get svc`
- 查看 Service 詳細資訊：
  `kubectl describe svc nginx-service`
- 刪除 Service：
  `kubectl delete svc nginx-service`

---

## 3. Namespace

### 3.1 定義與用途

- **Namespace** 用於將 Kubernetes 叢集中的資源分隔成多個虛擬群組，適合多租戶或環境隔離（如 dev、test、prod）。
- 預設有 `default`、`kube-system`、`kube-public` 等命名空間。

### 3.2 YAML 範例

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

### 3.3 常用 kubectl 指令

- 建立 Namespace：
  `kubectl apply -f namespace.yaml`
- 查詢 Namespace：
  `kubectl get ns`
- 指定 Namespace 操作：
  `kubectl get pods -n dev`
- 刪除 Namespace：
  `kubectl delete ns dev`

---

## 4. 物件間的關聯與設計建議

### 4.1 關聯說明

- **Pod** 是應用的運行單位，**Service** 負責將流量導向一組 Pod。
- **Namespace** 則將 Pod、Service 等資源分組隔離，便於權限控管與資源管理。
- Service 透過 Label Selector 綁定 Pod，Namespace 則作為資源的作用域。

### 4.2 實務設計建議

- **命名規範**：資源名稱應具描述性，便於維運。
- **Label 與 Selector**：善用 Label 管理與選取 Pod，提升彈性與可維護性。
- **Namespace 規劃**：依據團隊、專案或環境劃分 Namespace，避免資源混用。
- **資源隔離**：利用 Namespace 結合 RBAC 實現權限與資源隔離。

---

## 5. 專業工程師的最佳實踐

- **版本控管 YAML**：所有資源定義檔應納入版本控管（如 Git）。
- **自動化部署**：結合 CI/CD 工具自動化部署與管理。
- **監控與日誌**：部署監控（如 Prometheus）與日誌收集（如 ELK）確保可觀察性。
- **資源限制**：為 Pod 設定資源限制（CPU/Memory），避免資源爭用。
- **定期檢查與清理**：定期檢查未使用的資源，保持叢集整潔。
- **文件化**：維護清晰的文件，方便團隊協作與新成員上手。

---

> 本章重點：熟悉 Pod、Service、Namespace 的定義、操作與設計原則，是進階 Kubernetes 實戰的基礎。