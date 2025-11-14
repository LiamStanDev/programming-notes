# gRPC 與 Protocol Buffers

## Protocol Buffers 基礎

### 定義 .proto 文件

```protobuf
// user.proto
syntax = "proto3";

package user;

service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (stream User);
    rpc CreateUser(CreateUserRequest) returns (User);
    rpc StreamUsers(stream CreateUserRequest) returns (stream User);
}

message User {
    uint64 id = 1;
    string name = 2;
    string email = 3;
    repeated string roles = 4;
}

message GetUserRequest {
    uint64 id = 1;
}

message ListUsersRequest {
    uint32 page = 1;
    uint32 page_size = 2;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}
```

### Cargo 配置

```toml
[dependencies]
tonic = "0.11"
prost = "0.12"
tokio = { version = "1", features = ["full"] }

[build-dependencies]
tonic-build = "0.11"
```

### build.rs

```rust
fn main() {
    tonic_build::compile_protos("proto/user.proto")
        .unwrap_or_else(|e| panic!("Failed to compile protos {:?}", e));
}
```

## gRPC 服務端

### 實現服務

```rust
use tonic::{transport::Server, Request, Response, Status};

pub mod user {
    tonic::include_proto!("user");
}

use user::{
    user_service_server::{UserService, UserServiceServer},
    User, GetUserRequest, CreateUserRequest, ListUsersRequest,
};

#[derive(Default)]
pub struct MyUserService;

#[tonic::async_trait]
impl UserService for MyUserService {
    async fn get_user(
        &self,
        request: Request<GetUserRequest>,
    ) -> Result<Response<User>, Status> {
        let req = request.into_inner();
        
        Ok(Response::new(User {
            id: req.id,
            name: "John Doe".to_string(),
            email: "john@example.com".to_string(),
            roles: vec!["admin".to_string()],
        }))
    }
    
    type ListUsersStream = tokio_stream::wrappers::ReceiverStream<Result<User, Status>>;
    
    async fn list_users(
        &self,
        request: Request<ListUsersRequest>,
    ) -> Result<Response<Self::ListUsersStream>, Status> {
        let (tx, rx) = tokio::sync::mpsc::channel(4);
        
        tokio::spawn(async move {
            for i in 1..=10 {
                tx.send(Ok(User {
                    id: i,
                    name: format!("User {}", i),
                    email: format!("user{}@example.com", i),
                    roles: vec![],
                })).await.unwrap();
            }
        });
        
        Ok(Response::new(tokio_stream::wrappers::ReceiverStream::new(rx)))
    }
    
    async fn create_user(
        &self,
        request: Request<CreateUserRequest>,
    ) -> Result<Response<User>, Status> {
        let req = request.into_inner();
        
        Ok(Response::new(User {
            id: 999,
            name: req.name,
            email: req.email,
            roles: vec![],
        }))
    }
    
    type StreamUsersStream = tokio_stream::wrappers::ReceiverStream<Result<User, Status>>;
    
    async fn stream_users(
        &self,
        request: Request<tonic::Streaming<CreateUserRequest>>,
    ) -> Result<Response<Self::StreamUsersStream>, Status> {
        let mut stream = request.into_inner();
        let (tx, rx) = tokio::sync::mpsc::channel(4);
        
        tokio::spawn(async move {
            let mut id = 1;
            while let Some(req) = stream.message().await.unwrap() {
                tx.send(Ok(User {
                    id,
                    name: req.name,
                    email: req.email,
                    roles: vec![],
                })).await.unwrap();
                id += 1;
            }
        });
        
        Ok(Response::new(tokio_stream::wrappers::ReceiverStream::new(rx)))
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "127.0.0.1:50051".parse()?;
    let service = MyUserService::default();
    
    println!("gRPC server listening on {}", addr);
    
    Server::builder()
        .add_service(UserServiceServer::new(service))
        .serve(addr)
        .await?;
    
    Ok(())
}
```

## gRPC 客戶端

```rust
use user::user_service_client::UserServiceClient;
use user::{GetUserRequest, CreateUserRequest};

pub mod user {
    tonic::include_proto!("user");
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = UserServiceClient::connect("http://127.0.0.1:50051").await?;
    
    // Unary call
    let response = client.get_user(GetUserRequest { id: 1 }).await?;
    println!("User: {:?}", response.into_inner());
    
    // Server streaming
    let mut stream = client.list_users(user::ListUsersRequest {
        page: 1,
        page_size: 10,
    }).await?.into_inner();
    
    while let Some(user) = stream.message().await? {
        println!("Streamed user: {:?}", user);
    }
    
    // Client streaming
    let requests = vec![
        CreateUserRequest { name: "Alice".to_string(), email: "alice@example.com".to_string() },
        CreateUserRequest { name: "Bob".to_string(), email: "bob@example.com".to_string() },
    ];
    
    let request_stream = tokio_stream::iter(requests);
    let response = client.stream_users(request_stream).await?;
    let mut response_stream = response.into_inner();
    
    while let Some(user) = response_stream.message().await? {
        println!("Created user: {:?}", user);
    }
    
    Ok(())
}
```

## 中間件與攔截器

```rust
use tonic::{Request, Status};

// 認證攔截器
fn auth_interceptor(mut req: Request<()>) -> Result<Request<()>, Status> {
    let token = req.metadata().get("authorization")
        .and_then(|v| v.to_str().ok());
    
    if token == Some("Bearer SECRET") {
        Ok(req)
    } else {
        Err(Status::unauthenticated("Invalid token"))
    }
}

// 客戶端使用
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let channel = tonic::transport::Channel::from_static("http://127.0.0.1:50051")
        .connect()
        .await?;
    
    let mut client = user::user_service_client::UserServiceClient::with_interceptor(
        channel,
        auth_interceptor,
    );
    
    Ok(())
}
```

---

## 參考資料

1. [tonic Documentation](https://docs.rs/tonic/)
2. [Protocol Buffers](https://developers.google.com/protocol-buffers)
3. [gRPC Official](https://grpc.io/)
