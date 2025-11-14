# 基本使用
### 定義接口
```go
type Usb interface {
	plugin()
	unplug()
}
```
* 不用寫func關鍵字

### 實現接口
```go
type Phone struct {
	Name string
}

func (p Phone) plugin() {
	fmt.Println("Plugin...")
}

func (p Phone) unplug() {
	fmt.Println("Unplug...")
}
```
* 實現該接口要實現該接口的所有方法
* 類不用(不能)顯示標注接口名稱

### 使用接口
```go
func RunUsb(p Usb) {
	p.plugin()
	p.unplug()
}

func main() {
	var p = Phone{"LiamPhone"}
	RunUsb(p)
}
```
> 注意：若實現接口的為指針接收者，無法使用上面的方式要使用以下方法
```go
type Phone struct {
	Name string
}

func (p *Phone) plugin() {
	fmt.Println("Plugin...")
}

func (p *Phone) unplug() {
	fmt.Println("Unplug...")
}

func main() {
	var p = &Phone{"LiamPhone"} // 需要使用指針
	RunUsb(p)
}
```

# 空接口
---
空接口中不用實現任意類型，表示所有類型均實現該接口，與Java, C#中的Object類型概念相似，但本身類型不會顯示為interface{}, 而是原本類型。
## 基本使用
### 用於動態變量
表示該變量可以接收所有數據類型。
```go
type B interface{}


func main() {
	var a interface{} // 方式一
	a = 20
	fmt.Printf("value: %v, type: %T\n", a, a)
	a = "Hi"
	fmt.Printf("value: %v, type: %T\n", a, a)
	a = true
	fmt.Printf("value: %v, type: %T\n", a, a)

	var b B // 方式二
	b = 20
	fmt.Printf("value: %v, type: %T\n", b, b)
	b = "Hi"
	fmt.Printf("value: %v, type: %T\n", b, b)
	b = true
	fmt.Printf("value: %v, type: %T\n", b, b)
}
```
#### 仿照javascript寫法（好玩）
* 將any作為interface{}的類型別名。
```go
type any = interface{}

func main() {
	var a any
	a = 20
	fmt.Printf("value: %v, type: %T\n", a, a)
	a = "Hi"
	fmt.Printf("value: %v, type: %T\n", a, a)
	a = true
	fmt.Printf("value: %v, type: %T\n", a, a)
}

```

### 用於函數動態類型
```go
func Show(v interface{}) {
	fmt.Printf("value: %v, type: %T\n", v, v)
}
```

### 將slice與map可接收多類型
#### slice

```go
var s = make([]interface{}, 0)
s = append(s, "Hi")
s = append(s, true)
for i := range s {
	fmt.Println(i)
}
```

#### map
```go
var m = make(map[string]interface{})
m["Hi"] = 22
m["Liam"] = true

for k, v := range m {
	fmt.Println(k, v)
}
```


## 類型斷言
有以下功能:
1. 用來判斷該空接口是否屬於某個指定類型,
2. 空接口類型轉回原始類型。

### 類型判斷
#### 用於if-else語句
```go
var a interface{}
a = "Hello, world"
v, ok := a.(string) // 類型斷言

if (ok) { // 為指定類型
	fmt.Println("a is a string type, a: %v", v) // v 就是a的值
} else { // 不為指定類型 
	fmt.Println("Type assertion fail.")
}
```

#### 用於switch語句
```go
	var a interface{}
	a = 22

	switch a.(type) {
	case string:
		fmt.Println("x is a string")
	case int:
		fmt.Println("x is int")
	default:
		fmt.Println("wrong type")
	}
```

#### 轉回原始類型
 概念同C++的dynamic_cast
```go
package main

import "fmt"

type Usb interface {
	start()
	stop()
}

type Phone struct {
	Name string
}

func (p Phone) SayName() {
	fmt.Println(p.Name)
}

func (p Phone) start() {
	fmt.Println("Phone start")
}

func (p Phone) stop() {
	fmt.Println("Phone stop")
}

func main() {
	var usb Usb = Phone{"Liam"}

	var v, ok = usb.(Phone)
	if ok {
		// usb.SayName() // Error
		v.SayName()
	}
}
```


# 接口繼承
---
```go
type Usb interface {...}
type TypeC interface {...}

type Phone  interface {
	Usb 
	TypeC
}
```