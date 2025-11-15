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

待完整實現...
