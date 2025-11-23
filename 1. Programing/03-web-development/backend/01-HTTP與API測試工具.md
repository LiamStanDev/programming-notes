# HTTP 與 API 測試工具

## 目錄
- [curl](#curl)
- [grpcurl](#grpcurl)
- [websocat](#websocat)

---

## curl

### 命令說明

curl (Client URL) 是最常用的命令行 HTTP 客戶端,支援多種協議 (HTTP, HTTPS, FTP 等),用於發送請求、測試 API、下載檔案。

**基本語法:**
```bash
curl [options] <url>
```

### 常用選項

| 選項 | 說明 |
|------|------|
| `-X, --request` | 指定 HTTP 方法 (GET, POST, PUT, DELETE) |
| `-H, --header` | 添加請求標頭 |
| `-d, --data` | POST 請求資料 |
| `-F, --form` | 表單上傳 (multipart/form-data) |
| `-o, --output` | 輸出到檔案 |
| `-O` | 使用遠端檔名儲存 |
| `-L, --location` | 追蹤重定向 |
| `-v, --verbose` | 顯示詳細資訊 |
| `-s, --silent` | 靜默模式 |
| `-w, --write-out` | 自訂輸出格式 |
| `-k, --insecure` | 忽略 SSL 憑證驗證 |
| `-u, --user` | HTTP 基本認證 |
| `-b, --cookie` | 發送 Cookie |
| `-c, --cookie-jar` | 儲存 Cookie |
| `-A, --user-agent` | 設定 User-Agent |
| `-I, --head` | 只取得標頭 |
| `--connect-timeout` | 連接超時時間 |
| `-m, --max-time` | 最大執行時間 |

### 使用場景

1. **API 開發測試**: 測試 RESTful API 的各種請求
2. **介面除錯**: 檢查請求/回應內容
3. **效能分析**: 測量回應時間
4. **自動化腳本**: 批次請求處理
5. **下載檔案**: 命令行下載

---

### 操作範例

#### 基本 GET 請求

```bash
$ curl https://api.example.com/users

[
  {"id": 1, "name": "Alice"},
  {"id": 2, "name": "Bob"}
]
```

#### 帶標頭的 GET 請求

```bash
$ curl -H "Authorization: Bearer TOKEN123" \
       -H "Accept: application/json" \
       https://api.example.com/users/1

{"id": 1, "name": "Alice", "email": "alice@example.com"}
```

#### JSON POST 請求

```bash
$ curl -X POST https://api.example.com/users \
       -H "Content-Type: application/json" \
       -d '{"name": "Charlie", "email": "charlie@example.com"}'

{"id": 3, "name": "Charlie", "email": "charlie@example.com", "created_at": "2024-01-15T10:30:00Z"}
```

**輸出說明:**
- 回應為新建立的資源,包含伺服器產生的 `id` 和 `created_at`

#### 從檔案讀取請求資料

```bash
$ cat request.json
{"name": "David", "email": "david@example.com"}

$ curl -X POST https://api.example.com/users \
       -H "Content-Type: application/json" \
       -d @request.json
```

#### PUT 更新請求

```bash
$ curl -X PUT https://api.example.com/users/1 \
       -H "Content-Type: application/json" \
       -d '{"name": "Alice Updated"}'
```

#### DELETE 請求

```bash
$ curl -X DELETE https://api.example.com/users/1 \
       -H "Authorization: Bearer TOKEN123"
```

#### 表單上傳檔案

```bash
$ curl -X POST https://api.example.com/upload \
       -F "file=@/path/to/document.pdf" \
       -F "description=My Document"

{"file_id": "abc123", "size": 102400, "url": "https://cdn.example.com/abc123.pdf"}
```

#### 查看詳細請求/回應

```bash
$ curl -v https://api.example.com/health

*   Trying 93.184.216.34:443...
* Connected to api.example.com (93.184.216.34) port 443 (#0)
* SSL connection using TLSv1.3
> GET /health HTTP/2
> Host: api.example.com
> User-Agent: curl/7.81.0
> Accept: */*
>
< HTTP/2 200
< content-type: application/json
< date: Mon, 15 Jan 2024 10:30:00 GMT
<
{"status": "healthy", "version": "1.2.3"}
```

**輸出說明:**
- `>` 開頭: 發送的請求
- `<` 開頭: 收到的回應
- `*` 開頭: 連接資訊

#### 只查看回應標頭

```bash
$ curl -I https://api.example.com/users

HTTP/2 200
content-type: application/json
content-length: 1234
cache-control: max-age=3600
x-request-id: req-abc123
```

#### 追蹤重定向

```bash
$ curl -L -v http://example.com/old-path

* Following redirect to https://example.com/new-path
< HTTP/1.1 301 Moved Permanently
< Location: https://example.com/new-path
```

---

### 效能測量

#### 時間分析

建立格式檔案 `curl-format.txt`:
```
     time_namelookup:  %{time_namelookup}s\n
        time_connect:  %{time_connect}s\n
     time_appconnect:  %{time_appconnect}s\n
    time_pretransfer:  %{time_pretransfer}s\n
       time_redirect:  %{time_redirect}s\n
  time_starttransfer:  %{time_starttransfer}s\n
                     ----------\n
          time_total:  %{time_total}s\n
```

執行測量:
```bash
$ curl -w "@curl-format.txt" -o /dev/null -s https://api.example.com/users

     time_namelookup:  0.012s
        time_connect:  0.045s
     time_appconnect:  0.123s
    time_pretransfer:  0.124s
       time_redirect:  0.000s
  time_starttransfer:  0.234s
                     ----------
          time_total:  0.345s
```

**輸出說明:**
- `time_namelookup`: DNS 解析時間
- `time_connect`: TCP 連接時間
- `time_appconnect`: SSL/TLS 握手時間
- `time_starttransfer`: 首位元組時間 (TTFB)
- `time_total`: 總時間

#### 簡化版時間測量

```bash
$ curl -w "DNS:%{time_namelookup}s Connect:%{time_connect}s TTFB:%{time_starttransfer}s Total:%{time_total}s\n" \
       -o /dev/null -s https://api.example.com/users

DNS:0.012s Connect:0.045s TTFB:0.234s Total:0.345s
```

---

### 認證方式

#### HTTP Basic Auth

```bash
$ curl -u username:password https://api.example.com/admin
```

#### Bearer Token

```bash
$ curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
       https://api.example.com/users
```

#### API Key

```bash
$ curl -H "X-API-Key: your-api-key" \
       https://api.example.com/data
```

---

### Cookie 處理

#### 發送 Cookie

```bash
$ curl -b "session_id=abc123; user_id=456" https://example.com/dashboard
```

#### 儲存並使用 Cookie

```bash
# 登入並儲存 Cookie
$ curl -c cookies.txt -X POST https://example.com/login \
       -d "username=user&password=pass"

# 使用儲存的 Cookie
$ curl -b cookies.txt https://example.com/dashboard
```

---

### 實戰技巧

#### 模擬瀏覽器請求

```bash
$ curl -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
       -H "Accept-Language: zh-TW,zh;q=0.9,en;q=0.8" \
       -H "Accept: text/html,application/xhtml+xml" \
       https://example.com
```

#### 忽略 SSL 憑證 (測試環境)

```bash
$ curl -k https://localhost:8443/api
```

#### 設定超時

```bash
# 連接超時 5 秒,總時間 30 秒
$ curl --connect-timeout 5 -m 30 https://api.example.com/slow-endpoint
```

#### 重試機制

```bash
$ curl --retry 3 --retry-delay 2 https://api.example.com/unstable
```

#### 壓縮請求

```bash
$ curl --compressed https://api.example.com/large-response
```

#### 輸出 HTTP 狀態碼

```bash
$ curl -s -o /dev/null -w "%{http_code}" https://api.example.com/health
200
```

#### 條件判斷腳本

```bash
#!/bin/bash
status=$(curl -s -o /dev/null -w "%{http_code}" https://api.example.com/health)
if [ "$status" -eq 200 ]; then
    echo "Service is healthy"
else
    echo "Service is down: $status"
    exit 1
fi
```

---

## grpcurl

### 命令說明

grpcurl 是 gRPC 服務的命令行測試工具,類似 curl 但用於 gRPC 協議。支援服務反射 (Server Reflection) 和 proto 檔案。

**基本語法:**
```bash
grpcurl [options] <host:port> <method>
```

### 常用選項

| 選項 | 說明 |
|------|------|
| `-plaintext` | 使用明文連接 (非 TLS) |
| `-d` | 請求資料 (JSON 格式) |
| `-H` | 添加 metadata |
| `-proto` | 指定 proto 檔案 |
| `-import-path` | proto 導入路徑 |
| `-v` | 詳細輸出 |

### 使用場景

1. **gRPC API 測試**: 測試 gRPC 服務端點
2. **服務探索**: 列出可用服務和方法
3. **除錯**: 檢查 gRPC 請求/回應

---

### 操作範例

#### 列出服務 (需啟用反射)

```bash
$ grpcurl -plaintext localhost:50051 list

grpc.health.v1.Health
grpc.reflection.v1alpha.ServerReflection
myapp.UserService
```

#### 列出服務方法

```bash
$ grpcurl -plaintext localhost:50051 list myapp.UserService

myapp.UserService.GetUser
myapp.UserService.CreateUser
myapp.UserService.ListUsers
```

#### 查看方法簽名

```bash
$ grpcurl -plaintext localhost:50051 describe myapp.UserService.GetUser

myapp.UserService.GetUser is a method:
rpc GetUser ( .myapp.GetUserRequest ) returns ( .myapp.User );
```

#### 呼叫 Unary 方法

```bash
$ grpcurl -plaintext -d '{"user_id": "123"}' \
          localhost:50051 myapp.UserService/GetUser

{
  "id": "123",
  "name": "Alice",
  "email": "alice@example.com"
}
```

#### 帶 Metadata 的請求

```bash
$ grpcurl -plaintext \
          -H "authorization: Bearer TOKEN123" \
          -d '{"name": "Bob"}' \
          localhost:50051 myapp.UserService/CreateUser
```

#### 使用 Proto 檔案

```bash
$ grpcurl -proto user.proto \
          -import-path ./protos \
          -d '{"user_id": "123"}' \
          localhost:50051 myapp.UserService/GetUser
```

#### 串流請求

```bash
# Server streaming
$ grpcurl -plaintext -d '{"query": "test"}' \
          localhost:50051 myapp.SearchService/Search

# 持續輸出結果...
```

---

## websocat

### 命令說明

websocat 是 WebSocket 的命令行客戶端,用於測試 WebSocket 服務。

**基本語法:**
```bash
websocat [options] <url>
```

### 常用選項

| 選項 | 說明 |
|------|------|
| `-n` | 不換行 |
| `-1` | 一次性模式 |
| `-t` | 文字模式 |
| `-b` | 二進位模式 |
| `-H` | 添加標頭 |

### 使用場景

1. **WebSocket 測試**: 測試即時通訊服務
2. **除錯**: 檢查 WebSocket 訊息
3. **互動式測試**: 手動發送/接收訊息

---

### 操作範例

#### 連接 WebSocket

```bash
$ websocat ws://localhost:8080/ws

# 進入互動模式,輸入訊息後按 Enter 發送
Hello Server
{"type": "message", "content": "Hello Server"}

# 收到伺服器回應
{"type": "response", "content": "Hello Client"}
```

#### 帶認證的連接

```bash
$ websocat -H "Authorization: Bearer TOKEN123" \
           wss://api.example.com/ws
```

#### 發送單一訊息

```bash
$ echo '{"action": "subscribe", "channel": "updates"}' | \
  websocat ws://localhost:8080/ws
```

#### SSL 連接

```bash
$ websocat wss://secure.example.com/ws
```

#### 結合 jq 處理 JSON

```bash
$ websocat ws://localhost:8080/ws | jq '.data'
```

---

## 參考資料 (References)

1. [curl 官方文件](https://curl.se/docs/)
2. [grpcurl GitHub](https://github.com/fullstorydev/grpcurl)
3. [websocat GitHub](https://github.com/vi/websocat)
4. [Everything curl](https://everything.curl.dev/)
