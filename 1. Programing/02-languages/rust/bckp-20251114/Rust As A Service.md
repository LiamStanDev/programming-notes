# 1. Minimal HTTP Server
---
### 準備
* 安裝必要庫
```shell
cargo add tokio -F full # -F 表示 feature，Full 不是必須的但為了開發方便就全部引入
cargo add axum
```
> Axum 為 tokio 團隊開發的非常高效率的 web server stack，效率與 actix 不相上下，使用 tokio 作為 async runtime。

### 撰寫第一個 Web Server
```rust
use axum::{response::Html, routing::get, Router}
#[tokio::main]
async fn main() {
	let app = Router::new()
		.route("/", get(handler));
		
	let addr = "127.0.0.1:3000";
	let listener = tokio::net::TcpListener::bind(addr).await.unwrap(); // 使用 tokio 提供的異步 TcpListener

	prinln!("Listening on {}", addr);

	axum::server(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
	Html("<h1>Hello, World!</h1>")
}
```

# 2. Understanding the Serivce Stack
---
##### Layer 0: tokio
為 Axum 最底層的部份，提供異步的運行時，本質為無棧協程，用來 Schedule 任務，並不會進行等待而是移轉控制權給可以繼續運行的程式碼，可以配置為單線程或者多線程。

##### Layer 1: Hyper
提供 Http 服務，功能包括格式化與解析不同的 Header、Cookie等所有 Http 協議內容，提供強類型並且效能很高，被廣泛用於 Rust 的 Web service stack，如 Axum、Actix、Rocket 等。
> 可以直接使用 Hyper 來搭建 Http 服務，但是會十分痛苦。

##### Layer 2: Tower
為 Middleware 服務，可用於添加 timeout、併發限制、身份驗證、處理依賴注入等眾多功能。

### Lifecycle of Web Request
![[Pasted image 20240820231524.png|800]]
* Format Get: 在這裡其實 Browser 做了很多工作，包含連接地址與端口、主機資訊、用戶代理、接受的數據類型、接受的壓縮編碼、語言、Cookie
* Parse：是透過 Hyper 解析 Tcp 字節流，分析 Http 協議為可以操作的數據結構，裡面做了很多諸如 Https 解析等等內容。
* Layers：透過 Tower 實現，可以進行(解)壓縮、權限驗證等操作。
* Formatting: 透過 Hyper 進行轉換成 Tcp 友好的字節流。
> 裡面每一個步驟都是 await Boundary，在這些區間會透過異步使當前進程不會阻塞。


# 3. Extract
---
Extract 用於在 Handler 中提取 Http 中的資訊，用於給 Handler 做後續處理，可以使我的 Web 服務做出動態的內容，Extract 實做了 `FromRequest` 或 `FromRequestParts`，如下：
1. `axum::extract::Path`: 解析 Url 動態路徑
2. `axum::extract::Query`: 解析 Query String 
3. `axum::extract::Json`: 解析 body 中的 json
4. `axum::http::HeaderMap`: 解析 Header，本質也是 extract，但可能是因為可用於 request 與 response 所以沒放在 extract 模塊中。

### 使用
```rust
use axum::extract::Path;
use axum::extract::Query;
use axum::http::HeaderMap;
use axum::{response::Html, routing::get, Router};
use std::collections::HashMap;

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(handler))
        .route("/book/:id", get(path_extract)) // 使用 :id 表示為動態路徑
        .route("/book", get(query_extract))
        .route("/header", get(header_extract));

    let addr = "127.0.0.1:3000";
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    println!("Listening on {}", addr);
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}

async fn path_extract(Path(id): Path<u32>, // descructor 語法，並將 id 轉為 u32
) -> Html<String> {
    Html(format!("Hello, {}!", id))
}

async fn query_extract(Query(params): Query<HashMap<String, String>>) -> Html<String> {
    Html(format!("{:#?}", params))
}

async fn header_extract(headers: HeaderMap) -> Html<String> {
    Html(format!("{:#?}", headers))
}
```
> 若參數可空可再套一個 `Option<T>`

# 4. State and Extension
---
我們可以將一個配置透過注入的方式進行取得，Axum 底層採用 tower 達成，分為：
1. State: 表示全局狀態，只會綁定在一個 Router 上且只能有一個。(注入狀態) 
2. Extension: 將依賴注入到所有路由中，更加靈活。(注入服務)

```rust
#[tokio::main]
async fn main() {
    let shared_counter = Arc::new(MyCounter {
        counter: AtomicUsize::new(0),
    });
    let shared_config = Arc::new(MyConfig {
        text: "This is my configuration".to_owned(),
    });

    let app = Router::new()
        .route("/counter", get(handle_counter))
        .route("/extension", get(handle_extension)) 
        .with_state(Arc::clone(&shared_counter)) // 使用 with_state 注入 state (唯一)
        .layer(Extension(shared_counter)) // 使用 layer 注入 extension
        .layer(Extension(shared_config));

    let addr = "localhost:3000";
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    println!("Listening on {}", addr);
    axum::serve(listener, app).await.unwrap();
}

async fn handle_counter(State(mycounter): State<Arc<MyCounter>>) -> Html<String> {
    mycounter
        .counter
        .fetch_add(1, std::sync::atomic::Ordering::Relaxed);
    Html(format!(
        "<h1>Counter: {}</h1>",
        mycounter.counter.load(std::sync::atomic::Ordering::Relaxed)
    ))
}

async fn handle_extension(
    Extension(myconfig): Extension<Arc<MyConfig>>,
    Extension(mycounter): Extension<Arc<MyCounter>>,
) -> Html<String> {
    mycounter
        .counter
        .fetch_add(1, std::sync::atomic::Ordering::Relaxed);
    Html(format!(
        "<h1>Text: {}, Counter: {}</h1>",
        myconfig.text,
        mycounter.counter.load(std::sync::atomic::Ordering::Relaxed)
    ))
}
```


# 5. Mutiple Router
---
在項目進行模塊化後，我們會將路由進行分類，故我們可以使用 Axum 提供的 Nest Route 功能，讓我們將路由拆分成各個獨立的部份。

```rust
fn service_one() -> Router {
    Router::new().route("/", get(|| async { Html("Serive Once".to_owned()) }))
}

fn service_two() -> Router {
    Router::new().route("/", get(|| async { Html("Serive Two".to_owned()) }))
}

async fn main() {
    let app = Router::new()
        .nest("/service1", service_one())
        .nest("/service2", service_two())
        .route("/", get(handler))
        .route("/book/:id", get(path_extract))

    let addr = "localhost:3000";
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    println!("Listening on {}", addr);
    axum::serve(listener, app).await.unwrap();
}
```
> 注意: 使用 layer + Extension 會注入到所有的路由，而 State 只會在你指定的路由上才能獲取。


# 6. Calling Other Serivce
---
Rust 社區提供了同樣基於 tokio 的 Http client 庫稱為 `reqwest`，它可以進行 json 反序列化非常方便。添加方式如下
```shell
cargo add reqwest -F json # 提供 json 反序列化功能
```

### 使用
```rust
struct Counter {
    count: AtomicUsize,
}

#[tokio::main]
async fn main() {
    let counter = Arc::new(Counter {
        count: AtomicUsize::new(0),
    });

    let app = Router::new()
        .route("/", get(handler))
        .route("/inc", get(increament))
        .with_state(Arc::clone(&counter));

    let addr = "127.0.0.1:3000";
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    println!("Listening on: {}", addr);
    axum::serve(listener, app).await.unwrap();
}

async fn handler() -> Html<String> {
    println!("Sending GET request");
    let current_count = reqwest::get("http://localhost:3000/inc")
        .await
        .unwrap()
        .json::<usize>()
        .await
        .unwrap();
    Html(format!("<h1>Remote Counter: {}</h1>", current_count))
}

async fn increament(State(counter): State<Arc<Counter>>) -> Json<usize> {
    let current_value = counter.count.load(std::sync::atomic::Ordering::Relaxed);
    counter
        .count
        .fetch_add(1, std::sync::atomic::Ordering::Relaxed);
    Json(current_value)
}
```

# 7. 