# HTTP 客戶端與服務端

## HTTP 客戶端 (reqwest)

### 基礎用法

```toml
[dependencies]
reqwest = { version = "0.11", features = ["json"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

```rust
use reqwest;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // GET 請求
    let response = reqwest::get("https://httpbin.org/get").await?;
    let body = response.text().await?;
    println!("{}", body);
    
    // POST JSON
    let client = reqwest::Client::new();
    let res = client.post("https://httpbin.org/post")
        .json(&serde_json::json!({
            "key": "value"
        }))
        .send()
        .await?;
    
    println!("Status: {}", res.status());
    
    Ok(())
}
```

### 完整 HTTP 客戶端

```rust
use reqwest::{Client, header};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    email: String,
}

struct ApiClient {
    client: Client,
    base_url: String,
}

impl ApiClient {
    fn new(base_url: String) -> Self {
        let mut headers = header::HeaderMap::new();
        headers.insert(
            header::AUTHORIZATION,
            header::HeaderValue::from_static("Bearer TOKEN"),
        );
        
        let client = Client::builder()
            .default_headers(headers)
            .timeout(std::time::Duration::from_secs(30))
            .build()
            .unwrap();
        
        Self { client, base_url }
    }
    
    async fn get_user(&self, id: u64) -> Result<User, Box<dyn std::error::Error>> {
        let url = format!("{}/users/{}", self.base_url, id);
        let user = self.client.get(&url)
            .send()
            .await?
            .json::<User>()
            .await?;
        Ok(user)
    }
    
    async fn create_user(&self, user: &User) -> Result<User, Box<dyn std::error::Error>> {
        let url = format!("{}/users", self.base_url);
        let created = self.client.post(&url)
            .json(user)
            .send()
            .await?
            .json::<User>()
            .await?;
        Ok(created)
    }
}
```

### 文件上傳下載

```rust
use reqwest::multipart;
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

// 文件上傳
async fn upload_file() -> Result<(), Box<dyn std::error::Error>> {
    let form = multipart::Form::new()
        .text("field1", "value1")
        .file("file", "path/to/file.txt").await?;
    
    let client = reqwest::Client::new();
    client.post("https://httpbin.org/post")
        .multipart(form)
        .send()
        .await?;
    
    Ok(())
}

// 文件下載
async fn download_file(url: &str, path: &str) -> Result<(), Box<dyn std::error::Error>> {
    let response = reqwest::get(url).await?;
    let mut file = File::create(path).await?;
    let content = response.bytes().await?;
    file.write_all(&content).await?;
    Ok(())
}

// 流式下載
async fn stream_download(url: &str, path: &str) -> Result<(), Box<dyn std::error::Error>> {
    use futures::StreamExt;
    
    let response = reqwest::get(url).await?;
    let mut file = File::create(path).await?;
    let mut stream = response.bytes_stream();
    
    while let Some(chunk) = stream.next().await {
        file.write_all(&chunk?).await?;
    }
    
    Ok(())
}
```

## HTTP 服務端 (axum)

### 基礎路由

```toml
[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

```rust
use axum::{
    routing::{get, post},
    Router,
    Json,
    extract::Path,
};
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
}

async fn root() -> &'static str {
    "Hello, World!"
}

async fn get_user(Path(id): Path<u64>) -> Json<User> {
    Json(User {
        id,
        name: format!("User {}", id),
    })
}

async fn create_user(Json(user): Json<User>) -> Json<User> {
    Json(user)
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(root))
        .route("/users/:id", get(get_user))
        .route("/users", post(create_user));
    
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    
    axum::serve(listener, app).await.unwrap();
}
```

### 狀態管理

```rust
use axum::{
    Router,
    extract::State,
    routing::get,
    Json,
};
use std::sync::Arc;
use tokio::sync::RwLock;
use std::collections::HashMap;

#[derive(Clone)]
struct AppState {
    db: Arc<RwLock<HashMap<u64, String>>>,
}

async fn get_item(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> Option<Json<String>> {
    let db = state.db.read().await;
    db.get(&id).map(|v| Json(v.clone()))
}

#[tokio::main]
async fn main() {
    let state = AppState {
        db: Arc::new(RwLock::new(HashMap::new())),
    };
    
    let app = Router::new()
        .route("/items/:id", get(get_item))
        .with_state(state);
    
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### 中間件

```rust
use axum::{
    Router,
    middleware::{self, Next},
    http::Request,
    response::Response,
    routing::get,
};

async fn logging_middleware<B>(
    req: Request<B>,
    next: Next<B>,
) -> Response {
    println!("{} {}", req.method(), req.uri());
    next.run(req).await
}

async fn auth_middleware<B>(
    req: Request<B>,
    next: Next<B>,
) -> Response {
    // 驗證 token
    let token = req.headers()
        .get("authorization")
        .and_then(|v| v.to_str().ok());
    
    if token == Some("Bearer SECRET") {
        next.run(req).await
    } else {
        Response::builder()
            .status(401)
            .body("Unauthorized".into())
            .unwrap()
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(|| async { "Public" }))
        .route("/protected", get(|| async { "Protected" }))
        .layer(middleware::from_fn(logging_middleware))
        .route_layer(middleware::from_fn(auth_middleware));
    
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### 錯誤處理

```rust
use axum::{
    Router,
    routing::get,
    response::{IntoResponse, Response},
    http::StatusCode,
    Json,
};
use serde_json::json;

enum AppError {
    NotFound,
    InternalError(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "Not found"),
            AppError::InternalError(msg) => (StatusCode::INTERNAL_SERVER_ERROR, msg.as_str()),
        };
        
        (status, Json(json!({ "error": message }))).into_response()
    }
}

async fn handler() -> Result<Json<String>, AppError> {
    Err(AppError::NotFound)
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(handler));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

## 完整 REST API 範例

```rust
use axum::{
    Router,
    routing::{get, post, put, delete},
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::RwLock;
use std::collections::HashMap;

#[derive(Clone, Serialize, Deserialize)]
struct Todo {
    id: u64,
    title: String,
    completed: bool,
}

#[derive(Clone)]
struct AppState {
    todos: Arc<RwLock<HashMap<u64, Todo>>>,
    next_id: Arc<RwLock<u64>>,
}

// GET /todos
async fn list_todos(State(state): State<AppState>) -> Json<Vec<Todo>> {
    let todos = state.todos.read().await;
    Json(todos.values().cloned().collect())
}

// GET /todos/:id
async fn get_todo(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> Result<Json<Todo>, StatusCode> {
    let todos = state.todos.read().await;
    todos.get(&id)
        .cloned()
        .map(Json)
        .ok_or(StatusCode::NOT_FOUND)
}

// POST /todos
async fn create_todo(
    State(state): State<AppState>,
    Json(mut todo): Json<Todo>,
) -> (StatusCode, Json<Todo>) {
    let mut next_id = state.next_id.write().await;
    todo.id = *next_id;
    *next_id += 1;
    
    let mut todos = state.todos.write().await;
    todos.insert(todo.id, todo.clone());
    
    (StatusCode::CREATED, Json(todo))
}

// PUT /todos/:id
async fn update_todo(
    State(state): State<AppState>,
    Path(id): Path<u64>,
    Json(updated): Json<Todo>,
) -> Result<Json<Todo>, StatusCode> {
    let mut todos = state.todos.write().await;
    if todos.contains_key(&id) {
        todos.insert(id, updated.clone());
        Ok(Json(updated))
    } else {
        Err(StatusCode::NOT_FOUND)
    }
}

// DELETE /todos/:id
async fn delete_todo(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> StatusCode {
    let mut todos = state.todos.write().await;
    if todos.remove(&id).is_some() {
        StatusCode::NO_CONTENT
    } else {
        StatusCode::NOT_FOUND
    }
}

#[tokio::main]
async fn main() {
    let state = AppState {
        todos: Arc::new(RwLock::new(HashMap::new())),
        next_id: Arc::new(RwLock::new(1)),
    };
    
    let app = Router::new()
        .route("/todos", get(list_todos).post(create_todo))
        .route("/todos/:id", get(get_todo).put(update_todo).delete(delete_todo))
        .with_state(state);
    
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    println!("Server running on http://127.0.0.1:3000");
    axum::serve(listener, app).await.unwrap();
}
```

## 使用 actix-web

### 基礎服務端

```toml
[dependencies]
actix-web = "4"
serde = { version = "1", features = ["derive"] }
```

```rust
use actix_web::{web, App, HttpServer, HttpResponse, Responder};
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct User {
    name: String,
}

async fn index() -> impl Responder {
    HttpResponse::Ok().body("Hello, World!")
}

async fn echo(user: web::Json<User>) -> impl Responder {
    web::Json(user.into_inner())
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(index))
            .route("/echo", web::post().to(echo))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

## HTTP/2 與 HTTP/3

### hyper HTTP/2

```rust
use hyper::{Body, Request, Response, Server};
use hyper::service::{make_service_fn, service_fn};

async fn handle(_req: Request<Body>) -> Result<Response<Body>, hyper::Error> {
    Ok(Response::new(Body::from("Hello, HTTP/2!")))
}

#[tokio::main]
async fn main() {
    let make_svc = make_service_fn(|_conn| async {
        Ok::<_, hyper::Error>(service_fn(handle))
    });
    
    let addr = ([127, 0, 0, 1], 3000).into();
    let server = Server::bind(&addr).serve(make_svc);
    
    println!("Listening on http://{}", addr);
    server.await.unwrap();
}
```

---

## 參考資料

1. [reqwest Documentation](https://docs.rs/reqwest/)
2. [axum Documentation](https://docs.rs/axum/)
3. [actix-web Documentation](https://actix.rs/)
4. [hyper Documentation](https://hyper.rs/)
5. [HTTP/2 RFC 7540](https://datatracker.ietf.org/doc/html/rfc7540)
