# 內置基本類型
---
## 類型
### boolean
* 類型：`bool`
* 預設值：`false`

### integer
* 類型：
	* 有符號：int8, int16, int32, int64, int(依主機架構而定)
	* 無符號：uint8, uint16, uint32, uint64, uint(依主機架構而定)
	* 字符：byte(ASCII, uint8), rune(Unicode, int32)
	* unintptr: 類似於void*，但是一但指針被標示為unintptr就不會被gc識別。
### float
* 類型：
	* 實數：float32, float64 (默認類型)
	* 複數：complex64 (32實部+32虛部), complex128
### string
* 類型：`string`
* 以utf-8編碼

### enum
* 沒有枚舉關鍵字，他本身就是一個整數常量
```go
const (
	x = iota // x = 0
	y = iota // y = 1
	z // z = iota, z = 2
)
```
> iota表示自增長

## 類型轉換
* go 運算時需要相同類型, e.g. `int + uint8` 不能運算
```go
var a = 10
var b = 22.4
var z = float64(a) + b
fmt.Println(z)
```

# 內置容器
---
### array
* 數組
```go
var a [5]int
a[1] 10
```

### slice
* 動態大小的數組，底層是一個數組自己本身是一個胖指針。
```go
var a = make([]int, 10) // 初始話大小為10的切片
var length = len(a)
var capacity = cap(a)
append(s, 22) // 向後添加
```

### map
* 底層維護hashmap，自己本身是一個胖指針
```go
var m = make(map[string]int, 10)
m["Liam"] = 10
m["Lucy"] = 22
delete(m, "Liam")
v, ok := m["Liam"] // v = 0, ok = false
```

# 自定義類型
---
### 使用type
* 很像rust中的newtype概念，並不是類型別名
* 下面例子該類型與int並不相容，但有int所有性質與操作。
```go
type Myint int

func fn(n int) {
	fmt.Println(n)
}

func main() {
	var a Myint = 122
	fn(a) // Compile Error
}
```
##### 類型別明
```go
type Myint = int 
```

### 使用struct
* 可以將一捆數據打包成一個對象。
* 命名規則：
	* struct名大寫表示外部能訪問，小寫表示包內可訪問
	* 字段大寫表示外部能訪問，小寫表示包內能訪問
#### 定義
```go
type Person struct {
    name string
    age int
    sex string
}
```
#### 實例化
```go
var p1 Person
var p2 = Person{}                  // 與上面相同
var p3 = Person{"Liam", 25, "Man"} // 依照參數順序，給初始值
var p4 = new(Person)               // p4 為 *Person
var p5 = &Person{}                 // p5 為 *Person (與new相同)
var p6 = &Person{                  // 指定賦值給指定參數, 可以缺省
	name: "Liam",
	age:  25,
	sex:  "Man",
}
```
##### make vs. new
* make：
	* 只能用在slice, map, chanel上
	* 返回的是該類型
	* 這三個類型底層都有維護一個數據結構，所以make初始化底層結構(全部補零)，並將該類型(胖指針)進行設定。
* new : 
	* 可以用在所有類型
	* 開闢空間並將該空間填充0
	* 返回的是指針

### 添加方法
* 可以對所有的自定義類型添加方法, e.g. struct, type
* 接收者：
	* 對象本身：值傳遞，函數中的修改不會改變對象本身，對象本身被複製到接收者上
	* 對象指針：指針傳遞，函數中的修改會改變對象本身。
```go
package main

import (
	"fmt"
)

type Person struct {
	name string
	age  int
	sex  string
}

func (p *Person) ChangeName(name string) string {
	var oldName = p.name
	p.name = name
	return oldName
}

func (p Person) Introduce() {
	fmt.Printf("My name is %v and I'm %v\n", p.name, p.sex)
}

func main() {
	var p3 = Person{"Liam", 25, "Man"} // 給初始值
	p3.ChangeName("Jack")
	p3.Introduce()
}
```

#### 該如何選擇指針接收者還是值接收者？
1. 一致性：若該結構體中有使用指針接收者，那全部都該使用指針接收者。
2. 減少拷貝：使用指針接收者可以減少拷貝的次數。
3. 減少垃圾生成：使用值接收者，不會生產堆空間，可以減少垃圾回收負擔。
4. 不確定下：使用指針接收者。

