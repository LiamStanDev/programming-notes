# Day 3：數據庫連接與 SQL 操作（sqlx）

## 📚 學習目標

- 掌握 sqlx 的核心用法與優勢
- 深入理解位置參數與命名參數的選擇策略
- 理解 Struct Tag Mapping 自動綁定
- 實現 Repository 模式進行數據訪問
- 掌握參數轉換的內部機制與最佳實踐

---

## 1. sqlx 簡介

### 1.1 為什麼選擇 sqlx？

sqlx 是 `database/sql` 的超集，提供：
- **結構體映射**：自動將查詢結果映射到 Go 結構體
- **Named Query**：使用命名參數而非 `?` 占位符
- **批量操作**：支持 `Select` 和 `NamedExec` 簡化代碼
- **類型安全**：保持 `database/sql` 的類型安全性

### 1.2 安裝

```bash
go get github.com/jmoiron/sqlx
go get github.com/lib/pq  # PostgreSQL 驅動
```

---

## 2. 數據庫連接

### 2.1 基本連接

```go
package main

import (
    "fmt"
    "log"
    
    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"  // PostgreSQL 驅動
)

func main() {
    dsn := "host=localhost port=5432 user=postgres password=secret dbname=mydb sslmode=disable"
    
    db, err := sqlx.Connect("postgres", dsn)
    if err != nil {
        log.Fatalf("Failed to connect: %v", err)
    }
    defer db.Close()
    
    // 測試連接
    if err := db.Ping(); err != nil {
        log.Fatalf("Failed to ping: %v", err)
    }
    
    fmt.Println("Database connected!")
}
```

### 2.2 配置連接池

```go
func NewDB(dsn string) (*sqlx.DB, error) {
    db, err := sqlx.Connect("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("connect to database: %w", err)
    }
    
    // 連接池配置
    db.SetMaxOpenConns(25)                 // 最大打開連接數
    db.SetMaxIdleConns(5)                  // 最大空閒連接數
    db.SetConnMaxLifetime(5 * time.Minute) // 連接最大生命週期
    db.SetConnMaxIdleTime(10 * time.Minute)// 空閒連接最大存活時間
    
    return db, nil
}
```

**為什麼需要配置連接池？**

| 配置項 | 作用 | 不配置的風險 |
|------|------|------------|
| `MaxOpenConns` | 限制最大連接數 | 可能耗盡數據庫連接資源 |
| `MaxIdleConns` | 保持一定空閒連接 | 頻繁創建/銷毀連接，性能下降 |
| `ConnMaxLifetime` | 防止連接過期 | 使用過期連接導致錯誤 |
| `ConnMaxIdleTime` | 釋放長時間不用的連接 | 浪費數據庫資源 |

### 2.3 環境配置管理

```go
package config

import (
    "fmt"
    "os"
)

type DBConfig struct {
    Host     string
    Port     string
    User     string
    Password string
    DBName   string
    SSLMode  string
}

func LoadDBConfig() *DBConfig {
    return &DBConfig{
        Host:     getEnv("DB_HOST", "localhost"),
        Port:     getEnv("DB_PORT", "5432"),
        User:     getEnv("DB_USER", "postgres"),
        Password: getEnv("DB_PASSWORD", ""),
        DBName:   getEnv("DB_NAME", "mydb"),
        SSLMode:  getEnv("DB_SSLMODE", "disable"),
    }
}

func (c *DBConfig) DSN() string {
    return fmt.Sprintf(
        "host=%s port=%s user=%s password=%s dbname=%s sslmode=%s",
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

## 3. Struct Tag Mapping

### 3.1 基本映射

```go
// User 結構體
type User struct {
    ID        int       `db:"id"`
    Username  string    `db:"username"`
    Email     string    `db:"email"`
    CreatedAt time.Time `db:"created_at"`
    UpdatedAt time.Time `db:"updated_at"`
}

// 查詢單個用戶
func GetUser(db *sqlx.DB, id int) (*User, error) {
    var user User
    query := "SELECT id, username, email, created_at, updated_at FROM users WHERE id = $1"
    
    err := db.Get(&user, query, id)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("user not found")
        }
        return nil, fmt.Errorf("query user: %w", err)
    }
    
    return &user, nil
}
```

**為什麼使用 `db` tag？**

- **自動映射**：sqlx 根據 `db` tag 將數據庫列名映射到結構體字段
- **命名轉換**：支持 snake_case（數據庫）→ CamelCase（Go）的自動轉換
- **靈活性**：可以處理字段名與列名不一致的情況

### 3.2 查詢多行

```go
func GetAllUsers(db *sqlx.DB) ([]User, error) {
    var users []User
    query := "SELECT id, username, email, created_at, updated_at FROM users"
    
    err := db.Select(&users, query)
    if err != nil {
        return nil, fmt.Errorf("query users: %w", err)
    }
    
    return users, nil
}
```

**`Get` vs `Select` 的關鍵區別**：

| 方法 | 用途 | dest 類型 | 無結果時 |
|------|------|-----------|----------|
| `Get` | 查詢**單筆**資料 | `*User` | 返回 `sql.ErrNoRows` |
| `Select` | 查詢**多筆**資料 | `*[]User` | 返回空切片（不報錯） |

### 3.3 嵌套結構體

```go
type Address struct {
    Street  string `db:"street"`
    City    string `db:"city"`
    Country string `db:"country"`
}

type UserProfile struct {
    User              // 嵌入 User
    Address           // 嵌入 Address
    Bio     string `db:"bio"`
}

// 查詢需要使用 JOIN
func GetUserProfile(db *sqlx.DB, userID int) (*UserProfile, error) {
    var profile UserProfile
    query := `
        SELECT 
            u.id, u.username, u.email, u.created_at, u.updated_at,
            a.street, a.city, a.country,
            p.bio
        FROM users u
        JOIN addresses a ON u.id = a.user_id
        JOIN profiles p ON u.id = p.user_id
        WHERE u.id = $1
    `
    
    err := db.Get(&profile, query, userID)
    if err != nil {
        return nil, fmt.Errorf("query user profile: %w", err)
    }
    
    return &profile, nil
}
```

**嵌套結構體的優勢**：
- 代碼複用：避免重複定義字段
- 組合優於繼承：靈活組合不同數據模型
- 自動扁平化映射：sqlx 會自動處理嵌套結構的映射

---

## 4. 核心概念：位置參數 vs 命名參數

### 4.1 兩種參數類型的本質

#### 位置參數（Positional Parameters）
```go
// PostgreSQL 使用 $1, $2, $3
query := "INSERT INTO users (username, email) VALUES ($1, $2)"
db.Exec(query, "john", "john@example.com")

// MySQL 使用 ?
query := "INSERT INTO users (username, email) VALUES (?, ?)"
db.Exec(query, "john", "john@example.com")
```

#### 命名參數（Named Parameters）
```go
// 使用 :name 語法
query := "INSERT INTO users (username, email) VALUES (:username, :email)"
params := map[string]interface{}{
    "username": "john",
    "email": "john@example.com",
}
db.NamedExec(query, params)

// 或使用結構體
type CreateUserRequest struct {
    Username string `db:"username"`
    Email    string `db:"email"`
}
req := &CreateUserRequest{Username: "john", Email: "john@example.com"}
db.NamedExec(query, req)
```

### 4.2 優缺點深度對比

#### 位置參數（Positional Parameters）

**✅ 優點**：
1. **性能最優**：無需參數轉換，直接傳遞給數據庫
2. **簡單直接**：適合參數少且順序固定的查詢
3. **原生支持**：所有 SQL 驅動都原生支持

**❌ 缺點**：
1. **可讀性差**：參數多時難以對應
   ```go
   // 很難看出哪個參數對應哪個字段
   db.Exec(query, "john", "john@example.com", 25, "New York", "USA", time.Now(), true)
   ```

2. **維護困難**：SQL 調整順序時容易出錯
   ```go
   // 原始 SQL
   query := "INSERT INTO users (username, email) VALUES ($1, $2)"
   
   // 後來需要添加 age，必須同步修改參數順序
   query := "INSERT INTO users (username, age, email) VALUES ($1, $2, $3)"
   db.Exec(query, username, age, email)  // 必須調整順序
   ```

3. **動態查詢複雜**：構建動態 SQL 時需要手動管理索引
   ```go
   var conditions []string
   var args []interface{}
   index := 1
   
   if username != "" {
       conditions = append(conditions, fmt.Sprintf("username = $%d", index))
       args = append(args, username)
       index++
   }
   if email != "" {
       conditions = append(conditions, fmt.Sprintf("email = $%d", index))
       args = append(args, email)
       index++
   }
   ```

#### 命名參數（Named Parameters）

**✅ 優點**：
1. **可讀性極強**：參數名稱清晰明確
   ```go
   query := `
       INSERT INTO users (username, email, age, city, country)
       VALUES (:username, :email, :age, :city, :country)
   `
   // 一眼就能看出參數對應關係
   ```

2. **維護容易**：SQL 調整順序不影響代碼
   ```go
   // 調整 SQL 順序，代碼無需改動
   query := `
       INSERT INTO users (email, username, age)  -- 順序改變
       VALUES (:email, :username, :age)           -- 只要名稱對應即可
   `
   ```

3. **動態查詢友好**：構建動態 SQL 更簡單
   ```go
   args := make(map[string]interface{})
   query := "SELECT * FROM users WHERE 1=1"
   
   if username != "" {
       query += " AND username = :username"
       args["username"] = username
   }
   if email != "" {
       query += " AND email = :email"
       args["email"] = email
   }
   ```

4. **支持結構體**：直接使用 Go 結構體作為參數
   ```go
   type CreateUserRequest struct {
       Username string `db:"username"`
       Email    string `db:"email"`
   }
   db.NamedExec(query, req)  // 自動映射
   ```

**❌ 缺點**：
1. **性能開銷**：需要內部轉換為位置參數（雖然開銷很小）
2. **額外依賴**：依賴 sqlx 庫，標準庫不支持

### 4.3 參數轉換的內部機制

**命名參數是如何工作的？**

sqlx 內部執行 3 個步驟將命名參數轉換為位置參數：

```go
// 第 1 步：使用 sqlx.Named() 轉換
query := "SELECT * FROM users WHERE username = :username AND email = :email"
args := map[string]interface{}{
    "username": "john",
    "email": "john@example.com",
}

// Named() 將命名參數轉換為 ? 占位符
newQuery, newArgs, _ := sqlx.Named(query, args)
// newQuery = "SELECT * FROM users WHERE username = ? AND email = ?"
// newArgs = []interface{}{"john", "john@example.com"}

// 第 2 步：使用 db.Rebind() 適配數據庫
reboundQuery := db.Rebind(newQuery)
// PostgreSQL: "SELECT * FROM users WHERE username = $1 AND email = $2"
// MySQL:      "SELECT * FROM users WHERE username = ? AND email = ?"

// 第 3 步：執行查詢
db.Select(&users, reboundQuery, newArgs...)
```

**為什麼需要 `Rebind()`？**

不同數據庫使用不同的占位符語法：

| 數據庫 | 占位符語法 | 示例 |
|--------|-----------|------|
| PostgreSQL | `$1, $2, $3` | `WHERE id = $1 AND name = $2` |
| MySQL | `?` | `WHERE id = ? AND name = ?` |
| Oracle | `:1, :2, :3` | `WHERE id = :1 AND name = :2` |
| SQL Server | `@p1, @p2, @p3` | `WHERE id = @p1 AND name = @p2` |

`Rebind()` 自動將 `?` 轉換為目標數據庫的格式，確保跨數據庫兼容性。

### 4.4 使用場景決策指南

#### 🎯 使用位置參數的場景

1. **簡單查詢（參數 ≤ 3 個）**
   ```go
   db.Get(&user, "SELECT * FROM users WHERE id = $1", userID)
   db.Exec("DELETE FROM users WHERE id = $1", userID)
   ```

2. **性能關鍵路徑**
   ```go
   // 高頻查詢，避免轉換開銷
   for i := 0; i < 10000; i++ {
       db.Exec("INSERT INTO logs (message) VALUES ($1)", msg)
   }
   ```

3. **標準庫限制**（不使用 sqlx 時）
   ```go
   // database/sql 只支持位置參數
   db.Exec("UPDATE users SET status = $1 WHERE id = $2", status, id)
   ```

#### 🎯 使用命名參數的場景

1. **複雜插入/更新（參數 > 3 個）**
   ```go
   query := `
       INSERT INTO users (username, email, age, city, country, bio)
       VALUES (:username, :email, :age, :city, :country, :bio)
   `
   db.NamedExec(query, user)
   ```

2. **動態查詢構建**
   ```go
   query := "SELECT * FROM users WHERE 1=1"
   args := make(map[string]interface{})
   
   if filter.Username != nil {
       query += " AND username = :username"
       args["username"] = *filter.Username
   }
   if filter.Email != nil {
       query += " AND email = :email"
       args["email"] = *filter.Email
   }
   ```

3. **批量操作**
   ```go
   users := []User{
       {Username: "john", Email: "john@example.com"},
       {Username: "jane", Email: "jane@example.com"},
   }
   db.NamedExec("INSERT INTO users (...) VALUES (...)", users)
   ```

4. **使用結構體參數**
   ```go
   type UpdateUserRequest struct {
       ID       int    `db:"id"`
       Username string `db:"username"`
       Email    string `db:"email"`
   }
   db.NamedExec("UPDATE users SET username = :username, email = :email WHERE id = :id", req)
   ```

#### 📊 決策流程圖

```
開始查詢
    ↓
參數數量 ≤ 3？
    ├─ 是 → 是否高頻查詢？
    │         ├─ 是 → 使用位置參數（性能優先）
    │         └─ 否 → 使用命名參數（可讀性優先）
    └─ 否 → 是否動態查詢？
              ├─ 是 → 使用命名參數（必須）
              └─ 否 → 是否使用結構體？
                        ├─ 是 → 使用命名參數（必須）
                        └─ 否 → 使用命名參數（推薦）
```

---

## 5. sqlx 核心函數完整解析

### 5.1 查詢函數族

#### `db.Get(dest, query, args...)` - 查詢單行

```go
var user User
err := db.Get(&user, "SELECT * FROM users WHERE id = $1", userID)
```

**使用要點**：
- ✅ 查詢必須返回恰好 1 行
- ✅ 無結果時返回 `sql.ErrNoRows`
- ✅ dest 必須是結構體指針 `*User`
- ❌ 不能用於查詢多行

**為什麼用 `Get`？**
- 語義明確：明確表示期望單行結果
- 自動校驗：多行或無行都會報錯
- 簡化代碼：無需手動調用 `rows.Next()`

#### `db.Select(dest, query, args...)` - 查詢多行

```go
var users []User
err := db.Select(&users, "SELECT * FROM users")
```

**使用要點**：
- ✅ 查詢可以返回 0 到多行
- ✅ 無結果時返回空切片（不報錯）
- ✅ dest 必須是切片指針 `*[]User`
- ❌ 不能用於單行查詢（雖然可以工作，但語義不清）

**為什麼用 `Select`？**
- 批量處理：一次查詢多條記錄
- 內存效率：sqlx 內部優化了切片增長
- 自動迭代：無需手動處理 `rows.Next()` 循環

#### `Get` vs `Select` 實戰對比

```go
// ❌ 錯誤示例：用 Select 查詢單行
var users []User
db.Select(&users, "SELECT * FROM users WHERE id = $1", userID)
if len(users) > 0 {
    user := users[0]  // 多餘的切片操作
}

// ✅ 正確示例：用 Get 查詢單行
var user User
err := db.Get(&user, "SELECT * FROM users WHERE id = $1", userID)
if err == sql.ErrNoRows {
    // 明確處理未找到的情況
}

// ❌ 錯誤示例：用 Get 查詢可能的多行
var user User
db.Get(&user, "SELECT * FROM users LIMIT 1")  // 語義不清

// ✅ 正確示例：用 Select 查詢多行
var users []User
db.Select(&users, "SELECT * FROM users LIMIT 10")
```

### 5.2 執行函數族

#### `db.Exec(query, args...)` - 位置參數執行

```go
result, err := db.Exec("DELETE FROM users WHERE id = $1", userID)
rows, _ := result.RowsAffected()
```

**適用場景**：
- 簡單的 INSERT/UPDATE/DELETE
- 參數少（≤ 3 個）
- 不需要結構體映射

**為什麼檢查 `RowsAffected`？**
```go
result, err := db.Exec("UPDATE users SET status = $1 WHERE id = $2", status, id)
if err != nil {
    return err
}

rows, _ := result.RowsAffected()
if rows == 0 {
    return fmt.Errorf("user not found")  // 更新失敗，可能 ID 不存在
}
```

#### `db.NamedExec(query, arg)` - 命名參數執行

```go
// 使用 map
params := map[string]interface{}{
    "username": "john",
    "email": "john@example.com",
}
result, err := db.NamedExec(
    "INSERT INTO users (username, email) VALUES (:username, :email)",
    params,
)

// 使用結構體
type User struct {
    Username string `db:"username"`
    Email    string `db:"email"`
}
user := &User{Username: "john", Email: "john@example.com"}
result, err := db.NamedExec(
    "INSERT INTO users (username, email) VALUES (:username, :email)",
    user,
)

// 批量插入
users := []User{
    {Username: "john", Email: "john@example.com"},
    {Username: "jane", Email: "jane@example.com"},
}
result, err := db.NamedExec(
    "INSERT INTO users (username, email) VALUES (:username, :email)",
    users,
)
```

**適用場景**：
- 參數多（> 3 個）
- 使用結構體作為數據源
- 批量操作

#### `Exec` vs `NamedExec` 實戰對比

```go
// 場景 1：簡單刪除 → 使用 Exec
db.Exec("DELETE FROM users WHERE id = $1", id)

// 場景 2：複雜更新 → 使用 NamedExec
type UpdateUserRequest struct {
    ID       int    `db:"id"`
    Username string `db:"username"`
    Email    string `db:"email"`
    Age      int    `db:"age"`
    City     string `db:"city"`
}
req := &UpdateUserRequest{...}
db.NamedExec(`
    UPDATE users 
    SET username = :username, email = :email, age = :age, city = :city
    WHERE id = :id
`, req)

// 場景 3：批量插入 → 使用 NamedExec
users := []User{{...}, {...}, {...}}
db.NamedExec("INSERT INTO users (...) VALUES (...)", users)
```

### 5.3 預編譯函數

#### `db.PrepareNamed(query)` - 預編譯命名查詢

```go
stmt, err := db.PrepareNamed(`
    INSERT INTO users (username, email) 
    VALUES (:username, :email) 
    RETURNING id
`)
defer stmt.Close()

// 重複使用
var id1 int
stmt.Get(&id1, map[string]interface{}{"username": "john", "email": "john@example.com"})

var id2 int
stmt.Get(&id2, map[string]interface{}{"username": "jane", "email": "jane@example.com"})
```

**為什麼使用預編譯？**

1. **性能優化**：避免重複解析 SQL
   ```go
   // ❌ 低效：每次都解析 SQL
   for _, user := range users {
       db.NamedExec(query, user)  // 每次都解析
   }
   
   // ✅ 高效：預編譯一次，重複使用
   stmt, _ := db.PrepareNamed(query)
   defer stmt.Close()
   for _, user := range users {
       stmt.Exec(user)  // 只執行，不解析
   }
   ```

2. **SQL 注入防護**：參數化查詢
3. **數據庫優化**：數據庫可以緩存執行計劃

**`PrepareNamed` vs `NamedExec` 選擇**：

| 場景 | 使用 | 原因 |
|------|------|------|
| 執行 1 次 | `NamedExec` | 預編譯開銷 > 直接執行 |
| 執行 2-10 次 | 兩者皆可 | 性能差異不明顯 |
| 執行 > 10 次 | `PrepareNamed` | 預編譯帶來明顯收益 |
| 批量插入 | `NamedExec` + 切片 | sqlx 內部已優化 |

### 5.4 高級查詢函數

#### `sqlx.Named(query, arg)` - 參數轉換

```go
query := "SELECT * FROM users WHERE username = :username AND email = :email"
args := map[string]interface{}{
    "username": "john",
    "email": "john@example.com",
}

// 轉換為位置參數
newQuery, newArgs, err := sqlx.Named(query, args)
// newQuery = "SELECT * FROM users WHERE username = ? AND email = ?"
// newArgs = []interface{}{"john", "john@example.com"}
```

**為什麼需要 `Named()`？**

命名參數是 sqlx 的擴展，數據庫本身只認位置參數。`Named()` 是橋樑：

```
命名參數 (:name)  →  [Named()] →  位置參數 (?)  →  [Rebind()] →  數據庫占位符 ($1)
```

**實際使用場景**：動態查詢構建

```go
func (r *userRepository) Search(filter *UserFilter) ([]User, error) {
    query := "SELECT * FROM users WHERE 1=1"
    args := make(map[string]interface{})
    
    if filter.Username != nil {
        query += " AND username ILIKE :username"
        args["username"] = "%" + *filter.Username + "%"
    }
    
    if filter.Email != nil {
        query += " AND email = :email"
        args["email"] = *filter.Email
    }
    
    // 轉換命名參數 → 位置參數
    namedQuery, namedArgs, err := sqlx.Named(query, args)
    if err != nil {
        return nil, err
    }
    
    // 適配數據庫
    namedQuery = r.db.Rebind(namedQuery)
    
    // 執行查詢
    var users []User
    err = r.db.Select(&users, namedQuery, namedArgs...)
    return users, err
}
```

#### `db.Rebind(query)` - 占位符轉換

```go
// 通用查詢（使用 ?）
query := "SELECT * FROM users WHERE id = ?"

// PostgreSQL 自動轉換為 $1
reboundQuery := db.Rebind(query)
// reboundQuery = "SELECT * FROM users WHERE id = $1"

// MySQL 保持不變
reboundQuery := db.Rebind(query)
// reboundQuery = "SELECT * FROM users WHERE id = ?"
```

**為什麼需要 `Rebind()`？**

跨數據庫兼容性：
```go
// 編寫通用代碼
query := sqlx.In("SELECT * FROM users WHERE id IN (?)", ids)

// PostgreSQL 部署
query = db.Rebind(query)  // 轉為 $1, $2, $3

// MySQL 部署
query = db.Rebind(query)  // 保持 ?, ?, ?
```

#### `sqlx.In(query, args...)` - IN 查詢展開

```go
ids := []int{1, 2, 3, 4, 5}

// 使用 sqlx.In 展開
query, args, err := sqlx.In("SELECT * FROM users WHERE id IN (?)", ids)
// query = "SELECT * FROM users WHERE id IN (?, ?, ?, ?, ?)"
// args = []interface{}{1, 2, 3, 4, 5}

// 適配數據庫
query = db.Rebind(query)
// PostgreSQL: "SELECT * FROM users WHERE id IN ($1, $2, $3, $4, $5)"

// 執行查詢
var users []User
db.Select(&users, query, args...)
```

**為什麼需要 `In()`？**

SQL 不支持數組參數：
```go
// ❌ 錯誤：SQL 不支持數組
ids := []int{1, 2, 3}
db.Select(&users, "SELECT * FROM users WHERE id IN ($1)", ids)

// ✅ 正確：使用 In() 展開
query, args, _ := sqlx.In("SELECT * FROM users WHERE id IN (?)", ids)
query = db.Rebind(query)
db.Select(&users, query, args...)
```

**完整的 IN 查詢流程**：

```go
func (r *userRepository) GetByIDs(ids []int) ([]User, error) {
    // 第 1 步：展開 IN 參數
    query, args, err := sqlx.In("SELECT * FROM users WHERE id IN (?)", ids)
    if err != nil {
        return nil, fmt.Errorf("build IN query: %w", err)
    }
    
    // 第 2 步：適配數據庫
    query = r.db.Rebind(query)
    
    // 第 3 步：執行查詢
    var users []User
    err = r.db.Select(&users, query, args...)
    if err != nil {
        return nil, fmt.Errorf("execute IN query: %w", err)
    }
    
    return users, nil
}
```

### 5.5 事務函數

#### `db.Beginx()` - 開始事務

```go
tx, err := db.Beginx()
if err != nil {
    return fmt.Errorf("begin transaction: %w", err)
}
defer tx.Rollback()  // 保底回滾

// 執行操作
tx.Exec(...)
tx.NamedExec(...)

// 提交事務
if err := tx.Commit(); err != nil {
    return fmt.Errorf("commit failed: %w", err)
}
```

**為什麼 `defer tx.Rollback()`？**

保證異常時自動回滾：
```go
tx, _ := db.Beginx()
defer tx.Rollback()  // 1. 如果 Commit 成功，Rollback 是空操作
                      // 2. 如果出錯提前返回，自動回滾

tx.Exec("INSERT INTO users ...")
if err != nil {
    return err  // 自動觸發 Rollback
}

tx.Commit()  // 成功提交
```

**事務最佳實踐包裝器**：

```go
func WithTransaction(db *sqlx.DB, fn func(*sqlx.Tx) error) error {
    tx, err := db.Beginx()
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    
    defer func() {
        if p := recover(); p != nil {
            tx.Rollback()
            panic(p)  // 重新拋出 panic
        }
    }()
    
    if err := fn(tx); err != nil {
        tx.Rollback()
        return err
    }
    
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit transaction: %w", err)
    }
    
    return nil
}

// 使用
err := WithTransaction(db, func(tx *sqlx.Tx) error {
    _, err := tx.Exec("INSERT INTO users ...")
    if err != nil {
        return err
    }
    
    _, err = tx.Exec("INSERT INTO profiles ...")
    return err
})
```

### 5.6 函數選擇決策表

| 需求 | 使用函數 | 參數類型 | 示例 |
|------|----------|----------|------|
| 查詢單行 | `Get` | 位置參數 | `db.Get(&user, query, id)` |
| 查詢多行 | `Select` | 位置參數 | `db.Select(&users, query)` |
| 簡單 DML（≤3 參數） | `Exec` | 位置參數 | `db.Exec(query, id)` |
| 複雜 DML（>3 參數） | `NamedExec` | 命名參數 | `db.NamedExec(query, user)` |
| 批量操作 | `NamedExec` | 命名參數 + 切片 | `db.NamedExec(query, users)` |
| 重複執行（>10 次） | `PrepareNamed` | 命名參數 | `stmt.Exec(user)` |
| 動態查詢 | `Named` + `Rebind` | 命名參數 | 見動態查詢示例 |
| IN 查詢 | `In` + `Rebind` | 位置參數 | `sqlx.In(query, ids)` |
| 事務操作 | `Beginx` | - | `tx.Exec(...)` |

---

## 6. Repository 模式

### 6.1 為什麼使用 Repository 模式？

**問題場景**：
```go
// ❌ 業務邏輯層直接操作 SQL
func CreateOrder(db *sqlx.DB, order *Order) error {
    // 業務邏輯和數據訪問混在一起
    tx, _ := db.Beginx()
    defer tx.Rollback()
    
    _, err := tx.Exec("INSERT INTO orders (...) VALUES (...)", ...)
    if err != nil {
        return err
    }
    
    _, err = tx.Exec("UPDATE inventory SET stock = stock - $1 WHERE product_id = $2", ...)
    if err != nil {
        return err
    }
    
    tx.Commit()
    return nil
}
```

**Repository 模式的優勢**：
1. **關注點分離**：業務邏輯與數據訪問分離
2. **可測試性**：可以 mock Repository 進行單元測試
3. **可維護性**：數據訪問邏輯集中管理
4. **可擴展性**：易於切換數據源（SQL → NoSQL）

### 6.2 定義接口

```go
package repository

type UserRepository interface {
    Create(user *User) (int, error)
    GetByID(id int) (*User, error)
    GetByEmail(email string) (*User, error)
    List(limit, offset int) ([]User, error)
    Update(user *User) error
    Delete(id int) error
}
```

### 6.3 實現 Repository

```go
package repository

import (
    "database/sql"
    "fmt"
    "time"
    
    "github.com/jmoiron/sqlx"
)

type userRepository struct {
    db *sqlx.DB
}

func NewUserRepository(db *sqlx.DB) UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) Create(user *User) (int, error) {
    query := `
        INSERT INTO users (username, email, password, created_at)
        VALUES (:username, :email, :password, :created_at)
        RETURNING id
    `
    
    user.CreatedAt = time.Now()
    user.UpdatedAt = time.Now()
    
    stmt, err := r.db.PrepareNamed(query)
    if err != nil {
        return 0, fmt.Errorf("prepare insert: %w", err)
    }
    defer stmt.Close()
    
    var id int
    err = stmt.Get(&id, user)
    if err != nil {
        return 0, fmt.Errorf("execute insert: %w", err)
    }
    
    return id, nil
}

func (r *userRepository) GetByID(id int) (*User, error) {
    var user User
    query := `
        SELECT id, username, email, password, created_at, updated_at
        FROM users
        WHERE id = $1
    `
    
    err := r.db.Get(&user, query, id)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("user not found")
        }
        return nil, fmt.Errorf("query user: %w", err)
    }
    
    return &user, nil
}

func (r *userRepository) GetByEmail(email string) (*User, error) {
    var user User
    query := `
        SELECT id, username, email, password, created_at, updated_at
        FROM users
        WHERE email = $1
    `
    
    err := r.db.Get(&user, query, email)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("user not found")
        }
        return nil, fmt.Errorf("query user: %w", err)
    }
    
    return &user, nil
}

func (r *userRepository) List(limit, offset int) ([]User, error) {
    var users []User
    query := `
        SELECT id, username, email, created_at, updated_at
        FROM users
        ORDER BY created_at DESC
        LIMIT $1 OFFSET $2
    `
    
    err := r.db.Select(&users, query, limit, offset)
    if err != nil {
        return nil, fmt.Errorf("query users: %w", err)
    }
    
    return users, nil
}

func (r *userRepository) Update(user *User) error {
    user.UpdatedAt = time.Now()
    
    query := `
        UPDATE users
        SET username = :username,
            email = :email,
            updated_at = :updated_at
        WHERE id = :id
    `
    
    result, err := r.db.NamedExec(query, user)
    if err != nil {
        return fmt.Errorf("update user: %w", err)
    }
    
    rows, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("get rows affected: %w", err)
    }
    
    if rows == 0 {
        return fmt.Errorf("user not found")
    }
    
    return nil
}

func (r *userRepository) Delete(id int) error {
    query := "DELETE FROM users WHERE id = $1"
    
    result, err := r.db.Exec(query, id)
    if err != nil {
        return fmt.Errorf("delete user: %w", err)
    }
    
    rows, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("get rows affected: %w", err)
    }
    
    if rows == 0 {
        return fmt.Errorf("user not found")
    }
    
    return nil
}
```

---

## 7. 高級查詢

### 7.1 動態查詢構建

```go
type UserFilter struct {
    Username *string
    Email    *string
    MinAge   *int
    MaxAge   *int
}

func (r *userRepository) Search(filter *UserFilter) ([]User, error) {
    query := "SELECT * FROM users WHERE 1=1"
    args := make(map[string]interface{})
    
    if filter.Username != nil {
        query += " AND username ILIKE :username"
        args["username"] = "%" + *filter.Username + "%"
    }
    
    if filter.Email != nil {
        query += " AND email = :email"
        args["email"] = *filter.Email
    }
    
    if filter.MinAge != nil {
        query += " AND age >= :min_age"
        args["min_age"] = *filter.MinAge
    }
    
    if filter.MaxAge != nil {
        query += " AND age <= :max_age"
        args["max_age"] = *filter.MaxAge
    }
    
    namedQuery, namedArgs, err := sqlx.Named(query, args)
    if err != nil {
        return nil, fmt.Errorf("build named query: %w", err)
    }
    
    namedQuery = r.db.Rebind(namedQuery)
    
    var users []User
    err = r.db.Select(&users, namedQuery, namedArgs...)
    if err != nil {
        return nil, fmt.Errorf("execute search: %w", err)
    }
    
    return users, nil
}
```

**為什麼動態查詢必須用命名參數？**

```go
// ❌ 位置參數版本 - 索引管理噩夢
query := "SELECT * FROM users WHERE 1=1"
args := []interface{}{}
index := 1

if username != "" {
    query += fmt.Sprintf(" AND username = $%d", index)
    args = append(args, username)
    index++
}
if email != "" {
    query += fmt.Sprintf(" AND email = $%d", index)
    args = append(args, email)
    index++
}

// ✅ 命名參數版本 - 清晰簡潔
query := "SELECT * FROM users WHERE 1=1"
args := make(map[string]interface{})

if username != "" {
    query += " AND username = :username"
    args["username"] = username
}
if email != "" {
    query += " AND email = :email"
    args["email"] = email
}
```

### 7.2 IN 查詢

```go
func (r *userRepository) GetByIDs(ids []int) ([]User, error) {
    query, args, err := sqlx.In("SELECT * FROM users WHERE id IN (?)", ids)
    if err != nil {
        return nil, fmt.Errorf("build IN query: %w", err)
    }
    
    query = r.db.Rebind(query)
    
    var users []User
    err = r.db.Select(&users, query, args...)
    if err != nil {
        return nil, fmt.Errorf("execute IN query: %w", err)
    }
    
    return users, nil
}
```

### 7.3 聚合查詢

```go
type UserStats struct {
    TotalUsers  int     `db:"total_users"`
    ActiveUsers int     `db:"active_users"`
    AvgAge      float64 `db:"avg_age"`
}

func (r *userRepository) GetStats() (*UserStats, error) {
    var stats UserStats
    query := `
        SELECT 
            COUNT(*) as total_users,
            COUNT(*) FILTER (WHERE is_active = true) as active_users,
            AVG(age) as avg_age
        FROM users
    `
    
    err := r.db.Get(&stats, query)
    if err != nil {
        return nil, fmt.Errorf("query stats: %w", err)
    }
    
    return &stats, nil
}
```

---

## 8. 事務處理

### 8.1 基本事務

```go
func TransferBalance(db *sqlx.DB, fromUserID, toUserID int, amount float64) error {
    tx, err := db.Beginx()
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    defer tx.Rollback()  // 如果沒有提交則回滾
    
    // 扣除發送方餘額
    _, err = tx.Exec(
        "UPDATE accounts SET balance = balance - $1 WHERE user_id = $2",
        amount, fromUserID,
    )
    if err != nil {
        return fmt.Errorf("deduct balance: %w", err)
    }
    
    // 增加接收方餘額
    _, err = tx.Exec(
        "UPDATE accounts SET balance = balance + $1 WHERE user_id = $2",
        amount, toUserID,
    )
    if err != nil {
        return fmt.Errorf("add balance: %w", err)
    }
    
    // 記錄交易
    _, err = tx.Exec(
        "INSERT INTO transactions (from_user_id, to_user_id, amount) VALUES ($1, $2, $3)",
        fromUserID, toUserID, amount,
    )
    if err != nil {
        return fmt.Errorf("insert transaction: %w", err)
    }
    
    // 提交事務
    if err = tx.Commit(); err != nil {
        return fmt.Errorf("commit transaction: %w", err)
    }
    
    return nil
}
```

### 8.2 事務包裝器

```go
func WithTransaction(db *sqlx.DB, fn func(*sqlx.Tx) error) error {
    tx, err := db.Beginx()
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    
    defer func() {
        if p := recover(); p != nil {
            tx.Rollback()
            panic(p)  // 重新拋出 panic
        }
    }()
    
    if err := fn(tx); err != nil {
        tx.Rollback()
        return err
    }
    
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit transaction: %w", err)
    }
    
    return nil
}

// 使用
err := WithTransaction(db, func(tx *sqlx.Tx) error {
    _, err := tx.Exec("INSERT INTO users ...")
    if err != nil {
        return err
    }
    
    _, err = tx.Exec("INSERT INTO profiles ...")
    return err
})
```

---

## 9. 實戰練習

### 練習 1：實現完整的 CRUD Repository

```go
// TODO: 為 Product 實體實現完整的 Repository
type Product struct {
    ID          int       `db:"id"`
    Name        string    `db:"name"`
    Description string    `db:"description"`
    Price       float64   `db:"price"`
    Stock       int       `db:"stock"`
    CreatedAt   time.Time `db:"created_at"`
    UpdatedAt   time.Time `db:"updated_at"`
}

type ProductRepository interface {
    Create(product *Product) (int, error)
    GetByID(id int) (*Product, error)
    List(limit, offset int) ([]Product, error)
    Update(product *Product) error
    Delete(id int) error
    UpdateStock(id int, quantity int) error
}
```

### 練習 2：實現分頁查詢

```go
type Pagination struct {
    Page     int
    PageSize int
    Total    int
}

type UserListResult struct {
    Users      []User
    Pagination Pagination
}

// TODO: 實現帶分頁的用戶列表查詢
func (r *userRepository) ListWithPagination(page, pageSize int) (*UserListResult, error) {
    // 1. 查詢總數
    // 2. 計算 offset
    // 3. 查詢當前頁數據
    // 4. 組裝結果
}
```

### 練習 3：實現級聯刪除

```go
// TODO: 刪除用戶時同時刪除相關數據
func (r *userRepository) DeleteWithRelations(id int) error {
    return WithTransaction(r.db, func(tx *sqlx.Tx) error {
        // 1. 刪除用戶的訂單
        // 2. 刪除用戶的地址
        // 3. 刪除用戶本身
    })
}
```

---

## 10. 最佳實踐總結

### ✅ Do's

1. **使用 Struct Tag 自動映射**
   - 減少手動字段賦值代碼
   - 提高可維護性

2. **根據場景選擇參數類型**
   - 簡單查詢（≤3 參數）→ 位置參數
   - 複雜操作（>3 參數）→ 命名參數
   - 動態查詢 → 命名參數

3. **實現 Repository 模式分離數據訪問邏輯**
   - 提高代碼可測試性
   - 關注點分離

4. **使用事務包裝器簡化事務管理**
   - 避免忘記 Rollback
   - 統一錯誤處理

5. **使用連接池配置優化性能**
   - 避免連接耗盡
   - 平衡性能與資源

6. **使用 `sql.ErrNoRows` 判斷記錄不存在**
   ```go
   if err == sql.ErrNoRows {
       return nil, fmt.Errorf("user not found")
   }
   ```

7. **檢查 `RowsAffected` 確保操作成功**
   ```go
   rows, _ := result.RowsAffected()
   if rows == 0 {
       return fmt.Errorf("update failed")
   }
   ```

### ❌ Don'ts

1. **避免在業務邏輯層直接操作 SQL**
   - 使用 Repository 模式封裝

2. **不要忘記處理 `RowsAffected` 檢查更新/刪除是否成功**
   - UPDATE/DELETE 可能影響 0 行

3. **不要在循環中執行查詢**
   - 使用 IN 查詢或 JOIN 批量處理

4. **不要忽略連接池配置**
   - 可能導致連接耗盡或性能問題

5. **不要在長事務中執行耗時操作**
   - 縮短事務持有鎖的時間

6. **不要濫用命名參數**
   - 簡單查詢用位置參數性能更好

7. **不要混用 `Get` 和 `Select`**
   - 單行用 `Get`，多行用 `Select`

---

## 11. 核心函數速查表

| 函數 | 參數類型 | 返回 | 適用場景 | 示例 |
|------|----------|------|----------|------|
| `Get` | 位置 | 單行→結構體 | 查詢單筆 | `db.Get(&user, query, id)` |
| `Select` | 位置 | 多行→切片 | 查詢多筆 | `db.Select(&users, query)` |
| `Exec` | 位置 | Result | 簡單 DML | `db.Exec(query, id)` |
| `NamedExec` | 命名 | Result | 複雜 DML/批量 | `db.NamedExec(query, user)` |
| `PrepareNamed` | 命名 | NamedStmt | 重複執行 | `stmt.Exec(user)` |
| `Named` | 命名 | 轉換後查詢 | 動態查詢 | `Named(query, args)` |
| `Rebind` | - | 適配查詢 | 跨數據庫 | `db.Rebind(query)` |
| `In` | 位置 | 展開查詢 | IN 查詢 | `In(query, ids)` |
| `Beginx` | - | Tx | 事務 | `tx, _ := db.Beginx()` |

---

## 12. 延伸閱讀

- [sqlx Documentation](https://jmoiron.github.io/sqlx/)
- [Go database/sql Tutorial](https://go.dev/doc/database/sql-injection)
- [PostgreSQL Best Practices](https://wiki.postgresql.org/wiki/Don't_Do_This)

---

**上一篇**: [Day 2 - 錯誤處理與資源管理](02-錯誤處理與資源管理.md)  
**下一篇**: [Day 4 - Goroutine 與 Channel 核心機制](../02-併發篇/04-Goroutine與Channel核心機制.md)
