# 第十三章：CI/CD 與自動化

## 1. GitOps 與 Pipeline 原理與應用

### 1.1 GitOps 概念

GitOps 是以 Git 為單一事實來源（Single Source of Truth），透過 Git 儲存庫管理基礎設施與應用程式部署設定，並由自動化工具（如 ArgoCD、Flux）持續監控與同步叢集狀態。其核心理念為「所有變更皆經由 Pull Request，並自動化部署」。

#### 1.1.1 ArgoCD

- **原理**：持續監控 Git 儲存庫，將 Kubernetes 叢集狀態與 Git 內容自動同步。
- **應用**：適合多環境部署、權限審核、回溯與審計。

#### 1.1.2 Flux

- **原理**：自動偵測 Git 變更，並將變更套用至 Kubernetes 叢集。
- **應用**：支援多種自動化策略，與 Helm、Kustomize 整合良好。

### 1.2 Pipeline（Tekton、Jenkins X）

#### 1.2.1 Tekton

- **原理**：以 Kubernetes CRD 定義 CI/CD Pipeline，彈性高、原生整合。
- **應用**：適合雲原生環境，支援多階段建置、測試、部署。

#### 1.2.2 Jenkins X

- **原理**：基於 Jenkins，專為 Kubernetes 設計，結合 GitOps 流程。
- **應用**：自動化應用程式建置、測試、部署，支援多雲與多環境。

---

## 2. 自動化部署與滾動更新策略

### 2.1 自動化部署流程

1. 開發者提交程式碼至 Git。
2. CI 工具（如 Tekton、Jenkins X）自動建置、測試。
3. 成功後將映像檔推送至 Registry。
4. CD 工具（如 ArgoCD、Flux）偵測到 Git 設定變更，自動部署至 Kubernetes。

### 2.2 滾動更新（Rolling Update）

- **說明**：逐步將新版本 Pod 取代舊版本，確保服務不中斷。
- **Kubernetes 實作**：Deployment 預設支援滾動更新，可設定 maxSurge、maxUnavailable。

#### YAML 範例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    spec:
      containers:
      - name: my-app
        image: my-app:v2
```

---

## 3. 各方案 YAML/指令範例與常用操作

### 3.1 ArgoCD

#### 安裝

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

#### 建立 Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/example/repo.git'
    targetRevision: HEAD
    path: k8s
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

#### 常用指令

```bash
# 登入 ArgoCD CLI
argocd login <ARGOCD_SERVER>
# 同步 Application
argocd app sync my-app
# 查看狀態
argocd app get my-app
```

---

### 3.2 Flux

#### 安裝

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
flux install
```

#### 建立 GitRepository 與 Kustomization

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/example/repo.git
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  path: ./k8s
  prune: true
  sourceRef:
    kind: GitRepository
    name: my-app
```

#### 常用指令

```bash
# 查看同步狀態
flux get kustomizations
# 手動同步
flux reconcile kustomization my-app
```

---

### 3.3 Tekton

#### 安裝

```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

#### Pipeline YAML 範例

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-and-deploy
spec:
  tasks:
    - name: build
      taskRef:
        name: build-docker-image
    - name: deploy
      runAfter:
        - build
      taskRef:
        name: deploy-to-k8s
```

#### 常用指令

```bash
# 查看 PipelineRun
kubectl get pipelineruns
# 觸發 Pipeline
kubectl create -f pipeline-run.yaml
```

---

### 3.4 Jenkins X

#### 安裝

```bash
jx admin operator --username admin
```

#### 建立 Pipeline

Jenkins X 會自動產生 `jenkins-x.yml`，範例如下：

```yaml
buildPack: kubernetes
pipelineConfig:
  pipelines:
    release:
      steps:
        - name: build
          image: golang:1.18
          command: make build
        - name: deploy
          image: gcr.io/cloud-builders/kubectl
          command: make deploy
```

#### 常用指令

```bash
# 查看 Pipeline 狀態
jx get activities
# 觸發 Pipeline
jx start pipeline
```

---

## 4. CI/CD 與自動化的實務建議與最佳實踐

- **以 Git 為單一事實來源**：所有設定與部署皆應透過 Git 管理，確保可追蹤與審計。
- **自動化測試與驗證**：每次提交皆應自動執行測試，確保品質。
- **分離 CI 與 CD 流程**：建議將建置（CI）與部署（CD）分開，提升彈性與安全性。
- **權限與審核機制**：善用 Pull Request、Code Review，避免未經審核的變更直接進入生產環境。
- **監控與回滾**：部署後應即時監控，並設計自動回滾機制以降低風險。
- **環境一致性**：開發、測試、正式環境應盡量一致，避免部署問題。
- **資源與密鑰管理**：敏感資訊應使用 Kubernetes Secret 或外部密鑰管理服務。
- **文件化流程**：將 CI/CD 流程、部署步驟、異常處理等文件化，方便團隊協作與維運。

---

本章涵蓋 CI/CD 與自動化的核心原理、主流程、主流工具操作與 YAML 範例，並提供實務建議，協助讀者建立穩健的自動化部署體系。