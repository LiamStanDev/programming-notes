# Concepts
---
### Channels
Rust 發布有三個 Channels，分別為
1. stable: 每六周發布一次
2. Beta: 下一個 stable 的版本
3. Nightly: 每晚發布的版本
rustup 默認會使用每個 channel 最新的發布，但也可以指定特定的版本 e.g. `nightly-2020-07-27`。

```shell
# 下載 nightly
rustup toolchain install nightly
# 設定為當前默認
rustup default nightly # 之後 cargo 默認使用這個版本

rustup update # 會更新 default 的 channel 
```

### Toolchains
工具鏈表示 rust 編譯器的安裝，來源有
1. 官方的發布的 channels: stable, beta, and nightly
2. 官方的 archive
3. 其他來源

```rust
rustup toolchain install stable-x86_64-pc-windows-msvc
```

### Componenets
每個工具鏈都有幾個組件，有一些是必備的 (e.g. rustc)，另一些是可選配的 (e.g. Clippy)。

```shell
rustup componenet list # 查看已經安裝與可安裝的組件
```
#### 常見組件
- `rustc`: rust 編譯器與 Rustdoc.
- `cargo` 
- `rustfmt`
- `rust-std`: rust 標準庫
- `rust-docs`: 可以運行 `rustup doc` 指令
- `rust-analyzer`: lsp
- `clippy`: linter
- `miri`: rust 實驗性的 intepreter
- `rust-src`: 標準庫源代碼
- `rust-mingw`: 用於 x86_64 的 windows 工具。
- `llvm-tools`: LLMV 工具.

```shell
rustup component add llvm-tools-preview
rustup component add rust-src
```
# Cross-compliation
---
Rust 支援大量的平台，在默認安裝 tool chain 時會使用當前主機的平台，若要交叉編譯需要透過 `rustup target add` 指令完成。

```shell
rustup target list # 查看所有支援的平台
rustup target add riscv64gc-unknown-none-elf # 使用指定平台
```

