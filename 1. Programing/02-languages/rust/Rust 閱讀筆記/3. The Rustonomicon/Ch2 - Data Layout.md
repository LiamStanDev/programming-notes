# Repr(Rust)
---
Rust 提供了以下幾個方式來佈局複合類型:
- structs (named product types): 命名複合類型
- tuples (anonymous product types): 匿名複合類型
- arrays (homogeneous product types): 同值複合類型
- enums (named sum types -- tagged unions): 命名總和類型 -- 有標籤的聯合體
- unions (untagged unions): 無標籤的聯合體

在默認情況下複合類型的對齊方式等於最大的字段 e.g.
```rust
struct A {
		a: u8,
		b: u32,
		c: u16,
}
```
實際布局以 b 的 u32 的進行布局，如下
```rust
struct A {
		a: u8,
		_pad1: [u8, 3], // 注意 pad 都是以一字節的倍數進行對齊
		b: u32,
		c: u16,
		_pad2: [u8, 2],
}
```
_Rust 並不保證會按照字段的順序進行布局_，若按造順序可能會造成空間浪費。

# Exotically Sized Types (非正常大小類型)
---
### Dynamically Sized Types (DSTs)
Rust 支持 DST，這些類型沒有在編譯時期(statically)已知的大小或佈局，他們只能存在在一個指針的背後，任何指向 DST 的指針都會變成包含完善 DST 信息的 wide pointer (胖指針)。
有兩種主要的 DST 類型:
* trait objects: `dyn MyTrait`
	* Trait object 代表某種類型，他實現了指定的 Trait。實際的類型被插除而是使用指針指向 vtalbe，vtable 中紀錄所有必需的資訊以操作該類型，而運行時的對象大小可以從 vtable 獲取。
* slice: `[T]`, `str` 等
	* 一個切片就是對一段連續的存儲空間的視圖，通常是一個數組或 Vec。切片本身僅包含一個指針，指向數據的開始位置，以及指向數據的長度，這個長度告訴程序切片包含多少個元素。
###### Struct 可以存放 DST
Rust 允許在 struct 最後一個字段聲明為 DST，但該 struct 就會變成 DST，這樣的形式沒什麼用
```rust
struct MySuperSlice<T: ?Sized> {
		info: u32,
		data: T
}
```
> `?Sized` 表示可以為 `Sized` 或者 `!Sized`

### Zero Sized Types (ZSTs)
Rust 也允許不佔空間的類型如下
```rust
struct Nothing; // No field = no size

// All Field have no size = no size
struct LotsOfNoting {
		foo: Nothing,
		qux: (), // 空元組
		baz: [u8; 0], // 空數組
}
```
在 Rust 中這些 ZSTs 並不會佔用空間，實際應用在 `HashMap<Key, Value>` 可以將 Value 設定為空元組，`type Set<Key> = HashMap<Key, ()>`，在其他語言中可能會占用空間，但在 Rust 中並不會。

# Alternative representations
---
### repr(C)
這是最重要的 repr，他會按照 C 語言布局的形式布局: 按字段順序的布局方式，任何通過 FFI 邊界的類型應該都有 repr(C)。

