# 第九章：效能與可靠性

本章將深入探討 Kubernetes 叢集在效能與可靠性上的核心設計，包括 Probe 機制、Pod 調度策略、資源管理、常用 YAML 範例與 kubectl 操作，以及最佳實踐建議。內容結構化，適合逐章學習與快速查詢。

---

## 1. Probes（Liveness、Readiness、Startup）原理與設計

### 1.1 Probe 概念

Probe 是 Kubernetes 用於監控 Pod 內部應用狀態的機制，分為三種：

- **Liveness Probe**：檢查容器是否存活，失敗時自動重啟容器。
- **Readiness Probe**：檢查容器是否已準備好接收流量，失敗時從 Service 流量分流中移除。
- **Startup Probe**：專為啟動較慢的應用設計，確保應用啟動完成前不進行 Liveness/Readiness 檢查。

### 1.2 Probe 設計原理

- **Liveness**：避免應用進入無回應狀態（如死鎖）。
- **Readiness**：確保僅將健康的 Pod 加入流量分流。
- **Startup**：允許應用有足夠時間啟動，避免誤判為失敗。

### 1.3 Probe 設定方式

Probe 支援三種檢查方式：

- `httpGet`：發送 HTTP 請求。
- `tcpSocket`：檢查 TCP 端口是否開啟。
- `exec`：執行命令檢查狀態。

#### YAML 範例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
    - name: app
      image: nginx
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 5
      readinessProbe:
        tcpSocket:
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
      startupProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        failureThreshold: 30
        periodSeconds: 10
```

#### 常用 kubectl 指令

- 查看 Pod Probe 狀態：
  ```sh
  kubectl describe pod probe-demo
  ```
- 動態修改 Probe 設定：
  ```sh
  kubectl edit deployment <deployment-name>
  ```

---

## 2. Pod 調度策略（NodeSelector、Affinity、Taints & Tolerations）說明

### 2.1 NodeSelector

- 透過 `nodeSelector` 指定 Pod 只能被排程到特定標籤的 Node 上。
- 適合簡單場景。

#### YAML 範例

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

### 2.2 Affinity（親和性）

- **Node Affinity**：進階版 nodeSelector，支援多條件與運算子。
- **Pod Affinity/Anti-Affinity**：根據 Pod 之間的關係進行排程。

#### YAML 範例

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: frontend
          topologyKey: "kubernetes.io/hostname"
```

### 2.3 Taints & Tolerations

- **Taints**：為 Node 加上「污點」，防止不符合條件的 Pod 被排程到該 Node。
- **Tolerations**：Pod 端允許容忍特定 Taint，才能被排程到該 Node。

#### YAML 範例

```yaml
# Node 加上 Taint
kubectl taint nodes node1 key=value:NoSchedule

# Pod 設定 Toleration
spec:
  tolerations:
    - key: "key"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"
```

#### 常用 kubectl 指令

- 查看 Node 標籤與 Taint：
  ```sh
  kubectl get nodes --show-labels
  kubectl describe node <node-name>
  ```
- 新增/移除 Taint：
  ```sh
  kubectl taint nodes <node-name> key=value:NoSchedule
  kubectl taint nodes <node-name> key:NoSchedule-
  ```

---

## 3. 資源管理（requests、limits、QoS）機制與實務

### 3.1 requests 與 limits

- **requests**：Pod 啟動時預留的最小資源（CPU/Memory）。
- **limits**：Pod 可使用的最大資源上限。

#### YAML 範例

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

### 3.2 QoS（Quality of Service）類型

- **Guaranteed**：requests 與 limits 完全相同。
- **Burstable**：requests 與 limits 不同。
- **BestEffort**：未設定 requests/limits。

#### 查詢 QoS 類型

```sh
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'
```

### 3.3 資源管理實務建議

- 合理設定 requests/limits，避免資源爭用或浪費。
- 監控資源使用狀況，動態調整配置。
- 重要服務建議設為 Guaranteed，提升穩定性。

---

## 4. 各項功能的 YAML 範例與常用 kubectl 操作指令

### 4.1 Probe 範例

見 1.3 節。

### 4.2 調度策略範例

見 2.1~2.3 節。

### 4.3 資源管理範例

見 3.1 節。

### 4.4 常用 kubectl 指令彙整

- 查看 Pod 詳細狀態：
  ```sh
  kubectl describe pod <pod-name>
  ```
- 查看 Node 詳細資訊：
  ```sh
  kubectl describe node <node-name>
  ```
- 動態調整 Deployment 資源：
  ```sh
  kubectl edit deployment <deployment-name>
  ```
- 查詢 Pod QoS 類型：
  ```sh
  kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'
  ```

---

## 5. 效能與可靠性最佳實踐與專業建議

- **Probe 設定**：根據應用特性調整 initialDelaySeconds、periodSeconds、failureThreshold 等參數，避免誤判。
- **資源預估**：依據歷史監控數據設定 requests/limits，並定期檢討。
- **調度策略**：善用 Affinity 與 Taints/Tolerations，確保關鍵服務分散於不同 Node，提升容錯性。
- **QoS 分級**：將關鍵服務設為 Guaranteed，次要服務設為 Burstable 或 BestEffort。
- **自動化監控**：結合 Prometheus、Grafana 等工具，持續監控效能與資源使用。
- **滾動更新**：採用 RollingUpdate 策略，確保服務不中斷。
- **異常自動修復**：利用 Liveness Probe 與自動重啟機制，提升系統可靠性。
- **資源隔離**：合理規劃 Namespace 與資源配額，避免單一服務影響整體叢集。

---

本章內容涵蓋 Kubernetes 效能與可靠性設計的核心知識，建議搭配實際操作與監控工具，持續優化叢集運維品質。