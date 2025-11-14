# 第五章：工作負載管理

## 1. 工作負載物件定義、用途與差異

### Deployment
- **定義**：用於管理無狀態應用的部署與升級，確保指定數量的 Pod 持續運行。
- **用途**：自動滾動更新、回滾、擴縮容，適合 Web 服務等 stateless 應用。
- **差異**：不保證 Pod 名稱與順序，適合大多數一般服務。

### ReplicaSet
- **定義**：確保指定數量的 Pod 副本持續運行。
- **用途**：通常由 Deployment 管理，單獨使用較少。
- **差異**：無升級、回滾等高階功能，僅維持副本數。

### StatefulSet
- **定義**：管理有狀態應用，提供穩定的 Pod 名稱、順序與持久化儲存。
- **用途**：資料庫、分散式系統等需穩定身分的服務。
- **差異**：Pod 有固定名稱與順序，支援持久化 Volume。

### DaemonSet
- **定義**：確保每個節點上都運行一個 Pod。
- **用途**：節點監控、日誌收集等需全節點部署的服務。
- **差異**：每個節點僅一個 Pod，節點增減自動調整。

### Job
- **定義**：一次性任務，確保指定數量的 Pod 成功執行結束。
- **用途**：批次處理、資料遷移等短暫任務。
- **差異**：執行完畢即結束，不會自動重啟。

### CronJob
- **定義**：定時排程執行 Job。
- **用途**：定時備份、排程任務等。
- **差異**：類似 Linux cron，定時產生 Job 執行。

---

## 2. 自動擴縮容機制

### HPA（Horizontal Pod Autoscaler）
- **說明**：根據 CPU、記憶體等指標自動調整 Pod 數量。
- **用途**：應對流量高峰，提升資源利用率。
- **常見指標**：CPU 使用率、記憶體、Custom Metrics。

### VPA（Vertical Pod Autoscaler）
- **說明**：自動調整 Pod 的 CPU/記憶體資源限制。
- **用途**：動態調整單一 Pod 資源，減少資源浪費。
- **注意事項**：調整時會重啟 Pod，適合非即時服務。

---

## 3. YAML 範例與常用 kubectl 指令

### Deployment 範例
```yaml
apiVersion: apps/v1 # deployment api 版本
kind: Deployment # 資源類別
metadata:
  name: nginx-deploy # deployment 名稱
  namespace: default # 所在的命名空間
  labels: # deployment 標籤(與 Pod 無關)
    app: nginx-deploy
spec:
  selector: # 選擇器
    matchLabels: # 按照標籤匹配
      app: nginx-deploy # 匹配哪些 Pod 標籤
      
  replicas: 1 # 期望副本數量
  
  revisionHistoryLimit: 10 # 進行滾動更新後，保留歷史版本數 (保存十個版本)
  strategy: # 更新策略
    type: RollingUpdate # 指定更新策略 e.g. Recreate, RollingUpdate
    rollingUpdate: # 滾動更新策略
      maxSurge: 25% # 滾動更新時，更新的個數最多能為原本的多少比例(新的版本會多建立)
      maxUnavailable: 25% # 滾動更新時最大不可用比例
      
  # Pod 模板    
  template: 
    metadata: # Pod 元信息
      labels: # Pod 的標籤 (需要對應 selector 中的規則)
        app: nginx-deploy
    spec: # Pod 規格
      containers:
        - image: nginx:1.29
          imagePullPolicy: IfNotPresent
          name: nginx
      restartPolicy: Always
```
**常用指令**
- 建立：`kubectl apply -f deployment.yaml`
- 查詢：`kubectl get deployments`
- 滾動：
	- 歷史：`kubectl rollout history deployment/nginx-deploy `
	- 停止：`kubectl rollout pause deployment/nginx-deploy`
	- 重啓：`kubectl rollout resume deployment/nginx-deploy`
	- 回退：
		- 上一版：`kubectl rollout undo deployment/nginx-deploy`
		- 指定版本：`kubectl rollout undo deployment/nginx-deploy --to-revision=2`
> 這邊建議回滾還是建立一個 deployment yaml (`nginx-deployment-2024-05-06-toversionv3.yaml` )，然後使用 apply 會更好

---

### StatefulSet 範例
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 2
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-pvc
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-pvc
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```
**常用指令**
- 建立：`kubectl apply -f statefulset.yaml`
- 查詢：`kubectl get statefulsets`
- 擴容：`kubectl scale statefulset mysql --replicas=3`

---

### DaemonSet 範例
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.16
```
**常用指令**
- 建立：`kubectl apply -f daemonset.yaml`
- 查詢：`kubectl get daemonsets`
- 刪除：`kubectl delete daemonset fluentd`

---

### Job 範例
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-job
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```
**常用指令**
- 建立：`kubectl apply -f job.yaml`
- 查詢：`kubectl get jobs`
- 查詢 Pod：`kubectl get pods --selector=job-name=pi-job`
- 查看結果：`kubectl logs <pod-name>`

---

### CronJob 範例
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```
**常用指令**
- 建立：`kubectl apply -f cronjob.yaml`
- 查詢：`kubectl get cronjobs`
- 查詢 Job：`kubectl get jobs --watch`
- 查詢 Pod：`kubectl get pods --selector=job-name=hello`

---

### HPA 範例
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```
**常用指令**
- 建立：`kubectl apply -f hpa.yaml`
- 查詢：`kubectl get hpa`
- 監控：`kubectl describe hpa nginx-hpa`

---

## 4. 工作負載設計與資源調度實務建議

- **選擇合適的工作負載物件**：無狀態服務用 Deployment，有狀態服務用 StatefulSet，需全節點部署用 DaemonSet，批次任務用 Job/CronJob。
- **資源限制與請求**：每個 Pod 建議設置 `resources.requests` 與 `resources.limits`，避免資源爭用或浪費。
- **自動擴縮容**：善用 HPA/VPA，根據實際負載自動調整資源，提升彈性與效率。
- **滾動更新與回滾**：Deployment/StatefulSet 支援滾動更新，建議搭配 `readinessProbe`、`livenessProbe` 提升可用性。
- **持久化儲存**：StatefulSet 搭配 PVC，確保資料不因 Pod 重建而遺失。
- **監控與日誌**：部署 DaemonSet 進行全節點監控與日誌收集，提升可觀測性。
- **命名規則與標籤管理**：統一命名與標籤，方便資源查詢與管理。
- **資源調度策略**：可利用 nodeSelector、affinity、taints/tolerations 等進階調度功能，提升資源利用率與服務穩定性。

---

> 本章涵蓋 Kubernetes 工作負載管理的核心知識，適合逐章學習與快速查詢，建議搭配實際操作加深理解。