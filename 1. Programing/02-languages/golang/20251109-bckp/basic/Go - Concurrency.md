# goroutine介紹
---
Go語言中原生支持Coroutine(協程)，協程為用戶態線程，操作系統不知道協程的存在。
### Mulit-Thread vs. Coroutine
多線程：在線程切換時，有較大的開銷e.g. Java中為2MB
多協程：在協程切換時，有較少的開銷e.g. Go中為2KB


# 協程使用
---
### 創建一個協程
```go
func test() {
	for i := 0; i < 10; i++ {
		fmt.Println("test(), Golang", i)
		time.Sleep(time.Millisecond * 100)
	}
}

func main() {
	go test() // 開啟協程

	for i := 0; i < 10; i++ {
		fmt.Println("main(), Golang", i)
		time.Sleep(time.Millisecond * 100)
	}
}
```
> 這樣開啟的協程部會管理主進程資源釋放，所以當主進程結束協程也會結束。

### 使用協程計數器
協程計數器用於統計共有幾個協程在運作，用來主進程等待協程完成。
* package: `sync`
* struct: WaitGroup
```go
// 協程計數器
var wg WaitGroup // 全局的

func test() {
	defer wg.Done() // 計數器-1
	for i := 0; i < 10; i++ {
		fmt.Println("test(), Golang", i)
		time.Sleep(time.Millisecond * 100)
	}
	// wg.Done() // 計數器-1
}

func main() {
	wg.Add(1) // 計數器+1
	go test() // 開啟協程

	for i := 0; i < 10; i++ {
		fmt.Println("main(), Golang", i)
		time.Sleep(time.Millisecond * 100)
	}
	wg.Wait() // 等待協程計數器為0
}
```
> 上面我使用defer表示defer下面的全部語句結束完才會進行defer，我個人覺得這種寫法比較好一點。

### 設置協程佔用cpu數量
用來設定要有多少OS線程來運行用戶態線程。
* 默認值：cpu的核心數, 取得核心數 `runtime.NumCpu()`
* 設定方法：`runtime.GOMAXPROCS(7)`
```go
import "runtime"
func main() {
	var cpuNum = runtime.NumCpu()
	runtime.GOMAXPROCS(cpuNum - 1)
}
```


# Channel
---
作為語言級別提供的goroutine的通訊方式，可以使用channel在多個goroutine之間進行溝通。

