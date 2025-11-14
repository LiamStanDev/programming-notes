# Modules
---

### 多文件多模塊
Rust 提供三種方式布置模塊:
1. modules in their own file: 本身模塊聲明在同名的 .rs 文件中。 
2. modules in their own directory with mod.rs: 子模塊聲明在父模塊的 mod.rs 中
3. modules in their own file with supplementary directory: 子模塊聲名在父模塊中，但放在子文件夾中。

```text
.
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── main.rs
│   ├── plant_structures
│   │   ├── leaves.rs
│   │   ├── mod.rs
│   │   ├── roots.rs
│   │   ├── stems
│   │   │   ├── phloem.rs
│   │   │   └── xylem.rs
│   │   └── strems.rs
│   └── spores.rs
```

```rust
// main.rs
mod plant_structures;

// spores.rs                       // Method 1
mod spores; 

// plant_sturctures/mod.rs         // Method 2
pub mod leaves;
pub mod roots;
pub mod stems;

// plant_sturctures/stems.rs       // Method 3
pub mod xylem;
pub mod phloem;
```

### Paths and Imports
Rust 採用路徑的方式來導入模塊、函數、結構體、特徵、全局常量與全局靜態變量。
* `use crate::` : 表示從該 crate 的根開始
* `use self::` : 表示從目前位置開始
* `use super::` : 表示從父位置開始
* `use ::` : 表示從外部包開始
	* 用於本地路徑與外部包路徑相同時 e.g.
		`use ::image::Pixels` (外部 crate) vs. `use self::image::Sampler`
###### use 是甚麼?
本質上 use 就是 alias，原本需要寫全路徑，但使用 use 後只需要從 use 後面開始寫，大大減少我們 coding 時的麻煩。

###### pub use 是甚麼?
use 只會作用在該模塊中，在別的模塊依舊還是需要寫全路徑，可以使用 pub use 給外部也使用這些 alias。
### Prelude
