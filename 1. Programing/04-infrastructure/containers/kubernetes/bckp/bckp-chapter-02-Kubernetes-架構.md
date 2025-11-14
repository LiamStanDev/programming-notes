# Kubernetes 架構

## 架構總覽

Kubernetes 採用分層式架構，主要分為 **Control Plane（控制平面）** 與 **Node（工作節點）**。控制平面負責整體叢集管理與調度，工作節點則執行實際的應用容器。

### 架構圖（官方風格）

```markdown
+-------------------------------+         +-------------------+
|         Control Plane         |         |      Node 1       |
|-------------------------------|         |-------------------|
| +-------------------------+   |         | +-------------+   |
| |      etcd (Cluster)     |<--+-------> | |  kubelet    |   |
| +-------------------------+   |         | +-------------+   |
| +-------------------------+   |         | +-------------+   |
| |    kube-apiserver       |<--+-------> | | kube-proxy  |   |
| +-------------------------+   |         | +-------------+   |
| +-------------------------+   |         | +-------------+   |
| | kube-controller-manager |   |         | | Container   |   |
| +-------------------------+   |         | |  Runtime    |   |
| +-------------------------+   |         | +-------------+   |
| |    kube-scheduler       |   |         +-------------------+
| +-------------------------+   |
+-------------------------------+         |      Node 2       |
                                          |-------------------|
                                          | kubelet           |
                                          | kube-proxy        |
                                          | Container Runtime |
                                          +-------------------+
```
> **說明：** Control Plane 可多實例部署（高可用），etcd 建議組成叢集，所有 Node 透過 kubelet/kube-proxy 與控制平面互動。

---
## 安裝方式總覽

Kubernetes 叢集常見安裝方式有四種：**kubeadm**、**k3s**、**minikube**、**kind**。以下分別說明其理論基礎、官方文件風格指令、適用場景比較與實務建議。

---

### 1. kubeadm

- **理論說明**
  - 官方支援的標準安裝工具，協助快速部署生產等級 Kubernetes 叢集。
  - 提供初始化（init）、節點加入（join）、升級、重設等功能。
  - 適合自訂需求、學習原生架構、企業生產環境。

- **官方文件風格指令範例**
  ```bash
  # 安裝 kubeadm、kubelet、kubectl
  sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl
  sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
  echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
  sudo apt-get update
  sudo apt-get install -y kubelet kubeadm kubectl
  sudo apt-mark hold kubelet kubeadm kubectl

  # 初始化控制平面
  sudo kubeadm init --pod-network-cidr=10.244.0.0/16

  # 設定 kubectl
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

  # 節點加入
  sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
  ```

- **適用場景與選擇建議**
  - 適合企業生產環境、需高度自訂、學習原生架構。
  - 適用於 VM、實體機、雲端主機。
  - 需自行處理網路、儲存、HA 等進階設定。

- **實務建議與常見問題**
  - 建議搭配 CNI（如 Flannel、Calico）安裝網路插件。
  - 控制平面建議多台主機部署（高可用）。
  - 常見問題：初始化失敗多因 swap 未關閉、網路未設定、時區不同步。

---

### 2. k3s

- **理論說明**
  - 輕量化 Kubernetes 發行版，由 Rancher 開發，專為 IoT、邊緣運算、小型叢集設計。
  - 單一二進位檔，內建多數元件，安裝快速。
  - 支援 ARM 架構，資源需求低。

- **官方文件風格指令範例**
  ```bash
  # 一鍵安裝（單節點）
  curl -sfL https://get.k3s.io | sh -

  # 查看狀態
  sudo k3s kubectl get node

  # 取得 kubeconfig
  sudo cat /etc/rancher/k3s/k3s.yaml

  # 節點加入
  curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
  ```

- **適用場景與選擇建議**
  - 適合資源有限環境、IoT、邊緣運算、測試、開發。
  - 不建議用於大型生產叢集（>50節點）。
  - 適合快速部署、簡化維運。

- **實務建議與常見問題**
  - 預設內建 SQLite，生產環境建議改用外部資料庫（如 MySQL）。
  - 注意內建元件版本與相容性。
  - 常見問題：防火牆未開啟必要 port、token 輸入錯誤。

---

### 3. minikube

- **理論說明**
  - 官方維護的本機單節點 Kubernetes，適合學習、開發、測試。
  - 支援多種虛擬化後端（VirtualBox、KVM、Docker）。
  - 可模擬多節點，但主要用於單機環境。

- **官方文件風格指令範例**
  ```bash
  # 安裝 minikube
  curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
  sudo install minikube-linux-amd64 /usr/local/bin/minikube

  # 啟動本機叢集
  minikube start --driver=docker

  # 查詢狀態
  minikube status

  # 停止/刪除叢集
  minikube stop
  minikube delete
  ```

- **適用場景與選擇建議**
  - 適合本機開發、CI/CD 測試、教學演示。
  - 不建議用於生產環境。
  - 支援多平台（Windows/Mac/Linux）。

- **實務建議與常見問題**
  - 建議搭配 Docker driver，效能較佳。
  - 常見問題：虛擬化未啟用、資源不足導致啟動失敗。

---

### 4. kind (Kubernetes IN Docker)

- **理論說明**
  - 以 Docker 容器模擬多節點 Kubernetes 叢集，適合開發、測試 CI/CD。
  - 不需額外虛擬機，快速建立輕量多節點環境。
  - 官方支援，常用於專案自動化測試。

- **官方文件風格指令範例**
  ```bash
  # 安裝 kind
  GO111MODULE="on" go install sigs.k8s.io/kind@latest

  # 建立叢集
  kind create cluster --name demo

  # 查詢節點
  kubectl get nodes

  # 刪除叢集
  kind delete cluster --name demo
  ```

- **適用場景與選擇建議**
  - 適合 CI/CD pipeline、自動化測試、多節點模擬。
  - 不建議用於生產環境或需持久化儲存場景。
  - 適合開發者快速驗證 Kubernetes 功能。

- **實務建議與常見問題**
  - 需安裝 Docker，且主機資源需足夠。
  - 常見問題：Docker 版本不符、網路設定衝突。

---

### 安裝方式比較與選擇建議

| 工具      | 適用場景         | 優點                   | 限制/注意事項           |
|-----------|------------------|------------------------|-------------------------|
| kubeadm   | 生產、客製化     | 彈性高、原生支援       | 安裝較繁瑣、需自管元件   |
| k3s       | IoT、邊緣、輕量 | 輕量、安裝快、資源低   | 不適合大型叢集          |
| minikube  | 本機開發、教學   | 易用、跨平台           | 單節點為主、非生產用途   |
| kind      | 測試、自動化     | 多節點模擬、快速重建   | 僅限於 Docker 容器內    |

- **選擇建議**
  - 生產環境建議 kubeadm，需高彈性與原生支援。
  - IoT/邊緣或資源有限建議 k3s。
  - 本機開發、教學可用 minikube。
  - 測試、CI/CD pipeline 推薦 kind。

- **專業工程師實務建議**
  - 生產環境務必考慮高可用、備份、監控。
  - 測試/開發環境可用 k3s、minikube、kind 提升效率。
  - 安裝前確認主機資源、網路、防火牆設定。
  - 常見問題多與網路、資源、相依套件有關，建議先查閱官方 FAQ。

## 基本物件

### 1. Pod

- **理論說明**
  - Pod 是 Kubernetes 最小的可部署單元，通常包含一個或多個緊密相關的容器，這些容器共享網路、儲存與生命週期。
  - 適合部署單一應用或緊密耦合的多容器應用（如 sidecar 模式）。
- **官方文件風格 YAML 範例**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx-pod
  spec:
    containers:
      - name: nginx
        image: nginx:1.25
        ports:
          - containerPort: 80
  ```
- **常見應用場景與設計建議**
  - 單一微服務、測試環境、短暫任務（如 Job）。
  - 建議以 Deployment、Job、DaemonSet 等高階控制器管理 Pod，避免直接操作裸 Pod。
- **專業工程師實務建議／常見問題**
  - 裸 Pod 無自動修復能力，建議僅用於測試或特殊需求。
  - 常見問題：Pod CrashLoopBackOff 多因資源不足、鏡像錯誤或探針設定不當。

---

### 2. Service

- **理論說明**
  - Service 提供穩定的網路存取入口，將流量導向一組 Pod，實現負載平衡與服務發現。
  - 支援 ClusterIP（預設）、NodePort、LoadBalancer、ExternalName 等型態。
- **官方文件風格 YAML 範例**
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
- **常見應用場景與設計建議**
  - 微服務間通訊、對外暴露服務、負載平衡。
  - 建議搭配 label selector 精確選取目標 Pod。
  - 對外服務建議使用 Ingress 或 LoadBalancer，提升彈性與安全性。
- **專業工程師實務建議／常見問題**
  - Service IP 為虛擬 IP，僅於叢集內部可用（除非設為 NodePort/LoadBalancer）。
  - 常見問題：selector 設定錯誤導致無後端 Pod、NodePort 衝突、LoadBalancer 需雲端支援。

---
---
## 網路與服務發現

### 1. Service 類型理論說明

- **ClusterIP**
  - 預設型態，僅於叢集內部可存取。
  - 提供虛擬 IP，Pod 間服務發現與負載平衡。
  - 適用：微服務內部通訊、後端服務。
  - **YAML 範例**
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: backend
    spec:
      selector:
        app: backend
      ports:
        - port: 8080
          targetPort: 8080
      type: ClusterIP
    ```

- **NodePort**
  - 於每個 Node 開放指定 port（30000-32767），外部可透過 NodeIP:NodePort 存取。
  - 適用：測試、簡易對外服務、無雲端 LoadBalancer 環境。
  - **YAML 範例**
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: nodeport-demo
    spec:
      selector:
        app: demo
      ports:
        - port: 80
          targetPort: 8080
          nodePort: 30080
      type: NodePort
    ```

- **LoadBalancer**
  - 整合雲端平台（如 GCP、AWS、Azure）自動建立外部 Load Balancer。
  - 適用：正式環境對外服務、需彈性擴展與高可用。
  - **YAML 範例**
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: lb-demo
    spec:
      selector:
        app: demo
      ports:
        - port: 80
          targetPort: 8080
      type: LoadBalancer
    ```

- **Ingress**
  - 以 7 層（HTTP/HTTPS）代理方式統一管理多服務路由、TLS、虛擬主機。
  - 需搭配 Ingress Controller（如 NGINX、Traefik）。
  - 適用：多網站入口、API Gateway、SSL 統一管理。
  - **YAML 範例**
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: web-ingress
    spec:
      rules:
        - host: www.example.com
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: web
                    port:
                      number: 80
    ```

- **設計建議與實務經驗**
  - 內部服務建議用 ClusterIP，對外服務優先考慮 Ingress 或 LoadBalancer。
  - NodePort 僅用於測試或特殊需求，避免大量暴露 port。
  - Ingress 可集中管理 SSL、路由，提升安全與維運效率。
  - LoadBalancer 需雲端支援，裸機環境可用 MetalLB 等方案。

---

### 2. CNI（Container Network Interface）原理與常見方案

- **原理說明**
  - CNI 定義容器網路插件標準，負責 Pod 網路分配、IP 管理、跨主機通訊。
  - 每個 Pod 會分配唯一 IP，跨 Node 需 Overlay/路由技術。
  - 常見功能：Pod-to-Pod 通訊、NetworkPolicy 支援、IPAM。

- **常見方案**
  - **Flannel**
    - 最簡單、易部署，採用 VXLAN/UDP Overlay。
    - 適合小型叢集、學習環境。
    - 不支援 NetworkPolicy。
    - 安裝指令（kubeadm 範例）：
      ```bash
      kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
      ```
  - **Calico**
    - 支援 L3 路由、NetworkPolicy、BGP，效能佳。
    - 適合生產環境、中大型叢集。
    - 支援純路由與 Overlay 模式。
    - 安裝指令（kubeadm 範例）：
      ```bash
      kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
      ```
  - **Cilium**
    - 基於 eBPF，支援高效能網路、NetworkPolicy、API-aware 安全。
    - 適合高安全需求、雲原生、Service Mesh 整合。
    - 支援透明加密、流量觀察。
    - 安裝指令（kubeadm 範例）：
      ```bash
      kubectl create -f https://raw.githubusercontent.com/cilium/cilium/v1.14.0/install/kubernetes/quick-install.yaml
      ```

- **設計建議與實務經驗**
  - 生產環境建議 Calico 或 Cilium，Flannel 適合學習/測試。
  - 需考慮 NetworkPolicy、效能、維運複雜度。
  - Overlay 網路效能略低於純路由，需評估流量需求。

---

### 3. Service Mesh（服務網格）簡介

- **核心概念**
  - Service Mesh 以 sidecar 代理（如 Envoy）統一管理服務間流量、認證、觀察、流量控制。
  - 解耦應用程式與網路治理，提升可觀察性與安全性。

- **常見方案**
  - **Istio**
    - 功能最完整，支援流量治理、金絲雀發布、熔斷、認證、監控。
    - 適合大型、複雜微服務架構。
    - 安裝指令（官方快速安裝）：
      ```bash
      curl -L https://istio.io/downloadIstio | sh -
      cd istio-*
      export PATH=$PWD/bin:$PATH
      istioctl install --set profile=demo -y
      ```
  - **Linkerd**
    - 輕量、易用，專注於核心流量治理與可觀察性。
    - 適合中小型團隊、快速導入。
    - 安裝指令（官方快速安裝）：
      ```bash
      curl -sL https://run.linkerd.io/install | sh
      export PATH=$PATH:$HOME/.linkerd2/bin
      linkerd install | kubectl apply -f -
      ```

- **應用場景與設計建議**
  - 需細緻流量控制、零信任安全、跨語言追蹤時建議導入 Service Mesh。
  - 小型專案可先用 Ingress + NetworkPolicy，逐步導入 Mesh。
  - Service Mesh 增加複雜度，建議先於測試環境驗證效益。

---

### 4. 常見應用場景與專業工程師實務經驗

- **微服務內部通訊**：ClusterIP + NetworkPolicy 控制流量，提升安全。
- **對外 API/網站服務**：Ingress + LoadBalancer，集中管理流量與 SSL。
- **多租戶/多環境隔離**：Namespace + NetworkPolicy + Ingress。
- **高安全需求**：Cilium/Calico + Service Mesh，實現細緻存取控制與流量加密。
- **裸機/自建環境**：可用 MetalLB 提供 LoadBalancer 功能。
- **設計建議**
  - 依需求選擇網路方案，評估效能、維運、社群活躍度。
  - 生產環境建議啟用 NetworkPolicy，並定期審查規則。
  - Service Mesh 導入需考慮團隊維運能力與效益。

---

### 3. Namespace

- **理論說明**
  - Namespace 用於將叢集資源邏輯分隔，適合多租戶、環境隔離（如 dev/stage/prod）。
  - 各 Namespace 內資源名稱可重複，資源預設僅在同一 Namespace 內可見。
- **官方文件風格 YAML 範例**
  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: dev
  ```
- **常見應用場景與設計建議**
  - 多團隊協作、權限隔離、資源配額（ResourceQuota）管理。
  - 建議為不同環境、專案、團隊建立獨立 Namespace。
- **專業工程師實務建議／常見問題**
  - 預設有 default、kube-system、kube-public 等 Namespace，勿隨意刪除。
  - 常見問題：資源建立於錯誤 Namespace、RBAC 權限未正確設置、跨 Namespace 通訊需額外設計。

---
## 各元件說明

### Control Plane 元件

- **etcd**
  分散式 key-value 儲存系統，保存整個叢集狀態資料。
  **最佳實踐：** 建議部署於奇數台主機，並定期備份。

- **kube-apiserver**
  所有操作的入口，負責處理 REST API 請求，協調各元件溝通。
  **最佳實踐：** 啟用 RBAC，限制 API 存取權限。

- **kube-controller-manager**
  執行各種控制器（如副本、節點、命名空間等），確保叢集狀態符合期望。
  **最佳實踐：** 監控 controller log，及時發現異常。

- **kube-scheduler**
  負責將 Pod 分配到合適的 Node 上。
  **最佳實踐：** 合理設計資源請求與限制，提升調度效率。

### Node 元件

- **kubelet**
  節點代理程式，負責與控制平面溝通並管理本機 Pod。
  **最佳實踐：** 定期更新 kubelet，並監控其健康狀態。

- **kube-proxy**
  負責 Pod 網路服務的流量轉發與負載平衡。
  **最佳實踐：** 根據網路規模選擇合適模式（iptables/ipvs）。

- **Container Runtime**
  如 containerd、Docker，負責容器的建立與執行。
  **最佳實踐：** 建議使用 CNCF 認證的 Runtime，並定期更新。

---

## 架構設計原則與高可用性

Kubernetes 架構設計強調「去中心化」、「彈性擴展」與「高可用性」。高可用性（HA）設計重點如下：

- **多 Master 架構**：Control Plane 元件（kube-apiserver、controller-manager、scheduler）可多實例部署，避免單點故障。
- **etcd 叢集**：建議部署奇數台（如 3 或 5 台）etcd，確保資料一致性與容錯。
- **負載平衡**：kube-apiserver 前可加設 HAProxy、Nginx 或雲端 Load Balancer，讓 Node 與用戶端流量自動分流。
- **資料備援**：定期備份 etcd，並測試還原流程。
- **自動修復**：利用 controller-manager 監控與自動修復異常資源。

### 高可用性部署指令與 YAML 範例

#### 1. 建立 etcd 叢集（以三台為例）

```bash
# 於三台主機分別啟動 etcd，並設定 initial-cluster
etcd --name etcd1 --initial-advertise-peer-urls http://etcd1:2380 \
  --listen-peer-urls http://etcd1:2380 \
  --listen-client-urls http://etcd1:2379 \
  --initial-cluster etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380 \
  --initial-cluster-state new
```

#### 2. kube-apiserver 加入外部負載平衡

```yaml
# 範例：kube-apiserver service 透過 LoadBalancer
apiVersion: v1
kind: Service
metadata:
  name: kube-apiserver-lb
  namespace: kube-system
spec:
  type: LoadBalancer
  ports:
    - port: 6443
      targetPort: 6443
  selector:
    component: kube-apiserver
```

#### 3. 節點註冊時指定多個 apiserver

```bash
# kubelet 啟動參數可指定多個 apiserver
kubelet --api-servers=https://apiserver-lb:6443
```

#### 4. 檢查高可用狀態

```bash
# 檢查 etcd cluster 健康
etcdctl endpoint health --cluster

# 檢查多個 apiserver 狀態
kubectl get endpoints -n kube-system kube-apiserver
```

## 架構運作原理與流程

1. 使用者透過 `kubectl` 或 API 請求操作 Kubernetes。
2. **kube-apiserver** 接收請求並驗證、授權。
3. 狀態資訊儲存於 **etcd**。
4. **kube-controller-manager** 監控叢集狀態，執行自動修正。
5. **kube-scheduler** 根據資源與策略，將 Pod 分配至合適 Node。
6. **kubelet** 於 Node 上建立/管理 Pod，並回報狀態。
7. **kube-proxy** 處理服務流量，確保網路連通。

---

## 指令與 YAML 範例

### 查詢各元件狀態

```bash
# 查詢控制平面元件
kubectl get componentstatuses

# 查詢所有 Node 狀態
kubectl get nodes

# 查詢 Pod 狀態
kubectl get pods -A
```

### 範例：Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
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
        image: nginx:1.25
        ports:
        - containerPort: 80
```

---

## 進階實務建議與最佳實踐

- 控制平面建議隔離於專用主機，提升安全性與穩定性。
- Control Plane、etcd、Node 建議分開不同主機或 VM，避免資源競爭。
- 定期備份 etcd，並測試還原流程。
- 監控所有元件健康狀態，設置告警機制（可用 Prometheus + Alertmanager）。
- 合理規劃資源請求與限制，避免資源爭用與 OOM。
- 定期更新 Kubernetes 版本，修補安全漏洞，升級前先於測試環境驗證。
- 使用 RBAC 與 NetworkPolicy 強化存取與網路安全。
- 建議啟用 PodDisruptionBudget、PodAntiAffinity，提升服務可用性。
- 關鍵服務建議多副本部署，並設計 readiness/liveness probe。
- 重要元件（如 kubelet、kube-proxy）建議設為 systemd 服務並啟用自動重啟。
- 叢集規模擴展時，建議先壓力測試與資源預估。

## 專業工程師實務建議

- 控制平面建議隔離於專用主機，提升安全性與穩定性。
- 定期備份 etcd，並測試還原流程。
- 監控所有元件健康狀態，設置告警機制。
- 合理規劃資源請求與限制，避免資源爭用。
- 定期更新 Kubernetes 版本，修補安全漏洞。
- 使用 RBAC 與 NetworkPolicy 強化存取與網路安全。

---

## 小結

本章節完整說明 Kubernetes 架構、各元件職責、運作流程，並提供官方風格圖解、指令與 YAML 範例，以及實務建議，協助讀者快速掌握架構核心知識。
## 工作負載管理

### 1. Deployment
- **理論說明**：用於管理無狀態應用的部署與升級，確保指定數量的 Pod 持續運行，支援滾動更新與回滾。
- **YAML 範例**：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
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
        image: nginx:1.25
```
- **應用場景**：Web 服務、API、微服務等無狀態應用。
- **設計建議**：配合 Service 使用，設定適當的 `replicas`，善用 `strategy` 控制升級方式。
- **實務建議/常見問題**：避免將有狀態應用部署為 Deployment，升級時注意資源限制與健康檢查設定。

---

### 2. ReplicaSet
- **理論說明**：確保指定數量的 Pod 副本持續運行，通常由 Deployment 管理，不建議單獨使用。
- **YAML 範例**：
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 2
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
        image: nginx:1.25
```
- **應用場景**：特殊情境下需手動管理 Pod 副本。
- **設計建議**：一般應由 Deployment 管理，避免直接操作 ReplicaSet。
- **實務建議/常見問題**：直接操作 ReplicaSet 會失去升級與回滾能力。

---

### 3. StatefulSet
- **理論說明**：用於有狀態應用，提供穩定的網路識別、順序部署、持久化儲存。
- **YAML 範例**：
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
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
        image: nginx:1.25
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
- **應用場景**：資料庫、分散式系統（如 Kafka、ZooKeeper）。
- **設計建議**：需搭配 PersistentVolume，Pod 數量擴縮時注意資料一致性。
- **實務建議/常見問題**：升級順序受限，刪除 StatefulSet 不會自動刪除 PVC。

---

### 4. DaemonSet
- **理論說明**：確保每個（或指定）節點上都運行一個 Pod，常用於系統層級服務。
- **YAML 範例**：
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.16
```
- **應用場景**：日誌收集、監控代理、網路插件。
- **設計建議**：可用 `nodeSelector`、`tolerations` 控制部署節點。
- **實務建議/常見問題**：節點數量變動時會自動調整，注意資源消耗。

---

### 5. Job
- **理論說明**：用於一次性任務，確保任務成功完成指定次數。
- **YAML 範例**：
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
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
- **應用場景**：批次資料處理、資料遷移、初始化任務。
- **設計建議**：設定合理的 `backoffLimit`，避免重複執行造成資源浪費。
- **實務建議/常見問題**：Job 完成後 Pod 不會自動刪除，需設定 TTL。

---

### 6. CronJob
- **理論說明**：定時排程任務，類似 Linux cron，週期性執行 Job。
- **YAML 範例**：
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
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
- **應用場景**：定時備份、定期報表、排程維護任務。
- **設計建議**：注意排程頻率與資源消耗，避免重疊執行。
- **實務建議/常見問題**：預設保留成功/失敗 Job 數量有限，需調整 `successfulJobsHistoryLimit`。

---

### 7. Scaling（HPA、VPA）
- **理論說明**：
  - **HPA（Horizontal Pod Autoscaler）**：根據 CPU、記憶體或自訂指標自動調整 Pod 數量。
  - **VPA（Vertical Pod Autoscaler）**：自動調整 Pod 的資源限制（CPU/記憶體）。
- **YAML 範例（HPA）**：
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
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```
- **YAML 範例（VPA）**：
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       Deployment
    name:       nginx-deployment
  updatePolicy:
    updateMode: "Auto"
```
- **應用場景**：流量彈性調整、資源最佳化。
- **設計建議**：HPA 適用於無狀態服務，VPA 適合資源需求變動大但不易水平擴展的應用。
- **實務建議/常見問題**：HPA 與 VPA 不建議同時作用於同一 Pod，監控指標需正確設置。

---
## 儲存與資料管理

### 1. 理論說明

- **Volume**
  - Pod 內部或多容器間共享資料的抽象，生命週期隨 Pod 存在。
  - 支援多種型態（emptyDir、hostPath、configMap、secret、persistentVolumeClaim 等）。
- **PersistentVolume（PV）**
  - 叢集級別的儲存資源，由管理員預先配置或動態供應。
  - 支援 NFS、iSCSI、雲端磁碟（如 AWS EBS、GCE PD）、本地磁碟等多種後端。
- **PersistentVolumeClaim（PVC）**
  - 使用者對儲存資源的請求，聲明所需容量、存取模式等。
  - 綁定至 PV 後，Pod 可掛載並存取資料。
- **StorageClass**
  - 定義儲存供應策略（如效能、備份、快照等），支援動態供應（Dynamic Provisioning）。
  - 常見參數：provisioner、parameters（如 type、zone）、reclaimPolicy。
- **動態供應（Dynamic Provisioning）**
  - 當 PVC 建立時，若無合適 PV，StorageClass 會自動建立 PV 並綁定。
  - 提升自動化與彈性，減少手動管理負擔。

---

### 2. 官方文件風格 YAML 範例

#### StorageClass

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

#### PersistentVolume（靜態）

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /srv/nfs
    server: 10.0.0.1
  persistentVolumeReclaimPolicy: Retain
```

#### PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: ""
```

#### PersistentVolumeClaim（動態供應）

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast
```

#### Pod 掛載 PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: busybox
      volumeMounts:
        - mountPath: "/data"
          name: data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: fast-pvc
```

---

### 3. StatefulSet 資料持久化設計重點

- 每個 Pod 會自動建立獨立 PVC，確保資料不因 Pod 重建而遺失。
- `volumeClaimTemplates` 用於自動產生 PVC，命名規則為 `<claim-name>-<statefulset-name>-<ordinal>（如 www-web-0）`。
- PVC 生命週期獨立於 Pod，刪除 StatefulSet 不會自動刪除 PVC，需手動清理。
- 適合部署需資料一致性、獨立儲存的應用（如資料庫、分散式系統）。
- 建議搭配支援快照、備份的 StorageClass，提升資料安全。

#### StatefulSet 持久化 YAML 範例

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 3
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
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
      storageClassName: fast
```

---

### 4. 常見應用場景

- **資料庫服務**：MySQL、PostgreSQL、MongoDB 等需持久化資料的應用。
- **分散式系統**：Kafka、ZooKeeper、Elasticsearch 等需節點獨立儲存。
- **CI/CD 產物儲存**：Jenkins、GitLab Runner 等需保存構建產物。
- **共用檔案系統**：NFS、GlusterFS 提供多 Pod 共享資料。

---

### 5. 設計建議與專業工程師實務經驗

- 優先使用動態供應，減少手動 PV 管理。
- 生產環境建議 StorageClass 設定 `reclaimPolicy: Retain`，避免誤刪資料。
- PVC 容量不可縮減，僅能擴充，規劃時預留成長空間。
- 監控 PVC 使用率，定期備份重要資料。
- StatefulSet 擴縮容時，需評估資料一致性與應用支援度。
- 多租戶環境建議搭配 ResourceQuota、LimitRange 控制儲存資源。
- 常見問題：PVC 綁定失敗多因 StorageClass 名稱錯誤、資源不足、權限設定不當。
- 建議熟悉各雲端儲存方案（如 AWS EBS、GCP PD、Azure Disk）特性與限制，選擇最適合業務需求的後端。

---
## 配置與密鑰管理

### 1. ConfigMap

- **理論說明**
  - ConfigMap 用於儲存非機密型態的組態資料（如環境變數、設定檔），讓應用程式與組態解耦。
  - 支援 key-value、檔案、目錄等多種資料型態。
  - 可動態更新，Pod 重新載入後自動取得新值。

- **官方文件風格 YAML 範例**
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: app-config
  data:
    APP_ENV: production
    LOG_LEVEL: info
    config.yaml: |
      port: 8080
      debug: false
  ```

- **常見應用場景**
  - 儲存應用環境參數、外部 API 端點、非敏感設定檔。
  - 多環境部署（dev/stage/prod）組態切換。

- **設計建議**
  - 建議將組態與映像檔分離，提升可維運性。
  - 大型設定檔可用檔案型態存放於 ConfigMap。
  - 避免將敏感資訊（如密碼、金鑰）存於 ConfigMap。

- **專業工程師實務經驗**
  - 建議搭配 Deployment 的 `envFrom`、`volumeMounts` 使用，彈性高。
  - 動態變更 ConfigMap 後，需重啟 Pod 才會生效（部分應用支援自動 reload）。
  - 常見問題：ConfigMap 名稱拼寫錯誤、key 不存在導致應用啟動失敗。

---

### 2. Secret

- **理論說明**
  - Secret 用於儲存敏感資料（如密碼、API 金鑰、TLS 憑證），以 base64 編碼方式存放。
  - 支援 Opaque（預設）、docker-registry、tls 等型態。
  - 權限嚴格控管，避免敏感資訊外洩。

- **官方文件風格 YAML 範例**
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: db-secret
  type: Opaque
  data:
    username: YWRtaW4=      # admin
    password: cGFzc3dvcmQ=  # password
  ```

- **常見應用場景**
  - 儲存資料庫帳密、API Token、TLS 憑證。
  - 私有映像倉庫認證（docker-registry）。

- **設計建議**
  - 建議搭配 RBAC 限制 Secret 存取權限。
  - 機密資料應以 Secret 管理，避免硬編碼於映像檔或 ConfigMap。
  - 定期輪換密鑰，並監控 Secret 使用狀態。

- **專業工程師實務經驗**
  - 建議搭配 `envFrom`、`volumeMounts` 將 Secret 注入 Pod。
  - 可結合外部密鑰管理系統（如 HashiCorp Vault、雲端 KMS）自動同步 Secret。
  - 常見問題：base64 編碼錯誤、權限設置不當導致應用無法存取。

---

### 3. RBAC（Role-Based Access Control）

- **理論說明**
  - RBAC 用於細緻控制 Kubernetes 物件的存取權限，依據角色（Role/ClusterRole）與綁定（Binding/ClusterBinding）授權。
  - 支援 Namespace 級（Role/RoleBinding）與叢集級（ClusterRole/ClusterRoleBinding）權限。

- **官方文件風格 YAML 範例**
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    namespace: dev
    name: pod-reader
  rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "list", "watch"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: read-pods
    namespace: dev
  subjects:
    - kind: User
      name: alice
      apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: Role
    name: pod-reader
    apiGroup: rbac.authorization.k8s.io
  ```

- **常見應用場景**
  - 多團隊協作、權限隔離、最小權限原則落實。
  - 自動化工具、CI/CD Pipeline 權限分離。

- **設計建議**
  - 採用最小權限原則（Least Privilege），僅授權必要操作。
  - 建議以 Namespace 為單位設計 Role，叢集級操作才用 ClusterRole。
  - 定期審查與調整權限，避免權限漂移。

- **專業工程師實務經驗**
  - 建議結合 ServiceAccount 管理自動化流程權限。
  - 可用 `kubectl auth can-i` 測試權限設定。
  - 常見問題：Role/Binding 名稱拼寫錯誤、subjects 設定不符、權限過寬。

---

### 4. ServiceAccount

- **理論說明**
  - ServiceAccount 為 Pod 提供身分識別，常用於自動化流程、應用程式存取 API。
  - 每個 Namespace 預設有 default ServiceAccount，亦可自訂多個。
  - 搭配 RBAC 控制 ServiceAccount 權限。

- **官方文件風格 YAML 範例**
  ```yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: ci-bot
    namespace: dev
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: ci-bot-binding
    namespace: dev
  subjects:
    - kind: ServiceAccount
      name: ci-bot
      namespace: dev
  roleRef:
    kind: Role
    name: pod-reader
    apiGroup: rbac.authorization.k8s.io
  ```

- **常見應用場景**
  - CI/CD Pipeline、Job、CronJob 等自動化任務。
  - 應用程式需存取 Kubernetes API。

- **設計建議**
  - 建議每個自動化流程/應用建立獨立 ServiceAccount，分離權限。
  - 搭配 RBAC 精細控管 ServiceAccount 可執行操作。
  - 避免所有 Pod 共用 default ServiceAccount。

- **專業工程師實務經驗**
  - 建議結合 Secret 管理 Token，提升安全性。
  - 可用 `automountServiceAccountToken: false` 降低敏感應用風險。
  - 常見問題：ServiceAccount 未正確綁定、權限不足導致任務失敗。

---

### 5. 綜合設計建議與實務經驗

- 配置與密鑰建議分層管理，敏感資料一律用 Secret，組態用 ConfigMap。
- RBAC 與 ServiceAccount 應搭配設計，落實最小權限原則。
- 建議定期檢查未使用的 Secret/ConfigMap，避免資源堆積與潛在風險。
- 生產環境建議結合外部密鑰管理系統，提升安全性與自動化能力。
- 常見問題多與權限設計、資源命名、資料注入方式有關，建議建立標準化流程與自動化檢查。

---
## 效能與可靠性

### 1. Probes（Liveness、Readiness、Startup）

- **理論說明**
  - **Liveness Probe**：偵測容器是否存活，失敗時自動重啟容器，避免應用進入無回應狀態。
  - **Readiness Probe**：判斷容器是否已準備好接收流量，未就緒時不會將流量導向該 Pod。
  - **Startup Probe**：專為啟動較慢的應用設計，啟動期間只檢查 Startup Probe，成功後才啟用 Liveness/Readiness。
- **YAML 範例**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: probe-demo
  spec:
    containers:
    - name: app
      image: busybox
      args: ["sh", "-c", "sleep 3600"]
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5
      readinessProbe:
        tcpSocket:
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
      startupProbe:
        exec:
          command: ["cat", "/tmp/ready"]
        failureThreshold: 30
        periodSeconds: 10
  ```
- **設計建議與實務經驗**
  - 建議所有生產服務皆設置 Liveness/Readiness Probe。
  - Startup Probe 適用於啟動時間長的應用（如 JVM）。
  - 避免 Probe 設定過於嚴格導致頻繁重啟。
  - 常見問題：Probe 路徑/端口錯誤、初始延遲設太短。

---

### 2. Pod 調度策略（NodeSelector、Affinity、Taints & Tolerations）

- **理論說明**
  - **NodeSelector**：最簡單的節點選擇方式，根據 label 指定 Pod 部署目標節點。
  - **Node Affinity**：進階節點選擇，支援 required/ preferred 條件，語法彈性高。
  - **Pod Affinity/Anti-Affinity**：根據其他 Pod label 決定同/不同節點部署，提升高可用性。
  - **Taints & Tolerations**：Node 設定 Taint 後，僅允許有對應 Toleration 的 Pod 調度至該節點。
- **YAML 範例**
  - NodeSelector
    ```yaml
    spec:
      nodeSelector:
        disktype: ssd
    ```
  - Node Affinity
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
    ```
  - Pod Anti-Affinity
    ```yaml
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - frontend
            topologyKey: "kubernetes.io/hostname"
    ```
  - Taints & Tolerations
    ```yaml
    # 節點加 Taint
    kubectl taint nodes node1 key=value:NoSchedule
    # Pod 設定 Toleration
    spec:
      tolerations:
      - key: "key"
        operator: "Equal"
        value: "value"
        effect: "NoSchedule"
    ```
- **設計建議與實務經驗**
  - 關鍵服務建議搭配 PodAntiAffinity 分散部署，提升容錯。
  - Taints 常用於專用節點（如 GPU/高效能），避免一般工作負載誤調度。
  - Node Affinity 適合多區域/異質硬體環境。
  - 常見問題：label 拼寫錯誤、Taint 未設 Toleration 導致 Pod 無法調度。

---

### 3. 資源管理（requests、limits、QoS）

- **理論說明**
  - **requests**：Pod 啟動與調度時保證分配的最小資源（CPU/記憶體）。
  - **limits**：Pod 可使用的最大資源上限，超過時會被限制（CPU）或 OOM Kill（記憶體）。
  - **QoS（Quality of Service）**：根據 requests/limits 設定自動分級，影響資源爭用時的優先順序。
    - Guaranteed：CPU/記憶體 requests=limits，最高優先。
    - Burstable：部分設 requests/limits，彈性調度。
    - BestEffort：未設 requests/limits，最低優先。
- **YAML 範例**
  ```yaml
  spec:
    containers:
    - name: app
      image: nginx
      resources:
        requests:
          memory: "128Mi"
          cpu: "250m"
        limits:
          memory: "256Mi"
          cpu: "500m"
  ```
- **設計建議與實務經驗**
  - 生產環境建議所有 Pod 設定 requests/limits，避免資源爭用。
  - 關鍵服務建議設為 Guaranteed，提升穩定性。
  - 定期監控實際資源用量，調整 requests/limits。
  - 常見問題：requests 設過低導致調度延遲，limits 設過低導致 OOM。

---

### 4. 常見應用場景、設計建議與專業工程師實務經驗

- **應用場景**
  - 金融、電商等高可用服務：建議多區域部署、PodAntiAffinity、資源預留。
  - AI/ML、GPU 工作負載：Taints & Tolerations 管理專用節點，Node Affinity 指定硬體。
  - 多租戶/多環境：合理規劃 QoS、資源配額，避免資源爭用。
- **設計建議**
  - 重要服務務必設置 Probe 與資源限制，並分散部署。
  - 定期壓力測試，根據監控數據調整資源配置。
  - 使用 HPA/VPA 動態調整資源，提升彈性與效能。
  - 關鍵節點建議設 Taint，僅允許特定 Pod 部署。
- **專業工程師實務經驗**
  - 生產環境常見問題多與 Probe 設定錯誤、資源配置不當、調度策略未落實有關。
  - 建議建立標準化 YAML 範本，並結合 CI/CD 自動檢查資源與 Probe 設定。
  - 監控與告警（如 Prometheus + Alertmanager）是維持效能與可靠性的關鍵。
  - 經驗法則：寧願資源預留多一點，避免 OOM/資源爭用導致服務中斷。
## 進階功能

### 1. Operators

- **理論說明**
  - Operator 是以 Kubernetes 原生 API 擴充自動化運維的設計模式，結合 CRD（CustomResourceDefinition）與 Controller，將人類運維知識程式化，實現應用全生命週期管理（部署、升級、備份、恢復等）。
  - 常見於資料庫、分散式系統等需複雜狀態管理的應用。

- **官方文件風格 YAML 範例**
  ```yaml
  apiVersion: "etcd.database.coreos.com/v1beta2"
  kind: EtcdCluster
  metadata:
    name: example
  spec:
    size: 3
    version: "3.4.13"
  ```
- **常見應用場景**
  - 自動化部署/升級/備份資料庫（如 etcd、PostgreSQL、MongoDB）。
  - 複雜分散式應用（如 Kafka、ElasticSearch）。
- **設計建議與實務經驗**
  - 建議選用社群活躍、維護良好的 Operator。
  - 生產環境需評估 Operator 權限與安全性，避免過度授權。
  - 可用 Operator SDK、Kubebuilder 開發自訂 Operator。

---

### 2. CRD（CustomResourceDefinition）

- **理論說明**
  - CRD 允許用戶自訂 Kubernetes API 物件，擴充原生資源型態，配合 Controller 實現自動化管理。
  - CRD 物件可用 `kubectl` 操作，支援 YAML/JSON 定義。

- **官方文件風格 YAML 範例**
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
        schema:
          openAPIV3Schema:
            type: object
            properties:
              spec:
                type: object
                properties:
                  cronSpec:
                    type: string
                  image:
                    type: string
                  replicas:
                    type: integer
    scope: Namespaced
    names:
      plural: crontabs
      singular: crontab
      kind: Crontab
      shortNames:
      - ct
  ```
- **常見應用場景**
  - 擴充叢集 API，管理自訂資源（如 CronTab、Backup、AppConfig）。
- **設計建議與實務經驗**
  - 建議設計明確的 schema，善用 validation，提升穩定性。
  - CRD 版本管理建議採用 `versions` 欄位，便於升級與相容性維護。
  - Controller 必須處理 CRD 物件的全生命週期，避免殘留資源。

---

### 3. Admission Controller

- **理論說明**
  - Admission Controller 於 API Server 處理請求時進行攔截、驗證、修改，分為 Mutating（可修改）與 Validating（僅驗證）兩類。
  - 常用於安全加固、資源標準化、政策落實（如強制加 label、驗證映像來源）。

- **官方文件風格 YAML 範例（Webhook 註冊）**
  ```yaml
  apiVersion: admissionregistration.k8s.io/v1
  kind: ValidatingWebhookConfiguration
  metadata:
    name: example-validating-webhook
  webhooks:
    - name: validate.example.com
      clientConfig:
        service:
          name: example-webhook
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
- **常見應用場景**
  - 強制命名規範、資源配額、映像安全掃描、Pod 安全政策（PSP/Opa Gatekeeper/Kyverno）。
- **設計建議與實務經驗**
  - 建議 Mutating/Validating Webhook 服務需高可用，避免阻塞 API Server。
  - 生產環境建議設置逾時與失敗策略（如 `failurePolicy: Ignore`）。
  - 可用 OPA Gatekeeper、Kyverno 實現進階政策管理。

---

### 4. API Server 擴充

- **理論說明**
  - 除 CRD 外，Kubernetes 支援 Aggregated API Server，可註冊外部 API Server，實現更進階的 API 擴充。
  - 適合需獨立認證、複雜邏輯或高效能需求的自訂資源。

- **官方文件風格 YAML 範例（APIService 註冊）**
  ```yaml
  apiVersion: apiregistration.k8s.io/v1
  kind: APIService
  metadata:
    name: v1beta1.example.com
  spec:
    service:
      name: example-apiserver
      namespace: default
    group: example.com
    version: v1beta1
    insecureSkipTLSVerify: true
    groupPriorityMinimum: 2000
    versionPriority: 10
  ```
- **常見應用場景**
  - 需自訂認證/授權、跨叢集 API 聚合、複雜業務邏輯。
- **設計建議與實務經驗**
  - Aggregated API Server 適合大型平台級擴充，需考慮維運與升級複雜度。
  - 建議先評估 CRD 是否足夠，僅於必要時採用 Aggregated API Server。

---

### 5. Helm

- **理論說明**
  - Helm 是 Kubernetes 的套件管理工具，透過 Chart（模板）自動化部署、升級、回滾應用，提升維運效率與一致性。
  - 支援參數化、版本管理、依賴管理。

- **官方文件風格指令範例**
  ```bash
  # 安裝 Helm
  curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

  # 新增 Chart 倉庫
  helm repo add bitnami https://charts.bitnami.com/bitnami

  # 搜尋套件
  helm search repo mysql

  # 安裝套件
  helm install my-mysql bitnami/mysql --set auth.rootPassword=secretpassword

  # 升級套件
  helm upgrade my-mysql bitnami/mysql --set auth.rootPassword=newpassword

  # 移除套件
  helm uninstall my-mysql
  ```
- **常見應用場景**
  - 標準化部署中間件（如 MySQL、Redis、Prometheus）。
  - 企業內部自訂應用模板管理。
- **設計建議與實務經驗**
  - 建議建立企業內部 Chart 倉庫，統一管理模板。
  - Chart 參數建議分層設計，區分環境與敏感資訊。
  - 生產環境建議搭配 GitOps（如 ArgoCD、Flux）自動化部署。

---

### 6. 綜合應用場景與專業工程師實務經驗

- **Operators/CRD**
  - 適合自動化複雜應用生命週期管理，減少人為操作錯誤。
  - 建議先評估現有 Operator，必要時再自訂開發。
- **Admission Controller**
  - 強化安全、標準化資源，建議逐步導入並測試政策影響。
- **API Server 擴充**
  - 僅於 CRD/Operator 無法滿足需求時採用，需評估維運負擔。
- **Helm**
  - 適合團隊協作、標準化部署，建議結合 CI/CD 流程。
- **設計建議**
  - 進階功能多涉及權限與安全，建議搭配 RBAC、審計日誌。
  - 生產環境建議先於測試叢集驗證，並建立回滾/備份機制。
  - 持續關注社群動態，定期更新 Operator/Chart 版本，避免安全風險。

---
---

## 監控與日誌

### 1. 監控（Metrics Server、Prometheus、Grafana）

- **理論說明**
  - **Metrics Server**：Kubernetes 內建資源監控元件，提供 CPU/記憶體即時指標，支援 HPA/VPA。
  - **Prometheus**：CNCF 主流監控系統，支援多維度指標收集、查詢、告警，與 Kubernetes 深度整合。
  - **Grafana**：可視化平台，支援多種資料來源（含 Prometheus），用於儀表板展示與告警。

- **部署指令與 YAML 範例**
  - Metrics Server
    ```bash
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    ```
  - Prometheus（Helm 安裝）
    ```bash
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    helm install prometheus prometheus-community/kube-prometheus-stack
    ```
  - Grafana（隨 Prometheus Stack 一併部署，或單獨安裝）
    ```bash
    kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
    kubectl port-forward svc/prometheus-grafana 3000:80
    # 瀏覽器開啟 http://localhost:3000
    ```

- **常見應用場景**
  - HPA/VPA 自動擴縮、資源瓶頸分析、服務健康監控、異常告警。
- **設計建議與實務經驗**
  - 生產環境建議用 Prometheus + Alertmanager，並設置持久化儲存。
  - 儀表板可用社群模板（Grafana.com），自訂告警規則。
  - 監控指標建議分層（系統、應用、業務），便於定位問題。

---

### 2. 日誌管理（EFK/ELK Stack）

- **架構說明**
  - **EFK/ELK Stack**：Elasticsearch（儲存/搜尋）、Fluentd/Logstash（收集/轉換）、Kibana（可視化）。
  - EFK（Fluentd）更適合 Kubernetes，Logstash 適用於複雜轉換需求。

- **部署重點與 YAML 範例**
  - Fluentd DaemonSet（收集所有 Node 日誌）
    ```yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: fluentd
      namespace: kube-system
    spec:
      selector:
        matchLabels:
          name: fluentd
      template:
        metadata:
          labels:
            name: fluentd
        spec:
          containers:
          - name: fluentd
            image: fluent/fluentd:v1.16
            env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
    ```
  - Elasticsearch（Helm 安裝）
    ```bash
    helm repo add elastic https://helm.elastic.co
    helm install elasticsearch elastic/elasticsearch
    ```
  - Kibana（Helm 安裝）
    ```bash
    helm install kibana elastic/kibana
    ```

- **常見應用場景**
  - 集中日誌查詢、異常追蹤、稽核合規、故障排查。
- **設計建議與實務經驗**
  - 日誌建議標準化格式（JSON），便於搜尋與分析。
  - 生產環境建議設儲存備援、定期清理舊日誌。
  - Fluent Bit 可替代 Fluentd，資源消耗更低。

---

### 3. 分散式追蹤（Jaeger）

- **理論說明**
  - Jaeger 為 CNCF 分散式追蹤系統，支援 OpenTracing 標準，適用於微服務架構下的請求鏈路追蹤、瓶頸分析。

- **部署指令與 YAML 範例**
  - Helm 安裝 Jaeger
    ```bash
    helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
    helm repo update
    helm install jaeger jaegertracing/jaeger
    ```
  - 服務注入追蹤 sidecar（以 Deployment 為例）
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: app
    spec:
      template:
        spec:
          containers:
          - name: app
            image: my-app:latest
            env:
            - name: JAEGER_AGENT_HOST
              value: "jaeger-agent"
          - name: jaeger-agent
            image: jaegertracing/jaeger-agent
    ```

- **常見應用場景**
  - 微服務請求鏈路追蹤、效能瓶頸定位、跨服務延遲分析。
- **設計建議與實務經驗**
  - 建議所有服務皆注入 tracing SDK，並標註關鍵業務資訊。
  - 生產環境建議設置持久化後端（如 Elasticsearch）。
  - 可結合 Grafana Tempo、OpenTelemetry 強化觀察性。

---

### 4. 官方文件風格 YAML/指令範例

- 監控與日誌元件建議以 Helm 部署，便於升級與參數化。
- DaemonSet 適合部署日誌收集器（Fluentd/Fluent Bit）。
- 監控/日誌/追蹤服務建議設置獨立 Namespace，便於管理與權限控管。

---

### 5. 常見應用場景、設計建議與專業工程師實務經驗

- **應用場景**
  - 自動擴縮（HPA）、異常告警、故障排查、稽核合規、效能分析。
- **設計建議**
  - 監控、日誌、追蹤三者建議整合（Observability），提升問題定位效率。
  - 指標、日誌、追蹤資料建議分層管理，並設置儲存備援與告警。
  - 生產環境建議設置多副本、持久化儲存、定期備份。
- **專業工程師實務經驗**
  - 監控指標過多會影響效能，建議聚焦核心業務與系統指標。
  - 日誌量大時建議設置過濾與分流，避免儲存壓力。
  - 追蹤系統建議與 APM（如 Grafana Tempo、OpenTelemetry）整合，提升全鏈路可觀察性。
  - 定期檢查監控、日誌、追蹤系統健康狀態，並測試告警通知流程。

---
---
## 安全性與治理

### 1. NetworkPolicy 網路政策
- **理論說明**：
  - 用於限制 Pod 間、Pod 與外部的網路流量，實現微服務間的最小權限原則。
  - 以 Namespace 為範圍，預設允許所有流量，需明確定義規則才會生效。
- **YAML 範例**：
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-nginx
    namespace: demo
  spec:
    podSelector:
      matchLabels:
        app: nginx
    policyTypes:
    - Ingress
    ingress:
    - from:
      - podSelector:
          matchLabels:
            access: "true"
      ports:
      - protocol: TCP
        port: 80
  ```
- **常見應用場景**：
  - 僅允許特定應用之間通訊，阻擋未授權存取。
  - 分層防禦，減少橫向移動風險。
- **設計建議與實務經驗**：
  - 建議預設拒絕（deny all），逐步開放必要流量。
  - 測試 NetworkPolicy 前，確認 CNI 插件支援。
  - 逐步導入，避免一次性全開導致服務中斷。

---

### 2. Pod Security Standards（PSS）
- **理論說明**：
  - Kubernetes 官方安全標準，分為 privileged、baseline、restricted 三層。
  - 透過 Admission Controller（如 PodSecurity Admission）強制執行。
- **YAML 範例**（Namespace 級別）：
  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: secure-ns
    labels:
      pod-security.kubernetes.io/enforce: "restricted"
      pod-security.kubernetes.io/enforce-version: "latest"
  ```
- **常見應用場景**：
  - 金融、醫療等高敏感產業，強制執行 restricted。
  - 開發環境可用 baseline，生產環境建議 restricted。
- **設計建議與實務經驗**：
  - 先以 warn 模式觀察違規情形，再切換 enforce。
  - 定期審查與更新政策，配合 CI/CD 自動檢查。

---

### 3. Secret 管理（Sealed Secrets、Vault）
- **理論說明**：
  - Secret 用於儲存敏感資訊（如密碼、API 金鑰）。
  - Sealed Secrets：將 Secret 加密後存於 Git，解密僅在 K8s 叢集內進行。
  - Vault：集中式密鑰管理，支援動態憑證、審計與權限控管。
- **Sealed Secrets YAML 範例**：
  ```yaml
  apiVersion: bitnami.com/v1alpha1
  kind: SealedSecret
  metadata:
    name: mysecret
    namespace: demo
  spec:
    encryptedData:
      password: AgB+...
  ```
- **Vault CLI 指令範例**：
  ```bash
  vault kv put secret/myapp password=123456
  vault kv get secret/myapp
  ```
- **常見應用場景**：
  - GitOps 流程下安全儲存 Secret。
  - 多團隊、多環境共用集中密鑰管理。
- **設計建議與實務經驗**：
  - 避免明文 Secret 存於 Git。
  - 定期輪換密鑰，並設置權限最小化。
  - Sealed Secrets 適合 GitOps，Vault 適合大規模多租戶。

---

### 4. 多租戶與 Namespace 隔離
- **理論說明**：
  - Namespace 提供邏輯隔離，適合多團隊、多專案共用叢集。
  - 可搭配 ResourceQuota、LimitRange、NetworkPolicy、PSS 強化隔離。
- **YAML 範例**（ResourceQuota）：
  ```yaml
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
- **常見應用場景**：
  - SaaS 平台多租戶隔離。
  - 企業內部多部門共用單一叢集。
- **設計建議與實務經驗**：
  - 每個租戶/團隊獨立 Namespace，並設置資源配額。
  - 結合 RBAC、NetworkPolicy、PSS，實現多層防護。
  - 定期審查 Namespace 資源使用與安全政策。

---
## CI/CD 與自動化

### 1. GitOps 理論與工具
- **GitOps 概念**：以 Git 作為唯一真相來源（Single Source of Truth），所有部署、設定皆透過 Git 版本控管，實現自動化部署與回溯。
- **核心流程**：
  - 開發者提交 PR → 合併至主分支 → GitOps 工具偵測變更 → 自動同步至叢集
- **常用工具**：
  - **ArgoCD**：支援多種 K8s 資源同步、視覺化 UI、RBAC 管理。
  - **Flux**：輕量、原生整合 Git，支援自動化鏡像更新。
- **YAML 範例（ArgoCD Application）**：
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: my-app
    namespace: argocd
  spec:
    project: default
    source:
      repoURL: 'https://github.com/org/repo.git'
      targetRevision: HEAD
      path: manifests/
    destination:
      server: 'https://kubernetes.default.svc'
      namespace: default
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
  ```
- **常見應用場景**：
  - 多環境部署（dev/stage/prod）
  - 快速回滾、審計追蹤
- **設計建議**：
  - 採用分支/資料夾分離多環境
  - 配合 RBAC 控制權限
  - 配合 Image Update Automation 實現自動升級

---

### 2. Pipeline 理論與工具
- **Pipeline 概念**：將軟體交付流程自動化，涵蓋建置、測試、部署等階段。
- **常用工具**：
  - **Tekton**：K8s 原生 CI/CD，資源型態（Pipeline/Task/Trigger），彈性高。
  - **Jenkins X**：結合 Jenkins 與 K8s，支援自動化 Preview Environment、GitOps。
- **YAML 範例（Tekton Pipeline）**：
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
        runAfter: [build]
        taskRef:
          name: deploy-to-k8s
  ```
- **常見應用場景**：
  - PR 驗證、單元測試、端對端測試
  - 多階段部署（Blue/Green、Canary）
- **設計建議**：
  - 任務拆分細緻，便於重用
  - 配合 Secret/ConfigMap 管理敏感資訊
  - Pipeline as Code，所有流程皆版本控管

---

### 3. 自動化部署與滾動更新策略
- **自動化部署**：結合 GitOps 與 Pipeline，實現從 Commit 到 Production 全自動化。
- **滾動更新（Rolling Update）**：
  - **Kubernetes Deployment** 預設支援滾動更新，逐步替換 Pod，確保服務不中斷。
  - **YAML 範例**：
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
            - name: app
              image: my-app:latest
    ```
- **進階策略**：
  - **Canary**：小流量先行，逐步放量
  - **Blue/Green**：新舊環境並存，切換流量
- **專業建議**：
  - 配合 Probe（Readiness/Liveness）確保健康檢查
  - 設定合理的 maxSurge/maxUnavailable
  - 監控指標（如錯誤率、延遲）自動回滾
  - 部署自動化需與審核、權限、審計流程整合

---

### 4. 工程師實務經驗分享
- **常見挑戰**：
  - YAML/流程複雜，建議模組化、範本化
  - 權限控管與審計需完善
  - 多團隊協作時，建議明確分工與責任歸屬
- **最佳實踐**：
  - 版本控管所有部署資源
  - 建立自動化測試與驗證流程
  - 定期演練回滾與異常處理
  - 文件與流程透明，便於新成員上手
---
## 維運與實務案例

### 1. 常見錯誤與排查方法

- **kubectl debug**
  - 啟動除錯 Pod，進行現場問題分析。
  - 指令範例：
    ```bash
    kubectl debug pod/<POD_NAME> -it --image=busybox
    ```
  - 適用：容器啟動失敗、網路異常、環境變數檢查。

- **kubectl logs**
  - 查詢 Pod/容器日誌，快速定位應用錯誤。
  - 指令範例：
    ```bash
    kubectl logs <POD_NAME>
    kubectl logs <POD_NAME> -c <CONTAINER_NAME>
    kubectl logs -f <POD_NAME>
    ```
  - 適用：CrashLoopBackOff、應用異常、啟動失敗。

- **kubectl describe**
  - 詳細檢視資源狀態、事件、錯誤訊息。
  - 指令範例：
    ```bash
    kubectl describe pod <POD_NAME>
    kubectl describe node <NODE_NAME>
    ```
  - 適用：資源調度失敗、事件追蹤、狀態異常。

- **常見排查流程**
  - 先查 logs，後 describe，必要時 debug 進入容器。
  - 檢查事件（Events）、狀態（Status）、資源（Resource）與網路（Network）。
  - 常見錯誤：ImagePullBackOff、CrashLoopBackOff、Pending、OOMKilled、Node NotReady。

---

### 2. 升級與版本管理重點

- **升級前準備**
  - 先於測試環境驗證升級流程與相容性。
  - 備份 etcd、重要資料與設定檔。
  - 查閱[官方升級指南](https://kubernetes.io/zh/docs/tasks/administer-cluster/cluster-upgrade/)。

- **升級指令範例（kubeadm）**
  ```bash
  # 查詢可用版本
  kubeadm version
  apt-cache madison kubeadm

  # 升級 kubeadm
  sudo apt-get update && sudo apt-get install -y kubeadm=<VERSION>

  # 升級控制平面
  sudo kubeadm upgrade plan
  sudo kubeadm upgrade apply v<VERSION>

  # 升級 kubelet/kubectl
  sudo apt-get install -y kubelet=<VERSION> kubectl=<VERSION>
  sudo systemctl restart kubelet
  ```

- **版本管理建議**
  - 控制平面與節點建議同版本或最多差一小版本（n、n-1）。
  - 定期追蹤[官方發布週期](https://kubernetes.io/releases/)。
  - 升級前先查詢相依元件（CNI、Storage、Ingress）相容性。
  - 建議使用 Helm 管理應用升級，便於回滾。

---

### 3. 災難恢復與高可用架構設計

- **etcd 備份與還原**
  - 定期備份 etcd，並測試還原流程。
  - 備份指令：
    ```bash
    ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key
    ```
  - 還原指令：
    ```bash
    ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db
    ```

- **高可用設計重點**
  - 多 Master 架構，etcd 奇數台部署。
  - kube-apiserver 前加負載平衡（HAProxy/Nginx/雲端 LB）。
  - 關鍵元件多副本部署，設置 PodAntiAffinity。
  - 定期演練備援與還原流程。

- **實務經驗**
  - 災難發生時，先確認 etcd、apiserver 狀態，再逐步恢復 Node 與應用。
  - 建議建立自動備份與監控告警機制。

---

### 4. 官方文件風格指令/範例

- **常用維運指令**
  ```bash
  # 查詢所有 Pod 狀態
  kubectl get pods -A

  # 查詢節點狀態
  kubectl get nodes

  # 重新啟動 Deployment
  kubectl rollout restart deployment/<DEPLOYMENT_NAME>

  # 查看資源事件
  kubectl get events -A --sort-by='.lastTimestamp'

  # 檢查資源使用量
  kubectl top node
  kubectl top pod -A
  ```

- **YAML 範例：Pod 除錯**
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: debug-pod
  spec:
    containers:
    - name: debug
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
  ```

---

### 5. 常見應用場景、設計建議與專業工程師實務經驗

- **應用場景**
  - 緊急修復：使用 debug/exec 進入容器，臨時修補設定。
  - 線上升級：滾動更新、金絲雀發布，確保不中斷服務。
  - 故障排查：結合 logs、describe、events 快速定位問題。
  - 資源瓶頸：top、metrics-server 分析資源用量，調整 requests/limits。
  - 災難恢復：etcd 快照還原、YAML 版控快速重建。

- **設計建議**
  - 建立標準化維運流程與自動化腳本。
  - 關鍵服務多副本、分散部署，提升容錯。
  - 定期備份與演練還原，降低災難風險。
  - 監控、日誌、告警三位一體，提升可觀察性。
  - 升級/變更前必做測試與備份。

- **專業工程師經驗**
  - 問題多發生於資源不足、權限設定、網路異常。
  - 建議建立 FAQ 與常見錯誤對照表，提升團隊解決效率。
  - 版本升級建議分階段、逐步驗證，避免一次性大幅變動。
  - 重要指令與 YAML 範本建議集中管理，便於查詢與複用。