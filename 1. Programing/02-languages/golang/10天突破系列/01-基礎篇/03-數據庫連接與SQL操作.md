# Day 3：數據庫連接與 SQL 操作（sqlx）

## 📚 學習目標

- 掌握 sqlx 的核心用法與優勢
- 理解 Struct Tag Mapping 自動綁定
- 實現 Repository 模式進行數據訪問
- 熟練使用 Named Query 和批量操作

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

---

## 4. Named Query（命名參數）

### 4.1 基本用法

```go
// 使用命名參數插入
func CreateUser(db *sqlx.DB, username, email string) (int, error) {
    query := `
        INSERT INTO users (username, email, created_at)
        VALUES (:username, :email, :created_at)
        RETURNING id
    `
    
    params := map[string]interface{}{
        "username":   username,
        "email":      email,
        "created_at": time.Now(),
    }
    
    stmt, err := db.PrepareNamed(query)
    if err != nil {
        return 0, fmt.Errorf("prepare statement: %w", err)
    }
    defer stmt.Close()
    
    var id int
    err = stmt.Get(&id, params)
    if err != nil {
        return 0, fmt.Errorf("execute insert: %w", err)
    }
    
    return id, nil
}
```

### 4.2 使用結構體作為參數

```go
type CreateUserRequest struct {
    Username string    `db:"username"`
    Email    string    `db:"email"`
    Password string    `db:"password"`
    Created  time.Time `db:"created_at"`
}

func CreateUserFromStruct(db *sqlx.DB, req *CreateUserRequest) (int, error) {
    req.Created = time.Now()
    
    query := `
        INSERT INTO users (username, email, password, created_at)
        VALUES (:username, :email, :password, :created_at)
        RETURNING id
    `
    
    stmt, err := db.PrepareNamed(query)
    if err != nil {
        return 0, fmt.Errorf("prepare statement: %w", err)
    }
    defer stmt.Close()
    
    var id int
    err = stmt.Get(&id, req)
    if err != nil {
        return 0, fmt.Errorf("execute insert: %w", err)
    }
    
    return id, nil
}
```

### 4.3 批量插入

```go
func BulkInsertUsers(db *sqlx.DB, users []CreateUserRequest) error {
    query := `
        INSERT INTO users (username, email, password, created_at)
        VALUES (:username, :email, :password, :created_at)
    `
    
    for i := range users {
        users[i].Created = time.Now()
    }
    
    _, err := db.NamedExec(query, users)
    if err != nil {
        return fmt.Errorf("bulk insert users: %w", err)
    }
    
    return nil
}
```

---

## 5. Repository 模式

### 5.1 定義接口

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

### 5.2 實現 Repository

```go
package repository

import (
    "database/sql"
    "fmt"
    
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

## 6. 高級查詢

### 6.1 動態查詢構建

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

### 6.2 IN 查詢

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

### 6.3 聚合查詢

```go
type UserStats struct {
    TotalUsers  int `db:"total_users"`
    ActiveUsers int `db:"active_users"`
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

## 7. 事務處理

### 7.1 基本事務

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

### 7.2 事務包裝器

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

## 8. 實戰練習

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

## 9. 最佳實踐總結

### ✅ Do's
1. **使用 Struct Tag 自動映射**
2. **使用 Named Query 提高可讀性**
3. **實現 Repository 模式分離數據訪問邏輯**
4. **使用事務包裝器簡化事務管理**
5. **使用連接池配置優化性能**
6. **使用 `sql.ErrNoRows` 判斷記錄不存在**

### ❌ Don'ts
1. **避免在業務邏輯層直接操作 SQL**
2. **不要忘記處理 `RowsAffected` 檢查更新/刪除是否成功**
3. **不要在循環中執行查詢（使用 IN 或 JOIN）**
4. **不要忽略連接池配置（可能導致連接耗盡）**
5. **不要在長事務中執行耗時操作**

---

## 10. 延伸閱讀

- [sqlx Documentation](https://jmoiron.github.io/sqlx/)
- [Go database/sql Tutorial](https://go.dev/doc/database/sql-injection)
- [PostgreSQL Best Practices](https://wiki.postgresql.org/wiki/Don't_Do_This)

---

**上一篇**: [Day 2 - 錯誤處理與資源管理](02-錯誤處理與資源管理.md)  
**下一篇**: [Day 4 - Goroutine 與 Channel 核心機制](../02-併發篇/04-Goroutine與Channel核心機制.md)
