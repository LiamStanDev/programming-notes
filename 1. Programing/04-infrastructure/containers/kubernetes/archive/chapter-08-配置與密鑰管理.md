# 第八章：配置與密鑰管理

## 目錄
1. [ConfigMap、Secret、RBAC、ServiceAccount 定義、用途與差異](#configmapsecretrbacserviceaccount-定義用途與差異)
2. [各物件 YAML 範例與常用 kubectl 操作指令](#各物件-yaml-範例與常用-kubectl-操作指令)
3. [配置與密鑰管理的安全性考量與實務建議](#配置與密鑰管理的安全性考量與實務建議)
4. [專業工程師的最佳實踐](#專業工程師的最佳實踐)

---

## ConfigMap、Secret、RBAC、ServiceAccount 定義、用途與差異

### 1. ConfigMap
- **定義**：用於儲存非機密的組態資料（如環境變數、設定檔）。
- **用途**：將應用程式設定與映像檔分離，便於管理與更新。
- **差異**：不適合存放敏感資訊。

### 2. Secret
- **定義**：用於儲存敏感資料（如密碼、API 金鑰、TLS 憑證）。
- **用途**：安全地將機密資訊注入 Pod。
- **差異**：內容經 base64 編碼，權限控管較嚴格。

### 3. RBAC（Role-Based Access Control）
- **定義**：基於角色的存取控制，限制用戶或服務帳號對 Kubernetes 資源的操作權限。
- **用途**：細緻化權限管理，提升安全性。
- **差異**：僅管理權限，無法存放資料。

### 4. ServiceAccount
- **定義**：Pod 在 Kubernetes 內部存取 API Server 時所使用的身份。
- **用途**：自動掛載於 Pod，配合 RBAC 控制權限。
- **差異**：屬於身份識別，非資料存放。

---

## 各物件 YAML 範例與常用 kubectl 操作指令

### 1. ConfigMap

#### YAML 範例
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-config
data:
  APP_ENV: production
  LOG_LEVEL: info
```

#### 常用指令
- 建立：
  `kubectl apply -f configmap.yaml`
- 查詢：
  `kubectl get configmap`
- 查看內容：
  `kubectl describe configmap example-config`
- 直接建立：
  `kubectl create configmap example-config --from-literal=APP_ENV=production`

---

### 2. Secret

#### YAML 範例
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQ=   # base64 編碼後的 "password"
```

#### 常用指令
- 建立：
  `kubectl apply -f secret.yaml`
- 查詢：
  `kubectl get secret`
- 查看內容（解碼）：
  `kubectl get secret example-secret -o yaml`
- 直接建立：
  `kubectl create secret generic example-secret --from-literal=DB_PASSWORD=password`

---

### 3. RBAC（Role、RoleBinding）

#### YAML 範例
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
subjects:
- kind: ServiceAccount
  name: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### 常用指令
- 建立：
  `kubectl apply -f rbac.yaml`
- 查詢角色：
  `kubectl get roles`
- 查詢綁定：
  `kubectl get rolebindings`
- 查看詳細：
  `kubectl describe role pod-reader`

---

### 4. ServiceAccount

#### YAML 範例
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
```

#### 常用指令
- 建立：
  `kubectl apply -f serviceaccount.yaml`
- 查詢：
  `kubectl get serviceaccount`
- 查看詳細：
  `kubectl describe serviceaccount my-service-account`

---

## 配置與密鑰管理的安全性考量與實務建議

- **最小權限原則**：僅授予必要權限，避免過度授權。
- **密鑰加密**：啟用 Kubernetes Secret 加密（Encryption at Rest）。
- **避免硬編碼**：敏感資訊不應寫死於映像檔或程式碼中。
- **審計與監控**：啟用審計日誌，定期檢查資源存取紀錄。
- **定期輪替密鑰**：定期更新密碼、API 金鑰等敏感資訊。
- **限制 Secret 存取**：僅允許授權 Pod 或 ServiceAccount 存取 Secret。

---

## 專業工程師的最佳實踐

1. **組態與密鑰分離**：將組態（ConfigMap）與密鑰（Secret）分開管理，便於權限控管與審計。
2. **自動化管理**：利用 CI/CD 工具自動化配置與密鑰的建立、更新與輪替。
3. **版本控管**：ConfigMap YAML 可納入版本控制，Secret 建議以加密方式儲存於安全倉庫。
4. **監控異常存取**：設置監控與警示，偵測異常存取行為。
5. **定期檢查權限**：定期審查 RBAC 規則與 ServiceAccount 權限，移除不必要的存取。
6. **教育與培訓**：定期對團隊進行安全意識訓練，提升整體安全水準。

---

> 本章節涵蓋 Kubernetes 配置與密鑰管理的核心知識，協助工程師建立安全、可維運的叢集環境。