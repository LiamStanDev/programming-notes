# TCP 與 UDP 基礎

## TCP Socket 編程

### 基礎服務端

```rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Server listening on port 8080");
    
    loop {
        let (mut socket, addr) = listener.accept().await?;
        println!("New connection from: {}", addr);
        
        tokio::spawn(async move {
            let mut buffer = [0; 1024];
            
            loop {
                match socket.read(&mut buffer).await {
                    Ok(0) => {
                        println!("Client {} disconnected", addr);
                        break;
                    }
                    Ok(n) => {
                        if socket.write_all(&buffer[..n]).await.is_err() {
                            eprintln!("Failed to write to {}", addr);
                            break;
                        }
                    }
                    Err(e) => {
                        eprintln!("Error reading from {}: {}", addr, e);
                        break;
                    }
                }
            }
        });
    }
}
```

### 基礎客戶端

```rust
use tokio::net::TcpStream;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut stream = TcpStream::connect("127.0.0.1:8080").await?;
    println!("Connected to server");
    
    // 發送數據
    stream.write_all(b"Hello, server!").await?;
    
    // 接收響應
    let mut buffer = [0; 1024];
    let n = stream.read(&mut buffer).await?;
    println!("Received: {}", String::from_utf8_lossy(&buffer[..n]));
    
    Ok(())
}
```

### 行式協議 (Line-Based Protocol)

```rust
use tokio::net::TcpListener;
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    
    loop {
        let (socket, _) = listener.accept().await?;
        tokio::spawn(async move {
            handle_connection(socket).await;
        });
    }
}

async fn handle_connection(socket: tokio::net::TcpStream) {
    let (reader, mut writer) = socket.into_split();
    let mut reader = BufReader::new(reader);
    let mut line = String::new();
    
    loop {
        line.clear();
        match reader.read_line(&mut line).await {
            Ok(0) => break,  // EOF
            Ok(_) => {
                // Echo 回客戶端
                if writer.write_all(line.as_bytes()).await.is_err() {
                    break;
                }
            }
            Err(e) => {
                eprintln!("Error: {}", e);
                break;
            }
        }
    }
}
```

### 分幀 (Framing)

```rust
use tokio::net::TcpStream;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use bytes::{Buf, BufMut, BytesMut};

// 長度前綴幀格式：[4 bytes length][payload]
async fn read_frame(socket: &mut TcpStream) -> std::io::Result<BytesMut> {
    // 讀取長度
    let mut len_bytes = [0u8; 4];
    socket.read_exact(&mut len_bytes).await?;
    let len = u32::from_be_bytes(len_bytes) as usize;
    
    // 讀取 payload
    let mut buffer = BytesMut::with_capacity(len);
    buffer.resize(len, 0);
    socket.read_exact(&mut buffer).await?;
    
    Ok(buffer)
}

async fn write_frame(socket: &mut TcpStream, data: &[u8]) -> std::io::Result<()> {
    // 寫入長度
    let len = data.len() as u32;
    socket.write_all(&len.to_be_bytes()).await?;
    
    // 寫入 payload
    socket.write_all(data).await?;
    
    Ok(())
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut stream = TcpStream::connect("127.0.0.1:8080").await?;
    
    write_frame(&mut stream, b"Hello, World!").await?;
    let response = read_frame(&mut stream).await?;
    println!("Received: {:?}", response);
    
    Ok(())
}
```

### Socket 選項

```rust
use tokio::net::TcpStream;
use std::time::Duration;

async fn configure_socket(stream: &TcpStream) -> std::io::Result<()> {
    // 設置 TCP_NODELAY（禁用 Nagle 算法）
    stream.set_nodelay(true)?;
    
    // 設置 keepalive
    let keepalive = socket2::TcpKeepalive::new()
        .with_time(Duration::from_secs(60))
        .with_interval(Duration::from_secs(10));
    
    let socket = socket2::SockRef::from(stream);
    socket.set_tcp_keepalive(&keepalive)?;
    
    // 設置接收緩衝區大小
    socket.set_recv_buffer_size(64 * 1024)?;
    socket.set_send_buffer_size(64 * 1024)?;
    
    Ok(())
}
```

## UDP Socket 編程

### 基礎服務端

```rust
use tokio::net::UdpSocket;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let socket = UdpSocket::bind("127.0.0.1:8080").await?;
    println!("UDP server listening on port 8080");
    
    let mut buffer = vec![0u8; 1024];
    
    loop {
        let (len, addr) = socket.recv_from(&mut buffer).await?;
        println!("Received {} bytes from {}", len, addr);
        
        // Echo 回客戶端
        socket.send_to(&buffer[..len], addr).await?;
    }
}
```

### 基礎客戶端

```rust
use tokio::net::UdpSocket;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let socket = UdpSocket::bind("0.0.0.0:0").await?;
    socket.connect("127.0.0.1:8080").await?;
    
    // 發送數據
    socket.send(b"Hello, UDP server!").await?;
    
    // 接收響應
    let mut buffer = vec![0u8; 1024];
    let n = socket.recv(&mut buffer).await?;
    println!("Received: {}", String::from_utf8_lossy(&buffer[..n]));
    
    Ok(())
}
```

### 廣播 (Broadcast)

```rust
use tokio::net::UdpSocket;
use std::net::SocketAddr;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let socket = UdpSocket::bind("0.0.0.0:0").await?;
    
    // 啟用廣播
    socket.set_broadcast(true)?;
    
    // 發送廣播消息
    let broadcast_addr: SocketAddr = "255.255.255.255:8080".parse()?;
    socket.send_to(b"Broadcast message", broadcast_addr).await?;
    
    Ok(())
}
```

### 組播 (Multicast)

```rust
use tokio::net::UdpSocket;
use std::net::{Ipv4Addr, SocketAddr};

// 組播接收者
async fn multicast_receiver() -> Result<(), Box<dyn std::error::Error>> {
    let multicast_addr: Ipv4Addr = "239.0.0.1".parse()?;
    let bind_addr: SocketAddr = "0.0.0.0:8080".parse()?;
    
    let socket = UdpSocket::bind(bind_addr).await?;
    socket.join_multicast_v4(multicast_addr, Ipv4Addr::UNSPECIFIED)?;
    
    let mut buffer = vec![0u8; 1024];
    loop {
        let (len, addr) = socket.recv_from(&mut buffer).await?;
        println!("Multicast from {}: {}", addr, String::from_utf8_lossy(&buffer[..len]));
    }
}

// 組播發送者
async fn multicast_sender() -> Result<(), Box<dyn std::error::Error>> {
    let socket = UdpSocket::bind("0.0.0.0:0").await?;
    let multicast_addr: SocketAddr = "239.0.0.1:8080".parse()?;
    
    loop {
        socket.send_to(b"Multicast message", multicast_addr).await?;
        tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
    }
}
```

## 連接池

### 簡單連接池

```rust
use tokio::net::TcpStream;
use tokio::sync::Mutex;
use std::sync::Arc;

struct ConnectionPool {
    connections: Mutex<Vec<TcpStream>>,
    addr: String,
    max_size: usize,
}

impl ConnectionPool {
    fn new(addr: String, max_size: usize) -> Self {
        Self {
            connections: Mutex::new(Vec::new()),
            addr,
            max_size,
        }
    }
    
    async fn acquire(&self) -> Result<TcpStream, Box<dyn std::error::Error>> {
        let mut pool = self.connections.lock().await;
        
        if let Some(conn) = pool.pop() {
            Ok(conn)
        } else {
            drop(pool);  // 釋放鎖
            TcpStream::connect(&self.addr).await.map_err(Into::into)
        }
    }
    
    async fn release(&self, conn: TcpStream) {
        let mut pool = self.connections.lock().await;
        if pool.len() < self.max_size {
            pool.push(conn);
        }
        // 超過最大大小則丟棄連接
    }
}

#[tokio::main]
async fn main() {
    let pool = Arc::new(ConnectionPool::new("127.0.0.1:8080".to_string(), 10));
    
    let pool_clone = pool.clone();
    tokio::spawn(async move {
        match pool_clone.acquire().await {
            Ok(mut conn) => {
                // 使用連接
                let _ = conn;
                // pool_clone.release(conn).await;
            }
            Err(e) => eprintln!("Failed to acquire connection: {}", e),
        }
    }).await.unwrap();
}
```

### 使用 deadpool

```rust
use deadpool::managed::{Manager, Pool, RecycleResult};
use tokio::net::TcpStream;
use async_trait::async_trait;

struct TcpConnectionManager {
    addr: String,
}

#[async_trait]
impl Manager for TcpConnectionManager {
    type Type = TcpStream;
    type Error = std::io::Error;
    
    async fn create(&self) -> Result<TcpStream, std::io::Error> {
        TcpStream::connect(&self.addr).await
    }
    
    async fn recycle(&self, _conn: &mut TcpStream) -> RecycleResult<std::io::Error> {
        // 檢查連接是否有效
        Ok(())
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let manager = TcpConnectionManager {
        addr: "127.0.0.1:8080".to_string(),
    };
    
    let pool = Pool::builder(manager)
        .max_size(10)
        .build()
        .unwrap();
    
    let conn = pool.get().await?;
    // 使用連接
    drop(conn);  // 自動歸還連接池
    
    Ok(())
}
```

## 協議實現範例

### Redis RESP 協議

```rust
use tokio::net::TcpStream;
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};

enum RespValue {
    SimpleString(String),
    Error(String),
    Integer(i64),
    BulkString(Option<Vec<u8>>),
    Array(Vec<RespValue>),
}

async fn send_command(stream: &mut TcpStream, args: &[&str]) -> std::io::Result<()> {
    let mut cmd = format!("*{}\r\n", args.len());
    for arg in args {
        cmd.push_str(&format!("${}\r\n{}\r\n", arg.len(), arg));
    }
    stream.write_all(cmd.as_bytes()).await
}

async fn read_response(reader: &mut BufReader<TcpStream>) -> std::io::Result<RespValue> {
    let mut line = String::new();
    reader.read_line(&mut line).await?;
    
    match line.chars().next() {
        Some('+') => Ok(RespValue::SimpleString(line[1..line.len()-2].to_string())),
        Some('-') => Ok(RespValue::Error(line[1..line.len()-2].to_string())),
        Some(':') => {
            let num = line[1..line.len()-2].parse().unwrap();
            Ok(RespValue::Integer(num))
        }
        Some('$') => {
            let len: i64 = line[1..line.len()-2].parse().unwrap();
            if len == -1 {
                Ok(RespValue::BulkString(None))
            } else {
                let mut buf = vec![0u8; len as usize];
                reader.read_exact(&mut buf).await?;
                reader.read_line(&mut line).await?;  // \r\n
                Ok(RespValue::BulkString(Some(buf)))
            }
        }
        _ => unimplemented!(),
    }
}
```

### HTTP/1.1 簡易解析

```rust
use tokio::io::{AsyncBufReadExt, BufReader};
use tokio::net::TcpStream;
use std::collections::HashMap;

struct HttpRequest {
    method: String,
    path: String,
    headers: HashMap<String, String>,
    body: Vec<u8>,
}

async fn parse_request(stream: TcpStream) -> std::io::Result<HttpRequest> {
    let mut reader = BufReader::new(stream);
    
    // 解析請求行
    let mut line = String::new();
    reader.read_line(&mut line).await?;
    let parts: Vec<&str> = line.trim().split_whitespace().collect();
    let (method, path) = (parts[0].to_string(), parts[1].to_string());
    
    // 解析 headers
    let mut headers = HashMap::new();
    loop {
        line.clear();
        reader.read_line(&mut line).await?;
        if line.trim().is_empty() {
            break;
        }
        
        if let Some((key, value)) = line.split_once(':') {
            headers.insert(
                key.trim().to_lowercase(),
                value.trim().to_string(),
            );
        }
    }
    
    // 讀取 body（如果有 Content-Length）
    let mut body = Vec::new();
    if let Some(len_str) = headers.get("content-length") {
        if let Ok(len) = len_str.parse::<usize>() {
            body.resize(len, 0);
            use tokio::io::AsyncReadExt;
            reader.read_exact(&mut body).await?;
        }
    }
    
    Ok(HttpRequest { method, path, headers, body })
}
```

## 錯誤處理

### 超時處理

```rust
use tokio::net::TcpStream;
use tokio::time::{timeout, Duration};

async fn connect_with_timeout(addr: &str) -> Result<TcpStream, Box<dyn std::error::Error>> {
    match timeout(Duration::from_secs(5), TcpStream::connect(addr)).await {
        Ok(Ok(stream)) => Ok(stream),
        Ok(Err(e)) => Err(Box::new(e)),
        Err(_) => Err("Connection timeout".into()),
    }
}
```

### 重連機制

```rust
use tokio::net::TcpStream;
use tokio::time::{sleep, Duration};

async fn connect_with_retry(addr: &str, max_retries: u32) -> Result<TcpStream, std::io::Error> {
    let mut attempts = 0;
    
    loop {
        match TcpStream::connect(addr).await {
            Ok(stream) => return Ok(stream),
            Err(e) if attempts < max_retries => {
                attempts += 1;
                eprintln!("Connection failed, retry {}/{}: {}", attempts, max_retries, e);
                sleep(Duration::from_secs(2_u64.pow(attempts))).await;
            }
            Err(e) => return Err(e),
        }
    }
}
```

---

## 參考資料

1. [Tokio TCP Documentation](https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html)
2. [Tokio UDP Documentation](https://docs.rs/tokio/latest/tokio/net/struct.UdpSocket.html)
3. [TCP/IP Illustrated](https://en.wikipedia.org/wiki/TCP/IP_Illustrated) (W. Richard Stevens)
4. [UNIX Network Programming](https://en.wikipedia.org/wiki/UNIX_Network_Programming) (W. Richard Stevens)
5. [Rust Network Programming](https://learning.oreilly.com/library/view/network-programming-with/9781788624893/)
