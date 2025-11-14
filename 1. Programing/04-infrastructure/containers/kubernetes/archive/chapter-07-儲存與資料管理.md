# 第七章：儲存與資料管理

本章將深入介紹 Kubernetes 儲存與資料管理相關的核心概念、物件定義、實務設計與最佳實踐，協助你建立穩健且可持續運作的資料持久化方案。

---

## 1. Volume、PersistentVolume、PersistentVolumeClaim、StorageClass、動態供應

### 1.1 Volume

**定義**：
Volume 是 Pod 內部可掛載的儲存空間，生命週期隨 Pod 而定。常見類型有 emptyDir、hostPath、configMap、secret 等。

**用途**：
- 暫存資料、共用檔案、存放設定檔等。
- Pod 重啟時資料會遺失（除非使用外部儲存）。

---

### 1.2 PersistentVolume (PV)

**定義**：
PersistentVolume 是叢集層級的儲存資源，由管理員預先配置或動態供應，獨立於 Pod 生命週期。

**用途**：
- 提供持久化儲存，支援 NFS、iSCSI、雲端磁碟等多種後端。
- 可被多個 Pod 透過 PVC 綁定使用。

---

### 1.3 PersistentVolumeClaim (PVC)

**定義**：
PersistentVolumeClaim 是使用者對儲存資源的請求，描述所需容量、存取模式等。

**用途**：
- Pod 透過 PVC 取得 PV，實現資料持久化。
- 支援動態供應（Dynamic Provisioning）。

---

### 1.4 StorageClass

**定義**：
StorageClass 定義不同儲存類型的供應策略（如 SSD、HDD、快照等），用於動態供應 PV。

**用途**：
- 自動化 PV 建立流程。
- 支援多種儲存後端與參數設定。

---

### 1.5 動態供應（Dynamic Provisioning）

**定義**：
當 PVC 指定 StorageClass 時，Kubernetes 會自動建立對應的 PV，無需管理員手動配置。

**用途**：
- 提升彈性與自動化程度。
- 減少人為操作錯誤。

---

## 2. StatefulSet 資料持久化設計與實務考量

### 2.1 StatefulSet 與資料持久化

- StatefulSet 適用於有狀態應用（如資料庫、分散式系統）。
- 每個 Pod 會有獨立的 PVC，確保資料不因 Pod 重建而遺失。
- PVC 命名規則：`<volume-claim-template-name>-<statefulset-name>-<ordinal>`

### 2.2 設計考量

- **儲存後端選擇**：需支援高可用、備份與還原。
- **資料一致性**：確保應用支援分散式鎖或同步機制。
- **備份策略**：定期快照與異地備份。
- **儲存擴充**：選擇可線性擴充的儲存解決方案。

---

## 3. 各物件 YAML 範例與常用 kubectl 操作指令

### 3.1 Volume 範例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo
spec:
  containers:
    - name: app
      image: busybox
      command: [ "sleep", "3600" ]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      emptyDir: {}
```

---

### 3.2 PersistentVolume (PV) 範例

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /srv/nfs
    server: 10.0.0.1
```

---

### 3.3 PersistentVolumeClaim (PVC) 範例

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

---

### 3.4 StorageClass 範例

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Delete
```

---

### 3.5 StatefulSet 搭配 PVC 範例

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
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
          image: nginx
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi
```

---

### 3.6 常用 kubectl 操作指令

```bash
# 建立 PV、PVC、StorageClass
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl apply -f storageclass.yaml

# 查詢 PV、PVC、StorageClass 狀態
kubectl get pv
kubectl get pvc
kubectl get storageclass

# 刪除 PV、PVC、StorageClass
kubectl delete pv <pv-name>
kubectl delete pvc <pvc-name>
kubectl delete storageclass <sc-name>

# 查看 PVC 綁定狀態
kubectl describe pvc <pvc-name>

# 查看 StatefulSet 及其 PVC
kubectl get statefulset
kubectl get pvc -l app=nginx
```

---

## 4. 儲存管理與資料持久化的專業建議與最佳實踐

- **選擇合適的儲存類型**：依應用需求選擇 SSD、HDD 或雲端儲存。
- **啟用動態供應**：簡化儲存管理，提升自動化效率。
- **設定適當的存取模式**：如 ReadWriteOnce、ReadOnlyMany、ReadWriteMany。
- **定期備份與測試還原**：確保資料安全與可用性。
- **監控儲存資源使用狀況**：預防容量不足或效能瓶頸。
- **遵循最小權限原則**：限制 Pod 存取敏感資料的權限。
- **規劃資料遷移與擴充策略**：確保未來可擴展性與維運彈性。
- **善用 StorageClass 參數**：如快照、加密、IOPS 等，提升資料安全與效能。

---

本章內容涵蓋 Kubernetes 儲存與資料管理的所有核心知識點，適合逐章學習與快速查詢。