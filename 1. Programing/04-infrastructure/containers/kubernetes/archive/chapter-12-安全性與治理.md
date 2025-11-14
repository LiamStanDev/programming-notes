# 第十二章：安全性與治理

本章將介紹 Kubernetes 叢集中的安全性與治理機制，涵蓋 NetworkPolicy、Pod Security Standards（PSS）、Secret 管理（Sealed Secrets、Vault）、多租戶與 Namespace 隔離等主題，並提供 YAML 範例、常用 kubectl 指令，以及實務建議與最佳實踐。

---

## 1. NetworkPolicy

### 原理與用途

NetworkPolicy 用於限制 Pod 之間或 Pod 與外部的網路流量，實現微服務間的網路隔離與安全防護。預設情況下，Pod 之間網路是全開的，需透過 NetworkPolicy 明確定義允許的流量。

### YAML 範例

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx-from-frontend
  namespace: demo
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 80
```

### 常用 kubectl 指令

- 建立 NetworkPolicy：
  ```sh
  kubectl apply -f networkpolicy.yaml
  ```
- 查詢現有 NetworkPolicy：
  ```sh
  kubectl get networkpolicy -n <namespace>
  ```
- 刪除 NetworkPolicy：
  ```sh
  kubectl delete networkpolicy <policy-name> -n <namespace>
  ```

---

## 2. Pod Security Standards（PSS）

### 原理與用途

Pod Security Standards（PSS）是 Kubernetes 官方針對 Pod 安全性的分級標準，分為 privileged、baseline、restricted 三個等級。可透過 Pod Security Admission（PSA）在 Namespace 層級強制執行。

### YAML 範例（Namespace 標註）

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-ns
  labels:
    pod-security.kubernetes.io/enforce: "restricted"
    pod-security.kubernetes.io/enforce-version: "latest"
```

### 常用 kubectl 指令

- 新增 Namespace 並套用 PSS：
  ```sh
  kubectl create ns secure-ns
  kubectl label ns secure-ns pod-security.kubernetes.io/enforce=restricted
  ```
- 查詢 Namespace 標籤：
  ```sh
  kubectl get ns --show-labels
  ```

---

## 3. Secret 管理

### 原理與用途

Secret 用於儲存敏感資訊（如密碼、API 金鑰）。Kubernetes 原生 Secret 僅 base64 編碼，建議搭配加密工具（如 Sealed Secrets、Vault）提升安全性。

### 3.1 Sealed Secrets

#### 原理

Sealed Secrets 由 Bitnami 提供，允許將加密後的 Secret 存放於 Git 等版本控制系統，僅能由指定叢集解密。

#### YAML 範例

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mysecret
  namespace: demo
spec:
  encryptedData:
    password: AgB2...
```

#### 常用 kubectl 指令

- 產生 SealedSecret：
  ```sh
  kubectl create secret generic mysecret --from-literal=password=123456 --dry-run=client -o yaml > mysecret.yaml
  kubeseal < mysecret.yaml -o yaml > mysealedsecret.yaml
  ```
- 套用 SealedSecret：
  ```sh
  kubectl apply -f mysealedsecret.yaml
  ```

### 3.2 Vault

#### 原理

HashiCorp Vault 提供動態密鑰管理與存取控制，支援自動輪換、審計等功能。可透過 CSI 驅動或外掛整合至 Kubernetes。

#### YAML 範例（Vault Agent Injector）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vault-demo
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "demo-role"
    vault.hashicorp.com/agent-inject-secret-config.txt: "secret/data/demo"
spec:
  containers:
    - name: app
      image: busybox
      command: ["sleep", "3600"]
```

#### 常用 kubectl 指令

- 部署 Vault Agent Injector：
  ```sh
  kubectl apply -f vault-agent-injector.yaml
  ```
- 查詢 Secret 掛載狀態：
  ```sh
  kubectl describe pod vault-demo
  ```

---

## 4. 多租戶與 Namespace 隔離

### 原理與用途

多租戶（Multi-tenancy）指在同一叢集內為不同團隊或專案提供隔離環境。Namespace 是最基本的隔離單位，可搭配 ResourceQuota、NetworkPolicy、RBAC 強化隔離。

### YAML 範例

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    cpu: "4"
    memory: 8Gi
    pods: "20"
```

### 常用 kubectl 指令

- 建立 Namespace：
  ```sh
  kubectl create ns team-a
  ```
- 設定資源配額：
  ```sh
  kubectl apply -f resourcequota.yaml
  ```
- 查詢 Namespace 內資源使用狀況：
  ```sh
  kubectl describe quota -n team-a
  ```

---

## 5. 安全性與治理的實務建議與最佳實踐

- **最小權限原則**：僅授權必要的資源存取權限，避免過度授權。
- **定期審查與輪換密鑰**：定期檢查 Secret、ServiceAccount、憑證等敏感資訊，並執行輪換。
- **啟用 NetworkPolicy**：預設拒絕所有流量，僅允許必要的網路通訊。
- **落實 Pod Security Standards**：根據應用需求選擇合適的 PSS 等級，並強制執行於 Namespace。
- **使用加密 Secret 工具**：如 Sealed Secrets、Vault，避免明文存放敏感資訊。
- **多租戶隔離**：不同團隊/專案應分配獨立 Namespace，並搭配資源配額、RBAC、NetworkPolicy。
- **監控與審計**：啟用審計日誌，監控異常行為，並定期檢查安全事件。
- **自動化安全檢查**：導入 CI/CD 安全掃描工具（如 kube-bench、kube-hunter）。

---

## 6. 參考資源

- [Kubernetes 官方文件 - Security](https://kubernetes.io/zh/docs/concepts/security/)
- [Pod Security Standards](https://kubernetes.io/zh/docs/concepts/security/pod-security-standards/)
- [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [HashiCorp Vault](https://www.vaultproject.io/docs/platform/k8s)
