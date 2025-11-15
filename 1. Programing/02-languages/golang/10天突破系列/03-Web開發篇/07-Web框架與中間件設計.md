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

## 5. 完整示例：RESTful API

見完整筆記...

---

**上一篇**: [Day 6 - 高級併發模式與外部通訊](../02-併發篇/06-高級併發模式與外部通訊.md)  
**下一篇**: [Day 8 - 數據持久化與容器化](08-數據持久化與容器化.md)
