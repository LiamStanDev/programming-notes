# 附錄 A：Go 工具鏈完全指南

## 📚 本篇內容

- `go test` 測試工具
- `go tool pprof` 性能分析
- `go vet` 靜態分析
- `go generate` 代碼生成
- `delve` 調試工具

---

## 1. go test

### 1.1 基本測試

```go
// user_test.go
package user

import "testing"

func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name  string
        email string
        want  bool
    }{
        {"valid email", "user@example.com", true},
        {"invalid email", "invalid", false},
        {"empty email", "", false},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := ValidateEmail(tt.email)
            if got != tt.want {
                t.Errorf("ValidateEmail(%q) = %v, want %v", tt.email, got, tt.want)
            }
        })
    }
}
```

### 1.2 基準測試

```go
func BenchmarkValidateEmail(b *testing.B) {
    for i := 0; i < b.N; i++ {
        ValidateEmail("user@example.com")
    }
}
```

執行：
```bash
go test -bench=. -benchmem
```

---

## 2. go tool pprof

### 2.1 CPU 性能分析

```go
import (
    "os"
    "runtime/pprof"
)

func main() {
    f, _ := os.Create("cpu.prof")
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()
    
    // 業務代碼
}
```

分析：
```bash
go tool pprof cpu.prof
```

### 2.2 內存分析

```go
import (
    "os"
    "runtime/pprof"
)

func main() {
    // 業務代碼
    
    f, _ := os.Create("mem.prof")
    pprof.WriteHeapProfile(f)
    f.Close()
}
```

---

## 3. go vet

靜態代碼分析：
```bash
go vet ./...
```

常見問題檢查：
- Printf 格式字符串錯誤
- 未使用的變量
- 錯誤的結構體 tag
- 死鎖風險

---

## 4. go generate

### 4.1 代碼生成

```go
//go:generate stringer -type=Status
type Status int

const (
    Pending Status = iota
    Active
    Inactive
)
```

執行：
```bash
go generate ./...
```

---

## 5. delve 調試

### 5.1 安裝

```bash
go install github.com/go-delve/delve/cmd/dlv@latest
```

### 5.2 使用

```bash
# 調試程序
dlv debug main.go

# 設置斷點
(dlv) break main.main
(dlv) continue
(dlv) print myVar
```

---

待完整實現...
