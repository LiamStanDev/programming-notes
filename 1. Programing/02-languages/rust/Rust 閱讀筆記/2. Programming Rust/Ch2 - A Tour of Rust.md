
### Rust 開發工具
* rustup: a tool for managin Rust installation, like NVM for Node.
* cargo: Rust's general-purpose tool.
* rustc: compiler.
* rustdoc: documentation tool.


### 處理命令行參數
* `std::env::args()`: 返回命令行參數的迭代器。
* `std::str::FromStr`: 實現這個 Trait 可以使用 `from_str()` 方法，用於將字符串轉為其他類型，如下
```rust
std::str::FromStr;

let num = u64::from_str("25");
```
