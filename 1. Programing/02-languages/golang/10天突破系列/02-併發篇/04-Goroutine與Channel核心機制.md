# Day 4：Goroutine 與 Channel 核心機制

## 📚 學習目標

- 理解 Goroutine 調度器 (GMP 模型) 原理
- 掌握 Channel 的有緩衝/無緩衝應用
- 熟練使用 `sync` 包進行同步控制
- 實現經典併發模式

---

## 1. Goroutine 基礎

### 1.1 創建 Goroutine

```go
package main

import (
    "fmt"
    "time"
)

func sayHello(name string) {
    fmt.Printf("Hello, %s!\n", name)
}

func main() {
    // 同步調用
    sayHello("Alice")
    
    // 異步調用（啟動 Goroutine）
    go sayHello("Bob")
    
    // 主 Goroutine 需要等待，否則程序會立即退出
    time.Sleep(1 * time.Second)
}
```

### 1.2 Goroutine 特性

- **輕量級**：初始棧大小僅 2KB（線程通常 1-2MB）
- **動態棧**：棧大小可自動增長/收縮
- **由 Go Runtime 調度**：M:N 調度模型

### 1.3 GMP 調度模型

```
G (Goroutine)：代表一個 Goroutine，包含棧、指令指針等
M (Machine)：代表一個操作系統線程
P (Processor)：代表調度的上下文，數量等於 GOMAXPROCS

[G] [G] [G]     ← 待執行的 Goroutine 隊列
  ↓   ↓   ↓
 [P] [P] [P]    ← Processor（默認等於 CPU 核心數）
  ↓   ↓   ↓
 [M] [M] [M]    ← 操作系統線程
```

**調度策略**：
- **Work Stealing**：空閒 P 會從其他 P 的隊列中竊取 G
- **Hand Off**：當 M 阻塞時，P 會轉移到其他 M
- **搶占式調度**：長時間運行的 G 會被搶占（Go 1.14+）

---

## 2. Channel 基礎

### 2.1 Channel 創建與操作

```go
// 創建無緩衝 Channel
ch := make(chan int)

// 創建有緩衝 Channel
buffered := make(chan string, 10)

// 發送數據
ch <- 42

// 接收數據
value := <-ch

// 接收並檢查是否關閉
value, ok := <-ch
if !ok {
    fmt.Println("Channel closed")
}

// 關閉 Channel
close(ch)
```

### 2.2 無緩衝 Channel（同步通道）

```go
func main() {
    ch := make(chan string)
    
    go func() {
        fmt.Println("Goroutine: about to send")
        ch <- "hello"  // 阻塞，直到有接收者
        fmt.Println("Goroutine: sent")
    }()
    
    time.Sleep(2 * time.Second)
    fmt.Println("Main: about to receive")
    msg := <-ch  // 阻塞，直到有發送者
    fmt.Println("Main: received", msg)
}

// 輸出:
// Goroutine: about to send
// (2秒後)
// Main: about to receive
// Main: received hello
// Goroutine: sent
```

### 2.3 有緩衝 Channel（異步通道）

```go
func main() {
    ch := make(chan int, 3)  // 緩衝大小為 3
    
    ch <- 1
    ch <- 2
    ch <- 3
    // ch <- 4  // 會阻塞，因為緩衝已滿
    
    fmt.Println(<-ch)  // 1
    fmt.Println(<-ch)  // 2
    fmt.Println(<-ch)  // 3
}
```

### 2.4 單向 Channel

```go
// 只接收
func consumer(ch <-chan int) {
    for num := range ch {
        fmt.Println("Received:", num)
    }
}

// 只發送
func producer(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)
}

func main() {
    ch := make(chan int, 5)
    go producer(ch)
    consumer(ch)
}
```

---

## 3. Channel 操作模式

### 3.1 Select 多路複用

```go
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)
    
    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "from ch1"
    }()
    
    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "from ch2"
    }()
    
    for i := 0; i < 2; i++ {
        select {
        case msg1 := <-ch1:
            fmt.Println("Received", msg1)
        case msg2 := <-ch2:
            fmt.Println("Received", msg2)
        }
    }
}
```

### 3.2 Select 的 default 子句（非阻塞）

```go
func main() {
    ch := make(chan int, 1)
    
    select {
    case ch <- 42:
        fmt.Println("Sent")
    default:
        fmt.Println("Channel full, skipping")
    }
    
    select {
    case val := <-ch:
        fmt.Println("Received:", val)
    default:
        fmt.Println("Channel empty, skipping")
    }
}
```

### 3.3 超時控制

```go
func main() {
    ch := make(chan string)
    
    go func() {
        time.Sleep(2 * time.Second)
        ch <- "result"
    }()
    
    select {
    case res := <-ch:
        fmt.Println("Received:", res)
    case <-time.After(1 * time.Second):
        fmt.Println("Timeout!")
    }
}
```

### 3.4 range 遍歷 Channel

```go
func main() {
    ch := make(chan int, 5)
    
    go func() {
        for i := 0; i < 5; i++ {
            ch <- i
        }
        close(ch)  // 必須關閉，否則 range 會死鎖
    }()
    
    for num := range ch {
        fmt.Println("Received:", num)
    }
}
```

---

## 4. sync 包：同步原語

### 4.1 WaitGroup（等待組）

```go
import "sync"

func main() {
    var wg sync.WaitGroup
    
    for i := 0; i < 5; i++ {
        wg.Add(1)  // 計數器 +1
        
        go func(id int) {
            defer wg.Done()  // 計數器 -1
            fmt.Printf("Worker %d starting\n", id)
            time.Sleep(time.Second)
            fmt.Printf("Worker %d done\n", id)
        }(i)
    }
    
    wg.Wait()  // 阻塞，直到計數器為 0
    fmt.Println("All workers done")
}
```

**注意事項**：
- `Add()` 必須在 Goroutine 啟動前調用
- 每個 Goroutine 必須調用 `Done()`
- `Wait()` 只能調用一次

### 4.2 Mutex（互斥鎖）

```go
import "sync"

type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}

func main() {
    var wg sync.WaitGroup
    counter := &Counter{}
    
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Increment()
        }()
    }
    
    wg.Wait()
    fmt.Println("Final value:", counter.Value())  // 輸出: 1000
}
```

### 4.3 RWMutex（讀寫鎖）

```go
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func NewCache() *Cache {
    return &Cache{
        data: make(map[string]string),
    }
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()  // 讀鎖（多個 Goroutine 可同時持有）
    defer c.mu.RUnlock()
    
    val, ok := c.data[key]
    return val, ok
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()  // 寫鎖（獨占）
    defer c.mu.Unlock()
    
    c.data[key] = value
}

func (c *Cache) Delete(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    delete(c.data, key)
}
```

**選擇原則**：
- **讀多寫少**：使用 `RWMutex`
- **寫多**：使用 `Mutex`

### 4.4 Once（單次執行）

```go
var (
    instance *Database
    once     sync.Once
)

func GetDatabase() *Database {
    once.Do(func() {
        fmt.Println("Initializing database...")
        instance = &Database{
            // 初始化邏輯
        }
    })
    return instance
}

func main() {
    var wg sync.WaitGroup
    
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            db := GetDatabase()
            fmt.Println(db)
        }()
    }
    
    wg.Wait()
    // "Initializing database..." 只會打印一次
}
```

### 4.5 Cond（條件變量）

```go
type Queue struct {
    mu    sync.Mutex
    cond  *sync.Cond
    items []int
}

func NewQueue() *Queue {
    q := &Queue{}
    q.cond = sync.NewCond(&q.mu)
    return q
}

func (q *Queue) Enqueue(item int) {
    q.mu.Lock()
    defer q.mu.Unlock()
    
    q.items = append(q.items, item)
    q.cond.Signal()  // 喚醒一個等待的 Goroutine
}

func (q *Queue) Dequeue() int {
    q.mu.Lock()
    defer q.mu.Unlock()
    
    for len(q.items) == 0 {
        q.cond.Wait()  // 等待被喚醒
    }
    
    item := q.items[0]
    q.items = q.items[1:]
    return item
}
```

---

## 5. 經典併發模式

### 5.1 Worker Pool

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        time.Sleep(time.Second)
        results <- job * 2
    }
}

func main() {
    const numWorkers = 3
    const numJobs = 10
    
    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)
    
    // 啟動 workers
    for w := 0; w < numWorkers; w++ {
        go worker(w, jobs, results)
    }
    
    // 發送任務
    for j := 0; j < numJobs; j++ {
        jobs <- j
    }
    close(jobs)
    
    // 收集結果
    for r := 0; r < numJobs; r++ {
        result := <-results
        fmt.Println("Result:", result)
    }
}
```

### 5.2 Pipeline（流水線）

```go
// Stage 1: 生成數字
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

// Stage 2: 平方
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Stage 3: 打印
func print(in <-chan int) {
    for n := range in {
        fmt.Println(n)
    }
}

func main() {
    // 構建 Pipeline
    nums := generate(1, 2, 3, 4, 5)
    squared := square(nums)
    print(squared)
}
```

### 5.3 Fan-out / Fan-in

```go
// Fan-out: 將任務分發給多個 worker
func fanOut(in <-chan int, workers int) []<-chan int {
    channels := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        channels[i] = worker(in)
    }
    return channels
}

func worker(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
            time.Sleep(time.Millisecond * 100)
        }
        close(out)
    }()
    return out
}

// Fan-in: 合併多個 channel
func fanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for n := range c {
                out <- n
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}

func main() {
    in := make(chan int)
    
    go func() {
        for i := 0; i < 10; i++ {
            in <- i
        }
        close(in)
    }()
    
    // Fan-out 到 3 個 worker
    workers := fanOut(in, 3)
    
    // Fan-in 合併結果
    results := fanIn(workers...)
    
    for result := range results {
        fmt.Println(result)
    }
}
```

---

## 6. 常見併發陷阱

### 6.1 Goroutine 洩漏

```go
// ❌ 錯誤：Goroutine 永遠阻塞
func leak() {
    ch := make(chan int)
    go func() {
        val := <-ch  // 永遠等待
        fmt.Println(val)
    }()
    // ch 沒有發送數據，Goroutine 永遠不會退出
}

// ✅ 正確：使用超時或取消機制
func noLeak(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case val := <-ch:
            fmt.Println(val)
        case <-ctx.Done():
            return
        }
    }()
}
```

### 6.2 循環變量捕獲

```go
// ❌ 錯誤：所有 Goroutine 使用同一個變量
func wrong() {
    for i := 0; i < 5; i++ {
        go func() {
            fmt.Println(i)  // 可能全部打印 5
        }()
    }
    time.Sleep(time.Second)
}

// ✅ 正確：傳遞變量副本
func correct() {
    for i := 0; i < 5; i++ {
        go func(id int) {
            fmt.Println(id)
        }(i)
    }
    time.Sleep(time.Second)
}
```

### 6.3 關閉已關閉的 Channel

```go
// ❌ 錯誤：關閉已關閉的 Channel 會 panic
ch := make(chan int)
close(ch)
close(ch)  // panic!

// ✅ 正確：使用 sync.Once
var once sync.Once
var ch = make(chan int)

once.Do(func() {
    close(ch)
})
```

---

## 7. 實戰練習

### 練習 1：實現速率限制器

```go
type RateLimiter struct {
    rate   int
    ticker *time.Ticker
}

func NewRateLimiter(requestsPerSecond int) *RateLimiter {
    // TODO: 實現
}

func (r *RateLimiter) Allow() bool {
    // TODO: 實現
}
```

### 練習 2：實現併發安全的 LRU Cache

```go
type LRUCache struct {
    capacity int
    mu       sync.Mutex
    cache    map[string]*list.Element
    list     *list.List
}

func NewLRUCache(capacity int) *LRUCache {
    // TODO: 實現
}

func (c *LRUCache) Get(key string) (interface{}, bool) {
    // TODO: 實現
}

func (c *LRUCache) Put(key string, value interface{}) {
    // TODO: 實現
}
```

### 練習 3：實現批量數據處理器

```go
// 每 100ms 或累積 10 個項目時批量處理
type Batcher struct {
    size     int
    interval time.Duration
}

func (b *Batcher) Process(items <-chan string) {
    // TODO: 實現批量處理邏輯
}
```

---

## 8. 最佳實踐總結

### ✅ Do's
1. **使用 Channel 傳遞數據，使用鎖保護狀態**
2. **優先使用無緩衝 Channel（明確同步點）**
3. **由發送方關閉 Channel**
4. **使用 WaitGroup 等待 Goroutine 完成**
5. **使用 `defer` 確保解鎖**

### ❌ Don'ts
1. **不要在接收方關閉 Channel**
2. **不要向已關閉的 Channel 發送數據**
3. **不要忘記處理 Goroutine 的退出條件**
4. **不要在持有鎖時執行長時間操作**
5. **不要過度創建 Goroutine（使用 Worker Pool）**

---

## 9. 性能調優

### 9.1 設置 GOMAXPROCS

```go
import "runtime"

func init() {
    // 設置最大並行數（默認為 CPU 核心數）
    runtime.GOMAXPROCS(runtime.NumCPU())
}
```

### 9.2 監控 Goroutine 數量

```go
fmt.Println("Goroutines:", runtime.NumGoroutine())
```

---

**上一篇**: [Day 3 - 數據庫連接與 SQL 操作](../01-基礎篇/03-數據庫連接與SQL操作.md)  
**下一篇**: [Day 5 - Context 與併發模式](05-Context與併發模式.md)
