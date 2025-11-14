# OAuth 2.0
---
## 介紹
OAuth 2.0 是一個開放的標準**授權協議** [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)，允許用戶提供一個安全的訪問令牌給第三方應用，可以避免用戶名與密碼暴露給第三方應用，第三方應用就可以代表用戶去訪問服務提供商的的信息。
![[Pasted image 20240102200821.png]]
* 可以實現 SSO 單點登入。
* 第三方應用可以利用令牌獲取用戶信息。

## 術語
1. Resource Owner: 指的是用戶本人
2. Resource Server: 指的是託管用戶數據的服務器，只有提供有效的訪問令牌後才會提供數據, e.g. Github, Facebook, Google, etc.
3. Client: 想要訪問 Resource Server 數據的第三方應用
4. Authorization Server: 負責驗證用戶身份並提供令牌給第三方應用，可與 Resource Server 相同。
## 授權方式
### 方式一： Authorization code (最常使用, 最安全)
#### 1. Authorization Request：
* 使用者訪問第三方應用，然後點擊要求訪問的資源
* 第三方應用程式將用戶重定向到授權伺服器（Authorization Server），並附帶以下參數

| 字段 | 描述 |
| ---- | ---- |
| response_type | 指定為 `code`，表示使用授權碼模式 |
| client_id | 授權服務器註冊第三方應用而得到的唯一識別 |
| redirect_uri | 用於接收授權碼的重定向URI |
| scope | 可選，指定需要訪問的資源範圍 |
| state | 可選（推薦），原樣返回給客戶端 |
> state: 在整個授權流程中保持不變的狀態值，用來防止CSRF攻擊。

```
GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz
        &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
Host: server.example.com
```

#### 2. Authorization Response：
用戶到達授權伺服器後，首先會被要求登錄（除非已登錄），然後確認是否同意第三方應用所請求的權限。如果用戶同意授權，授權伺服器將用戶重定向回第三方應用，重定向的URL中包含一個授權碼（Authorization Code）。參數如下

| 字段 | 描述 |
| ---- | ---- |
| code | Authorization code |
| state | 若用戶請求授權有提供，則會返回相同 state |
```
HTTP/1.1 302 Found
Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA
		   &state=xyz
```

#### 3. Access Token Request：
第三方應用收到授權碼後，會在**後臺向授權伺服器發請求**，用授權碼換取訪問令牌送請求。這個請求將包括*授權碼、客戶端ID、客戶端密鑰和重定向URI*。

| 字段 | 描述 |
| ---- | ---- |
| grant_type | 固定爲 `authorization_code` |
| code | Authorization code |
| redirect_uri | 必須與授權請求中使用的重定向URI相同 |
| client_id | 同上 |
```
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
```

> Authorization Heaer 爲由 base64 編碼後的 client_id 與 secret (向授權服務器申請時的)，用來驗證是哪個客戶端。
#### 4. Access Token Response
授權伺服器驗證請求有效後，會向第三方應用頒發訪問令牌（Access Token）和刷新令牌（Refresh Token）。訪問令牌是有時效性的，而當訪問令牌過期時，可以使用刷新令牌來獲取新的訪問令牌。返回值如下

| 字段 | 描述 |
| ---- | ---- |
| access_token | 訪問令牌 |
| token_type | 表示訪問令牌的類型，通常是 `Bearer` |
| expires_in | 訪問令牌的有效期（以秒為單位） |
| refresh_token | 可選，用於獲取新訪問令牌的刷新令牌。 |

```
 HTTP/1.1 200 OK
 Content-Type: application/json;charset=UTF-8
 Cache-Control: no-store
 Pragma: no-cache

 {
   "access_token":"2YotnFZFEjr1zCsicMWpAA",
   "token_type":"example",
   "expires_in":3600,
   "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
   "example_parameter":"example_value"
 }
```
#### 5. 應用訪問資源：
第三方應用現在可以使用訪問令牌向資源伺服器請求用戶數據了。

> URI vs. URL
> Uniform Resource Identifier:
      用於標示某一個網路資源名稱的字符串，用來**唯一標示該資源**
> Uniform Resource Locator：
> 	爲 URI 的子集，**提供找到該資源的方法**，e.g. http/https

### 方式二：Implicit
相較於 Authentication Code 簡化了先取得 Authorization Code 後才獲得 Access Token 的步驟。

#### 1. Authorization Request
* 用戶打開第三方應用程式，然後點擊要求訪問的資源
* Implicit授權模式不返回刷新令牌（Refresh Token），因為這樣做可能會導致安全風險。
* 第三方應用程式將用戶重定向到授權伺服器，附帶以下參數：

| 字段 | 描述 |
| ---- | ---- |
| response_type | 指定為 `token`，表示使用授權碼模式 |
| client_id | 授權服務器註冊第三方應用而得到的唯一識別 |
| redirect_uri | 用於接收授權碼的重定向URI |
| scope | 可選，指定需要訪問的資源範圍 |
| state | 可選（推薦），原樣返回給客戶端 |
> state: 在整個授權流程中保持不變的狀態值，用來防止CSRF攻擊。

```
GET /authorize?response_type=token&client_id=s6BhdRkqt3&state=xyz
        &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
Host: server.example.com
```

#### 2. Access Token Response
* 戶在授權伺服器上進行身份驗證並授權第三方應用程式訪問其資源。

| 字段 | 描述 |
| ---- | ---- |
| access_token | 訪問令牌 |
| token_type | 表示訪問令牌的類型，通常是 `Bearer` |
| expires_in | 訪問令牌的有效期（以秒為單位） |
| scope | 可選，指定需要訪問的資源範圍 |
| state | 可選（推薦），原樣返回給客戶端 |
```
 HTTP/1.1 302 Found
 Location: http://example.com/cb#access_token=2YotnFZFEjr1zCsicMWpAA
					 &state=xyz&token_type=example&expires_in=3600
```

### 方式三： Resource Owner Password Credentials (不安全)
* 不同於其他需要多個步驟和重定向的授權方式
* 允許客戶端應用程序直接使用用戶的用戶名和密碼來交換訪問令牌
#### 1. Authorization Request and Response
用戶直接向客戶端應用程序提供他們的用戶名和密碼。

#### 2. Access Token Request
客戶端應用程序向授權伺服器的令牌端點發送 POST 請求，其中包括以下參數：

| 字段 | 描述 |
| ---- | ---- |
| grant_type | 設置為 `password` |
| client_id | 客戶端應用程序的唯一識別碼 |
| client_secret | 用戶密鑰 |
| username | 用戶的用戶名或識別符 |
| password | 用戶的密碼 |
| scope | 指定所需的訪問範圍 |

```
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=password&username=johndoe&password=A3ddj3w
```

> Authorization Heaer 爲由 base64 編碼後的 client_id 與 secret (向授權服務器申請時的)，用來驗證是哪個客戶端。
#### 3. Access Token Response
如果提供的憑證有效且經授權，授權伺服器將回應包括訪問令牌和刷新令牌。

| 字段 | 描述 |
| ---- | ---- |
| access_token | 設置為 `password` |
| token_type | 客戶端應用程序的唯一識別碼 |
| expires_in | 用戶密鑰 |
| refresh_token | 用戶的用戶名或識別符 |
```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
 "access_token":"2YotnFZFEjr1zCsicMWpAA",
 "token_type":"example",
 "expires_in":3600,
 "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
 "example_parameter":"example_value"
}
```

### 方式四：Client Credential
#### 1. Authorization Request and Response
客戶端對於用戶進行驗證。

#### 2. Access Token Request
客戶端應用程序使用其自身的憑證（通常是客戶端ID和客戶端密碼）向授權伺服器的令牌端點發送請求，請求訪問特定資源。參數如下：

| 字段 | 描述 |
| ---- | ---- |
| grant_type | 指定爲 `client_credentials` |
| scope | 指定所需的訪問範圍 |
```
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
```

> Authorization Heaer 爲由 base64 編碼後的 client_id 與 secret (向授權服務器申請時的)，用來驗證是哪個客戶端。
#### 3. Access Token Response
```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
 "access_token":"2YotnFZFEjr1zCsicMWpAA",
 "token_type":"example",
 "expires_in":3600,
 "example_parameter":"example_value"
}
```


# OpenID Connection (OIDC)
---
### 介紹
解決 Auth 2.0 本身並不直接提供身份驗證的問題。OpenID Connect（OIDC）是建立在 OAuth 2.0 協議之上的一個標準，用於身份驗證和授權。

### 特點
##### ID Token
爲 JWT 格式的安全令牌，裏面包含用戶的身份信息

| 字段 | 描述 |
| ---- | ---- |
| sub | 用戶的唯一標示符號 |
| iss | 令牌發行者 |
| exp | 令牌過期時間 |
```json
 {
   "iss": "https://server.example.com", // issuer
   "sub": "24400320", // subject 爲這個 Token 的唯一識別序號
   "aud": "s6BhdRkqt3", // audience 表示這個 Token 是要給誰
   "nonce": "n-0S6_WzA2Mj", // 隨機值
   "exp": 1311281970, // expire 時間戳
   "iat": 1311280970, // issue at 發行時間戳
   "auth_time": 1311280969, // 用戶驗證時間戳
   "acr": "urn:mace:incommon:iap:silver"
  }
```

### 流程
##### 客戶端註冊：
客戶端（應用程序）在認證提供者處註冊，並獲取客戶端識別符（Client ID）和客戶端密鑰（Client Secret）。
##### 發起認證請求：
客戶端將用戶重定向到 OIDC 認證提供者的授權端點，同時提供必要的參數，如 response_type、client_id、redirect_uri 等。
例如：`https://auth-server.com/auth?response_type=code&client_id=your-client-id&redirect_uri=your-redirect-uri&scope=openid profile&state=xyz`
##### 用戶登錄：
用戶被 OIDC 認證提供者引導到登錄頁面，輸入其憑據（用戶名和密碼）進行身份驗證。
##### 用戶授權：
用戶成功登錄後，OIDC 認證提供者將顯示一個授權頁面，詢問用戶是否允許客戶端訪問其信息。
用戶可以選擇授予或拒絕授權。
##### 頒發授權碼：
如果用戶授予了授權，OIDC 認證提供者將重定向用戶回到客戶端提供的 redirect_uri 並附帶授權碼（Authorization Code）。
例如：`https://your-redirect-uri.com/callback?code=authorization-code&state=xyz`
##### 交換授權碼：
客戶端使用收到的授權碼向 OIDC 認證提供者的令牌端點發起請求，以交換授權碼獲取訪問令牌（Access Token）和 ID 令牌（ID Token）。
##### 獲取令牌：
OIDC 認證提供者驗證授權碼，並返回包含 Access Token 和 ID Token 的響應。
##### 驗證 ID Token：
客戶端驗證 ID Token 的簽名，並確保其中的信息（例如簽發者、受眾等）是可接受的。
如果驗證通過，客戶端可以使用 ID Token 中的用戶信息完成用戶身份驗證。