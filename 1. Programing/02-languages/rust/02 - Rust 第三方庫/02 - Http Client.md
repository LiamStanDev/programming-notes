# reqwest
---
## 介紹
`reqwest` 是 Rust 中一個流行且強大的 HTTP 客戶端庫。它提供了簡單易用的 API，用於發送 HTTP 請求和處理 HTTP 響應。`reqwest` 支持同步和異步操作，並且內置了對 HTTPS、JSON、表單數據等的支持。

##### 特點
1. **簡單易用**：提供了直觀的 API，易於發送各種 HTTP 請求。
2. **異步支持**：內置對 `async/await` 的支持，適合用於高性能的異步應用。
3. **豐富的功能**：支持 GET、POST、PUT、DELETE 等常見的 HTTP 方法，並且支持處理 JSON、表單數據、文件上傳等。
4. **HTTPS 支持**：內置對 HTTPS 的支持，確保數據傳輸的安全性。

##### 安裝
在 `Cargo.toml` 中添加 `reqwest` 依賴：
```toml
[dependencies]
reqwest = { version = "0.11", features = ["json"] }
tokio = { version = "1", features = ["full"] }
```

## 使用
### 發送 GET 請求
若沒有其他的需求，可以直接透過 `reqwest::get` 函數就能發送 GET 請求。
```rust
use reqwest::Error;

#[tokio::main]
async fn main() -> Result<(), Error> {
    let response = reqwest::get("https://api.github.com/repos/rust-lang/rust")
        .await?
        .text()
        .await?;

    println!("Response: {}", response);
    Ok(())
}
```

### 發送複雜請求 - RequestBuilder
若希望要在請求中添加 body, header 等內容，需要使用 `reqwest::Client` 結構體。有幾個重點：
1. Client 結構體內部為 connection poll，所以請 **reuse it**
2. 不要使用 `Rc` 與 `Arc` 來包裹它來 reuse，因為它內部已經使用 `Arc`
	* 可以透過傳遞引用的方式進行覆用
3. `Client::post`, `Client::get` 等都會返回 `RequestBuilder`。
4. `RequestBuilder` 中可以
	1. 添加 body
	2. 添加 header
	3. 設置驗證
	4. 設置 timeout
	5. 設置 query string

#### Query string
```rust
let client = reqwest::Client::new();
// GET http://example.com/search?qery=rust&page=1
let params =[("query", "rust"), ("page", "1")];
let response: Result<Response, Error> = client
	.get("http://example.com/search")
	.query(&params)
	.send()
	.await;
```

#### 添加 Header
##### 一般
```rust
let client = reqwest::Client::new();
let mut headers = HeaderMap::new();
headers.insert(USER_AGNET, "APP")

let response = client
	.get("http://example.com/search")
	.headers(headers)
	.send()
	.await;
```

##### Basic 驗證
```rust
let client = reqwest::Client::new();
let res = client
	.get("http://example.com")
	.basic_auth("username", Some("password"))
	.send()
	.await;

```
##### Bearer 驗證
```rust
let client = reqwest::Client::new();
let res = client
    .get("http://example.com")
    .bearer_auth("your_token")
    .send()
    .await;
```

#### 添加 Body

##### 一般 (不會設置 Content-Type)
```rust
let client = reqwest::Client::new();
let res = client.post("http://expample.com")
	.body("some data")
	.send()
	.await;
```

##### application/x-www-form-urlencoded Body
* 會設置 `Content-Type: application/x-www-form-urlencoded`
* 用途：為 HTML 表單的默認編碼方式，用於普通提交，並不能處理二進制數據（e.g. 文件上傳）。
* 數據格式：同 query_string，表單以 key-value 方式存儲，key-value`=` 表示一組 key-value，組間分隔使用 `&`
```rust
let client = reqwest::Client::new();
let params = [("key1", "value1"), ("key2", "value2")];
let res = client
    .post("http://example.com")
    .form(&params)
    .send()
    .await?;
```

##### multipart/form Body
* 需要添加 `multipart` feature
* 會設置 `Content-Type: multipart/form-data`
* 用於：包含二進制內容的標單
* 數據格式：據被分割為多個部分（multipart），每個部分都有一個 Content-Disposition 標頭，用於描述該部分是屬於哪個表單字段。每個部分都可以包含其自己的MIME類型。
```rust
use reqwest::multipart;

let client = reqwest::Client::new();

let form = multipart::Form::new()
    .text("key1", "value1")
    .file("file", "/path/to/file")?;

let res = client
    .post("http://example.com/upload")
    .multipart(form)
    .send()
    .await;
```


> MIME 為 Multipurpose Internet Mail Extensions 網路標準，表示內容的類型
> * `text/plain`
> * `text/html`
> * `image/jpeg`
> * `application/json`
> * `application/pdf`


##### json
* 需要添加 `json` feature
* 會設置 `Content-Type: application/json`
```rust
use serde_json::json;

let client = reqwest::Client::new();
let data = json!({
    "key1": "value1",
    "key2": "value2"
});

let res = client
    .post("http://example.com")
    .json(&data)
    .send()
    .await;
```

### 處理響應 - Response
#### Http 相關
```rust
// 獲取 HTTP 狀態碼
let status = res.status();
println!("Status: {}", status);

// 獲取遠程地址（Remote addr）
if let Some(addr) = res.remote_addr() {
	println!("Remote Address: {}", addr);
}
```

#### Header 相關
```rust
// 獲取所有 headers
let headers = res.headers();
for (key, value) in headers.iter() {
	println!("Header: {} = {:?}", key, value);
}

// 獲取 content_length
if let Some(content_length) = res.content_length() {
	println!("Content-Length: {}", content_length);
}

// 獲取 cookies
if let Some(cookies) = res.cookies().next() {
	println!("Cookie: {:?}", cookies);
}
```

#### Body 

##### text
* 以文本的形式取得，以 Content-Type 中的 `charset` 指定的編碼，若沒有指定默認為 `utf-8` 編碼

```rust
// 以文本形式獲取 Body
let body_text = res.text().await;
println!("Body Text: {}", body_text);
```
##### json
* 需要添加 `json` feature
* 是透過 `serde_json::from_reader` 實現
```rust
#[derive(Deserialize)]
struct ApiResponse {
    key1: String,
    key2: String,
}


// 將 Body 解析為 JSON
let json_body: ApiResponse = res.json().await?;
println!("JSON Response: key1 = {}, key2 = {}", json_body.key1, json_body.key2);
```

##### bytes

###### 方式一：
使用 `chunk()` 方法。
```rust
// 透過 Stream 來讀 Body 為 bytes
let mut file = tokio::fs::File::create("output.bin").await?;
while let Some(chunk) = res.chunk().await? {
	file.write_all(&chunk).await?;
}
```
###### 方式二：
* 需要添加 `stream` feature
* 透過 Stream 來讀 body
```rust
use reqwest::Client;
use futures_util::StreamExt; // for stream extension methods
use tokio::io::AsyncWriteExt; // for async file operations

// 使用 bytes_stream 方法來讀取 Body
let mut stream = res.bytes_stream();

// 開啟一個文件來寫入下載的數據
let mut file = tokio::fs::File::create("output.bin").await?;

// 逐個 chunk 讀取數據並寫入文件
while let Some(chunk) = stream.next().await {
	let chunk = chunk?;
	file.write_all(&chunk).await?;
}
```


### 配置用戶 - ClientBuilder
可以用於統一配置 Http 請求/響應，如：
1. user agent
2. 默認 headers
3. cookies
4. 壓縮方式
5. proxy
6. timeout

```rust
let client = Client::builder()
	.user_agent("Custom User Agent")
	.default_headers(headers)
	.gzip(true) // 自動 reponse 解壓縮，若 Context-Encoding 有 gzip
	.cookie_store(true) // 啟用 cookie store，需要添加 cookies feature
	.proxy(Proxy::https("https://my-proxy.com"))
	.timeout(Duration::from_secs(30)) // 設定 30 秒超時請求
	.build();
```