# 第三章：Kubernetes 安裝方式總覽

本章將介紹 Kubernetes 常見的四種安裝方式：`kubeadm`、`k3s`、`minikube`、`kind`。內容包含各自適用場景、優缺點、主要安裝步驟與官方文件風格指令範例，並提供專業工程師的實務建議，協助讀者選擇最適合的部署方式。

---

## 3.1 安裝方式總覽

| 安裝方式     | 適用場景           | 優點               | 缺點                  |
| -------- | -------------- | ---------------- | ------------------- |
| kubeadm  | 正式環境、學習、測試     | 靈活、接近原生、官方支援     | 需自行處理網路、儲存等元件       |
| k3s      | 輕量化、IoT、邊緣運算   | 輕量、安裝快速、資源需求低    | 功能精簡，部分元件非原生        |
| minikube | 本機開發、測試        | 一鍵啟動、支援多種驅動、易於重置 | 僅適合單節點、資源消耗較高       |
| kind     | CI/CD、自動化測試、教學 | 容器內運行、快速建立多集群    | 僅支援 Docker、效能受限於容器層 |

---

## 3.2 kubeadm

### 適用場景

- 正式環境部署
- 學習 Kubernetes 原理
- 測試多節點集群

### 優缺點

- **優點**：高度可自訂、接近原生、官方長期支援
- **缺點**：需自行安裝網路、儲存、監控等元件，安裝步驟較繁瑣

### 主要安裝步驟

1. 安裝作業系統相依套件（如 Docker、containerd）
2. 安裝 kubeadm、kubelet、kubectl
3. 初始化 Master 節點
4. 安裝網路外掛（如 Calico、Flannel）
5. 加入 Worker 節點

### 官方文件風格指令範例

```bash
# 1. 安裝 kubeadm、kubelet、kubectl
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# 2. 初始化 Master 節點
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# 3. 設定 kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 4. 安裝網路外掛（以 Calico 為例）
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# 5. 加入 Worker 節點（於 Worker 節點執行 kubeadm join 指令）
```

---

## 3.3 k3s

### 適用場景

- 輕量化部署（IoT、邊緣運算）
- 測試、開發環境
- 資源有限的設備

### 優缺點

- **優點**：安裝快速、資源需求低、內建多數元件
- **缺點**：部分元件非原生、功能較精簡

### 主要安裝步驟

1. 下載並執行安裝腳本
2. 取得 kubeconfig 設定
3. （選用）加入 Agent 節點

### 官方文件風格指令範例

```bash
# 1. 安裝 k3s（單節點）
curl -sfL https://get.k3s.io | sh -

# 2. 查看安裝狀態
sudo k3s kubectl get node

# 3. 取得 kubeconfig
sudo cat /etc/rancher/k3s/k3s.yaml

# 4. 加入 Agent 節點（於 Agent 節點執行）
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<TOKEN> sh -
```

---

## 3.4 minikube

### 適用場景

- 本機開發、測試
- 學習 Kubernetes 操作

### 優缺點

- **優點**：一鍵啟動、支援多種 VM/容器驅動、易於重置
- **缺點**：僅支援單節點、資源消耗較高

### 主要安裝步驟

1. 安裝 minikube 與 kubectl
2. 啟動本地集群
3. 驗證安裝

### 官方文件風格指令範例

```bash
# 1. 安裝 minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# 2. 啟動集群（以 Docker 為驅動）
minikube start --driver=docker

# 3. 驗證節點狀態
kubectl get nodes
```

---

## 3.5 kind (Kubernetes IN Docker)

### 適用場景

- CI/CD 測試
- 自動化測試
- 教學、快速建立多集群

### 優缺點

- **優點**：容器內運行、快速建立、支援多集群
- **缺點**：僅支援 Docker、效能受限於容器層

### 主要安裝步驟

1. 安裝 Docker 與 kind
2. 建立 Kubernetes 集群
3. 驗證安裝

### 官方文件風格指令範例

```bash
# 1. 安裝 kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# 2. 建立集群
kind create cluster --name demo

# 3. 驗證節點狀態
kubectl get nodes
```

---

## 3.6 專業工程師實務建議與最佳實踐

- **選擇安裝方式時，應根據實際需求與環境資源評估**。正式環境建議使用 `kubeadm` 或企業級方案，開發測試可選擇 `minikube`、`kind` 或 `k3s`。
- **自動化安裝與版本控管**：建議將安裝流程寫成腳本，並記錄所用版本，方便日後維護與升級。
- **網路與儲存外掛選型**：kubeadm 須特別注意網路外掛（如 Calico、Flannel）與儲存方案的選擇，避免相容性問題。
- **資源監控與安全性**：無論哪種安裝方式，建議安裝資源監控（如 Prometheus）與加強安全性設置（如 RBAC、NetworkPolicy）。
- **多節點測試**：如需模擬多節點，`kind` 與 `kubeadm` 均可建立多節點集群，利於真實環境驗證。
- **定期備份與升級**：建議定期備份 etcd 資料與升級 Kubernetes 版本，確保集群穩定與安全。

---

> 本章內容涵蓋 Kubernetes 主要安裝方式，協助讀者根據需求選擇合適方案，並提供實務建議以提升部署效率與穩定性。