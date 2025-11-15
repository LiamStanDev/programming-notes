# Day 9：gRPC 與訊息佇列

## 📚 學習目標

- 掌握 Protobuf 語法與代碼生成
- 實現 gRPC Server/Client
- 理解 Kafka Consumer Group
- 實現 Offset 管理

---

## 1. Protobuf 定義

### 1.1 安裝工具

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

### 1.2 定義服務

**proto/user.proto**:
```protobuf
syntax = "proto3";

package user;

option go_package = "github.com/myapp/proto/user";

service UserService {
    rpc GetUser(GetUserRequest) returns (GetUserResponse);
    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
}

message GetUserRequest {
    int32 id = 1;
}

message GetUserResponse {
    int32 id = 1;
    string username = 2;
    string email = 3;
}

message ListUsersRequest {
    int32 page = 1;
    int32 page_size = 2;
}

message ListUsersResponse {
    repeated GetUserResponse users = 1;
    int32 total = 2;
}
```

### 1.3 生成代碼

```bash
protoc --go_out=. --go-grpc_out=. proto/user.proto
```

---

## 2. gRPC Server

```go
package main

import (
    "context"
    "log"
    "net"
    
    pb "github.com/myapp/proto/user"
    "google.golang.org/grpc"
)

type server struct {
    pb.UnimplementedUserServiceServer
    repo UserRepository
}

func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    user, err := s.repo.GetByID(int(req.Id))
    if err != nil {
        return nil, err
    }
    
    return &pb.GetUserResponse{
        Id:       int32(user.ID),
        Username: user.Username,
        Email:    user.Email,
    }, nil
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatal(err)
    }
    
    s := grpc.NewServer()
    pb.RegisterUserServiceServer(s, &server{})
    
    log.Println("gRPC server listening on :50051")
    log.Fatal(s.Serve(lis))
}
```

---

## 3. gRPC Client

```go
package main

import (
    "context"
    "log"
    
    pb "github.com/myapp/proto/user"
    "google.golang.org/grpc"
)

func main() {
    conn, err := grpc.Dial("localhost:50051", grpc.WithInsecure())
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()
    
    client := pb.NewUserServiceClient(conn)
    
    resp, err := client.GetUser(context.Background(), &pb.GetUserRequest{
        Id: 1,
    })
    if err != nil {
        log.Fatal(err)
    }
    
    log.Printf("User: %v", resp)
}
```

---

## 4. Kafka Consumer Group

```go
func (c *EventConsumer) ConsumeGroup(ctx context.Context, handler func(kafka.Message) error) error {
    for {
        msg, err := c.reader.FetchMessage(ctx)
        if err != nil {
            return err
        }
        
        if err := handler(msg); err != nil {
            log.Println("Handler error:", err)
            continue
        }
        
        if err := c.reader.CommitMessages(ctx, msg); err != nil {
            log.Println("Commit error:", err)
        }
    }
}
```

---

**上一篇**: [Day 8 - 數據持久化與容器化](../03-Web開發篇/08-數據持久化與容器化.md)  
**下一篇**: [Day 10 - 可觀測性與生產部署](10-可觀測性與生產部署.md)
