# Fork-join parallelism
---
### 簡介
Fork-join 平行化模式是透過將任務拆分成**可獨立運行的小任務**，透過 fork (分支，也就是啟動一個新的 Thread)，與 join(結合，等待 Thread 完成)，的方式達到平行化，加快整體任務完成的時間。
![[Pasted image 20241119192210.png|400]]
```rust
let mut thread_handles: Vec<JoinHandle = vec[]!;
// fork
for worklist in worklists {
	thread_handles.push(
		thread::spawn(move || process_files(worklist))
	);
}

// join
for handle in thread_handles {
	handle.join().unwrap()?;
}
```

### JoinHandle 是甚麼?
`JoinHandle` 可以透過 `join()` 方法做到以下事情:
1. 等待子線程完成: `join()` 會阻塞當前線程直到子線程完成。
2. 釋放資源: 若子線程沒完成主線程就退出，解構式不會被調用，子線程會直接被殺掉。
3. 錯誤處理: `join()` 的返回值為 `std::thread::Result<T>`，讓我們在主線程可以進一步處理，並且防止子線程異常擴散到整個進程。
	* C++: 會直接終止不做任何處理，主線程可能訪問到不該訪問的的內存。
	* Java/C#: 會拋出異常，但並不會要求主線程一定要進行處理。
4. 取得返回值: `std::thread::Result<T>` 中的 `T` 為 closure 的返回值。


### 多執行續不可變共享
```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>where
    F: FnOnce() -> T + Send + 'static,
    T: Send + 'static,
```
可以看到 `spawn` 接收的 closure 必須為 `'static` 生命期，故我們必須用獲取所有權的捕獲方式，但該變量需要被多個線程共享，且克隆又會導致很大的性能損耗時，就可以使用 atomic reference counting (`Arc`)。
```rust
fn process_files_in_parallel(filenames: Vec<String>, glossary: Arc<GigabyteMap>) {
	// ...
	for worklist in worklists {
		let glassary = Arc::clone(&glossary); // use variable shadowing
		thread_handles.push(
			thread::spawn(move || process_files(worklist, &glossary))
		);
	}
	// ...
}
```

### Rayon
Rayon 為 rust 社區中非常優秀的 join-fork 平行化 API，透過 work-stealing 技術，以動態平衡線程之間的工作負擔，讓所有 CPU 保持忙碌狀態，API 設計與 MapReduce 設計模型相似。
```rust
use rayon::prelude::*;

// 平行兩個任務
let (v1, v2) = rayon::join(f1, f2); // v1, v2 為返回值

// 平行多個任務

```

> work-stealing 技術:
> - **執行緒有自己的工作隊列**：每個執行緒會有一個本地的任務隊列，用於存放它自己需要處理的任務。
> - **空閒執行緒會竊取其他執行緒的任務**：如果一個執行緒完成了自己的任務且其隊列變空，它會嘗試從其他執行緒的隊列中竊取任務來繼續執行，避免 CPU 資源閒置。
# Channels
---
### 簡介
通道將值從一個線程送到另一個線程的單向管道。Rust 通道很快因為它是將值移動給接收方，而不是進行複製。


##### receiver 是 Iterator


##### 




# Shared mutable state
---
### 簡介
