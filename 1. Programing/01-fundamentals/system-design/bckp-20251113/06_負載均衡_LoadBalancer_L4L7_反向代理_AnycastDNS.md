# 負載均衡、L4/L7、反向代理與 Anycast DNS

## 1. 負載均衡理論解釋

### 負載均衡（Load Balancing）是什麼？
負載均衡是一種將流量或請求分散到多個伺服器或資源的技術，目的是提升系統的可用性、擴展性與容錯能力。常見應用於網站、API、資料庫等服務。

### L4 與 L7 負載均衡差異
- **L4（第四層，傳輸層）**
  依據 TCP/UDP 等協定的連線資訊（如 IP、Port）進行分流，速度快、效率高，但無法解析應用層內容。
- **L7（第七層，應用層）**
  依據 HTTP/HTTPS 等協定的應用層內容（如 URL、Header、Cookie）進行分流，可做更細緻的路由與策略，但效能較 L4 低。

| 層級 | 依據 | 優點 | 缺點 |
|------|------|------|------|
| L4   | IP/Port | 效能高、延遲低 | 無法解析應用層內容 |
| L7   | URL/Header/Cookie | 可做複雜路由、靈活性高 | 效能較低、延遲較高 |

### 反向代理（Reverse Proxy）
反向代理是一種代理伺服器，位於用戶端與伺服器之間，接收用戶請求後再轉發給後端伺服器。常用於：
- 隱藏內部架構
- 負載均衡
- SSL 終止
- 快取與安全性強化

### Anycast DNS
Anycast DNS 是一種網路路由技術，將同一個 IP 位址配置在多個地理位置的伺服器上，讓用戶端自動連到最近的節點。優點包括：
- 降低延遲
- 增強可用性與容錯
- 抵禦 DDoS 攻擊

---

## 2. 架構圖解

```mermaid
flowchart TD
    User(用戶端)
    subgraph Internet
      AnycastDNS[Anycast DNS]
    end
    L4[負載均衡器 (L4)]
    L7[負載均衡器/反向代理 (L7)]
    Backend1[後端伺服器 A]
    Backend2[後端伺服器 B]

    User --> AnycastDNS
    AnycastDNS --> L4
    L4 --> L7
    L7 --> Backend1
    L7 --> Backend2
```

- **Anycast DNS**：將流量導向最近的入口節點
- **L4 負載均衡**：根據 IP/Port 分流
- **L7 反向代理**：根據應用層內容進行進一步分流與處理

---

## 3. 真實世界配置範例

### Nginx L7 反向代理設定

```nginx
http {
    upstream backend {
        server backend1.example.com;
        server backend2.example.com;
    }

    server {
        listen 80;
        server_name www.example.com;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

### Anycast DNS 配置片段（以 BGP 為例）

> Anycast DNS 需 ISP 或雲端服務商支援，以下為 BGP 宣告範例（僅供參考）：

```bash
# BGP 宣告 Anycast IP
router bgp 65001
  network 203.0.113.1/32
  neighbor 192.0.2.1 remote-as 65002
  neighbor 192.0.2.1 route-reflector-client
```

---

## 4. 架構師實務建議與 Trade-off 分析

### 實務建議
- **多層負載均衡**：結合 Anycast DNS、L4、L7 可提升彈性與可用性。
- **監控與健康檢查**：確保各層節點健康，避免單點故障。
- **自動擴展**：搭配自動化工具（如 Kubernetes、Auto Scaling）動態調整資源。
- **安全性設計**：L7 層可整合 WAF、DDoS 防護，DNS 層建議啟用 DNSSEC。

### Trade-off 分析
- **效能 vs. 靈活性**：L4 效能高但功能有限，L7 靈活但資源消耗較大。
- **複雜度 vs. 可維護性**：多層架構提升彈性，但部署與維運複雜度增加。
- **成本考量**：Anycast DNS 需額外網路資源與 ISP 支援，成本較高。
- **單點故障風險**：每一層都需設計高可用，避免成為瓶頸。

---

## 參考資料
- [NGINX 負載均衡官方文件](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
- [Cloudflare Anycast 技術說明](https://www.cloudflare.com/zh-tw/learning/cdn/glossary/anycast/)
- [RFC 4786: Operation of Anycast Services](https://datatracker.ietf.org/doc/html/rfc4786)