# 第十四章：維運與實務案例

---

## 1. 常見錯誤與排查方法

Kubernetes 維運過程中，常見問題包括 Pod 啟動失敗、服務異常、資源不足等。以下介紹三大排查工具與實務建議：

### 1.1 kubectl logs

用於查詢 Pod 內部應用程式的標準輸出/錯誤日誌。

```bash
kubectl logs my-pod -n my-namespace
kubectl logs -f my-pod -c my-container
```

**建議：**

- 先檢查應用程式日誌，確認是否有明顯錯誤訊息。
- 若 Pod 有多個 container，需加上 `-c` 指定。

---

### 1.2 kubectl describe

用於詳細檢視資源狀態、事件(Event)與錯誤訊息。

```bash
kubectl describe pod my-pod -n my-namespace
```

**建議：**

- 觀察 Events 區塊，常見如 ImagePullBackOff、CrashLoopBackOff。
- 可用於檢查 Deployment、Service、Node 等各類資源。

---

### 1.3 kubectl debug

用於進行即時除錯，進入 Pod 進行診斷。

```bash
kubectl debug pod/my-pod -it --image=busybox --target=my-container
```

**建議：**

- 適合用於容器無 shell 或需臨時工具時。
- 可用於掛載不同映像檔進行診斷。

---

### 1.4 綜合排查流程建議

1. 先用 `kubectl get pod` 檢查狀態。
2. 用 `kubectl describe` 查看事件與詳細狀態。
3. 用 `kubectl logs` 查看應用日誌。
4. 必要時用 `kubectl debug` 進行容器內部診斷。

---

## 2. 升級與版本管理策略

Kubernetes 及其元件需定期升級以獲得新功能與安全修補。升級策略如下：

### 2.1 升級流程建議

1. **備份重要資料**（如 etcd、PersistentVolume）。
2. **檢查相容性**：參考官方[升級指南](https://kubernetes.io/zh/docs/tasks/administer-cluster/cluster-upgrade/)。
3. **分階段升級**：先升級 Master，再升級 Node。
4. **滾動升級工作負載**：確保服務不中斷。

#### 範例：升級 kubeadm 叢集

```bash
# 升級 kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.28.x-00

# 驗證升級計畫
sudo kubeadm upgrade plan

# 執行升級
sudo kubeadm upgrade apply v1.28.x

# 升級 kubelet 與 kubectl
sudo apt-get install -y kubelet=1.28.x-00 kubectl=1.28.x-00
sudo systemctl restart kubelet
```

**建議：**

- 升級前後皆需驗證叢集健康狀態。
- 使用版本鎖定避免自動升級造成不穩定。

---

### 2.2 版本管理實務

- **版本規劃**：建議每年至少升級一次，避免落後官方支援週期。
- **多環境驗證**：先於 staging 測試升級，再推至 production。
- **自動化腳本**：利用 CI/CD 工具自動化升級流程。

---

## 3. 災難恢復與高可用架構設計

### 3.1 etcd 備份與還原

etcd 為 Kubernetes 關鍵資料庫，需定期備份。

#### etcd 備份

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

#### etcd 還原

```bash
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir /var/lib/etcd-from-backup
```

**建議：**

- 定期自動備份，並將備份檔異地保存。
- 測試還原流程，確保備份可用。

---

### 3.2 高可用架構設計

- **多 Master 節點**：部署 3 個以上 Master，避免單點故障。
- **外部 etcd 叢集**：將 etcd 獨立部署，提升可靠性。
- **負載平衡器**：Master 前加設 Load Balancer，確保 API Server 可用性。
- **Pod 反親和性**：設定 Pod 反親和性，避免同一服務集中於單一 Node。

#### 範例：Pod 反親和性 YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - web
              topologyKey: "kubernetes.io/hostname"
      containers:
        - name: web
          image: nginx
```

---

### 3.3 災難恢復流程建議

1. 定期備份 etcd 與重要資料。
2. 預先規劃還原步驟並文件化。
3. 定期演練災難恢復流程。
4. 監控備份任務與叢集健康。

---

## 4. 各主題 YAML/指令範例與實務建議

### 4.1 常見排查指令

```bash
kubectl get pod -A
kubectl describe node <node-name>
kubectl get events --sort-by='.lastTimestamp'
```

### 4.2 範例：Deployment 滾動升級

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:2.0
```

**建議：**

- 使用 RollingUpdate 策略，確保升級不中斷。
- 設定 maxSurge、maxUnavailable 平衡可用性與升級速度。

---

### 4.3 範例：Pod 健康檢查

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

**建議：**

- 正確設置 liveness/readiness probe，提升服務穩定性。
- 定期檢查 probe 設定是否符合應用需求。

---

### 4.4 實務建議總結

- 建立標準化維運流程與文件。
- 定期檢查叢集健康與資源使用。
- 善用監控、告警與自動化工具。
- 持續學習官方文件與社群最佳實踐。

---
