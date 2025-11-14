# 第六章：網路與服務發現

---

## 1. ClusterIP、NodePort、LoadBalancer、Ingress 的原理、用途與差異

### 1.1 ClusterIP
- **原理**：預設的 Service 類型，僅在 Kubernetes 叢集內部提供虛擬 IP，Pod 可透過此 IP 互相存取。
- **用途**：內部服務通訊（如微服務間 API 呼叫）。
- **差異**：無法由叢集外部直接存取。

#### 範例 YAML
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-clusterip
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

#### 常用指令
```bash
kubectl get svc
kubectl describe svc my-clusterip
```

---

### 1.2 NodePort
- **原理**：在每個 Node 上開放一個指定範圍（30000-32767）的 port，將流量轉發至對應的 Service。
- **用途**：測試、開發環境，或簡易對外服務。
- **差異**：可由外部透過 Node IP + Port 存取，但不適合正式環境。

#### 範例 YAML
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

#### 常用指令
```bash
kubectl get svc
kubectl describe svc my-nodeport
```

---

### 1.3 LoadBalancer
- **原理**：結合雲端平台的負載平衡器，將外部流量導向 Service。
- **用途**：正式環境對外服務，需雲端支援（如 GCP、AWS）。
- **差異**：自動分配外部 IP，適合生產環境。

#### 範例 YAML
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

#### 常用指令
```bash
kubectl get svc
kubectl describe svc my-loadbalancer
```

---

### 1.4 Ingress
- **原理**：以 HTTP/HTTPS 方式統一管理多個 Service 的路由，通常搭配 Ingress Controller（如 NGINX）。
- **用途**：提供反向代理、TLS 終止、路由規則等進階功能。
- **差異**：可依路徑/主機名分流，適合複雜應用。

#### 範例 YAML
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

#### 常用指令
```bash
kubectl get ingress
kubectl describe ingress my-ingress
```

---

### 1.5 差異比較表

| 類型         | 內部存取 | 外部存取 | 負載平衡 | 進階路由 | 適用場景     |
|--------------|----------|----------|----------|----------|--------------|
| ClusterIP    | ✔        | ✘        | ✘        | ✘        | 內部服務     |
| NodePort     | ✔        | ✔        | ✘        | ✘        | 測試/開發    |
| LoadBalancer | ✔        | ✔        | ✔        | ✘        | 正式對外服務 |
| Ingress      | ✔        | ✔        | ✔        | ✔        | 複雜路由     |

---

## 2. CNI 原理與常見方案（Flannel、Calico、Cilium）

### 2.1 CNI（Container Network Interface）原理
- **說明**：CNI 是一套容器網路插件標準，負責 Pod 網路的分配、連線與管理。
- **運作流程**：Kubelet 啟動 Pod 時，呼叫 CNI 插件建立網路介面，分配 IP，並設定路由。

---

### 2.2 常見 CNI 方案

#### Flannel
- **原理**：以 Overlay Network（VXLAN）為主，簡單易用，適合小型叢集。
- **特色**：部署簡單，效能中等，功能較基礎。

#### Calico
- **原理**：以 BGP 為基礎，支援 L3 路由，具備網路政策（Network Policy）功能。
- **特色**：高效能、彈性高、支援大規模叢集與安全控管。

#### Cilium
- **原理**：基於 eBPF，支援 L3/L4/L7 網路安全與可觀察性。
- **特色**：高效能、進階安全、支援 Service Mesh 整合。

---

### 2.3 CNI 安裝與查詢指令

```bash
# 查詢目前 CNI 插件
kubectl get pods -n kube-system -o wide

# 以 manifest 安裝 Flannel 範例
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# 以 manifest 安裝 Calico 範例
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

---

## 3. Service Mesh（Istio、Linkerd）簡介

### 3.1 Service Mesh 定義
- **說明**：Service Mesh 是一種專為微服務架構設計的網路層，負責服務間通訊、流量管理、安全與可觀察性。
- **運作方式**：以 Sidecar Proxy（如 Envoy）注入至每個 Pod，攔截並管理流量。

---

### 3.2 常見方案

#### Istio
- **特色**：功能最完整，支援流量控制、認證、監控、金絲雀發布等。
- **適用**：大型、複雜的微服務架構。

#### Linkerd
- **特色**：輕量、易於部署，專注於核心功能（流量管理、可觀察性）。
- **適用**：中小型叢集，對效能有要求者。

---

### 3.3 基本操作指令

```bash
# 安裝 Istio（以 istioctl 為例）
istioctl install --set profile=demo

# 安裝 Linkerd（以 CLI 為例）
linkerd install | kubectl apply -f -

# 查詢 Sidecar 注入狀態
kubectl get pods --all-namespaces -o jsonpath='{.items[*].metadata.annotations}'
```

---

## 4. 各物件/方案的 YAML 範例與常用 kubectl 操作指令

### 4.1 Service 物件
- 參見前述 ClusterIP、NodePort、LoadBalancer 範例。

### 4.2 Ingress 物件
- 參見前述 Ingress 範例。

### 4.3 NetworkPolicy 範例（以 Calico 為例）
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx
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
```

#### 常用指令
```bash
kubectl get networkpolicy
kubectl describe networkpolicy allow-nginx
```

### 4.4 Cilium Hubble 可觀察性查詢
```bash
# 啟用 Hubble UI
cilium hubble enable
cilium hubble ui
```

---

## 5. 專業工程師的網路設計與服務發現實務建議

- **選擇合適的 Service 類型**：內部服務用 ClusterIP，對外服務建議搭配 Ingress 或 LoadBalancer。
- **CNI 選型**：小型叢集可用 Flannel，需安全與彈性則選 Calico，追求高效能與進階功能可考慮 Cilium。
- **善用 NetworkPolicy**：強化網路安全，僅允許必要流量。
- **導入 Service Mesh**：微服務架構建議導入 Service Mesh 提升流量管理與可觀察性。
- **監控與日誌**：結合 Prometheus、Grafana、Hubble 等工具，隨時掌握網路狀態。
- **服務發現**：Kubernetes 內建 DNS 支援自動服務發現，建議統一命名規則，方便維運。
- **彈性設計**：預留擴展空間，避免單點故障，善用多副本與負載平衡。

---
