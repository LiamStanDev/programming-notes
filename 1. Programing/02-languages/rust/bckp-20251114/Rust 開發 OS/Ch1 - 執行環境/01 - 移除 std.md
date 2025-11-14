### 編寫基本程序
為了方便起見我們在 `.cargo/config.toml` 添加
```toml
[build]
target = "riscv64gc-unknown-none-elf"
```
這樣就不用每次 compile 時都要 `cargo build --target riscv64gc-unknown-none-elf`，並且 rust-analyzer 也會進行該平台的檢查。

因為我們要自己寫操作系統，我們不能使用 rust 提供的 std，因為標準庫就是依賴操作系統的，我們只能使用 core。
```rust
// src/main.rs

#![no_std]
#![no_main]

mod lang_items; // 聲明子模塊
```
* `#![no_main]`: 
	因為運行 main 時需要有 runtime 在執行 main 之前就將寄存器初始化，也就是需要有基礎的運行環境，但我們沒有所以移除。
* `lang_items`: 
	是 Rust 語意中重要的概念，實際上是一組特殊的 Traits，基本的會由編譯器直接支持 (e.g. 加減乘除運算符)，但如 panic 處理與內存釋放等在沒有操作系統的情況下需要我們自己定義。

lang_items 子模塊如下，
```rust
// src/lang_items.rs

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

### 分析程序
為了查看能否順利運行，我們可以做以下分析。

1. `file output/binary` 查看文件格式
	可以看到為 RISC-V 64 可執行程序
2. `rust-readobj -h output/binary` 查看文件頭信息
```shell
  File: target/riscv64gc-unknown-none-elf/debug/os
   Format: elf64-littleriscv
   Arch: riscv64
   AddressSize: 64bit
   ......
   Type: Executable (0x2)
   Machine: EM_RISCV (0xF3)
   Version: 1
   Entry: 0x0 # 發現 Entry 為 nullptr 地址
   ......
   }
```
3. `rust-objdump -S output/binary` 進行反匯編
	發現甚麼都沒有，表示沒有生成匯編代碼。