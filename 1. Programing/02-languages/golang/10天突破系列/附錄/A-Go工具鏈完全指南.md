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

## 6. 測試覆蓋率

### 6.1 生成覆蓋率報告

```bash
# 運行測試並生成覆蓋率
go test -coverprofile=coverage.out ./...

# 查看覆蓋率統計
go tool cover -func=coverage.out

# 生成 HTML 報告
go tool cover -html=coverage.out -o coverage.html
```

### 6.2 設置覆蓋率門檻

```bash
#!/bin/bash
# check-coverage.sh

COVERAGE=$(go test -coverprofile=coverage.out ./... | grep "coverage:" | awk '{print $2}' | sed 's/%//')

if (( $(echo "$COVERAGE < 80" | bc -l) )); then
    echo "Coverage $COVERAGE% is below threshold 80%"
    exit 1
fi

echo "Coverage $COVERAGE% passed"
```

---

## 7. 其他實用工具

### 7.1 golangci-lint

```bash
# 安裝
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest

# 運行
golangci-lint run

# 配置文件 .golangci.yml
linters:
  enable:
    - gofmt
    - govet
    - errcheck
    - staticcheck
    - unused
    - gosimple
    - structcheck
    - varcheck
    - ineffassign
    - deadcode

linters-settings:
  errcheck:
    check-blank: true
```

### 7.2 goimports

```bash
# 安裝
go install golang.org/x/tools/cmd/goimports@latest

# 格式化並整理 import
goimports -w .
```

### 7.3 go mod graph

```bash
# 查看依賴圖
go mod graph

# 查看為什麼需要某個依賴
go mod why github.com/some/package
```

### 7.4 go build 優化

```bash
# 減小二進制大小
go build -ldflags="-s -w" -o app main.go

# 去除調試信息
# -s: 去除符號表
# -w: 去除 DWARF 調試信息

# 查看二進制大小
ls -lh app

# 進一步壓縮（使用 upx）
upx --best --lzma app
```

---

## 8. 性能分析實戰

### 8.1 CPU Profile 分析流程

```go
// 在代碼中埋點
import (
    "os"
    "runtime/pprof"
)

func main() {
    f, _ := os.Create("cpu.prof")
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()
    
    // 執行業務邏輯
    doWork()
}
```

```bash
# 分析
go tool pprof cpu.prof

# 常用命令
(pprof) top    # 顯示前 10 個最耗時的函數
(pprof) list functionName  # 查看函數源碼
(pprof) web    # 生成圖形化視圖（需要 graphviz）
```

### 8.2 Memory Profile

```go
import (
    "os"
    "runtime"
    "runtime/pprof"
)

func main() {
    // 執行業務邏輯
    doWork()
    
    // 手動觸發 GC
    runtime.GC()
    
    // 寫入內存 profile
    f, _ := os.Create("mem.prof")
    pprof.WriteHeapProfile(f)
    f.Close()
}
```

```bash
# 分析
go tool pprof -alloc_space mem.prof  # 查看分配的內存
go tool pprof -inuse_space mem.prof  # 查看使用中的內存
```

### 8.3 HTTP 服務性能分析

```go
import _ "net/http/pprof"

func main() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    
    // 主服務...
}
```

```bash
# 訪問 pprof 端點
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
go tool pprof http://localhost:6060/debug/pprof/heap
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

---

## 9. 調試技巧

### 9.1 使用 Delve 調試

```bash
# 調試測試
dlv test -- -test.v

# 調試特定測試
dlv test -- -test.run TestName

# 附加到運行中的進程
dlv attach <pid>

# 常用命令
(dlv) break main.main       # 設置斷點
(dlv) breakpoints          # 列出所有斷點
(dlv) continue             # 繼續執行
(dlv) next                 # 下一行
(dlv) step                 # 進入函數
(dlv) print varName        # 打印變量
(dlv) args                 # 打印函數參數
(dlv) locals               # 打印本地變量
(dlv) stack                # 打印調用棧
(dlv) goroutines           # 列出所有 goroutine
(dlv) goroutine 1          # 切換到特定 goroutine
```

### 9.2 條件斷點

```bash
(dlv) break main.go:10
(dlv) condition 1 i > 100  # 只在 i > 100 時觸發斷點 1
```

---

## 10. 最佳實踐總結

### ✅ Do's
1. **定期運行 `go mod tidy` 清理依賴**
2. **使用 `golangci-lint` 進行代碼檢查**
3. **編寫測試並保持高覆蓋率**
4. **使用 pprof 分析性能瓶頸**
5. **使用 `goimports` 自動格式化代碼**

### ❌ Don'ts
1. **不要忽略 `go vet` 的警告**
2. **不要在生產環境開啟 pprof 端點（或加權限控制）**
3. **不要提交未格式化的代碼**
4. **不要跳過性能測試**

---

## 11. 延伸閱讀

- [Go Testing Documentation](https://pkg.go.dev/testing)
- [Profiling Go Programs](https://go.dev/blog/pprof)
- [Delve Debugger](https://github.com/go-delve/delve)
- [golangci-lint](https://golangci-lint.run/)
