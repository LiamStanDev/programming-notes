# 基本介紹
---
Package 是多個Go源碼的集合，用來使程式可以重複使用。

### 包的分類
1. 標準庫（系統內置包）
	e.g. fmt, strconv, strings, sort, errors, time, encoding/json, os io等
2. 自定義包：自己寫的
3. 第三方包：別人寫的

# 建立自定義包
---
### 初始化項目
```shell
go mod init module_name
```
* 發佈到github項目通常會使用 `github.com/yourusername/myproject`

### 建立工具package
* 在該項目下添加文件夾 e.g. myproject/tools
* 在該文件夾創建go源文件 e.g. calc.go
```go
package tools

func Add(x int, y int) int { // public
	return x + y
}

func sub(x int, y int) int { // private
	return x - y
}
```
* 在main.go中使用
```go
package main

import (
	t "demo10/tools" // 給別名
	"fmt"
)

func main() {
	var res = t.Add(22, 10)

	fmt.Println(res)
}
```

#### package 規則
* 同一個文件夾下保持文件夾與內部package name相同
```go
mydir/
  mypackage/
    file1.go (package mypackage)
    file2.go (package mypackage)
  otherpackage/
    file3.go (package otherpackage)
    file4.go (package otherpackage)
```

### init()
用於導入包的初始化行為，也就是只要導入該包就會先執行該包的init()函數
#### 一些細節
* A包調用B包，調用順序為：B的init() -> A的init()


# 使用第三方包
---
* 查找包：[pkg.go.dev](https://pkg.go.dev/)

### 下載包
* 找到下載地址
* 運行以下命令
```shell
go get github.com/shopspring/decimal
```
* 會生成go.sum文件用來管理依賴
#### 下載位置 `cd $GOPATH`
```text
.
└── pkg
    ├── mod
    │   ├── github.com
    │   │   ├── shopspring
    │   │   └── ...
    ... 
	
```

### 使用包
```go
package main

import (
	"fmt"
	"github.com/shopspring/decimal"
)

func main() {
	var quentity = decimal.NewFromInt(3)

	fmt.Println(quentity)
}
```


