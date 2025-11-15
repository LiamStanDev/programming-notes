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

待完整實現...
