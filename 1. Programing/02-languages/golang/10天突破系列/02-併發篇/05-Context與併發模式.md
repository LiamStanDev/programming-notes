# Day 5：Context 與併發模式

## 📚 學習目標

- 掌握 Context 的傳遞鏈與取消機制
- 實現 Deadline/Timeout/Cancel 模式
- 使用 errgroup 進行錯誤收集
- 實現高級併發控制模式

---

## 1. Context 基礎

### 1.1 Context 的作用

Context 用於在 Goroutine 之間傳遞：
- **取消信號**：通知 Goroutine 停止工作
- **超時控制**：設置操作的最大執行時間
- **截止時間**：設置絕對的結束時間
- **請求範圍的值**：傳遞請求 ID、用戶信息等

### 1.2 創建 Context

```go
import "context"

// 1. Background Context（根 Context）
ctx := context.Background()

// 2. TODO Context（暫時佔位符）
ctx := context.TODO()

// 3. WithCancel（手動取消）
ctx, cancel := context.WithCancel(context.Background())
defer cancel()  // 確保資源釋放

// 4. WithTimeout（超時自動取消）
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// 5. WithDeadline（指定截止時間）
deadline := time.Now().Add(10 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

// 6. WithValue（攜帶值）
ctx := context.WithValue(context.Background(), "userID", 123)
```

---

## 2. Context 取消模式

### 2.1 基本取消

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    go func() {
        time.Sleep(2 * time.Second)
        cancel()  // 2 秒後取消
    }()
    
    doWork(ctx)
}

func doWork(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("Work cancelled:", ctx.Err())
            return
        default:
            fmt.Println("Working...")
            time.Sleep(500 * time.Millisecond)
        }
    }
}
```

### 2.2 超時控制

```go
func fetchData(ctx context.Context, url string) (string, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return "", err
    }
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()
    
    body, err := io.ReadAll(resp.Body)
    return string(body), err
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    
    data, err := fetchData(ctx, "https://api.example.com/data")
    if err != nil {
        if err == context.DeadlineExceeded {
            fmt.Println("Request timeout!")
        } else {
            fmt.Println("Error:", err)
        }
        return
    }
    
    fmt.Println("Data:", data)
}
```

### 2.3 級聯取消

```go
func main() {
    ctx := context.Background()
    
    ctx1, cancel1 := context.WithCancel(ctx)
    defer cancel1()
    
    ctx2, cancel2 := context.WithCancel(ctx1)
    defer cancel2()
    
    ctx3, cancel3 := context.WithTimeout(ctx2, 5*time.Second)
    defer cancel3()
    
    go worker(ctx3, "Worker 1")
    go worker(ctx3, "Worker 2")
    
    time.Sleep(2 * time.Second)
    cancel1()  // 取消 ctx1 會級聯取消 ctx2 和 ctx3
    
    time.Sleep(1 * time.Second)
}

func worker(ctx context.Context, name string) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("%s stopped: %v\n", name, ctx.Err())
            return
        default:
            fmt.Printf("%s working...\n", name)
            time.Sleep(500 * time.Millisecond)
        }
    }
}
```

---

## 3. Context 與值傳遞

### 3.1 傳遞請求上下文

```go
type contextKey string

const (
    RequestIDKey contextKey = "requestID"
    UserIDKey    contextKey = "userID"
)

// 設置值
func WithRequestID(ctx context.Context, requestID string) context.Context {
    return context.WithValue(ctx, RequestIDKey, requestID)
}

// 獲取值
func GetRequestID(ctx context.Context) (string, bool) {
    requestID, ok := ctx.Value(RequestIDKey).(string)
    return requestID, ok
}

// 使用
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    requestID := uuid.New().String()
    ctx := WithRequestID(r.Context(), requestID)
    
    processRequest(ctx)
}

func processRequest(ctx context.Context) {
    requestID, _ := GetRequestID(ctx)
    log.Printf("[%s] Processing request", requestID)
    
    // 傳遞到下游
    fetchDataFromDB(ctx)
}

func fetchDataFromDB(ctx context.Context) {
    requestID, _ := GetRequestID(ctx)
    log.Printf("[%s] Querying database", requestID)
}
```

### 3.2 最佳實踐

**✅ 適合存儲在 Context 中的數據**：
- 請求 ID
- 用戶認證信息
- 追蹤/日誌元數據

**❌ 不應存儲的數據**：
- 可選參數（應該作為函數參數）
- 業務邏輯數據
- 可變狀態

---

## 4. errgroup：錯誤收集

### 4.1 基本用法

```go
import (
    "context"
    "fmt"
    
    "golang.org/x/sync/errgroup"
)

func main() {
    g, ctx := errgroup.WithContext(context.Background())
    
    urls := []string{
        "https://api.example.com/users",
        "https://api.example.com/posts",
        "https://api.example.com/comments",
    }
    
    for _, url := range urls {
        url := url  // 捕獲變量
        g.Go(func() error {
            return fetchURL(ctx, url)
        })
    }
    
    // 等待所有 Goroutine 完成
    if err := g.Wait(); err != nil {
        fmt.Println("Error:", err)
    }
}

func fetchURL(ctx context.Context, url string) error {
    // 模擬 HTTP 請求
    time.Sleep(time.Second)
    fmt.Println("Fetched:", url)
    return nil
}
```

### 4.2 限制並發數量

```go
func main() {
    g, ctx := errgroup.WithContext(context.Background())
    g.SetLimit(3)  // 最多 3 個併發 Goroutine
    
    for i := 0; i < 10; i++ {
        i := i
        g.Go(func() error {
            return processTask(ctx, i)
        })
    }
    
    if err := g.Wait(); err != nil {
        fmt.Println("Error:", err)
    }
}

func processTask(ctx context.Context, id int) error {
    fmt.Printf("Task %d started\n", id)
    time.Sleep(time.Second)
    fmt.Printf("Task %d completed\n", id)
    return nil
}
```

### 4.3 快速失敗模式

```go
func fetchAllData(ctx context.Context) error {
    g, ctx := errgroup.WithContext(ctx)
    
    g.Go(func() error {
        return fetchUsers(ctx)
    })
    
    g.Go(func() error {
        return fetchPosts(ctx)
    })
    
    g.Go(func() error {
        time.Sleep(100 * time.Millisecond)
        return fmt.Errorf("fetch comments failed")  // 第一個錯誤
    })
    
    // 一旦有一個 Goroutine 返回錯誤，Context 會被取消
    // 其他 Goroutine 應該檢查 ctx.Done() 並退出
    return g.Wait()
}

func fetchUsers(ctx context.Context) error {
    for i := 0; i < 10; i++ {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // 模擬工作
            time.Sleep(50 * time.Millisecond)
        }
    }
    return nil
}
```

---

## 5. 高級併發模式

### 5.1 Semaphore（信號量）

```go
import "golang.org/x/sync/semaphore"

type Downloader struct {
    sem *semaphore.Weighted
}

func NewDownloader(maxConcurrent int64) *Downloader {
    return &Downloader{
        sem: semaphore.NewWeighted(maxConcurrent),
    }
}

func (d *Downloader) Download(ctx context.Context, url string) error {
    if err := d.sem.Acquire(ctx, 1); err != nil {
        return err
    }
    defer d.sem.Release(1)
    
    // 下載邏輯
    fmt.Println("Downloading:", url)
    time.Sleep(time.Second)
    return nil
}

func main() {
    ctx := context.Background()
    dl := NewDownloader(3)  // 最多 3 個併發下載
    
    var wg sync.WaitGroup
    urls := []string{"url1", "url2", "url3", "url4", "url5"}
    
    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()
            if err := dl.Download(ctx, u); err != nil {
                fmt.Println("Error:", err)
            }
        }(url)
    }
    
    wg.Wait()
}
```

### 5.2 或-完成模式 (Or-Done)

```go
func orDone(ctx context.Context, ch <-chan interface{}) <-chan interface{} {
    out := make(chan interface{})
    go func() {
        defer close(out)
        for {
            select {
            case <-ctx.Done():
                return
            case val, ok := <-ch:
                if !ok {
                    return
                }
                select {
                case out <- val:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    return out
}

// 使用
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    
    ch := make(chan interface{})
    go func() {
        for i := 0; i < 10; i++ {
            ch <- i
            time.Sleep(500 * time.Millisecond)
        }
        close(ch)
    }()
    
    for val := range orDone(ctx, ch) {
        fmt.Println(val)
    }
}
```

### 5.3 超時重試模式

```go
func RetryWithTimeout(
    ctx context.Context,
    attempts int,
    timeout time.Duration,
    fn func(context.Context) error,
) error {
    for i := 0; i < attempts; i++ {
        ctx, cancel := context.WithTimeout(ctx, timeout)
        
        err := fn(ctx)
        cancel()
        
        if err == nil {
            return nil
        }
        
        if ctx.Err() == context.Canceled {
            return ctx.Err()
        }
        
        if i < attempts-1 {
            backoff := time.Duration(i+1) * time.Second
            select {
            case <-time.After(backoff):
            case <-ctx.Done():
                return ctx.Err()
            }
        }
    }
    
    return fmt.Errorf("max retries exceeded")
}

// 使用
func main() {
    ctx := context.Background()
    
    err := RetryWithTimeout(ctx, 3, 2*time.Second, func(ctx context.Context) error {
        return fetchDataWithContext(ctx)
    })
    
    if err != nil {
        fmt.Println("Failed:", err)
    }
}
```

---

## 6. 實戰：HTTP 服務器優雅關閉

```go
func main() {
    srv := &http.Server{
        Addr:    ":8080",
        Handler: setupRouter(),
    }
    
    // 捕獲信號
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    
    // 啟動服務器
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server failed: %v", err)
        }
    }()
    
    log.Println("Server started on :8080")
    
    // 等待信號
    <-quit
    log.Println("Shutting down server...")
    
    // 優雅關閉（最多等待 10 秒）
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("Server shutdown failed: %v", err)
    }
    
    log.Println("Server stopped")
}

func setupRouter() http.Handler {
    mux := http.NewServeMux()
    
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(2 * time.Second)  // 模擬長時間操作
        w.Write([]byte("Hello, World!"))
    })
    
    return mux
}
```

---

## 7. 實戰練習

### 練習 1：實現帶 Context 的 Worker Pool

```go
type WorkerPool struct {
    workers int
    jobs    chan Job
}

type Job struct {
    ID   int
    Task func(context.Context) error
}

func NewWorkerPool(workers int) *WorkerPool {
    // TODO: 實現
}

func (p *WorkerPool) Start(ctx context.Context) {
    // TODO: 實現 worker 邏輯，檢查 ctx.Done()
}

func (p *WorkerPool) Submit(job Job) error {
    // TODO: 實現
}
```

### 練習 2：實現帶超時的緩存

```go
type CacheItem struct {
    Value     interface{}
    ExpiresAt time.Time
}

type TTLCache struct {
    mu    sync.RWMutex
    items map[string]CacheItem
}

func (c *TTLCache) Get(ctx context.Context, key string) (interface{}, error) {
    // TODO: 檢查 ctx.Done() 和過期時間
}

func (c *TTLCache) Set(ctx context.Context, key string, value interface{}, ttl time.Duration) error {
    // TODO: 實現
}
```

### 練習 3：實現並發資料聚合

```go
type UserData struct {
    Profile  *Profile
    Posts    []Post
    Comments []Comment
}

func FetchUserData(ctx context.Context, userID int) (*UserData, error) {
    // TODO: 使用 errgroup 並發獲取三種資料
    // 如果任意一個失敗，取消其他請求
}
```

---

## 8. 最佳實踐總結

### ✅ Do's
1. **始終傳遞 Context 作為第一個參數**
2. **使用 `defer cancel()` 確保資源釋放**
3. **在長時間操作中定期檢查 `ctx.Done()`**
4. **使用 WithValue 傳遞請求範圍的元數據**
5. **使用 errgroup 簡化錯誤處理**

### ❌ Don'ts
1. **不要將 Context 存儲在結構體中**
2. **不要傳遞 nil Context（使用 context.TODO()）**
3. **不要在 Context 中存儲業務邏輯數據**
4. **不要忽略 Context 的取消信號**
5. **不要在函數簽名中省略 Context 參數**

---

## 9. 延伸閱讀

- [Go Concurrency Patterns: Context](https://go.dev/blog/context)
- [errgroup Package](https://pkg.go.dev/golang.org/x/sync/errgroup)
- [Go Concurrency Patterns](https://go.dev/talks/2012/concurrency.slide)

---

**上一篇**: [Day 4 - Goroutine 與 Channel 核心機制](04-Goroutine與Channel核心機制.md)  
**下一篇**: [Day 6 - 高級併發模式與外部通訊](06-高級併發模式與外部通訊.md)
