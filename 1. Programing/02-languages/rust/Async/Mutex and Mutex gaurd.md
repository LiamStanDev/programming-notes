### Rust Mutex 的設計
###### 其他程式語言
在其他語言互斥鎖與共享資源是分開的，通常會提供 `lock` 與 `unlock` 兩個方法，如下:
```rust
// 非真實的 Rust 代碼
fn execlusive_access(mutex: &MyMutex, protected: &MyObject) {
		mutex.lock(); // 鎖上，防止其他線程訪問
		protected.do_something(); // 對共享資源進行操作
		mutex.unlock(); // 開鎖，讓其他線程能獲得鎖
}
```
###### Rust 語言設計
`std::sync::Mutex` 將互斥鎖與受鎖保護的共享資源綑綁在一起，所以 Mutex 持有被保護的共享資源，對 Mutex 上鎖獲得 `MutexGuard`。`MutexGuard` 有以下幾個性質:
* 實現 `deref` 特徵，解引用可以訪問保護的共享資源。
* 實現 `drop` 特徵，離開作用域會 `unlock`。
* 具有內部可變性

