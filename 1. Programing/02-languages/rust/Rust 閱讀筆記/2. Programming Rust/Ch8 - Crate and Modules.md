# Crate
---
Rust 程序由 crates 組成，每一個 crates 都是一個完整的編譯單元: 一個庫或者可執行文件的所有代碼 + 相關測試 + 示例 + 工具 + 配置 + 其他內容。

##### 依賴其他 Crates
可以在 Crates.io 中找到 crate，用於存放社區開源的 crate 的網站，crate.io 上面每個 crate 頁面都會顯示他們的 README.md 文件與到文檔和源代碼的鏈接。
* 進行 `cargo build` 時會從 crates.io 上下載指定版本的源碼，逐一讀取他們的 Cargo.toml 文件，並繼續下載依賴，遞迴操作。
* Rust 有極強的版本兼容性，在 Cargo.toml 文件指定 `edition = 2018`，表示使用 rustc 進行編譯，每個 crate 都會使用其指定的版本進行編譯，不會統一使用相同的版本編譯器，所以可以不同版本混編，所以 2005 也可以與 2008 編譯運行。


### 構建配置

| command line            | Cargo.toml section used |
| ----------------------- | ----------------------- |
| `cargo build`           | `[profile.dev]`         |
| `cargo build --release` | `[profile.release]`     |
| `cargo test`            | `[profile.test]`        |
> 其他請見 [Cargo 文檔](https://doc.rust-lang.org/cargo/reference/manifest.html)

# Module
---
**Crate 決定了項目之間的代碼共享，而模塊則為項目的內部的代碼組織**，模塊扮演了 Rust 中的命名空間(一種包含函數、類型、常量等內容的容器)，這些模塊組成 Rust 的程序或庫。

一個模塊如以下:
```rust
mod spore { // private mod
    pub struct Spore{} // public for all

    pub fn produce_spore(factory: &mut Sporangium) -> Spore { // public for all
    }

    pub(crate) fn genes(spore: &Spore) -> Vec<Gene> { // public only inside this crate
    }

    fn recombine(parent: &mut Cell) { // private
    }
}
```
* `pub`: 表示模塊外是可見，沒有標註為私有。
* `pub(crate)`: 表示在 crate 中都可見，但是在別的 crate 中不可見。
* `pub(super)`: 表示在父模塊可見。
* `pub(in crate::path::subpath)`: 可以指定在某模塊可見。
* 目前這個 spore 模塊是不可見的，想在 crate 外不使用需要使用 `pub mod`。

### 多文件多模塊
我們可以在一個文件寫上大量複雜的模塊關係，但這樣非常痛苦，我們也可以分文件編寫。

