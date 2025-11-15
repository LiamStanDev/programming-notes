# 附錄 B：常用標準庫速查

## 1. net/http

### 1.1 HTTP 服務器

```go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, World!")
})
http.ListenAndServe(":8080", nil)
```

### 1.2 HTTP 客戶端

```go
resp, err := http.Get("https://api.example.com")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()

body, err := io.ReadAll(resp.Body)
```

---

## 2. encoding/json

```go
// 編碼
data, err := json.Marshal(obj)

// 解碼
var obj MyStruct
err := json.Unmarshal(data, &obj)

// 美化輸出
data, err := json.MarshalIndent(obj, "", "  ")
```

---

## 3. time

```go
// 當前時間
now := time.Now()

// 格式化
formatted := now.Format("2006-01-02 15:04:05")

// 解析
t, err := time.Parse("2006-01-02", "2024-01-15")

// 時間運算
tomorrow := now.Add(24 * time.Hour)

// Sleep
time.Sleep(time.Second)
```

---

## 4. os

```go
// 讀取文件
data, err := os.ReadFile("file.txt")

// 寫入文件
err := os.WriteFile("file.txt", []byte("content"), 0644)

// 環境變量
value := os.Getenv("PATH")
os.Setenv("MY_VAR", "value")
```

---

## 5. io

```go
// 複製
_, err := io.Copy(dst, src)

// 讀取全部
data, err := io.ReadAll(reader)

// 管道
r, w := io.Pipe()
```

---

## 6. fmt

```go
// 格式化輸出
fmt.Printf("%d %s %.2f\n", 42, "hello", 3.14)

// 常用格式化動詞
%v    // 默認格式
%+v   // 添加字段名（struct）
%#v   // Go 語法表示
%T    // 類型
%t    // bool
%d    // 整數
%f    // 浮點數
%s    // 字符串
%p    // 指針
%q    // 帶引號的字符串
%x    // 十六進制

// 錯誤包裝
err := fmt.Errorf("failed to process: %w", originalErr)
```

---

## 7. strings

```go
import "strings"

// 常用操作
strings.Contains("hello", "ell")           // true
strings.HasPrefix("hello", "he")          // true
strings.HasSuffix("hello", "lo")          // true
strings.Index("hello", "ll")              // 2
strings.Split("a,b,c", ",")               // []string{"a", "b", "c"}
strings.Join([]string{"a", "b"}, "-")     // "a-b"
strings.Replace("hello", "l", "L", 1)     // "heLlo"
strings.ReplaceAll("hello", "l", "L")     // "heLLo"
strings.ToUpper("hello")                  // "HELLO"
strings.ToLower("HELLO")                  // "hello"
strings.TrimSpace("  hello  ")            // "hello"
strings.Trim("...hello...", ".")          // "hello"

// Builder（高效構建字符串）
var b strings.Builder
b.WriteString("hello")
b.WriteString(" ")
b.WriteString("world")
result := b.String()  // "hello world"
```

---

## 8. strconv

```go
import "strconv"

// 字符串轉數字
i, err := strconv.Atoi("123")           // int
f, err := strconv.ParseFloat("3.14", 64) // float64
b, err := strconv.ParseBool("true")     // bool

// 數字轉字符串
s := strconv.Itoa(123)                  // "123"
s = strconv.FormatFloat(3.14, 'f', 2, 64) // "3.14"
s = strconv.FormatBool(true)            // "true"
```

---

## 9. sync

```go
import "sync"

// Mutex
var mu sync.Mutex
mu.Lock()
// 臨界區
mu.Unlock()

// RWMutex
var rwMu sync.RWMutex
rwMu.RLock()
// 讀操作
rwMu.RUnlock()

rwMu.Lock()
// 寫操作
rwMu.Unlock()

// WaitGroup
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    // 工作
}()
wg.Wait()

// Once
var once sync.Once
once.Do(func() {
    // 只執行一次
})

// Pool
pool := &sync.Pool{
    New: func() interface{} {
        return new(Buffer)
    },
}
obj := pool.Get()
defer pool.Put(obj)
```

---

## 10. context

```go
import "context"

// 創建 context
ctx := context.Background()
ctx := context.TODO()

// WithCancel
ctx, cancel := context.WithCancel(parentCtx)
defer cancel()

// WithTimeout
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel()

// WithDeadline
deadline := time.Now().Add(10 * time.Second)
ctx, cancel := context.WithDeadline(parentCtx, deadline)
defer cancel()

// WithValue
ctx = context.WithValue(ctx, key, value)
val := ctx.Value(key)

// 檢查取消
select {
case <-ctx.Done():
    return ctx.Err()
default:
    // 繼續工作
}
```

---

## 11. errors

```go
import "errors"

// 創建錯誤
err := errors.New("something went wrong")

// 錯誤包裝
err = fmt.Errorf("failed to process: %w", originalErr)

// 錯誤檢查
if errors.Is(err, ErrNotFound) {
    // 處理特定錯誤
}

// 錯誤斷言
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println(pathErr.Path)
}
```

---

## 12. log

```go
import "log"

// 基本日誌
log.Println("info message")
log.Printf("formatted: %s", msg)
log.Fatal("fatal error")  // 輸出後調用 os.Exit(1)
log.Panic("panic error")  // 輸出後調用 panic()

// 自定義 logger
logger := log.New(os.Stdout, "[INFO] ", log.LstdFlags)
logger.Println("message")

// 設置前綴和標誌
log.SetPrefix("[APP] ")
log.SetFlags(log.Ldate | log.Ltime | log.Lshortfile)
```

---

## 13. flag

```go
import "flag"

// 定義命令行參數
var (
    port    = flag.Int("port", 8080, "server port")
    host    = flag.String("host", "localhost", "server host")
    verbose = flag.Bool("verbose", false, "verbose output")
)

func main() {
    flag.Parse()
    
    fmt.Printf("Starting server on %s:%d\n", *host, *port)
    
    // 獲取非標誌參數
    args := flag.Args()
}

// 使用: go run main.go -port=9000 -host=0.0.0.0 -verbose
```

---

## 14. regexp

```go
import "regexp"

// 編譯正則表達式
re := regexp.MustCompile(`\d+`)

// 匹配
matched := re.MatchString("abc123")  // true

// 查找
result := re.FindString("abc123def")  // "123"
results := re.FindAllString("a1b2c3", -1)  // []string{"1", "2", "3"}

// 替換
result = re.ReplaceAllString("a1b2c3", "X")  // "aXbXcX"

// 捕獲組
re = regexp.MustCompile(`(\d{4})-(\d{2})-(\d{2})`)
matches := re.FindStringSubmatch("2024-01-15")
// matches: ["2024-01-15", "2024", "01", "15"]
```

---

## 15. path/filepath

```go
import "path/filepath"

// 路徑操作
filepath.Join("dir", "subdir", "file.txt")  // dir/subdir/file.txt
filepath.Dir("/path/to/file.txt")           // /path/to
filepath.Base("/path/to/file.txt")          // file.txt
filepath.Ext("/path/to/file.txt")           // .txt
filepath.Abs("../file.txt")                 // 絕對路徑

// 遍歷目錄
filepath.Walk(".", func(path string, info os.FileInfo, err error) error {
    if err != nil {
        return err
    }
    fmt.Println(path)
    return nil
})

// 匹配模式
matched, err := filepath.Match("*.go", "main.go")  // true
```

---

## 16. 常用模式

### 16.1 讀取文件所有內容

```go
data, err := os.ReadFile("file.txt")
```

### 16.2 寫入文件

```go
err := os.WriteFile("file.txt", []byte("content"), 0644)
```

### 16.3 按行讀取文件

```go
file, _ := os.Open("file.txt")
defer file.Close()

scanner := bufio.NewScanner(file)
for scanner.Scan() {
    line := scanner.Text()
    fmt.Println(line)
}
```

### 16.4 HTTP 請求

```go
resp, err := http.Get("https://example.com")
if err != nil {
    return err
}
defer resp.Body.Close()

body, err := io.ReadAll(resp.Body)
```

### 16.5 JSON 序列化

```go
// 編碼
data, err := json.Marshal(obj)
data, err = json.MarshalIndent(obj, "", "  ")

// 解碼
var obj MyStruct
err = json.Unmarshal(data, &obj)
```

---

## 17. 延伸閱讀

- [Go Standard Library](https://pkg.go.dev/std)
- [Go by Example](https://gobyexample.com/)
