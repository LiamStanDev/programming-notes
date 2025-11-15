# Day 7：Web 框架與中間件設計（Fiber）

## 📚 學習目標

- 掌握 Fiber 框架核心 API
- 實現自定義中間件
- 設計 RESTful API 路由結構
- 實現請求驗證與錯誤處理

---

## 1. Fiber 快速開始

### 1.1 安裝

```bash
go get github.com/gofiber/fiber/v2
```

### 1.2 基本服務器

```go
package main

import (
    "github.com/gofiber/fiber/v2"
    "log"
)

func main() {
    app := fiber.New(fiber.Config{
        AppName: "My App v1.0.0",
    })
    
    app.Get("/", func(c *fiber.Ctx) error {
        return c.SendString("Hello, Fiber!")
    })
    
    log.Fatal(app.Listen(":3000"))
}
```

---

## 2. 路由設計

### 2.1 基本路由

```go
app.Get("/users", getUsers)
app.Get("/users/:id", getUserByID)
app.Post("/users", createUser)
app.Put("/users/:id", updateUser)
app.Delete("/users/:id", deleteUser)
```

### 2.2 路由分組

```go
func setupRoutes(app *fiber.App) {
    api := app.Group("/api/v1")
    
    // 用戶路由
    users := api.Group("/users")
    users.Get("/", getUsers)
    users.Get("/:id", getUserByID)
    users.Post("/", createUser)
    users.Put("/:id", updateUser)
    users.Delete("/:id", deleteUser)
    
    // 文章路由
    posts := api.Group("/posts")
    posts.Get("/", getPosts)
    posts.Get("/:id", getPostByID)
}
```

---

## 3. 中間件

### 3.1 內建中間件

```go
import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/logger"
    "github.com/gofiber/fiber/v2/middleware/recover"
    "github.com/gofiber/fiber/v2/middleware/cors"
)

func main() {
    app := fiber.New()
    
    // 日誌中間件
    app.Use(logger.New())
    
    // 恢復中間件（捕獲 panic）
    app.Use(recover.New())
    
    // CORS 中間件
    app.Use(cors.New(cors.Config{
        AllowOrigins: "http://localhost:3000",
        AllowHeaders: "Origin, Content-Type, Accept",
    }))
}
```

### 3.2 自定義中間件

```go
// 認證中間件
func AuthMiddleware(c *fiber.Ctx) error {
    token := c.Get("Authorization")
    
    if token == "" {
        return c.Status(401).JSON(fiber.Map{
            "error": "Unauthorized",
        })
    }
    
    // 驗證 token
    userID, err := validateToken(token)
    if err != nil {
        return c.Status(401).JSON(fiber.Map{
            "error": "Invalid token",
        })
    }
    
    // 將用戶 ID 存入 Locals
    c.Locals("userID", userID)
    
    return c.Next()
}

// 使用
api.Use(AuthMiddleware)
```

---

## 4. 請求處理

### 4.1 參數綁定

```go
type CreateUserRequest struct {
    Username string `json:"username" validate:"required,min=3"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=6"`
}

func createUser(c *fiber.Ctx) error {
    var req CreateUserRequest
    
    if err := c.BodyParser(&req); err != nil {
        return c.Status(400).JSON(fiber.Map{
            "error": "Invalid request body",
        })
    }
    
    // 驗證
    if err := validate.Struct(req); err != nil {
        return c.Status(400).JSON(fiber.Map{
            "error": err.Error(),
        })
    }
    
    // 業務邏輯...
    return c.JSON(fiber.Map{
        "message": "User created",
    })
}
```

---

## 5. 錯誤處理

### 5.1 自定義錯誤處理器

```go
func ErrorHandler(c *fiber.Ctx, err error) error {
    code := fiber.StatusInternalServerError
    message := "Internal Server Error"
    
    // 類型斷言檢查
    if e, ok := err.(*fiber.Error); ok {
        code = e.Code
        message = e.Message
    }
    
    // 記錄錯誤
    log.Printf("Error: %v", err)
    
    return c.Status(code).JSON(fiber.Map{
        "error": message,
    })
}

// 在 Fiber 配置中使用
app := fiber.New(fiber.Config{
    ErrorHandler: ErrorHandler,
})
```

### 5.2 統一響應格式

```go
type Response struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   string      `json:"error,omitempty"`
}

func SuccessResponse(c *fiber.Ctx, data interface{}) error {
    return c.JSON(Response{
        Success: true,
        Data:    data,
    })
}

func ErrorResponse(c *fiber.Ctx, code int, message string) error {
    return c.Status(code).JSON(Response{
        Success: false,
        Error:   message,
    })
}
```

---

## 6. 請求驗證

### 6.1 安裝驗證庫

```bash
go get github.com/go-playground/validator/v10
```

### 6.2 實現驗證中間件

```go
import "github.com/go-playground/validator/v10"

var validate = validator.New()

type CreateUserRequest struct {
    Username string `json:"username" validate:"required,min=3,max=20"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8"`
    Age      int    `json:"age" validate:"gte=0,lte=120"`
}

func createUser(c *fiber.Ctx) error {
    var req CreateUserRequest
    
    if err := c.BodyParser(&req); err != nil {
        return ErrorResponse(c, 400, "Invalid request body")
    }
    
    if err := validate.Struct(req); err != nil {
        return ErrorResponse(c, 400, err.Error())
    }
    
    // 業務邏輯
    user := &User{
        Username: req.Username,
        Email:    req.Email,
    }
    
    return SuccessResponse(c, user)
}
```

---

## 7. 完整示例：RESTful CRUD API

### 7.1 項目結構

```
myapp/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── handler/
│   │   └── user_handler.go
│   ├── service/
│   │   └── user_service.go
│   ├── repository/
│   │   └── user_repository.go
│   └── middleware/
│       ├── auth.go
│       └── logger.go
├── pkg/
│   └── validator/
│       └── validator.go
└── go.mod
```

### 7.2 Handler 層

```go
package handler

import (
    "github.com/gofiber/fiber/v2"
    "myapp/internal/service"
)

type UserHandler struct {
    service *service.UserService
}

func NewUserHandler(service *service.UserService) *UserHandler {
    return &UserHandler{service: service}
}

func (h *UserHandler) GetUser(c *fiber.Ctx) error {
    id, err := c.ParamsInt("id")
    if err != nil {
        return ErrorResponse(c, 400, "Invalid user ID")
    }
    
    user, err := h.service.GetUser(c.Context(), id)
    if err != nil {
        return ErrorResponse(c, 404, "User not found")
    }
    
    return SuccessResponse(c, user)
}

func (h *UserHandler) ListUsers(c *fiber.Ctx) error {
    page := c.QueryInt("page", 1)
    pageSize := c.QueryInt("page_size", 10)
    
    users, total, err := h.service.ListUsers(c.Context(), page, pageSize)
    if err != nil {
        return ErrorResponse(c, 500, "Failed to fetch users")
    }
    
    return SuccessResponse(c, fiber.Map{
        "users": users,
        "total": total,
        "page":  page,
    })
}

func (h *UserHandler) CreateUser(c *fiber.Ctx) error {
    var req CreateUserRequest
    
    if err := c.BodyParser(&req); err != nil {
        return ErrorResponse(c, 400, "Invalid request body")
    }
    
    if err := validate.Struct(req); err != nil {
        return ErrorResponse(c, 400, err.Error())
    }
    
    user, err := h.service.CreateUser(c.Context(), &req)
    if err != nil {
        return ErrorResponse(c, 500, "Failed to create user")
    }
    
    return c.Status(201).JSON(Response{
        Success: true,
        Data:    user,
    })
}

func (h *UserHandler) UpdateUser(c *fiber.Ctx) error {
    id, err := c.ParamsInt("id")
    if err != nil {
        return ErrorResponse(c, 400, "Invalid user ID")
    }
    
    var req UpdateUserRequest
    if err := c.BodyParser(&req); err != nil {
        return ErrorResponse(c, 400, "Invalid request body")
    }
    
    user, err := h.service.UpdateUser(c.Context(), id, &req)
    if err != nil {
        return ErrorResponse(c, 500, "Failed to update user")
    }
    
    return SuccessResponse(c, user)
}

func (h *UserHandler) DeleteUser(c *fiber.Ctx) error {
    id, err := c.ParamsInt("id")
    if err != nil {
        return ErrorResponse(c, 400, "Invalid user ID")
    }
    
    if err := h.service.DeleteUser(c.Context(), id); err != nil {
        return ErrorResponse(c, 500, "Failed to delete user")
    }
    
    return c.SendStatus(204)
}
```

### 7.3 路由設置

```go
package main

import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/logger"
    "github.com/gofiber/fiber/v2/middleware/recover"
    "myapp/internal/handler"
)

func setupRoutes(app *fiber.App, userHandler *handler.UserHandler) {
    // 中間件
    app.Use(logger.New())
    app.Use(recover.New())
    
    // API 版本分組
    api := app.Group("/api/v1")
    
    // 用戶路由
    users := api.Group("/users")
    users.Get("/", userHandler.ListUsers)
    users.Get("/:id", userHandler.GetUser)
    users.Post("/", userHandler.CreateUser)
    users.Put("/:id", userHandler.UpdateUser)
    users.Delete("/:id", userHandler.DeleteUser)
    
    // 健康檢查
    app.Get("/health", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{
            "status": "ok",
        })
    })
}
```

---

## 8. 高級特性

### 8.1 JWT 認證

```go
import (
    "github.com/gofiber/fiber/v2"
    jwtware "github.com/gofiber/jwt/v3"
    "github.com/golang-jwt/jwt/v4"
)

func JWTMiddleware(secret string) fiber.Handler {
    return jwtware.New(jwtware.Config{
        SigningKey: []byte(secret),
        ErrorHandler: func(c *fiber.Ctx, err error) error {
            return ErrorResponse(c, 401, "Unauthorized")
        },
    })
}

// 生成 Token
func generateToken(userID int, secret string) (string, error) {
    claims := jwt.MapClaims{
        "user_id": userID,
        "exp":     time.Now().Add(time.Hour * 24).Unix(),
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(secret))
}

// 使用
func setupAuthRoutes(app *fiber.App, secret string) {
    app.Post("/login", loginHandler)
    
    // 受保護的路由
    protected := app.Group("/api")
    protected.Use(JWTMiddleware(secret))
    protected.Get("/profile", profileHandler)
}
```

### 8.2 速率限制

```go
import "github.com/gofiber/fiber/v2/middleware/limiter"

func setupRateLimiting(app *fiber.App) {
    app.Use(limiter.New(limiter.Config{
        Max:        20,
        Expiration: 30 * time.Second,
        KeyGenerator: func(c *fiber.Ctx) string {
            return c.IP()
        },
        LimitReached: func(c *fiber.Ctx) error {
            return ErrorResponse(c, 429, "Too many requests")
        },
    }))
}
```

---

## 9. 實戰練習

### 練習 1：實現分頁中間件

```go
type PaginationMiddleware struct {
    DefaultPage     int
    DefaultPageSize int
    MaxPageSize     int
}

func (p *PaginationMiddleware) Handle(c *fiber.Ctx) error {
    // TODO: 解析並驗證 page 和 page_size 參數
    // TODO: 將結果存入 c.Locals()
    return c.Next()
}
```

### 練習 2：實現請求 ID 追蹤

```go
func RequestIDMiddleware(c *fiber.Ctx) error {
    // TODO: 生成或提取請求 ID
    // TODO: 添加到響應頭
    // TODO: 存入 c.Locals() 供後續使用
    return c.Next()
}
```

### 練習 3：實現 CORS 配置

```go
func setupCORS(app *fiber.App) {
    // TODO: 配置 CORS 中間件
    // 允許特定的 origins
    // 允許特定的 methods
    // 處理 preflight 請求
}
```

---

## 10. 最佳實踐總結

### ✅ Do's
1. **使用路由分組組織 API**
2. **實現統一的錯誤處理**
3. **使用中間件處理橫切關注點**
4. **驗證所有用戶輸入**
5. **使用結構化日誌**

### ❌ Don'ts
1. **不要在 Handler 中直接操作數據庫**
2. **不要忽略錯誤處理**
3. **不要返回敏感信息（如密碼）**
4. **不要使用阻塞操作**
5. **不要忘記設置請求超時**

---

## 11. 延伸閱讀

- [Fiber Documentation](https://docs.gofiber.io/)
- [RESTful API Design Best Practices](https://restfulapi.net/)
- [JWT Authentication](https://jwt.io/introduction)

---

**上一篇**: [Day 6 - 高級併發模式與外部通訊](../02-併發篇/06-高級併發模式與外部通訊.md)  
**下一篇**: [Day 8 - 數據持久化與容器化](08-數據持久化與容器化.md)
