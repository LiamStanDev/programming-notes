# Trait Object
---
### Trait Object 是胖指針
ref: [Rust 虚表布局规则介绍 - Lancern's Home](https://lancern.xyz/posts/2024/01/rust-vtable/)
Trait object (e.g. `&mut dyn Write`)是一個胖指針，就像引用 slice 一樣，Rust 會自動將該指針轉為胖指針，裡面包含兩個指針，一個為對象指針，另一個為 vtable 指針，布局如下:
![[Pasted image 20241114212135.png|800]]
* size + alignment: 組成 std::alloc::Layout 結構
* destructor + size + alignment: 可以釋放當前胖指針所引用的對象
* vtable 剩下字段代表 Trait 所定義的函數的指針
> 當定義一個 trait object 時，Rust 會為**每一個具體類型創建一個 vtable**。


### 生命期默認 `'static`
Trait Object 生命期默認為 `'static`，一般來說不用特別指定 (除了 closure)，Rust 考量每次使用 Trait object 都要進行生命期標註，大大簡化減少程式碼的可讀性，故使期默認為 `'static`，可以避免後續編寫的困難。

##### 手動指定更短的生命期
```rust
fn foo<'a>(data: &'a dyn SomeTrait) {  }
```




### 開銷
1. CPU 進行而外操作: 相比於普通方法，CPU 需透過查找 vtable 後才能確定方法位置，多一次查找 + 一次跳轉操作。
2. CPU 無法進行分支預測(branch prediction): 虛表的間接跳轉則無法被預測，因此 CPU 無法優化跳轉流程。
3. Cache miss: 指令快取的命中率降低，進而影響性能


# Generic
---

