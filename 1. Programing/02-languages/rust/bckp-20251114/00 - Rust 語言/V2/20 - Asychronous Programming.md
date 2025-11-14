* Thread 的 stack 通常會占用 100 KiB，而異步可以用一個 Thread 做到交錯執行獨立的非同步任務。
* 非同步 Rust 與同步 Rust 很相似，但需要額外處理會阻塞的操作。
* 標準庫提供的操作系統調用為同步的版本，操作系統處理時會進行阻塞等待
* 異步需要使用非同步版本的 I/O 程式庫

* Future: 
```rust
trait Future {
	type Output;
	fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>
}

enum Poll<T> {
	Ready(T),
	Pending,
}
```
* 非同步與同步版本的函數簽名基本上一樣，但是返回值會包一層 `Future`，如
```rust
fn read_to_string<'a>(&'a mut self, buf: &'a mut String) -> impl Future<Output = Result<usize>> + 'a; // 這邊表示該 future 生命期與 self, buf '一樣長'
```
* future 只能透過 poll 來敲打，直到吐出值。
* rust 輪詢只會發生在第一次調用與當 future 值得再次被輪詢時才進行，效率是很高的。
* 輪詢會從一個 await 到下一個 await，具體幾次要看其子 future 的行為。
* future 會記錄下一個 poll 應該在哪裡恢復執行、以及還原執行環境 (變量、參數)
* `async fn` 比普通 `fn` 多了暫停與恢復的功能。
* 在非同步函數中取得 future 的值只要 await 就好，但是該函數也返回 future，會一直往上拋總會有人要等待最上層的 future。
	* 會在同步函數中使用 `block_on` 接收 future，他會輪巡直到產生值(阻塞)，為非同步到同步的橋樑。
		* `block_on` 在收到 `Poll::Pending` 會阻塞線程進入休眠，並不會一直不斷進行 `poll` (效能高)。

### 在等待時做其他的工作
目前透過 `block_on` 我們可以在同步函數中等待一個 future 完成，但是這並沒有比同步的方式更好，因為我們並沒有讓 CPU 在等待時間做其他的事情。`async_std` 的 unstable feature 提供 `spawn_local`，可以在當前線程下將**添加異步任務至任務池**中，當一個任務 Pending 時就會切到另外一個任務。
* `spawn_local` 簽名
```rust
pub fn spawn_local<F, T>(future: F) -> JoinHandle<T> 
where
    F: Future<Output = T> + 'static,
    T: 'static,
```
* JoinHandle 是一個 Future，其值為參數 future 的結果值
* 閉包生命期必須為 `'static`

##### Example: 建立一個異步發送多個 request 的 HTTP client
```rust
use async_std::io::{ReadExt, WriteExt};

// 添加多個 request 至異步任務池中
async fn many_request(requests: Vec<(String, u16, String)>) -> Vec<std::io::Result<String>> {
    use async_std::task;
    let mut handles = vec![]; // 存放 async_std::task::JoinHandle，本身為 future
    for (host, port, path) in requests {
        handles.push(task::spawn_local(cheapo_owning_request(host, port, path))); // 添加任務
    }

    let mut results = vec![];
    for handle in handles {
        results.push(handle.await); // 透過 poll JoinHandle 獲取最終值
    }

    results
}

async fn cheapo_owning_request(host: String, port: u16, path: String) -> std::io::Result<String> {
    use async_std::net;
    eprintln!("[conect] request: {}:{}, path: {}", host, port, path);
    let mut socket = net::TcpStream::connect((&host as &str, port)).await?;

    let request = format!("GET {} HTTP/1.1\r\nHost: {}\r\n\r\n", path, host);
    socket.write_all(request.as_bytes()).await?;
    eprintln!("[write_all]");
    socket.shutdown(std::net::Shutdown::Write)?;

    let mut response = String::new();
    socket.read_to_string(&mut response).await?;
    eprintln!("[read_to_string]");
    Ok(response)
}

fn main() {
    let requests = vec![
        ("example.com".to_string(), 80, "/".to_string()),
        ("www.red-bean.com".to_string(), 80, "/".to_string()),
        ("en.wikipedia.org".to_string(), 80, "/".to_string()),
    ];

    let results = async_std::task::block_on(many_request(requests));
    for result in results {
        match result {
            Ok(response) => println!("{}", response),
            Err(err) => eprintln!("error: {}", err),
        }
    }
}
```


### async block
是一段包裹在 `async {}` 中的代碼，使你可以在區塊內使用 `.await` 運算子，為普通區塊陳述式的異步版本，如下
```rust
let serve_one: Future<Output = Result<...>> = async {
	let listener = net::TcpListener::bind("localhost:8080").await?;
	let (mut socket, _addr) = listener.accept().await?;
	// some code...
};
```
* async block 會返回最後一個運算式值 Future。
* 可以使用 `move` 來修飾 async block，使其以獲取所有權的方式捕獲外部變數。
```rust
```