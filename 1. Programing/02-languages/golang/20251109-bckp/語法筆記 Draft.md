### 可變數量參數
```go
package main

import "fmt"

func sum(nums ...int) {
	fmt.Println(nums, "")
	total := 0
	for _, num := range nums {
		total += num
	}
	fmt.Println(total)
}

func main() {
	sum(1, 2)
	sum(1, 2, 5)
	sum(1, 2, 2, 3, 4, 5)

	nums := []int{1, 2, 3, 4}
	sum(nums...)
}

```

### 類型斷言
**類型斷言（Type Assertion）** 是一種用於將接口類型（`interface`）的值轉換為具體類型的工具。它主要用於檢查或提取存儲在接口中的具體類型數據。
```go
value, ok := x.(T)
```
- **`x`**：表示一個接口類型的變數。
- **`T`**：表示要斷言的具體類型。
- **`value`**：如果斷言成功，將存儲轉換後的值。
- **`ok`**：布林值，表示斷言是否成功。

### Channel
#### 基本使用
```go
package main

import "fmt"

func main() {
	var messages = make(chan string)

	go func() { messages <- "ping" }()

	var msg = <-messages
	fmt.Println(msg)
}
```


#### Channel Direction
可以指定通道只讀或只寫
```go
func ping(pings <-chan string, pongs chan<- string) {}
```
* `pings` 為只讀 channel
* `pongs` 為只寫 channel
#### timeout
可以透過 `select` + `channel` 實現超時等待
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	c1 := make(chan string, 1)
	go func() {
		time.Sleep(2 * time.Second)
		c1 <- "result 1"
	}()

	select {
	case res := <-c1:
		fmt.Println(res)
	case <-time.After(1 * time.Second):
		fmt.Println("timeout 1")
	}
}
```
> `time.After()` 返回 `<-chan time.Time`

#### non-blocking channel
可以透過 `select` + `default` + `channel` 實現非阻塞通道
```go
select {
    case msg := <-messages:
        fmt.Println("received message", msg)
    default:
        fmt.Println("no message received")
    }
```

#### closing channel
關閉通道本質上是傳遞 close 信號給通道另外一端。
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	jobs := make(chan int, 5)
	done := make(chan bool)

	go func() {
		for {
			j, more := <-jobs // more is used to check chan status
			if more {
				fmt.Println("received job", j)
			} else {
				fmt.Println("received all jobs")
				done <- true
				return
			}
		}
	}()

	for j := 1; j <= 3; j++ {
		jobs <- j
		fmt.Println("send job", j)
		time.Sleep(1 * time.Second)
	}
	close(jobs)
	fmt.Println("send all jobs")
	<-done
}
```

#### range over channel
```go
package main

import "fmt"

func main() {
	queue := make(chan string, 2)
	queue <- "one"
	queue <- "two"
	close(queue)

	for elem := range queue {
		fmt.Println(elem)
	}
}
```

### Timer and Ticker
#### Timer
* 用途：用於在指定時間後執行一次操作。
* 行為：當計時器到期時，會向其 C 通道發送一個值（表示時間已到）。
* 主要方法：
	* time.NewTimer(d time.Duration)：創建一個新的 Timer，在 d 時間後觸發。
	* t.Stop()：停止計時器，避免它觸發。
	* t.Reset(d time.Duration)：重置計時器，使用新的持續時間。
範例：
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	// 創建一個 2 秒的 Timer
	timer := time.NewTimer(2 * time.Second)

	fmt.Println("Timer started...")

	// 等待 Timer 到期
	<-timer.C
	fmt.Println("Timer expired!")
}
```
Stop 方法的使用：
```go
func main() {
	timer := time.NewTimer(5 * time.Second)

	go func() {
		<-timer.C
		fmt.Println("Timer expired!")
	}()

	// 提前停止 Timer
	time.Sleep(2 * time.Second)
	if timer.Stop() {
		fmt.Println("Timer stopped before expiration!")
	}
```
#### Ticker
* 用途：用於定期執行某些操作。
* 行為：以固定的時間間隔，向其 C 通道發送時間值。
* 主要方法：
	* time.NewTicker(d time.Duration)：創建一個新的 Ticker，每隔 d 時間觸發一次。
	* t.Stop()：停止 Ticker，釋放相關資源。
範例：
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ticker := time.NewTicker(500 * time.Millisecond)
	done := make(chan bool)

	go func() {
		for {
			select {
			case <-done:
				return
			case t := <-ticker.C:
				fmt.Println("Tick at", t)
			}
		}
	}()

	time.Sleep(1600 * time.Millisecond)
	ticker.Stop()
	done <- true
	fmt.Println("Ticker stopped")
}
```

