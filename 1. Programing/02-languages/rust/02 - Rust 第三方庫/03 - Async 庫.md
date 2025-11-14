
> In [computer programming](https://en.wikipedia.org/wiki/Computer_programming "Computer programming"), the **async/await pattern** is a syntactic feature of many [programming languages](https://en.wikipedia.org/wiki/Programming_language "Programming language") that allows an [asynchronous](https://en.wikipedia.org/wiki/Asynchrony_\(computer_programming\) "Asynchrony (computer programming)"), [non-blocking](https://en.wikipedia.org/wiki/Non-blocking_I/O "Non-blocking I/O") [function](https://en.wikipedia.org/wiki/Subroutine "Subroutine") to be structured in a way similar to an ordinary synchronous function. It is semantically related to the concept of a [coroutine](https://en.wikipedia.org/wiki/Coroutine "Coroutine") and is often implemented using similar techniques, and is primarily intended to provide opportunities for the program to execute other code while waiting for a long-running, asynchronous task to complete, usually represented by [promises](https://en.wikipedia.org/wiki/Futures_and_promises "Futures and promises") or similar [data structures](https://en.wikipedia.org/wiki/Data_structure "Data structure").

# tokio
---

#### 不適合使用 tokio 的時機
* **Speeding up CPU-bound**: tokio 善於處裡應用存在 IO-bound，若是 CPU 密集應用應該使用 `rayon`
	* 註：兩者是可以混合使用的
* **Reading lot of files**: 雖然感覺是非同步 IO 適合的場景，但是 OS 通常不提供非同步檔案 API，所以使用 tokio 不會有額外的性能增加
* **Sending a single web request**: 若只是單獨發送那應該使用阻塞的 API，非同步適用於需要同時執行多個操作時才有意義。

#### Compile-time green thread
Rust 使用 async/await 提供非同步程式設計能力，其本質是：
> 在編譯期間產生的輕量任務（非同步狀態機），為 stackless coroutine 也就是 green thread 的一種實作（非搶占協作型、輕量級、多任務）。

```rust
async fn say_hello() {

}

#[tokio::main]
async fn main() {
	let op = say_world(); // op: Future<Output = ()>
	op.await;
}
```
`async fn` 雖然看起來與一般函數沒有什麼不同，但是呼叫並不會導致函數體被執行，而是返回一個值 (`Future`) 用來表示操作，概念上與無參數閉包相同。要執行 `async fn`  需要使用 `.await`。

#### Async `main` function
`async fn` 必須要被 runtime 執行。異步 runtime 包含：
* 非同步任務排程 (Scheduler)
* 提供 evented I/O (事件驅動 IO), timer, etc.

```rust
#[tokio::main]
async fn main() {
    println!("hello");
}

// 等價於
fn main() {
    let mut rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        println!("hello");
    })
}
```

### Spawn

#### `'static` Bound
* This means that the spawned task must **not contain any references to data owned outside the task**.
* 需要使用 `move` 讓捕獲到的變量被移動到閉包內部

#### Send bound
* This allows the Tokio runtime to move the tasks between threads while they are suspended at an `.await`
* Tasks are `Send` when **all** data that is held **across** `.await` calls is `Send`
編譯錯誤案例：
```rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        let rc = Rc::new("hello");

        // `rc` is used after `.await`.
        // the task's state.
        yield_now().await;

        println!("{}", rc);
    }); // rc drop here!
}
```
因為 `rc` 會在 .await 之後使用，tokio runtime 在 `.await` 之後可能將其放到其他執行續中，故所有狀態必須被保存到任務中，若保存的所有狀態都是 `Send`，表示任務本身可以跨執行續移動。
正確版本：
```rust
#[tokio::main]
async fn main() {
	tokio::spawn(async {
        {
            let rc = Rc::new("hello");
            println!("{}", rc);
        } // rc drop here!
		yield_now().await;
	});
}
```

> 注意 rust compile 尚未不支持使用 mem::drop() 來提前釋放，這不是邏輯錯誤而是 compiler 問題，故目前還是建議使用 scope 來限定存活期間。


### Do hold mutex across 


##### What is spurious wake-ups?
在 rust 的 async 模型中：
1. Future 只有在被 `poll()`  才會推進。
2. `poll()` 返回 `Poll::Ready(val)` 或者 `Poll::Pending`
3. 當 `poll()` 返回 `Poll::Pending` 時理論上只有準備好時才會被喚醒，但實際上：
	* 驅動程式可能會多次呼叫 `wake()`
	* 某些計時器 API 可能會多次觸發事件
	* 某些任務複雜任務導致多次喚醒

###### 如何避免 spurious wake-ups
在 Tokio 中會用 TaskFuture 把 Future 與其上一次 poll 的狀態包裹在一起，避免 `Ready` 後重複執行 `poll` 導致未定義錯誤與 CPU 空轉。
```rust
struct TaskFuture {
	// future 本身
    future: Pin<Box<dyn Future<Output = ()> + Send>>,
    // 上一次 poll 的結果
    poll: Poll<()>,
}
```





