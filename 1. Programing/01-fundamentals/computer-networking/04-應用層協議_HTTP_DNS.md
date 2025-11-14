# 應用層協議

## HTTP/HTTPS 協議深入解析

### HTTP 協議基礎

HTTP (HyperText Transfer Protocol) 是應用層的無狀態協議，基於 TCP 連接。

#### HTTP 請求結構
```
GET /api/users HTTP/1.1
Host: example.com
User-Agent: curl/7.68.0
Accept: application/json
Content-Type: application/json
Content-Length: 25

{"username": "john"}
```

#### HTTP 響應結構
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 45
Server: nginx/1.18.0

{"id": 123, "username": "john"}
```

### HTTP 方法與狀態碼

#### 常用 HTTP 方法
```bash
# GET - 獲取資源
curl -X GET https://api.example.com/users

# POST - 創建資源
curl -X POST -H "Content-Type: application/json" \
     -d '{"name":"john"}' https://api.example.com/users

# PUT - 更新資源
curl -X PUT -H "Content-Type: application/json" \
     -d '{"name":"jane"}' https://api.example.com/users/123

# DELETE - 刪除資源
curl -X DELETE https://api.example.com/users/123
```

#### 重要狀態碼分類
```
1xx - 資訊性響應
  100 Continue
  101 Switching Protocols

2xx - 成功響應
  200 OK
  201 Created
  204 No Content

3xx - 重定向
  301 Moved Permanently
  302 Found
  304 Not Modified

4xx - 客戶端錯誤
  400 Bad Request
  401 Unauthorized
  403 Forbidden
  404 Not Found
  429 Too Many Requests

5xx - 服務器錯誤
  500 Internal Server Error
  502 Bad Gateway
  503 Service Unavailable
  504 Gateway Timeout
```

### HTTP/1.1 vs HTTP/2 vs HTTP/3

```mermaid
graph TB
    subgraph "HTTP/1.1"
        A1[單一連接] --> B1[串行請求]
        B1 --> C1[Head-of-line blocking]
    end
    
    subgraph "HTTP/2"
        A2[多路復用] --> B2[併行請求]
        B2 --> C2[Server Push]
        C2 --> D2[Header 壓縮]
    end
    
    subgraph "HTTP/3"
        A3[QUIC over UDP] --> B3[更快連接建立]
        B3 --> C3[改進的多路復用]
        C3 --> D3[連接遷移]
    end
```

#### HTTP/2 特性實例
```bash
# 啟用 HTTP/2 的 curl 請求
curl --http2 -v https://example.com

# 檢查服務器 HTTP/2 支持
curl -I --http2 https://example.com
```

### HTTPS 與 TLS/SSL

#### TLS 握手過程
```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    
    C->>S: ClientHello (支持的密碼套件)
    S->>C: ServerHello (選擇的密碼套件)
    S->>C: Certificate (服務器證書)
    S->>C: ServerHelloDone
    C->>S: ClientKeyExchange (預主密鑰)
    C->>S: ChangeCipherSpec
    C->>S: Finished
    S->>C: ChangeCipherSpec
    S->>C: Finished
    
    Note over C,S: 加密通信開始
```

#### 證書驗證與檢查
```bash
# 檢查證書資訊
openssl s_client -connect example.com:443 -servername example.com

# 查看證書詳細資訊
echo | openssl s_client -connect example.com:443 2>/dev/null | \
openssl x509 -noout -text

# 檢查證書過期時間
echo | openssl s_client -connect example.com:443 2>/dev/null | \
openssl x509 -noout -dates

# 驗證證書鏈
openssl verify -CAfile ca-bundle.crt server.crt
```

### HTTP 持久連接與連接池

#### 連接管理策略
```bash
# 檢查 HTTP 連接
netstat -an | grep :80
ss -tuln | grep :80

# 查看連接池狀態
curl -w "time_namelookup: %{time_namelookup}\n\
time_connect: %{time_connect}\n\
time_appconnect: %{time_appconnect}\n\
time_pretransfer: %{time_pretransfer}\n\
time_redirect: %{time_redirect}\n\
time_starttransfer: %{time_starttransfer}\n\
time_total: %{time_total}\n" \
-o /dev/null -s https://example.com
```

## DNS 域名解析系統

### DNS 查詢流程

```mermaid
graph TB
    A[應用程序] --> B[本地 DNS 緩存]
    B --> C{緩存命中?}
    C -->|是| D[返回結果]
    C -->|否| E[本地 DNS 服務器]
    E --> F[根域名服務器]
    F --> G[頂級域名服務器]
    G --> H[權威域名服務器]
    H --> I[返回 IP 地址]
    I --> J[緩存並返回]
```

### DNS 記錄類型

#### 常見記錄類型
```bash
# A 記錄 - IPv4 地址
dig example.com A

# AAAA 記錄 - IPv6 地址
dig example.com AAAA

# CNAME 記錄 - 別名
dig www.example.com CNAME

# MX 記錄 - 郵件交換
dig example.com MX

# TXT 記錄 - 文本記錄
dig example.com TXT

# NS 記錄 - 名稱服務器
dig example.com NS

# PTR 記錄 - 反向查詢
dig -x 8.8.8.8
```

### DNS 查詢與除錯

#### 使用 dig 進行詳細查詢
```bash
# 完整查詢過程
dig +trace example.com

# 查詢特定 DNS 服務器
dig @8.8.8.8 example.com

# 短格式輸出
dig +short example.com

# 反向查詢
dig +short -x 8.8.8.8

# 查詢所有記錄
dig example.com ANY
```

#### 使用 nslookup
```bash
# 基本查詢
nslookup example.com

# 指定記錄類型
nslookup -type=MX example.com

# 反向查詢
nslookup 8.8.8.8
```

#### 使用 host 命令
```bash
# 基本查詢
host example.com

# 詳細資訊
host -v example.com

# 所有記錄類型
host -a example.com
```

### DNS 緩存管理

#### 系統 DNS 緩存
```bash
# Ubuntu/Debian - 重啟 systemd-resolved
sudo systemctl restart systemd-resolved

# 清除 systemd-resolved 緩存
sudo systemd-resolve --flush-caches

# 查看 DNS 統計
sudo systemd-resolve --statistics

# CentOS/RHEL - 重啟 NetworkManager
sudo systemctl restart NetworkManager
```

#### 瀏覽器 DNS 緩存
```bash
# Chrome - 訪問 chrome://net-internals/#dns
# Firefox - 在地址欄輸入 about:networking#dns
```

### DNS 安全與 DoH/DoT

#### DNS over HTTPS (DoH)
```bash
# 使用 curl 進行 DoH 查詢
curl -H 'accept: application/dns-json' \
'https://cloudflare-dns.com/dns-query?name=example.com&type=A'

# 配置 DoH 服務器
# /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com
DNSOverTLS=yes
```

#### DNS over TLS (DoT)
```bash
# 使用 kdig 進行 DoT 查詢
kdig +tls example.com @1.1.1.1

# 測試 DoT 連接
openssl s_client -connect 1.1.1.1:853
```

## 應用層協議除錯與監控

### HTTP 流量分析

#### 使用 tcpdump 捕獲 HTTP
```bash
# 捕獲 HTTP 流量
sudo tcpdump -i any -A 'port 80'

# 捕獲 HTTPS 握手
sudo tcpdump -i any 'port 443'

# 保存到文件
sudo tcpdump -i any -w http.pcap 'port 80'
```

#### 使用 Wireshark 分析
```bash
# 命令行 Wireshark
tshark -i any -f 'port 80' -V

# 過濾 HTTP 請求
tshark -i any -Y 'http.request.method=="GET"'
```

### 應用層性能監控

#### HTTP 請求時間分析
```bash
# 詳細時間分析
curl -w "@curl-format.txt" -o /dev/null -s https://example.com

# curl-format.txt 內容:
#     time_namelookup:  %{time_namelookup}\n
#        time_connect:  %{time_connect}\n
#     time_appconnect:  %{time_appconnect}\n
#    time_pretransfer:  %{time_pretransfer}\n
#       time_redirect:  %{time_redirect}\n
#  time_starttransfer:  %{time_starttransfer}\n
#                     ----------\n
#          time_total:  %{time_total}\n
```

#### DNS 解析時間監控
```bash
# 測量 DNS 解析時間
time nslookup example.com

# 使用 dig 測量
dig example.com | grep "Query time"
```

## 後端開發中的應用層協議

### RESTful API 設計原則

#### 資源導向設計
```bash
# 良好的 API 設計
GET    /api/users          # 獲取用戶列表
GET    /api/users/123      # 獲取特定用戶
POST   /api/users          # 創建用戶
PUT    /api/users/123      # 更新用戶
DELETE /api/users/123      # 刪除用戶

# 嵌套資源
GET    /api/users/123/posts     # 獲取用戶的文章
POST   /api/users/123/posts     # 為用戶創建文章
```

#### HTTP 緩存策略
```bash
# 設置緩存頭
Cache-Control: max-age=3600, public
ETag: "686897696a7c876b7e"
Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT

# 條件請求
If-None-Match: "686897696a7c876b7e"
If-Modified-Since: Wed, 21 Oct 2015 07:28:00 GMT
```

### 常見問題與解決方案

#### HTTP 連接問題
```bash
# 檢查連接狀態
ss -tuln | grep :80
netstat -an | grep :80

# 檢查服務器響應
curl -I https://example.com

# 測試 HTTP/2 支持
curl -I --http2 https://example.com
```

#### DNS 解析問題
```bash
# 檢查 DNS 配置
cat /etc/resolv.conf

# 測試不同 DNS 服務器
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com

# 檢查系統 DNS 緩存
sudo systemd-resolve --status
```

## 實戰練習

### 1. HTTP 服務器性能測試
```bash
# 使用 Apache Bench
ab -n 1000 -c 10 https://example.com/

# 使用 wrk
wrk -t12 -c400 -d30s https://example.com/

# 使用 hey
hey -n 1000 -c 50 https://example.com/
```

### 2. DNS 查詢優化測試
```bash
# 比較不同 DNS 服務器響應時間
for dns in 8.8.8.8 1.1.1.1 208.67.222.222; do
    echo "Testing $dns"
    dig @$dns example.com | grep "Query time"
done
```

### 3. HTTPS 證書監控
```bash
#!/bin/bash
# 證書過期監控腳本
domain="example.com"
expiry_date=$(echo | openssl s_client -connect $domain:443 -servername $domain 2>/dev/null | \
              openssl x509 -noout -enddate | cut -d= -f2)
echo "Certificate for $domain expires on: $expiry_date"
```

## 重點總結

1. **HTTP 協議理解**：掌握請求/響應結構、狀態碼含義
2. **HTTPS 安全性**：理解 TLS 握手過程和證書驗證
3. **DNS 解析機制**：熟悉查詢流程和記錄類型
4. **性能監控**：會使用各種工具分析應用層性能
5. **除錯技能**：能夠診斷和解決常見的應用層問題

## 下一章預告

下一章將深入探討 **Socket 程式設計與 I/O 模型**，學習如何在 Linux 環境下進行高效的網路程式設計。