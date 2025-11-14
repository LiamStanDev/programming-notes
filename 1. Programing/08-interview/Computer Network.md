## 基礎
---
##### 2. OSI 七層模型
* 物理層: 
	* 使用光、電或電磁波等方式將數據進行傳播，提供數據傳播的功能。
	* 數據: bit
	* 設備: 中繼器、集線器
* 數據鏈路層: 
	* 提供在同一網路中找到目標的功能。 
	* 數據: Frame (楨)，在封裝成楨時會添加 MAC 地址 (每個網卡獨有的)
	* 功能: 封裝成楨、差錯檢測與控制、流量控制
	* 設備: 交換機 (依照 MAC 地址找到對應的網卡)
* 網路層: 
	* 提供找到指定網路的功能
	* 數據: Datagram (包)
	* 功能: 尋址與路由選擇
	* 設備: 路由器
	* 協議: IP，ICMP
* 傳輸層:
	* 提供找到對應進程的功能 (端口)
	* 數據: Segment (段)
	* 功能: 流量控制、錯誤控制 (確定數據完整無誤)
	* 協議: TCP, UDP, QUIC
* 會話層:
	* 提供同步功能、管理狀態 (e.g. 登入狀態) 與恢復。
*  表示層:
	* 提供不同計算機之間表達方式的轉換，提供編碼與解碼功能 (加密解密) 與數據壓縮
* 應用層:
	* 協議: HTTP、FTP、SMTP
	* 與表示層和會話層的數據合併叫 Message (報文)
## 綜合
---
##### 3. browser 輸入 url 到顯示網頁過程?
1. DNS 解析，將域名轉換成對應的 IP 地址。
2. TCP 連接，Client 與 Server 進行 TCP 三次握手建立連接
3. 向服務器發送 HTTP 請求
4. 服務器返回 HTTP 響應
5. 瀏覽器渲染頁面
6. 斷開連接，執行 TCP 四次揮手


##### 4. 請說明 DNS 解析過程
以 www.google.com 爲例子
1. 查詢 Browser cache 是否有記錄
2. 若無記錄查詢本地 DNS 是否有記錄
3. 本地 DNS 向根域名服務器發送請求，得到頂級域名服務器 com 的 ip 地址
4. 本地 DNS 向 com 域名服務器發送請求，得到 google.com 的域名服務器地址
5. 本地 DNS 向 google.com 域名服務器發送請求，得到 www.google.com 的 ip 地址。

##### 5. WebSock 與 Socket 差別
Socket 是由 OS 提供的一套標準且簡單的 API，對 TCP/IP 的高度封裝，讓使用者可以不考慮底層細節，透過 Socket API 進行網路應用開發。
WebSocket 爲一個協議，建立在 TCP 連接之上提供雙向通訊的協議，他允許建立一個持久化的連接，使得服務端可以向客戶端進行數據推送，而不是採用輪詢的方式定時向服務器發送請求。

##### 6. 請說出常見的端口號？
* 21 FTP
* 22 SSH
* 53 DNS 解析
* 80 HTTP
* 443 HTTPS
* 3306 Mysql
* 5432 Postgresql
* 27017 mongodb
* 5672 Rabbitmq

## HTTP
---
##### 1. Http 狀態碼
* 1xx 信息狀態碼
	* 101 切換請求協議
* 2xx Success
	* 200 OK
	* 201 Created
	* 204 No Content
* 3xx Redirection
	* 301 Move Permanently
	* 302 Found 臨時重定向
	* 304 Not Modified 還尚未過期直接使用緩存
* 4xx Client Error
	* 400 Bad Request
	* 401 Unauthorized
	* 403 Forbidden
	* 404 Not Found
* 5xx Server Error
	* 500 Internal Server Error
	* 502 Bad Gateway
##### 2. HTTP 請求方式
* GET: 獲取資源請求
* POST: 提交數據請求
* PUT: 修改數據請求
* DELETE: 刪除數據請求
* OPTIONS: 常用來查詢是否支持能力，常見於瀏覽器的 CORS 政策，向服務端確認 `Access-Control-Allow-Headers`

##### 3. GET vs POST 有哪些區別
* 從請求來看， GET 將請求信息放在 request url 中，而 POST 放在 request body 中，所以 GET 能攜帶的數據大小有限制，POST則無。
* GET 能夠被緩存，且能夠添加進瀏覽器的書籤中。

##### 4. HTTP 請求與響應結構
###### Reqeust
* Request Line: 請求方法 url HTTP版本
* Request Header
* Request Body
###### Response
* Response Line: HTTP版本 狀態碼 狀態信息
* Response Header
* Response Body

###### 常見的頭部
* `Accept: text/html,application/xhtml+xml` 客戶端可接收的類型
* `Content-Type: application/json` body 內容
* `User-Agent: Mozilla/5.0` 代理用戶信息
* `Authorization: Bearer NlcjpwYXNzd29yZA==` 驗證信息
* `Cache-Control: no-cache, max-age=0` 緩存策略
* `Location: https://127.9.2.1/new-location` 用於 301 重定向或者 201 Created
* `Content-Length: 1024` body 的長度以byte爲單位
* `Set-Cookie: user_id=12345; Expires=Wed, 21 Oct 2022 07:28:00 GMT; Path=/` 服務端送 Set Cookie
* `Cookie: sessionID=abc123; userToken=xyz456` 爲客戶端請求時 Cookie


##### 5. HTTP 個版本區別
1.0 默認短連接，1.1 默認長連接，2.0 採用多路復用
HTTP 1.0: 每一次請求都要建立 TCP 連接，所以較沒效率。
HTTP 1.1: 可以使用 `Connection: keep-alive` 來保持連接，且 Body 經過壓縮，但有 HTTP 對頭阻塞的問題
HTTP 2.0: 採用多路復用，不用等待前一次響應，可以發起多個請求，Header 壓縮，支持服務器推送，採用hear 與 body 都採用二進制解析更快速。

##### 6. 甚麼是 HTTP 3.0
相較於前面的版本 HTTP 3.0 在傳輸層採用 UDP，使用 QUIC 來保證 UDP 的可靠性，因爲 HTTP 2.0 存在 TCP 對頭阻塞的問題，與 TCP 慢啓動問題等問題，是由 TCP 本身導致的，所以在 HTTP 3.0 採用 QUIC 作爲基礎發展。有以下特點
1. 使用 UDP
2. HTTPS 中的 TCP 三次握手與 TLS 多次握手，而 QUIC 減少到 3 次握手就能完成安全傳輸。

##### 7. HTTP vs HTTPS
* HTTP 是採用明文傳輸而 HTTPS 採用密文傳輸，保證數據安全
* HTTPS 要多進行 SSL/TLS 握手，協議密鑰
* HTTPS 需要向 CA 聲請證書，保證服務端的身份可性
* HTTP 端口爲 80, HTTPS 端口爲 443

##### 8. HTTPS 工作流
1. Client Hello: 發送支持的加密套件與第一隨機數
2. Server Hello: 發送選擇的加密套件與第二隨機數
3. Certificate: 發送服務器證書
4. Server key exchange: server 發送公鑰
5. Server Hello Down
6. Client Key exchange, change ciper spec: 客戶端生成第三隨機數，並使用公鑰加密，發送給 server
7. Encrypted handshake message: 雙方都取得三個隨機數，生成非對稱密鑰，之後都採用該密鑰加密訊息。

##### 9. Session 與 Cookie
兩者都是爲了解決 HTTP 無狀態，下保持狀態的方式。
* Cookie 是保存在客戶端中的文本數據，像是一組 key value pair，由服務端發送 Set-Cookie Header 要求客戶端保存，並之後每次請求客戶端都會攜帶該 Cookie，用來向服務端顯示目前狀態。
* Session 與 Cookie 不同，訊息是保存在服務端，而客戶端只保存 sessionid，然後透過 Cookie 發送 sessionid 由服務端查詢 sessionid 對應的狀態。

##### 10. JWT？
JWT 爲 json web token，JWT 是一種輕量緊湊無法修改又自包含的一串數據，可以解析 JWT 獲得資訊，用來保持用戶登入保存用戶信息，他的好處在於 Cookie 採用明文而Session又會佔用服務器空間，JWT 是存放在客戶端中，且簽名過後的令牌，用來表示用戶身份，相較於 Cookie 更安全，現在被廣泛使用在 OAuth 2.0 與 OpenID Connection 中。


