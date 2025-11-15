# Project 03：gRPC 數據庫微服務

## 🎯 項目目標

構建一個**生產級別**的 gRPC 數據庫微服務，涵蓋：
- ✅ gRPC Server/Client 實現
- ✅ Protobuf 數據模型定義
- ✅ sqlx 高性能數據訪問
- ✅ 連接池管理
- ✅ gRPC 攔截器（日誌、認證、限流）
- ✅ 錯誤處理與狀態碼
- ✅ 健康檢查與反射
- ✅ 單元測試與集成測試

---

## 📁 項目結構

```
project-03-database-service/
├── api/
│   └── proto/
│       └── user/
│           └── v1/
│               └── user.proto         # Protobuf 定義
├── cmd/
│   ├── server/
│   │   └── main.go                   # gRPC Server 入口
│   └── client/
│       └── main.go                   # 測試客戶端
├── internal/
│   ├── domain/                       # 領域層
│   │   ├── user.go
│   │   └── errors.go
│   ├── repository/                   # 數據訪問層
│   │   ├── user_repository.go
│   │   └── postgres/
│   │       └── user_repo_impl.go
│   ├── service/                      # gRPC 服務實現
│   │   └── user_service.go
│   └── interceptor/                  # gRPC 攔截器
│       ├── logger.go
│       ├── recovery.go
│       └── auth.go
├── pkg/
│   ├── logger/
│   │   └── logger.go
│   └── database/
│       └── postgres.go
├── migrations/
│   ├── 001_create_users.up.sql
│   └── 001_create_users.down.sql
├── tests/
│   ├── integration/
│   │   └── user_service_test.go
│   └── unit/
│       └── repository_test.go
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── .env.example
├── Makefile
├── buf.gen.yaml                      # Buf 代碼生成配置
├── go.mod
└── README.md
```

---

## 🚀 核心功能實現

### 1. Protobuf 定義

**api/proto/user/v1/user.proto**
```protobuf
syntax = "proto3";

package user.v1;

option go_package = "project-03/gen/user/v1;userv1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

// User 用戶實體
message User {
  int64 id = 1;
  string username = 2;
  string email = 3;
  bool is_active = 4;
  google.protobuf.Timestamp created_at = 5;
  google.protobuf.Timestamp updated_at = 6;
}

// CreateUserRequest 創建用戶請求
message CreateUserRequest {
  string username = 1;
  string email = 2;
  string password = 3;
}

// CreateUserResponse 創建用戶響應
message CreateUserResponse {
  User user = 1;
}

// GetUserRequest 獲取用戶請求
message GetUserRequest {
  int64 id = 1;
}

// GetUserResponse 獲取用戶響應
message GetUserResponse {
  User user = 1;
}

// ListUsersRequest 列表請求
message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
}

// ListUsersResponse 列表響應
message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
  int32 page = 3;
  int32 page_size = 4;
}

// UpdateUserRequest 更新用戶請求
message UpdateUserRequest {
  int64 id = 1;
  optional string username = 2;
  optional string email = 3;
  optional bool is_active = 4;
}

// UpdateUserResponse 更新用戶響應
message UpdateUserResponse {
  User user = 1;
}

// DeleteUserRequest 刪除用戶請求
message DeleteUserRequest {
  int64 id = 1;
}

// UserService 用戶服務
service UserService {
  // CreateUser 創建用戶
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  
  // GetUser 獲取用戶
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  
  // ListUsers 列出用戶
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  
  // UpdateUser 更新用戶
  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
  
  // DeleteUser 刪除用戶
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
  
  // StreamUsers 流式獲取用戶 (Server Streaming)
  rpc StreamUsers(ListUsersRequest) returns (stream User);
}
```

---

### 2. Buf 配置（推薦使用 Buf 管理 Protobuf）

**buf.gen.yaml**
```yaml
version: v1
plugins:
  - plugin: buf.build/protocolbuffers/go
    out: gen
    opt:
      - paths=source_relative
  - plugin: buf.build/grpc/go
    out: gen
    opt:
      - paths=source_relative
```

**生成代碼命令**
```bash
# 安裝 buf
go install github.com/bufbuild/buf/cmd/buf@latest

# 生成 Go 代碼
buf generate api/proto

# 或使用 protoc
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       api/proto/user/v1/user.proto
```

---

### 3. 領域層

**internal/domain/user.go**
```go
package domain

import (
	"time"
)

// User 用戶領域實體
type User struct {
	ID        int64
	Username  string
	Email     string
	Password  string // 密碼哈希
	IsActive  bool
	CreatedAt time.Time
	UpdatedAt time.Time
}

// CreateUserDTO 創建用戶 DTO
type CreateUserDTO struct {
	Username string
	Email    string
	Password string
}

// UpdateUserDTO 更新用戶 DTO
type UpdateUserDTO struct {
	ID       int64
	Username *string
	Email    *string
	IsActive *bool
}

// Validate 驗證創建請求
func (dto *CreateUserDTO) Validate() error {
	if dto.Username == "" {
		return ErrInvalidInput
	}
	if dto.Email == "" {
		return ErrInvalidInput
	}
	if len(dto.Password) < 8 {
		return ErrInvalidInput
	}
	return nil
}
```

**internal/domain/errors.go**
```go
package domain

import (
	"errors"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

var (
	// ErrNotFound 資源不存在
	ErrNotFound = errors.New("resource not found")
	
	// ErrDuplicateEntry 重複條目
	ErrDuplicateEntry = errors.New("duplicate entry")
	
	// ErrInvalidInput 無效輸入
	ErrInvalidInput = errors.New("invalid input")
	
	// ErrInternal 內部錯誤
	ErrInternal = errors.New("internal error")
)

// ToGRPCError 將領域錯誤轉換為 gRPC 錯誤
func ToGRPCError(err error) error {
	switch {
	case errors.Is(err, ErrNotFound):
		return status.Error(codes.NotFound, err.Error())
	case errors.Is(err, ErrDuplicateEntry):
		return status.Error(codes.AlreadyExists, err.Error())
	case errors.Is(err, ErrInvalidInput):
		return status.Error(codes.InvalidArgument, err.Error())
	default:
		return status.Error(codes.Internal, "internal server error")
	}
}
```

---

### 4. Repository 層

**internal/repository/user_repository.go**
```go
package repository

import (
	"context"
	"project-03/internal/domain"
)

// UserRepository 用戶倉庫接口
type UserRepository interface {
	Create(ctx context.Context, user *domain.User) error
	GetByID(ctx context.Context, id int64) (*domain.User, error)
	GetByEmail(ctx context.Context, email string) (*domain.User, error)
	List(ctx context.Context, limit, offset int) ([]*domain.User, error)
	Count(ctx context.Context) (int, error)
	Update(ctx context.Context, user *domain.User) error
	Delete(ctx context.Context, id int64) error
}
```

**internal/repository/postgres/user_repo_impl.go**
```go
package postgres

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	
	"github.com/jmoiron/sqlx"
	"github.com/lib/pq"
	"project-03/internal/domain"
	"project-03/internal/repository"
)

type userRepository struct {
	db *sqlx.DB
}

// NewUserRepository 創建 PostgreSQL 倉庫
func NewUserRepository(db *sqlx.DB) repository.UserRepository {
	return &userRepository{db: db}
}

func (r *userRepository) Create(ctx context.Context, user *domain.User) error {
	query := `
		INSERT INTO users (username, email, password_hash, is_active, created_at, updated_at)
		VALUES ($1, $2, $3, $4, $5, $6)
		RETURNING id
	`
	
	err := r.db.QueryRowContext(
		ctx, query,
		user.Username, user.Email, user.Password, user.IsActive,
		user.CreatedAt, user.UpdatedAt,
	).Scan(&user.ID)
	
	if err != nil {
		if pqErr, ok := err.(*pq.Error); ok {
			if pqErr.Code == "23505" { // unique_violation
				return domain.ErrDuplicateEntry
			}
		}
		return fmt.Errorf("create user: %w", err)
	}
	
	return nil
}

func (r *userRepository) GetByID(ctx context.Context, id int64) (*domain.User, error) {
	var user domain.User
	query := `
		SELECT id, username, email, password_hash, is_active, created_at, updated_at
		FROM users
		WHERE id = $1
	`
	
	err := r.db.GetContext(ctx, &user, query, id)
	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, domain.ErrNotFound
		}
		return nil, fmt.Errorf("get user: %w", err)
	}
	
	return &user, nil
}

func (r *userRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
	var user domain.User
	query := `
		SELECT id, username, email, password_hash, is_active, created_at, updated_at
		FROM users
		WHERE email = $1
	`
	
	err := r.db.GetContext(ctx, &user, query, email)
	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, domain.ErrNotFound
		}
		return nil, fmt.Errorf("get user by email: %w", err)
	}
	
	return &user, nil
}

func (r *userRepository) List(ctx context.Context, limit, offset int) ([]*domain.User, error) {
	var users []*domain.User
	query := `
		SELECT id, username, email, is_active, created_at, updated_at
		FROM users
		ORDER BY created_at DESC
		LIMIT $1 OFFSET $2
	`
	
	err := r.db.SelectContext(ctx, &users, query, limit, offset)
	if err != nil {
		return nil, fmt.Errorf("list users: %w", err)
	}
	
	return users, nil
}

func (r *userRepository) Count(ctx context.Context) (int, error) {
	var count int
	query := "SELECT COUNT(*) FROM users"
	
	err := r.db.GetContext(ctx, &count, query)
	if err != nil {
		return 0, fmt.Errorf("count users: %w", err)
	}
	
	return count, nil
}

func (r *userRepository) Update(ctx context.Context, user *domain.User) error {
	query := `
		UPDATE users
		SET username = $1, email = $2, is_active = $3, updated_at = $4
		WHERE id = $5
	`
	
	result, err := r.db.ExecContext(
		ctx, query,
		user.Username, user.Email, user.IsActive, user.UpdatedAt, user.ID,
	)
	if err != nil {
		if pqErr, ok := err.(*pq.Error); ok {
			if pqErr.Code == "23505" {
				return domain.ErrDuplicateEntry
			}
		}
		return fmt.Errorf("update user: %w", err)
	}
	
	rows, _ := result.RowsAffected()
	if rows == 0 {
		return domain.ErrNotFound
	}
	
	return nil
}

func (r *userRepository) Delete(ctx context.Context, id int64) error {
	query := "DELETE FROM users WHERE id = $1"
	
	result, err := r.db.ExecContext(ctx, query, id)
	if err != nil {
		return fmt.Errorf("delete user: %w", err)
	}
	
	rows, _ := result.RowsAffected()
	if rows == 0 {
		return domain.ErrNotFound
	}
	
	return nil
}
```

---

### 5. gRPC Service 實現

**internal/service/user_service.go**
```go
package service

import (
	"context"
	"time"
	
	"golang.org/x/crypto/bcrypt"
	"google.golang.org/protobuf/types/known/emptypb"
	"google.golang.org/protobuf/types/known/timestamppb"
	
	userv1 "project-03/gen/user/v1"
	"project-03/internal/domain"
	"project-03/internal/repository"
	"project-03/pkg/logger"
)

// UserServiceServer gRPC 用戶服務實現
type UserServiceServer struct {
	userv1.UnimplementedUserServiceServer
	repo   repository.UserRepository
	logger *logger.Logger
}

// NewUserServiceServer 創建服務
func NewUserServiceServer(repo repository.UserRepository, logger *logger.Logger) *UserServiceServer {
	return &UserServiceServer{
		repo:   repo,
		logger: logger,
	}
}

// CreateUser 創建用戶
func (s *UserServiceServer) CreateUser(ctx context.Context, req *userv1.CreateUserRequest) (*userv1.CreateUserResponse, error) {
	s.logger.Info("creating user", "username", req.Username)
	
	// 1. 驗證輸入
	dto := &domain.CreateUserDTO{
		Username: req.Username,
		Email:    req.Email,
		Password: req.Password,
	}
	if err := dto.Validate(); err != nil {
		return nil, domain.ToGRPCError(err)
	}
	
	// 2. 檢查郵箱是否已存在
	_, err := s.repo.GetByEmail(ctx, req.Email)
	if err == nil {
		return nil, domain.ToGRPCError(domain.ErrDuplicateEntry)
	}
	if err != domain.ErrNotFound {
		s.logger.Error("failed to check email", "error", err)
		return nil, domain.ToGRPCError(domain.ErrInternal)
	}
	
	// 3. 哈希密碼
	hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
	if err != nil {
		s.logger.Error("failed to hash password", "error", err)
		return nil, domain.ToGRPCError(domain.ErrInternal)
	}
	
	// 4. 創建用戶
	user := &domain.User{
		Username:  req.Username,
		Email:     req.Email,
		Password:  string(hashedPassword),
		IsActive:  true,
		CreatedAt: time.Now(),
		UpdatedAt: time.Now(),
	}
	
	if err := s.repo.Create(ctx, user); err != nil {
		s.logger.Error("failed to create user", "error", err)
		return nil, domain.ToGRPCError(err)
	}
	
	s.logger.Info("user created", "user_id", user.ID)
	
	return &userv1.CreateUserResponse{
		User: s.toProtoUser(user),
	}, nil
}

// GetUser 獲取用戶
func (s *UserServiceServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
	user, err := s.repo.GetByID(ctx, req.Id)
	if err != nil {
		return nil, domain.ToGRPCError(err)
	}
	
	return &userv1.GetUserResponse{
		User: s.toProtoUser(user),
	}, nil
}

// ListUsers 列出用戶
func (s *UserServiceServer) ListUsers(ctx context.Context, req *userv1.ListUsersRequest) (*userv1.ListUsersResponse, error) {
	// 參數驗證
	page := req.Page
	if page < 1 {
		page = 1
	}
	
	pageSize := req.PageSize
	if pageSize < 1 || pageSize > 100 {
		pageSize = 20
	}
	
	offset := int((page - 1) * pageSize)
	limit := int(pageSize)
	
	// 獲取用戶列表
	users, err := s.repo.List(ctx, limit, offset)
	if err != nil {
		s.logger.Error("failed to list users", "error", err)
		return nil, domain.ToGRPCError(err)
	}
	
	// 獲取總數
	total, err := s.repo.Count(ctx)
	if err != nil {
		s.logger.Error("failed to count users", "error", err)
		return nil, domain.ToGRPCError(err)
	}
	
	// 轉換為 Protobuf
	protoUsers := make([]*userv1.User, len(users))
	for i, user := range users {
		protoUsers[i] = s.toProtoUser(user)
	}
	
	return &userv1.ListUsersResponse{
		Users:    protoUsers,
		Total:    int32(total),
		Page:     page,
		PageSize: pageSize,
	}, nil
}

// UpdateUser 更新用戶
func (s *UserServiceServer) UpdateUser(ctx context.Context, req *userv1.UpdateUserRequest) (*userv1.UpdateUserResponse, error) {
	// 1. 獲取現有用戶
	user, err := s.repo.GetByID(ctx, req.Id)
	if err != nil {
		return nil, domain.ToGRPCError(err)
	}
	
	// 2. 更新字段
	if req.Username != nil {
		user.Username = *req.Username
	}
	if req.Email != nil {
		user.Email = *req.Email
	}
	if req.IsActive != nil {
		user.IsActive = *req.IsActive
	}
	
	user.UpdatedAt = time.Now()
	
	// 3. 保存更新
	if err := s.repo.Update(ctx, user); err != nil {
		s.logger.Error("failed to update user", "error", err)
		return nil, domain.ToGRPCError(err)
	}
	
	s.logger.Info("user updated", "user_id", user.ID)
	
	return &userv1.UpdateUserResponse{
		User: s.toProtoUser(user),
	}, nil
}

// DeleteUser 刪除用戶
func (s *UserServiceServer) DeleteUser(ctx context.Context, req *userv1.DeleteUserRequest) (*emptypb.Empty, error) {
	if err := s.repo.Delete(ctx, req.Id); err != nil {
		s.logger.Error("failed to delete user", "error", err)
		return nil, domain.ToGRPCError(err)
	}
	
	s.logger.Info("user deleted", "user_id", req.Id)
	
	return &emptypb.Empty{}, nil
}

// StreamUsers 流式返回用戶（Server Streaming）
func (s *UserServiceServer) StreamUsers(req *userv1.ListUsersRequest, stream userv1.UserService_StreamUsersServer) error {
	// 獲取所有用戶
	users, err := s.repo.List(stream.Context(), 1000, 0)
	if err != nil {
		return domain.ToGRPCError(err)
	}
	
	// 逐個發送
	for _, user := range users {
		if err := stream.Send(s.toProtoUser(user)); err != nil {
			s.logger.Error("failed to send user", "error", err)
			return err
		}
	}
	
	return nil
}

// toProtoUser 轉換為 Protobuf User
func (s *UserServiceServer) toProtoUser(user *domain.User) *userv1.User {
	return &userv1.User{
		Id:        user.ID,
		Username:  user.Username,
		Email:     user.Email,
		IsActive:  user.IsActive,
		CreatedAt: timestamppb.New(user.CreatedAt),
		UpdatedAt: timestamppb.New(user.UpdatedAt),
	}
}
```

---

### 6. gRPC 攔截器

**internal/interceptor/logger.go**
```go
package interceptor

import (
	"context"
	"time"
	
	"google.golang.org/grpc"
	"project-03/pkg/logger"
)

// Logger 日誌攔截器
func Logger(log *logger.Logger) grpc.UnaryServerInterceptor {
	return func(
		ctx context.Context,
		req interface{},
		info *grpc.UnaryServerInfo,
		handler grpc.UnaryHandler,
	) (interface{}, error) {
		start := time.Now()
		
		// 執行 RPC
		resp, err := handler(ctx, req)
		
		// 記錄日誌
		log.Info("grpc request",
			"method", info.FullMethod,
			"duration", time.Since(start).Milliseconds(),
			"error", err,
		)
		
		return resp, err
	}
}
```

**internal/interceptor/recovery.go**
```go
package interceptor

import (
	"context"
	
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"project-03/pkg/logger"
)

// Recovery panic 恢復攔截器
func Recovery(log *logger.Logger) grpc.UnaryServerInterceptor {
	return func(
		ctx context.Context,
		req interface{},
		info *grpc.UnaryServerInfo,
		handler grpc.UnaryHandler,
	) (resp interface{}, err error) {
		defer func() {
			if r := recover(); r != nil {
				log.Error("grpc panic recovered",
					"method", info.FullMethod,
					"panic", r,
				)
				err = status.Error(codes.Internal, "internal server error")
			}
		}()
		
		return handler(ctx, req)
	}
}
```

**internal/interceptor/auth.go**
```go
package interceptor

import (
	"context"
	"strings"
	
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
)

// Auth 認證攔截器（簡單 Token 驗證）
func Auth() grpc.UnaryServerInterceptor {
	return func(
		ctx context.Context,
		req interface{},
		info *grpc.UnaryServerInfo,
		handler grpc.UnaryHandler,
	) (interface{}, error) {
		// 跳過健康檢查
		if strings.Contains(info.FullMethod, "Health") {
			return handler(ctx, req)
		}
		
		// 獲取 metadata
		md, ok := metadata.FromIncomingContext(ctx)
		if !ok {
			return nil, status.Error(codes.Unauthenticated, "missing metadata")
		}
		
		// 檢查 Authorization header
		tokens := md.Get("authorization")
		if len(tokens) == 0 {
			return nil, status.Error(codes.Unauthenticated, "missing authorization token")
		}
		
		token := tokens[0]
		if !isValidToken(token) {
			return nil, status.Error(codes.Unauthenticated, "invalid token")
		}
		
		return handler(ctx, req)
	}
}

func isValidToken(token string) bool {
	// 簡單驗證（生產環境應使用 JWT）
	return token == "Bearer secret-token"
}
```

---

### 7. Server 主程序

**cmd/server/main.go**
```go
package main

import (
	"context"
	"fmt"
	"log"
	"net"
	"os"
	"os/signal"
	"syscall"
	"time"
	
	"github.com/jmoiron/sqlx"
	_ "github.com/lib/pq"
	"google.golang.org/grpc"
	"google.golang.org/grpc/health"
	"google.golang.org/grpc/health/grpc_health_v1"
	"google.golang.org/grpc/reflection"
	
	userv1 "project-03/gen/user/v1"
	"project-03/internal/interceptor"
	"project-03/internal/repository/postgres"
	"project-03/internal/service"
	"project-03/pkg/logger"
)

func main() {
	// 1. 初始化日誌
	appLogger := logger.New("development")
	appLogger.Info("starting grpc server")
	
	// 2. 連接數據庫
	dsn := "host=localhost port=5432 user=postgres password=secret dbname=userdb sslmode=disable"
	db, err := sqlx.Connect("postgres", dsn)
	if err != nil {
		appLogger.Error("failed to connect database", "error", err)
		os.Exit(1)
	}
	defer db.Close()
	
	// 配置連接池
	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(5)
	db.SetConnMaxLifetime(5 * time.Minute)
	
	appLogger.Info("database connected")
	
	// 3. 初始化依賴
	userRepo := postgres.NewUserRepository(db)
	userService := service.NewUserServiceServer(userRepo, appLogger)
	
	// 4. 創建 gRPC Server（鏈式攔截器）
	grpcServer := grpc.NewServer(
		grpc.ChainUnaryInterceptor(
			interceptor.Recovery(appLogger),
			interceptor.Logger(appLogger),
			// interceptor.Auth(), // 可選：啟用認證
		),
	)
	
	// 5. 註冊服務
	userv1.RegisterUserServiceServer(grpcServer, userService)
	
	// 6. 註冊健康檢查
	healthServer := health.NewServer()
	grpc_health_v1.RegisterHealthServer(grpcServer, healthServer)
	healthServer.SetServingStatus("", grpc_health_v1.HealthCheckResponse_SERVING)
	
	// 7. 註冊反射（用於 grpcurl）
	reflection.Register(grpcServer)
	
	// 8. 啟動服務器
	listener, err := net.Listen("tcp", ":50051")
	if err != nil {
		appLogger.Error("failed to listen", "error", err)
		os.Exit(1)
	}
	
	go func() {
		appLogger.Info("grpc server listening", "address", ":50051")
		if err := grpcServer.Serve(listener); err != nil {
			appLogger.Error("grpc server error", "error", err)
			os.Exit(1)
		}
	}()
	
	// 9. Graceful Shutdown
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	
	<-quit
	appLogger.Info("shutting down server...")
	
	grpcServer.GracefulStop()
	
	appLogger.Info("server stopped")
}
```

---

### 8. Client 示例

**cmd/client/main.go**
```go
package main

import (
	"context"
	"log"
	"time"
	
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	
	userv1 "project-03/gen/user/v1"
)

func main() {
	// 1. 連接 gRPC Server
	conn, err := grpc.Dial("localhost:50051",
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
	if err != nil {
		log.Fatalf("failed to connect: %v", err)
	}
	defer conn.Close()
	
	client := userv1.NewUserServiceClient(conn)
	
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	
	// 2. 創建用戶
	createResp, err := client.CreateUser(ctx, &userv1.CreateUserRequest{
		Username: "alice",
		Email:    "alice@example.com",
		Password: "password123",
	})
	if err != nil {
		log.Fatalf("CreateUser failed: %v", err)
	}
	log.Printf("Created user: %v", createResp.User)
	
	// 3. 獲取用戶
	getResp, err := client.GetUser(ctx, &userv1.GetUserRequest{
		Id: createResp.User.Id,
	})
	if err != nil {
		log.Fatalf("GetUser failed: %v", err)
	}
	log.Printf("Got user: %v", getResp.User)
	
	// 4. 列出用戶
	listResp, err := client.ListUsers(ctx, &userv1.ListUsersRequest{
		Page:     1,
		PageSize: 10,
	})
	if err != nil {
		log.Fatalf("ListUsers failed: %v", err)
	}
	log.Printf("Total users: %d", listResp.Total)
	
	// 5. 流式獲取用戶
	stream, err := client.StreamUsers(ctx, &userv1.ListUsersRequest{})
	if err != nil {
		log.Fatalf("StreamUsers failed: %v", err)
	}
	
	for {
		user, err := stream.Recv()
		if err != nil {
			break
		}
		log.Printf("Streamed user: %s", user.Username)
	}
}
```

---

### 9. Docker 配置

**docker/docker-compose.yml**
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: grpc_user_db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: userdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  grpc_server:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    container_name: grpc_user_service
    ports:
      - "50051:50051"
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_USER: postgres
      DB_PASSWORD: secret
      DB_NAME: userdb
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

volumes:
  postgres_data:
```

---

### 10. Makefile

```makefile
.PHONY: help proto run-server run-client test docker-up docker-down

help: ## 顯示幫助
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

proto: ## 生成 Protobuf 代碼
	buf generate api/proto

run-server: ## 運行 gRPC Server
	go run cmd/server/main.go

run-client: ## 運行測試客戶端
	go run cmd/client/main.go

test: ## 運行測試
	go test -v -race ./tests/...

docker-up: ## 啟動 Docker 容器
	docker-compose -f docker/docker-compose.yml up -d

docker-down: ## 停止 Docker 容器
	docker-compose -f docker/docker-compose.yml down

grpcurl-health: ## 測試健康檢查
	grpcurl -plaintext localhost:50051 grpc.health.v1.Health/Check

grpcurl-list: ## 列出服務
	grpcurl -plaintext localhost:50051 list

migrate-up: ## 執行數據庫遷移
	migrate -path migrations -database "postgresql://postgres:secret@localhost:5432/userdb?sslmode=disable" up

lint: ## 代碼檢查
	golangci-lint run
```

---

## 🎓 學習要點

### 1. Protobuf 設計
- **消息定義**：使用 `message` 定義數據模型
- **服務定義**：使用 `service` 定義 RPC 接口
- **版本管理**：通過 package 路徑管理 API 版本
- **可選字段**：使用 `optional` 標記可選字段

### 2. gRPC 模式
- **Unary RPC**：單請求-單響應（CreateUser, GetUser）
- **Server Streaming**：單請求-流響應（StreamUsers）
- **Client Streaming**：流請求-單響應
- **Bidirectional Streaming**：雙向流

### 3. 攔截器（Interceptor）
- **日誌記錄**：記錄每個 RPC 請求
- **錯誤恢復**：捕獲 panic 並返回錯誤
- **認證授權**：驗證 Token
- **限流**：控制請求速率

### 4. 錯誤處理
- **gRPC 狀態碼**：使用 `codes` 包定義錯誤類型
- **錯誤映射**：將領域錯誤轉換為 gRPC 錯誤
- **錯誤詳情**：使用 `status.Error` 添加錯誤描述

### 5. 健康檢查與反射
- **Health Check**：實現 gRPC 健康檢查協議
- **Reflection**：支持 grpcurl 等工具動態調用

---

## 🚀 快速開始

```bash
# 1. 生成 Protobuf 代碼
make proto

# 2. 啟動數據庫
make docker-up

# 3. 執行數據庫遷移
make migrate-up

# 4. 運行 Server
make run-server

# 5. 測試（新終端）
make run-client

# 6. 使用 grpcurl 測試
make grpcurl-health
grpcurl -plaintext -d '{"username":"bob","email":"bob@test.com","password":"password123"}' \
  localhost:50051 user.v1.UserService/CreateUser
```

---

## 📝 擴展方向

- [ ] 添加 JWT 認證
- [ ] 實現 Client Streaming（批量創建用戶）
- [ ] 實現 Bidirectional Streaming（實時聊天）
- [ ] 添加 TLS 加密
- [ ] 集成 OpenTelemetry 追蹤
- [ ] 添加 Prometheus 指標
- [ ] 實現 gRPC Gateway（HTTP to gRPC）
- [ ] 添加限流與熔斷
