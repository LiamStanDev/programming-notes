# 第十章：進階功能

## 1. Operators

### 原理與用途
Operators 是一種利用 Kubernetes API 擴充自動化運維的模式，將人類操作流程程式化，讓複雜應用（如資料庫、訊息佇列）能自動部署、升級、備份與修復。

- **原理**：基於自定義控制器（Controller）與自定義資源（CRD），監控資源狀態並自動執行操作。
- **用途**：自動化應用生命週期管理（如自動備份、故障修復、升級等）。

### YAML 範例
```yaml
apiVersion: "etcd.database.coreos.com/v1beta2"
kind: "EtcdCluster"
metadata:
  name: "example"
spec:
  size: 3
  version: "3.2.13"
```

### 常用操作指令
```shell
kubectl apply -f etcd-cluster.yaml
kubectl get etcdclusters
```

---

## 2. CRD（Custom Resource Definition）

### 原理與用途
CRD 讓使用者自定義 Kubernetes API 物件，擴充原生資源類型，讓控制器能監控並管理這些自定義物件。

- **原理**：註冊新資源類型到 API Server，並可搭配自定義控制器進行自動化管理。
- **用途**：擴充 Kubernetes 功能，支援特殊業務需求。

### YAML 範例
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: Crontab
    shortNames:
    - ct
```

### 常用操作指令
```shell
kubectl apply -f crd.yaml
kubectl get crd
kubectl get crontabs
```

---

## 3. Admission Controller

### 原理與用途
Admission Controller 是一組在 API Server 內部運作的攔截器，於資源請求進入叢集時進行驗證、修改或拒絕。

- **原理**：分為 Mutating（可修改請求）與 Validating（僅驗證請求）兩類。
- **用途**：強化安全性、政策控管、自動補全欄位等。

### YAML 範例（ValidatingWebhookConfiguration）
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: example-validator
webhooks:
  - name: validate.example.com
    clientConfig:
      service:
        name: example-service
        namespace: default
        path: "/validate"
      caBundle: <CA_BUNDLE>
    rules:
      - apiGroups: ["apps"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["deployments"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
```

### 常用操作指令
```shell
kubectl apply -f webhook.yaml
kubectl get validatingwebhookconfigurations
```

---

## 4. API Server 擴充

### 原理與用途
API Server 可透過 Aggregated API Server（AA）機制擴充，允許外部服務註冊為新的 API 群組，實現自定義 API。

- **原理**：外部服務註冊至 Kubernetes API Server，經過認證與授權後，提供額外 API。
- **用途**：支援複雜業務邏輯、整合外部系統。

### YAML 範例（APIService）
```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.example.com
spec:
  service:
    name: example-api
    namespace: default
  group: example.com
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 1000
  versionPriority: 15
```

### 常用操作指令
```shell
kubectl apply -f apiservice.yaml
kubectl get apiservices
```

---

## 5. Helm 與套件管理

### 原理與用途
Helm 是 Kubernetes 的套件管理工具，將多個 YAML 檔案打包成 Chart，方便部署、升級、回滾與管理複雜應用。

- **原理**：以 Chart 描述應用結構，支援參數化與版本控管。
- **用途**：簡化部署流程、促進團隊協作、提升可維運性。

### Helm 與其他套件管理差異
- **Helm**：專為 Kubernetes 設計，支援模板化、版本管理、依賴管理。
- **傳統套件管理（如 apt、yum）**：針對作業系統層級，無法直接管理 Kubernetes 物件。

### 常用操作指令
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-release bitnami/nginx
helm upgrade my-release bitnami/nginx
helm rollback my-release 1
helm uninstall my-release
```

---

## 6. 進階功能實務應用與專業建議

### Operators
- 適用於需自動化運維的複雜應用（如資料庫、分散式系統）。
- 建議選用社群成熟 Operator，避免自行開發維護負擔。

### CRD
- 適合擴充 Kubernetes 處理特殊業務需求。
- 建議搭配控制器，確保資源狀態自動同步。

### Admission Controller
- 強化安全與合規性（如禁止特定映像、強制資源標籤）。
- 建議測試 webhook 效能與可用性，避免影響 API Server 穩定。

### API Server 擴充
- 適合需自定義 API 或整合外部系統。
- 建議評估維運複雜度與安全性。

### Helm
- 適用於團隊協作與複雜應用部署。
- 建議維護自有 Chart 倉庫，確保版本一致與安全。

---

## 7. 小結

本章介紹 Kubernetes 進階功能，包含 Operators、CRD、Admission Controller、API Server 擴充、Helm 與套件管理，並提供 YAML 範例與實務建議。熟悉這些功能有助於提升叢集自動化、擴充性與維運效率。