### Rust 與 Async Runtime
Rust 在語言層面提供與 Async Runtime 交互的基本介面，但並不提供 Async Runtime 本身而是透過社區提供。

* Rust 提供 `Future` 與 `async/.await` 支持
* 熱門的 Async Runtime 有 `tokio`, `async-std` 等。
* Async Runtime 可以是單線程也可以是多線程
* Future 是與 Async Runtime 的交互介面, e.g. `poll` 方法, `Context` 結構體, `Poll` 枚舉
	* Async Runtime 透過 `poll` 方法執行 Future
	* Future 透過 `Poll` 枚舉告知 Async Runtime 執行情況
	* Future 通過 `Context` 結構體告訴 Async runtime 自己就緒

### Future 本質剖析
##### 為甚麼 Async function 不透過 .await 不會執行?
所有的 Async 方法/函數**返回一個 Future 的 instance**，表示會接收一個對象，並不是調用一個方法
```rust
async fn sleep() {
		// do something
}

// 等價於
fn sleep() -> impl Future<Output = ()> {
		SleepFuture{}
}
```


##### 手寫一個 Ready, Pending, YieldNow
###### 手寫 Future 
```rust
/// Ready
struct Ready;

impl Future for Ready {
    type Output = ();

    fn poll(
        self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Self::Output> {
        println!("Ready: Poll");
        Poll::Ready(()) // 直接返回 Ready
    }
}

pub fn ready() -> impl Future<Output = ()> {
    Ready {}
}

/// Pending
struct Pending;

impl Future for Pending {
    type Output = ();

    fn poll(self: std::pin::Pin<&mut Self>, cx: &mut std::task::Context<'_>) -> Poll<Self::Output> {
        println!("Pending: Poll");

        Poll::Pending
    }
}

pub fn pending() -> impl Future<Output = ()> {
    Pending {}
}

/// YieldNow
struct YieldNow {
    yielded: bool,
}

impl Future for YieldNow {
    type Output = ();

    fn poll(
        mut self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
    ) -> Poll<Self::Output> {
        println!("YieldNow: Poll");

        if self.yielded {
            return Poll::Ready(());
        }

        self.yielded = true;

        cx.waker().wake_by_ref();

        Poll::Pending
    }
}

pub fn yield_now() -> impl Future<Output = ()> {
    YieldNow { yielded: false }
}
```

###### 運行 Future
```rust
#[tokio::main]
async main() {
		println!("Before ready() .await");
    ready().await;
    println!("After ready() .await\n");
    
    println!("Before yield_now() .await");
    yield_now().await;
    println!("After yield_now() .await\n");

		println!("Before pending() .await");
    pending().await;
    println!("After pending() .await");
}
```
結果為
```text
Before ready() .await
Ready: Poll
After ready() .await

Before yield_now() .await
YieldNow: Poll  
YieldNow: Poll
After yield_now() .await

Before pending() .await
Pending: Poll
會卡在這邊不會停止
```

###### tokio::main 本質
```rust
fn main() {
		let body = async {
				println!("Before ready() .await");
		    ready().await;
		    println!("After ready() .await\n");
		}

    return tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .expect("Failed buiding the Runtime")
        .block_on(body);
}
```


### Spawn
* 生成任務 (spawn): 創建新的任務，但可能要等到運行時空閒才會輪詢 (可能要等)。
* 等待任務 (.await): 在當前任務中執行，立即輪詢，並暫停當前任務，進行子任務。
###### 簽名
```rust
pub fn spawn<F>(future: F) -> JoinHandle<F::Output>
where 
		F: Future + Send + 'static
		F::Output: Send + 'static
```
Trait bound 表明
1. `Future`: 因為 `spawn` 就是啟動異步任務，所以要傳遞一個 Future
2. `Send`: 因為 Runtime 為多線程下，Future 就需要再多個線程中傳遞
3. `'static`: 需要擁有靜態生命週期




### 參考資料
1. [how I finally understood async/await in Rust](https://hegdenu.net/posts/understanding-async-await-1/) 絕大部分參考
2. [B站 - 手寫一個 Future](https://www.bilibili.com/video/BV1qh4y1f7LK/?spm_id_from=333.337.search-card.all.click&vd_source=81381cbff2baf8dfd1b22050d029f496)

