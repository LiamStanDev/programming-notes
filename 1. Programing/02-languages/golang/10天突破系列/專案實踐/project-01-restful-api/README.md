# Project 01：企業級 RESTful API 服務

## 🎯 項目目標（Senior Engineer Level）

構建一個**生產級別**的用戶管理 RESTful API，涵蓋：
- ✅ Clean Architecture 分層設計
- ✅ Fiber + sqlx 技術棧
- ✅ 完整的錯誤處理與驗證
- ✅ 結構化日誌與可觀測性
- ✅ Graceful Shutdown
- ✅ 單元測試與集成測試
- ✅ Docker 容器化部署
- ✅ API 文檔自動生成

---

## 📁 項目結構（Clean Architecture）

```
project-01-restful-api/
├── cmd/
│   └── api/
│       └── main.go                 # 應用入口
├── internal/
│   ├── domain/                     # 領域層（業務實體）
│   │   ├── user.go                 # User 實體
│   │   └── errors.go               # 領域錯誤
│   ├── repository/                 # 數據訪問層
│   │   ├── user_repository.go      # User Repository 接口
│   │   └── postgres/
│   │       └── user_repo_impl.go   # PostgreSQL 實現
│   ├── service/                    # 業務邏輯層
│   │   └── user_service.go         # User Service
│   ├── handler/                    # HTTP 處理層
│   │   └── user_handler.go         # User Handler
│   ├── middleware/                 # 中間件
│   │   ├── logger.go               # 日誌中間件
│   │   ├── recovery.go             # 恢復中間件
│   │   ├── rate_limiter.go         # 限流中間件
│   │   └── request_id.go           # 請求追蹤
│   └── config/                     # 配置管理
│       └── config.go
├── pkg/                            # 可重用公共庫
│   ├── logger/                     # 結構化日誌
│   │   └── logger.go
│   ├── validator/                  # 數據驗證
│   │   └── validator.go
│   └── response/                   # 統一響應格式
│       └── response.go
├── migrations/                     # 數據庫遷移
│   ├── 001_create_users_table.up.sql
│   └── 001_create_users_table.down.sql
├── tests/                          # 測試
│   ├── integration/
│   │   └── user_api_test.go
│   └── unit/
│       └── user_service_test.go
├── docker/
│   ├── Dockerfile                  # 多階段構建
│   └── docker-compose.yml
├── .env.example
├── Makefile
├── go.mod
└── README.md
```

---

## 🚀 核心功能實現

### 1. 領域層（Domain Layer）

**internal/domain/user.go**
```go
package domain

import (
	"errors"
	"time"
)

// User 領域實體
type User struct {
	ID        int       `json:"id" db:"id"`
	Username  string    `json:"username" db:"username"`
	Email     string    `json:"email" db:"email"`
	Password  string    `json:"-" db:"password_hash"` // 不序列化到 JSON
	IsActive  bool      `json:"is_active" db:"is_active"`
	CreatedAt time.Time `json:"created_at" db:"created_at"`
	UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

// CreateUserRequest DTO
type CreateUserRequest struct {
	Username string `json:"username" validate:"required,min=3,max=50"`
	Email    string `json:"email" validate:"required,email"`
	Password string `json:"password" validate:"required,min=8"`
}

// UpdateUserRequest DTO
type UpdateUserRequest struct {
	Username *string `json:"username,omitempty" validate:"omitempty,min=3,max=50"`
	Email    *string `json:"email,omitempty" validate:"omitempty,email"`
	IsActive *bool   `json:"is_active,omitempty"`
}

// UserResponse 視圖模型（隱藏敏感信息）
type UserResponse struct {
	ID        int       `json:"id"`
	Username  string    `json:"username"`
	Email     string    `json:"email"`
	IsActive  bool      `json:"is_active"`
	CreatedAt time.Time `json:"created_at"`
}

// ToResponse 轉換為響應模型
func (u *User) ToResponse() *UserResponse {
	return &UserResponse{
		ID:        u.ID,
		Username:  u.Username,
		Email:     u.Email,
		IsActive:  u.IsActive,
		CreatedAt: u.CreatedAt,
	}
}
```

**internal/domain/errors.go**
```go
package domain

import "errors"

var (
	// ErrNotFound 資源不存在
	ErrNotFound = errors.New("resource not found")
	
	// ErrDuplicateEntry 重複條目
	ErrDuplicateEntry = errors.New("duplicate entry")
	
	// ErrInvalidInput 無效輸入
	ErrInvalidInput = errors.New("invalid input")
	
	// ErrUnauthorized 未授權
	ErrUnauthorized = errors.New("unauthorized")
	
	// ErrForbidden 禁止訪問
	ErrForbidden = errors.New("forbidden")
)
```

---

### 2. Repository 層

**internal/repository/user_repository.go**
```go
package repository

import (
	"context"
	"project-01/internal/domain"
)

// UserRepository 用戶數據訪問接口
type UserRepository interface {
	Create(ctx context.Context, user *domain.User) (int, error)
	GetByID(ctx context.Context, id int) (*domain.User, error)
	GetByEmail(ctx context.Context, email string) (*domain.User, error)
	List(ctx context.Context, limit, offset int) ([]*domain.User, error)
	Update(ctx context.Context, user *domain.User) error
	Delete(ctx context.Context, id int) error
	Count(ctx context.Context) (int, error)
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
	"project-01/internal/domain"
	"project-01/internal/repository"
)

type userRepository struct {
	db *sqlx.DB
}

// NewUserRepository 創建 PostgreSQL 用戶倉庫
func NewUserRepository(db *sqlx.DB) repository.UserRepository {
	return &userRepository{db: db}
}

func (r *userRepository) Create(ctx context.Context, user *domain.User) (int, error) {
	query := `
		INSERT INTO users (username, email, password_hash, is_active, created_at, updated_at)
		VALUES ($1, $2, $3, $4, $5, $6)
		RETURNING id
	`
	
	var id int
	err := r.db.QueryRowContext(
		ctx, query,
		user.Username, user.Email, user.Password, user.IsActive,
		user.CreatedAt, user.UpdatedAt,
	).Scan(&id)
	
	if err != nil {
		// 處理唯一約束衝突
		if pqErr, ok := err.(*pq.Error); ok {
			if pqErr.Code == "23505" { // unique_violation
				return 0, domain.ErrDuplicateEntry
			}
		}
		return 0, fmt.Errorf("create user: %w", err)
	}
	
	return id, nil
}

func (r *userRepository) GetByID(ctx context.Context, id int) (*domain.User, error) {
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
		return nil, fmt.Errorf("get user by id: %w", err)
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
	
	rows, err := result.RowsAffected()
	if err != nil {
		return fmt.Errorf("get rows affected: %w", err)
	}
	
	if rows == 0 {
		return domain.ErrNotFound
	}
	
	return nil
}

func (r *userRepository) Delete(ctx context.Context, id int) error {
	query := "DELETE FROM users WHERE id = $1"
	
	result, err := r.db.ExecContext(ctx, query, id)
	if err != nil {
		return fmt.Errorf("delete user: %w", err)
	}
	
	rows, err := result.RowsAffected()
	if err != nil {
		return fmt.Errorf("get rows affected: %w", err)
	}
	
	if rows == 0 {
		return domain.ErrNotFound
	}
	
	return nil
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
```

---

### 3. Service 層（業務邏輯）

**internal/service/user_service.go**
```go
package service

import (
	"context"
	"fmt"
	"time"
	
	"golang.org/x/crypto/bcrypt"
	"project-01/internal/domain"
	"project-01/internal/repository"
	"project-01/pkg/logger"
)

type UserService struct {
	repo   repository.UserRepository
	logger *logger.Logger
}

func NewUserService(repo repository.UserRepository, logger *logger.Logger) *UserService {
	return &UserService{
		repo:   repo,
		logger: logger,
	}
}

// CreateUser 創建用戶（包含密碼哈希）
func (s *UserService) CreateUser(ctx context.Context, req *domain.CreateUserRequest) (*domain.UserResponse, error) {
	// 1. 檢查郵箱是否已存在
	_, err := s.repo.GetByEmail(ctx, req.Email)
	if err == nil {
		return nil, domain.ErrDuplicateEntry
	}
	if err != domain.ErrNotFound {
		s.logger.Error("failed to check email existence", "error", err)
		return nil, fmt.Errorf("check email: %w", err)
	}
	
	// 2. 哈希密碼
	hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
	if err != nil {
		return nil, fmt.Errorf("hash password: %w", err)
	}
	
	// 3. 創建用戶實體
	user := &domain.User{
		Username:  req.Username,
		Email:     req.Email,
		Password:  string(hashedPassword),
		IsActive:  true,
		CreatedAt: time.Now(),
		UpdatedAt: time.Now(),
	}
	
	// 4. 持久化
	id, err := s.repo.Create(ctx, user)
	if err != nil {
		s.logger.Error("failed to create user", "error", err)
		return nil, err
	}
	
	user.ID = id
	s.logger.Info("user created successfully", "user_id", id, "username", user.Username)
	
	return user.ToResponse(), nil
}

// GetUser 獲取用戶詳情
func (s *UserService) GetUser(ctx context.Context, id int) (*domain.UserResponse, error) {
	user, err := s.repo.GetByID(ctx, id)
	if err != nil {
		return nil, err
	}
	
	return user.ToResponse(), nil
}

// ListUsers 獲取用戶列表（分頁）
func (s *UserService) ListUsers(ctx context.Context, page, pageSize int) ([]*domain.UserResponse, int, error) {
	if page < 1 {
		page = 1
	}
	if pageSize < 1 || pageSize > 100 {
		pageSize = 20
	}
	
	offset := (page - 1) * pageSize
	
	users, err := s.repo.List(ctx, pageSize, offset)
	if err != nil {
		return nil, 0, err
	}
	
	total, err := s.repo.Count(ctx)
	if err != nil {
		return nil, 0, err
	}
	
	responses := make([]*domain.UserResponse, len(users))
	for i, user := range users {
		responses[i] = user.ToResponse()
	}
	
	return responses, total, nil
}

// UpdateUser 更新用戶信息
func (s *UserService) UpdateUser(ctx context.Context, id int, req *domain.UpdateUserRequest) (*domain.UserResponse, error) {
	user, err := s.repo.GetByID(ctx, id)
	if err != nil {
		return nil, err
	}
	
	// 選擇性更新字段
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
	
	if err := s.repo.Update(ctx, user); err != nil {
		s.logger.Error("failed to update user", "user_id", id, "error", err)
		return nil, err
	}
	
	s.logger.Info("user updated successfully", "user_id", id)
	return user.ToResponse(), nil
}

// DeleteUser 刪除用戶
func (s *UserService) DeleteUser(ctx context.Context, id int) error {
	if err := s.repo.Delete(ctx, id); err != nil {
		s.logger.Error("failed to delete user", "user_id", id, "error", err)
		return err
	}
	
	s.logger.Info("user deleted successfully", "user_id", id)
	return nil
}
```

---

### 4. Handler 層（HTTP 處理）

**internal/handler/user_handler.go**
```go
package handler

import (
	"errors"
	"strconv"
	
	"github.com/gofiber/fiber/v2"
	"project-01/internal/domain"
	"project-01/internal/service"
	"project-01/pkg/response"
	"project-01/pkg/validator"
)

type UserHandler struct {
	service   *service.UserService
	validator *validator.Validator
}

func NewUserHandler(service *service.UserService, validator *validator.Validator) *UserHandler {
	return &UserHandler{
		service:   service,
		validator: validator,
	}
}

// CreateUser godoc
// @Summary 創建用戶
// @Tags users
// @Accept json
// @Produce json
// @Param request body domain.CreateUserRequest true "Create User Request"
// @Success 201 {object} response.Response{data=domain.UserResponse}
// @Failure 400 {object} response.Response
// @Failure 409 {object} response.Response
// @Router /api/v1/users [post]
func (h *UserHandler) CreateUser(c *fiber.Ctx) error {
	var req domain.CreateUserRequest
	
	if err := c.BodyParser(&req); err != nil {
		return response.Error(c, fiber.StatusBadRequest, "invalid request body", err)
	}
	
	if err := h.validator.Validate(&req); err != nil {
		return response.ValidationError(c, err)
	}
	
	user, err := h.service.CreateUser(c.Context(), &req)
	if err != nil {
		if errors.Is(err, domain.ErrDuplicateEntry) {
			return response.Error(c, fiber.StatusConflict, "email already exists", err)
		}
		return response.Error(c, fiber.StatusInternalServerError, "failed to create user", err)
	}
	
	return response.Success(c, fiber.StatusCreated, "user created successfully", user)
}

// GetUser godoc
// @Summary 獲取用戶詳情
// @Tags users
// @Produce json
// @Param id path int true "User ID"
// @Success 200 {object} response.Response{data=domain.UserResponse}
// @Failure 404 {object} response.Response
// @Router /api/v1/users/{id} [get]
func (h *UserHandler) GetUser(c *fiber.Ctx) error {
	id, err := strconv.Atoi(c.Params("id"))
	if err != nil {
		return response.Error(c, fiber.StatusBadRequest, "invalid user id", err)
	}
	
	user, err := h.service.GetUser(c.Context(), id)
	if err != nil {
		if errors.Is(err, domain.ErrNotFound) {
			return response.Error(c, fiber.StatusNotFound, "user not found", err)
		}
		return response.Error(c, fiber.StatusInternalServerError, "failed to get user", err)
	}
	
	return response.Success(c, fiber.StatusOK, "success", user)
}

// ListUsers godoc
// @Summary 獲取用戶列表
// @Tags users
// @Produce json
// @Param page query int false "Page number" default(1)
// @Param page_size query int false "Page size" default(20)
// @Success 200 {object} response.PaginatedResponse{data=[]domain.UserResponse}
// @Router /api/v1/users [get]
func (h *UserHandler) ListUsers(c *fiber.Ctx) error {
	page := c.QueryInt("page", 1)
	pageSize := c.QueryInt("page_size", 20)
	
	users, total, err := h.service.ListUsers(c.Context(), page, pageSize)
	if err != nil {
		return response.Error(c, fiber.StatusInternalServerError, "failed to list users", err)
	}
	
	return response.Paginated(c, fiber.StatusOK, users, total, page, pageSize)
}

// UpdateUser godoc
// @Summary 更新用戶
// @Tags users
// @Accept json
// @Produce json
// @Param id path int true "User ID"
// @Param request body domain.UpdateUserRequest true "Update User Request"
// @Success 200 {object} response.Response{data=domain.UserResponse}
// @Failure 404 {object} response.Response
// @Router /api/v1/users/{id} [put]
func (h *UserHandler) UpdateUser(c *fiber.Ctx) error {
	id, err := strconv.Atoi(c.Params("id"))
	if err != nil {
		return response.Error(c, fiber.StatusBadRequest, "invalid user id", err)
	}
	
	var req domain.UpdateUserRequest
	if err := c.BodyParser(&req); err != nil {
		return response.Error(c, fiber.StatusBadRequest, "invalid request body", err)
	}
	
	if err := h.validator.Validate(&req); err != nil {
		return response.ValidationError(c, err)
	}
	
	user, err := h.service.UpdateUser(c.Context(), id, &req)
	if err != nil {
		if errors.Is(err, domain.ErrNotFound) {
			return response.Error(c, fiber.StatusNotFound, "user not found", err)
		}
		return response.Error(c, fiber.StatusInternalServerError, "failed to update user", err)
	}
	
	return response.Success(c, fiber.StatusOK, "user updated successfully", user)
}

// DeleteUser godoc
// @Summary 刪除用戶
// @Tags users
// @Param id path int true "User ID"
// @Success 204
// @Failure 404 {object} response.Response
// @Router /api/v1/users/{id} [delete]
func (h *UserHandler) DeleteUser(c *fiber.Ctx) error {
	id, err := strconv.Atoi(c.Params("id"))
	if err != nil {
		return response.Error(c, fiber.StatusBadRequest, "invalid user id", err)
	}
	
	if err := h.service.DeleteUser(c.Context(), id); err != nil {
		if errors.Is(err, domain.ErrNotFound) {
			return response.Error(c, fiber.StatusNotFound, "user not found", err)
		}
		return response.Error(c, fiber.StatusInternalServerError, "failed to delete user", err)
	}
	
	return c.SendStatus(fiber.StatusNoContent)
}

// RegisterRoutes 註冊路由
func (h *UserHandler) RegisterRoutes(router fiber.Router) {
	users := router.Group("/users")
	{
		users.Post("/", h.CreateUser)
		users.Get("/", h.ListUsers)
		users.Get("/:id", h.GetUser)
		users.Put("/:id", h.UpdateUser)
		users.Delete("/:id", h.DeleteUser)
	}
}
```

---

### 5. 公共庫（pkg）

**pkg/response/response.go**
```go
package response

import (
	"github.com/gofiber/fiber/v2"
)

type Response struct {
	Success bool        `json:"success"`
	Message string      `json:"message"`
	Data    interface{} `json:"data,omitempty"`
	Error   string      `json:"error,omitempty"`
}

type PaginatedResponse struct {
	Success    bool        `json:"success"`
	Message    string      `json:"message"`
	Data       interface{} `json:"data"`
	Pagination Pagination  `json:"pagination"`
}

type Pagination struct {
	Page      int `json:"page"`
	PageSize  int `json:"page_size"`
	Total     int `json:"total"`
	TotalPage int `json:"total_page"`
}

func Success(c *fiber.Ctx, status int, message string, data interface{}) error {
	return c.Status(status).JSON(Response{
		Success: true,
		Message: message,
		Data:    data,
	})
}

func Error(c *fiber.Ctx, status int, message string, err error) error {
	errMsg := ""
	if err != nil {
		errMsg = err.Error()
	}
	
	return c.Status(status).JSON(Response{
		Success: false,
		Message: message,
		Error:   errMsg,
	})
}

func ValidationError(c *fiber.Ctx, err error) error {
	return c.Status(fiber.StatusBadRequest).JSON(Response{
		Success: false,
		Message: "validation failed",
		Error:   err.Error(),
	})
}

func Paginated(c *fiber.Ctx, status int, data interface{}, total, page, pageSize int) error {
	totalPage := (total + pageSize - 1) / pageSize
	
	return c.Status(status).JSON(PaginatedResponse{
		Success: true,
		Message: "success",
		Data:    data,
		Pagination: Pagination{
			Page:      page,
			PageSize:  pageSize,
			Total:     total,
			TotalPage: totalPage,
		},
	})
}
```

**pkg/logger/logger.go**
```go
package logger

import (
	"log/slog"
	"os"
)

type Logger struct {
	*slog.Logger
}

func New(env string) *Logger {
	var handler slog.Handler
	
	if env == "production" {
		handler = slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
			Level: slog.LevelInfo,
		})
	} else {
		handler = slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
			Level: slog.LevelDebug,
		})
	}
	
	return &Logger{
		Logger: slog.New(handler),
	}
}
```

**pkg/validator/validator.go**
```go
package validator

import (
	"github.com/go-playground/validator/v10"
)

type Validator struct {
	validate *validator.Validate
}

func New() *Validator {
	return &Validator{
		validate: validator.New(),
	}
}

func (v *Validator) Validate(data interface{}) error {
	return v.validate.Struct(data)
}
```

---

### 6. 中間件

**internal/middleware/logger.go**
```go
package middleware

import (
	"time"
	
	"github.com/gofiber/fiber/v2"
	"project-01/pkg/logger"
)

func Logger(log *logger.Logger) fiber.Handler {
	return func(c *fiber.Ctx) error {
		start := time.Now()
		
		// 處理請求
		err := c.Next()
		
		// 記錄日誌
		log.Info("http request",
			"method", c.Method(),
			"path", c.Path(),
			"status", c.Response().StatusCode(),
			"duration", time.Since(start).Milliseconds(),
			"ip", c.IP(),
			"request_id", c.Locals("request_id"),
		)
		
		return err
	}
}
```

**internal/middleware/recovery.go**
```go
package middleware

import (
	"github.com/gofiber/fiber/v2"
	"project-01/pkg/logger"
	"project-01/pkg/response"
)

func Recovery(log *logger.Logger) fiber.Handler {
	return func(c *fiber.Ctx) error {
		defer func() {
			if r := recover(); r != nil {
				log.Error("panic recovered",
					"error", r,
					"path", c.Path(),
					"request_id", c.Locals("request_id"),
				)
				
				response.Error(c, fiber.StatusInternalServerError, "internal server error", nil)
			}
		}()
		
		return c.Next()
	}
}
```

**internal/middleware/request_id.go**
```go
package middleware

import (
	"github.com/gofiber/fiber/v2"
	"github.com/google/uuid"
)

func RequestID() fiber.Handler {
	return func(c *fiber.Ctx) error {
		requestID := c.Get("X-Request-ID")
		if requestID == "" {
			requestID = uuid.New().String()
		}
		
		c.Locals("request_id", requestID)
		c.Set("X-Request-ID", requestID)
		
		return c.Next()
	}
}
```

**internal/middleware/rate_limiter.go**
```go
package middleware

import (
	"github.com/gofiber/fiber/v2"
	"github.com/gofiber/fiber/v2/middleware/limiter"
	"time"
)

func RateLimiter() fiber.Handler {
	return limiter.New(limiter.Config{
		Max:        20,
		Expiration: 1 * time.Minute,
		KeyGenerator: func(c *fiber.Ctx) string {
			return c.IP()
		},
		LimitReached: func(c *fiber.Ctx) error {
			return c.Status(fiber.StatusTooManyRequests).JSON(fiber.Map{
				"success": false,
				"message": "too many requests",
			})
		},
	})
}
```

---

### 7. 配置管理

**internal/config/config.go**
```go
package config

import (
	"fmt"
	"os"
	"strconv"
)

type Config struct {
	Server   ServerConfig
	Database DatabaseConfig
	App      AppConfig
}

type ServerConfig struct {
	Host string
	Port int
}

type DatabaseConfig struct {
	Host     string
	Port     int
	User     string
	Password string
	DBName   string
	SSLMode  string
}

type AppConfig struct {
	Environment string
	LogLevel    string
}

func Load() (*Config, error) {
	port, err := strconv.Atoi(getEnv("SERVER_PORT", "8080"))
	if err != nil {
		return nil, fmt.Errorf("invalid SERVER_PORT: %w", err)
	}
	
	dbPort, err := strconv.Atoi(getEnv("DB_PORT", "5432"))
	if err != nil {
		return nil, fmt.Errorf("invalid DB_PORT: %w", err)
	}
	
	return &Config{
		Server: ServerConfig{
			Host: getEnv("SERVER_HOST", "0.0.0.0"),
			Port: port,
		},
		Database: DatabaseConfig{
			Host:     getEnv("DB_HOST", "localhost"),
			Port:     dbPort,
			User:     getEnv("DB_USER", "postgres"),
			Password: getEnv("DB_PASSWORD", ""),
			DBName:   getEnv("DB_NAME", "userdb"),
			SSLMode:  getEnv("DB_SSLMODE", "disable"),
		},
		App: AppConfig{
			Environment: getEnv("APP_ENV", "development"),
			LogLevel:    getEnv("LOG_LEVEL", "debug"),
		},
	}, nil
}

func (c *DatabaseConfig) DSN() string {
	return fmt.Sprintf(
		"host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
		c.Host, c.Port, c.User, c.Password, c.DBName, c.SSLMode,
	)
}

func getEnv(key, fallback string) string {
	if value := os.Getenv(key); value != "" {
		return value
	}
	return fallback
}
```

---

### 8. 主程序入口

**cmd/api/main.go**
```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"
	
	"github.com/gofiber/fiber/v2"
	"github.com/gofiber/fiber/v2/middleware/cors"
	"github.com/jmoiron/sqlx"
	_ "github.com/lib/pq"
	
	"project-01/internal/config"
	"project-01/internal/handler"
	"project-01/internal/middleware"
	"project-01/internal/repository/postgres"
	"project-01/internal/service"
	"project-01/pkg/logger"
	"project-01/pkg/validator"
)

func main() {
	// 1. 加載配置
	cfg, err := config.Load()
	if err != nil {
		log.Fatalf("Failed to load config: %v", err)
	}
	
	// 2. 初始化日誌
	appLogger := logger.New(cfg.App.Environment)
	
	// 3. 連接數據庫
	db, err := sqlx.Connect("postgres", cfg.Database.DSN())
	if err != nil {
		appLogger.Error("failed to connect database", "error", err)
		os.Exit(1)
	}
	defer db.Close()
	
	// 配置連接池
	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(5)
	db.SetConnMaxLifetime(5 * time.Minute)
	
	appLogger.Info("database connected successfully")
	
	// 4. 初始化依賴
	v := validator.New()
	userRepo := postgres.NewUserRepository(db)
	userService := service.NewUserService(userRepo, appLogger)
	userHandler := handler.NewUserHandler(userService, v)
	
	// 5. 創建 Fiber 應用
	app := fiber.New(fiber.Config{
		ErrorHandler: customErrorHandler,
		AppName:      "User API v1.0",
	})
	
	// 6. 註冊全局中間件
	app.Use(middleware.RequestID())
	app.Use(middleware.Logger(appLogger))
	app.Use(middleware.Recovery(appLogger))
	app.Use(middleware.RateLimiter())
	app.Use(cors.New())
	
	// 7. 健康檢查
	app.Get("/health", func(c *fiber.Ctx) error {
		return c.JSON(fiber.Map{
			"status": "healthy",
			"timestamp": time.Now().Unix(),
		})
	})
	
	// 8. 註冊路由
	api := app.Group("/api/v1")
	userHandler.RegisterRoutes(api)
	
	// 9. 啟動服務器（支持 Graceful Shutdown）
	addr := fmt.Sprintf("%s:%d", cfg.Server.Host, cfg.Server.Port)
	
	go func() {
		appLogger.Info("server starting", "address", addr)
		if err := app.Listen(addr); err != nil {
			appLogger.Error("server failed to start", "error", err)
			os.Exit(1)
		}
	}()
	
	// 10. Graceful Shutdown
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	
	<-quit
	appLogger.Info("shutting down server...")
	
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	
	if err := app.ShutdownWithContext(ctx); err != nil {
		appLogger.Error("server shutdown failed", "error", err)
	}
	
	appLogger.Info("server exited")
}

func customErrorHandler(c *fiber.Ctx, err error) error {
	code := fiber.StatusInternalServerError
	
	if e, ok := err.(*fiber.Error); ok {
		code = e.Code
	}
	
	return c.Status(code).JSON(fiber.Map{
		"success": false,
		"message": "request failed",
		"error":   err.Error(),
	})
}
```

---

### 9. 數據庫遷移

**migrations/001_create_users_table.up.sql**
```sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_created_at ON users(created_at DESC);
```

**migrations/001_create_users_table.down.sql**
```sql
DROP TABLE IF EXISTS users;
```

---

### 10. Docker 配置

**docker/Dockerfile**
```dockerfile
# 多階段構建

# Stage 1: 構建階段
FROM golang:1.21-alpine AS builder

WORKDIR /app

# 安裝依賴
RUN apk add --no-cache git

# 複製依賴文件
COPY go.mod go.sum ./
RUN go mod download

# 複製源代碼
COPY . .

# 構建二進制文件
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main ./cmd/api

# Stage 2: 運行階段
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

# 從構建階段複製二進制文件
COPY --from=builder /app/main .
COPY --from=builder /app/.env.example .env

EXPOSE 8080

CMD ["./main"]
```

**docker/docker-compose.yml**
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: user_api_db
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

  api:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    container_name: user_api
    ports:
      - "8080:8080"
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_USER: postgres
      DB_PASSWORD: secret
      DB_NAME: userdb
      SERVER_PORT: 8080
      APP_ENV: production
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

volumes:
  postgres_data:
```

---

### 11. 測試

**tests/unit/user_service_test.go**
```go
package unit

import (
	"context"
	"testing"
	
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
	"project-01/internal/domain"
	"project-01/internal/service"
	"project-01/pkg/logger"
)

// MockUserRepository 模擬倉庫
type MockUserRepository struct {
	mock.Mock
}

func (m *MockUserRepository) Create(ctx context.Context, user *domain.User) (int, error) {
	args := m.Called(ctx, user)
	return args.Int(0), args.Error(1)
}

func (m *MockUserRepository) GetByID(ctx context.Context, id int) (*domain.User, error) {
	args := m.Called(ctx, id)
	if args.Get(0) == nil {
		return nil, args.Error(1)
	}
	return args.Get(0).(*domain.User), args.Error(1)
}

func (m *MockUserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
	args := m.Called(ctx, email)
	if args.Get(0) == nil {
		return nil, args.Error(1)
	}
	return args.Get(0).(*domain.User), args.Error(1)
}

// 其他方法省略...

func TestCreateUser_Success(t *testing.T) {
	mockRepo := new(MockUserRepository)
	log := logger.New("test")
	svc := service.NewUserService(mockRepo, log)
	
	req := &domain.CreateUserRequest{
		Username: "testuser",
		Email:    "test@example.com",
		Password: "password123",
	}
	
	// 設置期望
	mockRepo.On("GetByEmail", mock.Anything, req.Email).Return(nil, domain.ErrNotFound)
	mockRepo.On("Create", mock.Anything, mock.AnythingOfType("*domain.User")).Return(1, nil)
	
	// 執行
	user, err := svc.CreateUser(context.Background(), req)
	
	// 斷言
	assert.NoError(t, err)
	assert.NotNil(t, user)
	assert.Equal(t, req.Username, user.Username)
	
	mockRepo.AssertExpectations(t)
}
```

---

### 12. Makefile

**Makefile**
```makefile
.PHONY: help run build test docker-up docker-down migrate-up migrate-down

help: ## 顯示幫助信息
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

run: ## 運行應用
	go run cmd/api/main.go

build: ## 構建二進制文件
	go build -o bin/api cmd/api/main.go

test: ## 運行測試
	go test -v -race -coverprofile=coverage.out ./...

test-coverage: test ## 生成測試覆蓋率報告
	go tool cover -html=coverage.out

docker-up: ## 啟動 Docker 容器
	docker-compose -f docker/docker-compose.yml up -d

docker-down: ## 停止 Docker 容器
	docker-compose -f docker/docker-compose.yml down

docker-logs: ## 查看日誌
	docker-compose -f docker/docker-compose.yml logs -f

migrate-up: ## 執行數據庫遷移
	migrate -path migrations -database "postgresql://postgres:secret@localhost:5432/userdb?sslmode=disable" up

migrate-down: ## 回滾數據庫遷移
	migrate -path migrations -database "postgresql://postgres:secret@localhost:5432/userdb?sslmode=disable" down

lint: ## 運行代碼檢查
	golangci-lint run

tidy: ## 整理依賴
	go mod tidy
```

---

## 🎓 學習要點（Senior Level）

### 1. 架構設計
- **Clean Architecture**：嚴格分層，依賴倒置
- **依賴注入**：通過構造函數注入依賴
- **接口隔離**：Repository/Service 都是接口

### 2. 錯誤處理
- **Sentinel Errors**：使用 `errors.Is()` 判斷特定錯誤
- **Error Wrapping**：使用 `fmt.Errorf("%w")` 包裝錯誤
- **統一響應格式**：通過 `pkg/response` 統一處理

### 3. 可觀測性
- **結構化日誌**：使用 `slog` 記錄結構化日誌
- **Request ID**：請求追蹤
- **健康檢查**：`/health` 端點

### 4. 安全性
- **密碼哈希**：使用 `bcrypt`
- **輸入驗證**：使用 `validator`
- **限流**：使用 Fiber 限流中間件

### 5. 性能優化
- **連接池配置**：合理設置數據庫連接池
- **索引優化**：為常用查詢字段創建索引
- **分頁查詢**：避免全表掃描

### 6. 部署
- **多階段構建**：減小 Docker 鏡像體積
- **Graceful Shutdown**：優雅關閉服務
- **健康檢查**：支持 K8s readiness/liveness probe

---

## 🚀 快速開始

```bash
# 1. 啟動數據庫
make docker-up

# 2. 執行遷移
make migrate-up

# 3. 運行應用
make run

# 4. 測試 API
curl -X POST http://localhost:8080/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","email":"alice@example.com","password":"password123"}'
```

---

## 📝 後續擴展

- [ ] 添加 JWT 認證
- [ ] 集成 Redis 緩存
- [ ] 添加 Swagger 文檔
- [ ] 實現分布式追蹤（OpenTelemetry）
- [ ] 添加 Prometheus 指標
- [ ] 實現 API 版本控制
