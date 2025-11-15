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

## 5. gRPC 進階特性

### 5.1 攔截器（Interceptor）

```go
// 日誌攔截器
func LoggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()
    
    log.Printf("Method: %s, Request: %v", info.FullMethod, req)
    
    resp, err := handler(ctx, req)
    
    log.Printf("Method: %s, Duration: %v, Error: %v",
        info.FullMethod, time.Since(start), err)
    
    return resp, err
}

// 使用攔截器
s := grpc.NewServer(
    grpc.UnaryInterceptor(LoggingInterceptor),
)
```

### 5.2 錯誤處理

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    user, err := s.repo.GetByID(int(req.Id))
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            return nil, status.Error(codes.NotFound, "user not found")
        }
        return nil, status.Error(codes.Internal, "internal error")
    }
    
    return &pb.GetUserResponse{
        Id:       int32(user.ID),
        Username: user.Username,
        Email:    user.Email,
    }, nil
}
```

### 5.3 流式 RPC

```go
// 服務端流式
service UserService {
    rpc StreamUsers(StreamUsersRequest) returns (stream User);
}

func (s *server) StreamUsers(req *pb.StreamUsersRequest, stream pb.UserService_StreamUsersServer) error {
    users, err := s.repo.List(100, 0)
    if err != nil {
        return err
    }
    
    for _, user := range users {
        if err := stream.Send(&pb.User{
            Id:       int32(user.ID),
            Username: user.Username,
        }); err != nil {
            return err
        }
    }
    
    return nil
}

// 客戶端流式
service UserService {
    rpc BulkCreateUsers(stream CreateUserRequest) returns (BulkCreateResponse);
}

func (s *server) BulkCreateUsers(stream pb.UserService_BulkCreateUsersServer) error {
    count := 0
    
    for {
        req, err := stream.Recv()
        if err == io.EOF {
            return stream.SendAndClose(&pb.BulkCreateResponse{
                Count: int32(count),
            })
        }
        if err != nil {
            return err
        }
        
        if err := s.repo.Create(&User{
            Username: req.Username,
            Email:    req.Email,
        }); err != nil {
            return err
        }
        
        count++
    }
}
```

---

## 6. Kafka 進階

### 6.1 手動 Offset 管理

```go
type ManualConsumer struct {
    reader *kafka.Reader
}

func (c *ManualConsumer) Consume(ctx context.Context) error {
    for {
        msg, err := c.reader.FetchMessage(ctx)
        if err != nil {
            return err
        }
        
        // 處理消息
        if err := c.processMessage(msg); err != nil {
            log.Printf("Failed to process message: %v", err)
            continue  // 不提交 offset，下次重試
        }
        
        // 成功處理後提交 offset
        if err := c.reader.CommitMessages(ctx, msg); err != nil {
            log.Printf("Failed to commit offset: %v", err)
        }
    }
}
```

### 6.2 批量處理

```go
type BatchProcessor struct {
    reader    *kafka.Reader
    batchSize int
    timeout   time.Duration
}

func (p *BatchProcessor) ProcessBatch(ctx context.Context) error {
    batch := make([]kafka.Message, 0, p.batchSize)
    timer := time.NewTimer(p.timeout)
    
    for {
        select {
        case <-timer.C:
            if len(batch) > 0 {
                if err := p.processBatch(batch); err != nil {
                    return err
                }
                batch = batch[:0]
            }
            timer.Reset(p.timeout)
            
        default:
            msg, err := p.reader.FetchMessage(ctx)
            if err != nil {
                return err
            }
            
            batch = append(batch, msg)
            
            if len(batch) >= p.batchSize {
                if err := p.processBatch(batch); err != nil {
                    return err
                }
                batch = batch[:0]
                timer.Reset(p.timeout)
            }
        }
    }
}

func (p *BatchProcessor) processBatch(batch []kafka.Message) error {
    // 批量處理邏輯
    for _, msg := range batch {
        log.Printf("Processing: %s", string(msg.Value))
    }
    
    // 提交最後一條消息的 offset
    if len(batch) > 0 {
        return p.reader.CommitMessages(context.Background(), batch[len(batch)-1])
    }
    
    return nil
}
```

### 6.3 死信隊列

```go
type DeadLetterQueue struct {
    writer *kafka.Writer
}

func (d *DeadLetterQueue) Send(msg kafka.Message, err error) error {
    deadLetter := kafka.Message{
        Key:   msg.Key,
        Value: msg.Value,
        Headers: []kafka.Header{
            {Key: "original-topic", Value: []byte(msg.Topic)},
            {Key: "error", Value: []byte(err.Error())},
            {Key: "timestamp", Value: []byte(time.Now().Format(time.RFC3339))},
        },
    }
    
    return d.writer.WriteMessages(context.Background(), deadLetter)
}

// 使用
func (c *Consumer) Consume(ctx context.Context, dlq *DeadLetterQueue) error {
    for {
        msg, err := c.reader.FetchMessage(ctx)
        if err != nil {
            return err
        }
        
        if err := c.processMessage(msg); err != nil {
            log.Printf("Failed to process, sending to DLQ: %v", err)
            dlq.Send(msg, err)
        }
        
        c.reader.CommitMessages(ctx, msg)
    }
}
```

---

## 7. 實戰：微服務架構

### 7.1 服務間通信

```go
// User Service (gRPC)
type UserGRPCService struct {
    pb.UnimplementedUserServiceServer
    repo repository.UserRepository
}

// Order Service (REST) 調用 User Service (gRPC)
type OrderService struct {
    userClient pb.UserServiceClient
}

func (s *OrderService) CreateOrder(ctx context.Context, userID int, items []Item) error {
    // 通過 gRPC 驗證用戶
    user, err := s.userClient.GetUser(ctx, &pb.GetUserRequest{
        Id: int32(userID),
    })
    if err != nil {
        return fmt.Errorf("failed to get user: %w", err)
    }
    
    // 創建訂單
    order := &Order{
        UserID: userID,
        Items:  items,
    }
    
    // 發布事件到 Kafka
    if err := s.publishOrderCreated(order); err != nil {
        return err
    }
    
    return nil
}
```

### 7.2 事件驅動架構

```go
// 事件定義
type OrderCreatedEvent struct {
    OrderID   string    `json:"order_id"`
    UserID    int       `json:"user_id"`
    Total     float64   `json:"total"`
    CreatedAt time.Time `json:"created_at"`
}

// 發布者
type EventPublisher struct {
    producer *kafka.Writer
}

func (p *EventPublisher) PublishOrderCreated(event *OrderCreatedEvent) error {
    data, _ := json.Marshal(event)
    
    return p.producer.WriteMessages(context.Background(), kafka.Message{
        Topic: "order.created",
        Key:   []byte(event.OrderID),
        Value: data,
    })
}

// 訂閱者（通知服務）
type NotificationService struct {
    consumer *kafka.Reader
}

func (s *NotificationService) HandleOrderCreated(ctx context.Context) error {
    return s.consumer.Consume(ctx, func(msg kafka.Message) error {
        var event OrderCreatedEvent
        if err := json.Unmarshal(msg.Value, &event); err != nil {
            return err
        }
        
        // 發送通知
        return s.sendNotification(event.UserID, fmt.Sprintf(
            "Your order %s has been created", event.OrderID,
        ))
    })
}
```

---

## 8. 實戰練習

### 練習 1：實現 gRPC 健康檢查

```go
import "google.golang.org/grpc/health/grpc_health_v1"

type healthServer struct {
    grpc_health_v1.UnimplementedHealthServer
}

func (s *healthServer) Check(ctx context.Context, req *grpc_health_v1.HealthCheckRequest) (*grpc_health_v1.HealthCheckResponse, error) {
    // TODO: 實現健康檢查邏輯
    return &grpc_health_v1.HealthCheckResponse{
        Status: grpc_health_v1.HealthCheckResponse_SERVING,
    }, nil
}

// 註冊健康檢查服務
grpc_health_v1.RegisterHealthServer(s, &healthServer{})
```

### 練習 2：實現 Kafka 消息重試機制

```go
type RetryConsumer struct {
    reader      *kafka.Reader
    maxRetries  int
    retryDelay  time.Duration
}

func (c *RetryConsumer) ConsumeWithRetry(ctx context.Context) error {
    // TODO: 實現重試邏輯
    // 1. 嘗試處理消息
    // 2. 如果失敗，延遲後重試
    // 3. 超過最大重試次數後發送到 DLQ
}
```

### 練習 3：實現服務發現

```go
type ServiceRegistry struct {
    services map[string][]string
    mu       sync.RWMutex
}

func (r *ServiceRegistry) Register(serviceName, address string) error {
    // TODO: 註冊服務實例
}

func (r *ServiceRegistry) Discover(serviceName string) ([]string, error) {
    // TODO: 發現服務實例
}

func (r *ServiceRegistry) Deregister(serviceName, address string) error {
    // TODO: 註銷服務實例
}
```

---

## 9. 最佳實踐總結

### ✅ Do's
1. **使用攔截器實現橫切關注點**
2. **實現優雅的錯誤處理**
3. **使用流式 RPC 處理大量數據**
4. **實現消息重試和死信隊列**
5. **手動管理 Kafka Offset 確保可靠性**

### ❌ Don'ts
1. **不要在 gRPC 中傳輸大文件（使用流式）**
2. **不要忽略 Offset 提交失敗**
3. **不要在消息處理中執行長時間操作**
4. **不要忘記設置超時**
5. **不要在生產環境使用 `grpc.WithInsecure()`**

---

## 10. 延伸閱讀

- [gRPC Go Documentation](https://grpc.io/docs/languages/go/)
- [Protobuf Guide](https://developers.google.com/protocol-buffers)
- [Kafka Documentation](https://kafka.apache.org/documentation/)
- [Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)

---

**上一篇**: [Day 8 - 數據持久化與容器化](../03-Web開發篇/08-數據持久化與容器化.md)  
**下一篇**: [Day 10 - 可觀測性與生產部署](10-可觀測性與生產部署.md)
