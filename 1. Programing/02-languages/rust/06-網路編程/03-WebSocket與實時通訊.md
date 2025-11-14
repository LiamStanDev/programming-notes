# WebSocket 與實時通訊

## WebSocket 基礎

### 客戶端 (tokio-tungstenite)

```toml
[dependencies]
tokio-tungstenite = "0.21"
tokio = { version = "1", features = ["full"] }
futures-util = "0.3"
```

```rust
use tokio_tungstenite::{connect_async, tungstenite::Message};
use futures_util::{StreamExt, SinkExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "ws://localhost:8080/ws";
    let (ws_stream, _) = connect_async(url).await?;
    println!("WebSocket connected");
    
    let (mut write, mut read) = ws_stream.split();
    
    // 發送消息
    write.send(Message::Text("Hello, Server!".to_string())).await?;
    
    // 接收消息
    while let Some(msg) = read.next().await {
        let msg = msg?;
        if msg.is_text() || msg.is_binary() {
            println!("Received: {:?}", msg);
        }
    }
    
    Ok(())
}
```

### 服務端 (axum)

```rust
use axum::{
    Router,
    routing::get,
    extract::{
        ws::{WebSocket, WebSocketUpgrade, Message},
    },
    response::IntoResponse,
};
use futures::{sink::SinkExt, stream::StreamExt};

async fn ws_handler(ws: WebSocketUpgrade) -> impl IntoResponse {
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(msg) = socket.recv().await {
        if let Ok(msg) = msg {
            match msg {
                Message::Text(text) => {
                    println!("Received: {}", text);
                    if socket.send(Message::Text(format!("Echo: {}", text))).await.is_err() {
                        break;
                    }
                }
                Message::Binary(data) => {
                    if socket.send(Message::Binary(data)).await.is_err() {
                        break;
                    }
                }
                Message::Close(_) => break,
                _ => {}
            }
        }
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/ws", get(ws_handler));
    
    let listener = tokio::net::TcpListener::bind("127.0.0.1:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

## 聊天室實現

```rust
use axum::{
    Router,
    routing::get,
    extract::{
        ws::{WebSocket, WebSocketUpgrade, Message},
        State,
    },
    response::IntoResponse,
};
use std::sync::Arc;
use tokio::sync::{broadcast, RwLock};
use std::collections::HashMap;

type UserName = String;
type Users = Arc<RwLock<HashMap<UserName, broadcast::Sender<String>>>>;

#[derive(Clone)]
struct AppState {
    users: Users,
    tx: broadcast::Sender<String>,
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<AppState>,
) -> impl IntoResponse {
    ws.on_upgrade(|socket| handle_socket(socket, state))
}

async fn handle_socket(socket: WebSocket, state: AppState) {
    let (mut sender, mut receiver) = socket.split();
    let mut rx = state.tx.subscribe();
    
    // 接收用戶名
    let username = if let Some(Ok(Message::Text(name))) = receiver.next().await {
        name
    } else {
        return;
    };
    
    println!("{} joined", username);
    state.tx.send(format!("{} joined the chat", username)).unwrap();
    
    // 發送消息任務
    let username_clone = username.clone();
    let mut send_task = tokio::spawn(async move {
        while let Ok(msg) = rx.recv().await {
            if sender.send(Message::Text(msg)).await.is_err() {
                break;
            }
        }
    });
    
    // 接收消息任務
    let tx = state.tx.clone();
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(Message::Text(text))) = receiver.next().await {
            let msg = format!("{}: {}", username_clone, text);
            tx.send(msg).unwrap();
        }
    });
    
    // 等待任一任務完成
    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    };
    
    println!("{} left", username);
    state.tx.send(format!("{} left the chat", username)).unwrap();
}

#[tokio::main]
async fn main() {
    let (tx, _) = broadcast::channel(100);
    
    let state = AppState {
        users: Arc::new(RwLock::new(HashMap::new())),
        tx,
    };
    
    let app = Router::new()
        .route("/ws", get(ws_handler))
        .with_state(state);
    
    let listener = tokio::net::TcpListener::bind("127.0.0.1:8080").await.unwrap();
    println!("Chat server running on ws://127.0.0.1:8080/ws");
    axum::serve(listener, app).await.unwrap();
}
```

## Server-Sent Events (SSE)

```rust
use axum::{
    Router,
    routing::get,
    response::sse::{Event, Sse},
};
use futures::stream::{self, Stream};
use std::time::Duration;
use std::convert::Infallible;

async fn sse_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = stream::repeat_with(|| {
        Event::default().data(format!("Current time: {}", chrono::Utc::now()))
    })
    .map(Ok)
    .throttle(Duration::from_secs(1));
    
    Sse::new(stream).keep_alive(
        axum::response::sse::KeepAlive::new()
            .interval(Duration::from_secs(1))
            .text("keep-alive-text"),
    )
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/events", get(sse_handler));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

---

## 參考資料

1. [tokio-tungstenite Documentation](https://docs.rs/tokio-tungstenite/)
2. [axum WebSocket](https://docs.rs/axum/latest/axum/extract/ws/index.html)
3. [WebSocket RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455)
4. [Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
