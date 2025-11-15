# 附錄 C：Go 慣例與設計哲學

## 1. Go 的核心哲學

### 1.1 簡單性（Simplicity）

> "Simplicity is complicated." - Rob Pike

Go 選擇簡單而非強大：
- 只有 25 個關鍵字
- 沒有繼承，只有組合
- 顯式錯誤處理而非異常

### 1.2 正交性（Orthogonality）

特性之間相互獨立，可以自由組合：
```go
// 接口、Goroutine、Channel 可以自由組合
type Reader interface { Read([]byte) (int, error) }
type Writer interface { Write([]byte) (int, error) }
type ReadWriter interface { Reader; Writer }
```

### 1.3 組合優於繼承

```go
// ❌ 其他語言的繼承
class Manager extends Employee { }

// ✅ Go 的組合
type Manager struct {
    Employee
    Reports []Employee
}
```

---

## 2. 命名慣例

### 2.1 包命名

- **小寫、單數、簡短**
  - ✅ `http`, `json`, `user`
  - ❌ `HTTP`, `users`, `userPackage`

### 2.2 接口命名

- **單方法接口使用 -er 後綴**
  - `Reader`, `Writer`, `Stringer`

### 2.3 變量命名

- **短作用域用短名稱**
  ```go
  // ✅ 短循環
  for i := 0; i < 10; i++ { }
  
  // ✅ 長作用域用描述性名稱
  var maxConnectionsPerUser int
  ```

---

## 3. 錯誤處理慣例

### 3.1 顯式錯誤檢查

```go
// ✅ 立即檢查錯誤
result, err := doSomething()
if err != nil {
    return fmt.Errorf("do something: %w", err)
}

// ❌ 延遲檢查
result, err := doSomething()
// ... 其他代碼
if err != nil { ... }
```

### 3.2 錯誤包裝

```go
// ✅ 使用 %w 保留錯誤鏈
return fmt.Errorf("failed to load config: %w", err)

// ❌ 使用 %v 丟失錯誤鏈
return fmt.Errorf("failed to load config: %v", err)
```

---

## 4. 接口設計原則

### 4.1 小接口

```go
// ✅ 小而專注
type Reader interface {
    Read(p []byte) (n int, err error)
}

// ❌ 過大的接口
type DataProcessor interface {
    Read() error
    Write() error
    Validate() error
    Transform() error
    // ...10 個方法
}
```

### 4.2 接受接口，返回結構

```go
// ✅ 函數接受接口，返回具體類型
func NewReader(r io.Reader) *MyReader { }

// ❌ 返回接口
func NewReader(r io.Reader) io.Reader { }
```

---

## 5. 併發慣例

### 5.1 不要通過共享內存來通信

```go
// ✅ 通過通信共享內存
ch := make(chan int)
go func() { ch <- compute() }()
result := <-ch

// ❌ 通過共享內存通信（需要鎖）
var mu sync.Mutex
var result int
go func() {
    mu.Lock()
    result = compute()
    mu.Unlock()
}()
```

### 5.2 由發送者關閉 Channel

```go
// ✅ 生產者關閉
func producer(ch chan<- int) {
    defer close(ch)
    for i := 0; i < 10; i++ {
        ch <- i
    }
}

// ❌ 消費者關閉（會導致 panic）
func consumer(ch <-chan int) {
    for range ch { }
    close(ch)  // panic: close of receive-only channel
}
```

---

## 6. Proverbs（諺語）

Rob Pike 的 Go Proverbs：

1. **Don't communicate by sharing memory, share memory by communicating**
2. **Concurrency is not parallelism**
3. **Channels orchestrate; mutexes serialize**
4. **The bigger the interface, the weaker the abstraction**
5. **Make the zero value useful**
6. **interface{} says nothing**
7. **Gofmt's style is no one's favorite, yet gofmt is everyone's favorite**
8. **A little copying is better than a little dependency**
9. **Clear is better than clever**
10. **Errors are values**

---

## 7. 設計模式在 Go 中的實現

### 7.1 單例模式

```go
var (
    instance *Database
    once     sync.Once
)

func GetDatabase() *Database {
    once.Do(func() {
        instance = &Database{
            // 初始化
        }
    })
    return instance
}
```

### 7.2 工廠模式

```go
type Storage interface {
    Save(key string, value interface{}) error
}

func NewStorage(storageType string) Storage {
    switch storageType {
    case "memory":
        return &MemoryStorage{}
    case "redis":
        return &RedisStorage{}
    default:
        return &MemoryStorage{}
    }
}
```

### 7.3 選項模式

```go
type Server struct {
    host string
    port int
    timeout time.Duration
}

type ServerOption func(*Server)

func WithHost(host string) ServerOption {
    return func(s *Server) {
        s.host = host
    }
}

func WithPort(port int) ServerOption {
    return func(s *Server) {
        s.port = port
    }
}

func NewServer(opts ...ServerOption) *Server {
    s := &Server{
        host: "localhost",
        port: 8080,
        timeout: 30 * time.Second,
    }
    
    for _, opt := range opts {
        opt(s)
    }
    
    return s
}

// 使用
server := NewServer(
    WithHost("0.0.0.0"),
    WithPort(9000),
)
```

---

## 8. 代碼組織原則

### 8.1 包的職責

**✅ 好的包設計**：
```go
// package user - 只處理用戶相關邏輯
package user

type User struct { }
type Repository interface { }
type Service struct { }
```

**❌ 不好的包設計**：
```go
// package models - 包含所有模型（職責不清）
package models

type User struct { }
type Post struct { }
type Comment struct { }
type Order struct { }
```

### 8.2 分層架構

```
項目/
├── cmd/               # 應用入口
│   └── server/
│       └── main.go
├── internal/          # 私有代碼
│   ├── domain/       # 領域模型
│   ├── repository/   # 數據訪問
│   ├── service/      # 業務邏輯
│   └── handler/      # HTTP 處理器
├── pkg/              # 公共庫
│   ├── logger/
│   └── validator/
└── config/           # 配置
```

---

## 9. 性能優化原則

### 9.1 避免不必要的內存分配

```go
// ❌ 每次都分配新的 slice
func processItems(items []Item) []Result {
    var results []Result
    for _, item := range items {
        results = append(results, process(item))
    }
    return results
}

// ✅ 預分配容量
func processItems(items []Item) []Result {
    results := make([]Result, 0, len(items))
    for _, item := range items {
        results = append(results, process(item))
    }
    return results
}
```

### 9.2 使用 strings.Builder

```go
// ❌ 字符串拼接效率低
func buildString(parts []string) string {
    result := ""
    for _, part := range parts {
        result += part
    }
    return result
}

// ✅ 使用 Builder
func buildString(parts []string) string {
    var b strings.Builder
    for _, part := range parts {
        b.WriteString(part)
    }
    return b.String()
}
```

### 9.3 複用對象（sync.Pool）

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func processData(data []byte) {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()
    
    // 使用 buffer
    buf.Write(data)
}
```

---

## 10. 安全性原則

### 10.1 避免 SQL 注入

```go
// ❌ 不安全
query := fmt.Sprintf("SELECT * FROM users WHERE id = %s", userID)

// ✅ 使用參數化查詢
query := "SELECT * FROM users WHERE id = $1"
db.Query(query, userID)
```

### 10.2 驗證用戶輸入

```go
func validateEmail(email string) error {
    if email == "" {
        return errors.New("email is required")
    }
    
    re := regexp.MustCompile(`^[a-z0-9._%+\-]+@[a-z0-9.\-]+\.[a-z]{2,}$`)
    if !re.MatchString(email) {
        return errors.New("invalid email format")
    }
    
    return nil
}
```

### 10.3 避免敏感信息洩露

```go
// ❌ 不要在日誌中記錄敏感信息
log.Printf("User login: email=%s, password=%s", email, password)

// ✅ 只記錄必要信息
log.Printf("User login attempt: email=%s", email)
```

---

## 11. 測試原則

### 11.1 表格驅動測試

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"positive numbers", 1, 2, 3},
        {"negative numbers", -1, -2, -3},
        {"mixed", -1, 2, 1},
        {"zero", 0, 0, 0},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.want {
                t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

### 11.2 使用接口進行測試

```go
// 生產代碼
type UserRepository interface {
    GetByID(id int) (*User, error)
}

type UserService struct {
    repo UserRepository
}

// 測試代碼
type mockUserRepository struct {
    user *User
    err  error
}

func (m *mockUserRepository) GetByID(id int) (*User, error) {
    return m.user, m.err
}

func TestUserService_GetUser(t *testing.T) {
    mockRepo := &mockUserRepository{
        user: &User{ID: 1, Name: "Alice"},
    }
    
    service := &UserService{repo: mockRepo}
    user, err := service.GetUser(1)
    
    // 斷言...
}
```

---

## 12. 實用技巧

### 12.1 使用 iota 定義常量

```go
type Status int

const (
    StatusPending Status = iota  // 0
    StatusActive                 // 1
    StatusInactive              // 2
    StatusDeleted               // 3
)

const (
    _  = iota  // 跳過 0
    KB = 1 << (10 * iota)  // 1024
    MB                      // 1048576
    GB                      // 1073741824
)
```

### 12.2 使用空 struct 節省內存

```go
// Set 實現
type Set map[string]struct{}

func (s Set) Add(key string) {
    s[key] = struct{}{}  // struct{} 不佔用內存
}

func (s Set) Contains(key string) bool {
    _, exists := s[key]
    return exists
}
```

### 12.3 使用 embed 嵌入資源

```go
import _ "embed"

//go:embed config.json
var configData []byte

//go:embed templates/*
var templates embed.FS

func loadConfig() {
    // 使用 configData
}
```

---

## 13. 總結：Go 之道

1. **簡單勝於複雜** - 寫簡單的代碼，解決複雜的問題
2. **組合勝於繼承** - 使用嵌入和接口組合功能
3. **顯式勝於隱式** - 錯誤處理、類型轉換都要顯式
4. **並發不是並行** - 理解 Goroutine 和並行的區別
5. **接口要小** - 單一職責，易於實現
6. **錯誤是值** - 像處理其他值一樣處理錯誤
7. **少即是多** - 語言特性少，但功能強大

---

## 14. 推薦閱讀

- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
- [Go Proverbs](https://go-proverbs.github.io/)
- [Practical Go](https://dave.cheney.net/practical-go/presentations/qcon-china.html)
- [Uber Go Style Guide](https://github.com/uber-go/guide)

---

**恭喜！你已經完成了 Go 語言 10 天特訓的所有內容！** 🎉
