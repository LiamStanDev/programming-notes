編成界中最偉大的發現之一就是可以編寫**處裡多種不同類型**的的代碼，這就是所謂的 Polymorphism，而 Rust 使用兩個特性來支持多態: trait 與 generic，雖然很多語言支持多態，但 Rust 採用 Haskell 的 typeclass 啟發的新方法。

* trait: 是 Rust 中的 interface 或者 abstract class，但他功能遠不只這樣。
* generic: 是 Rust 中的模板，比模板更強的是他支持約束

# Trait Basic
---
### Trait 
通常一個 Trait 代表一種能力，表示一個類型能做的事。
* 實現 std::io::Write 的值可以寫入字節
* 實現 std::iter::Iterator 的值可以成生值的序列
* 實現 std::clone::Clone 的值可以在內存中克隆
* 實現 std::fmt::Debug 可以使用 `println!()` 的 {:?} 格式化符進行打印

以上都是標準庫的 traits，有很多類型實現他們 e.g. 
* std::fs::File 實現 Write，可以把字節寫入到本地文件、TcpStream 寫入到網路連接、`Vec<u8>` 可以用 `.write()` 在尾部添加數據。
* `Range<i32>` (e.g. 0..10) 、切片、哈希表都實現 Iterator。
* 大多數標準庫類型實現 Clone，除 TcpStream 這種不僅僅表示內存的類型以外。
* 大多數標準庫類型支持 Debug。

##### 細節
* **Trait 必須在作用域中**，否則無法調用 trait 的方法。
	* 你可以先編譯，編譯錯誤會提示你添加指定的 Trait
	* 原因: 因為怕 Trait 中的方法命名衝突
		* 可以使用完全限定方法調用來解決
* trait 並不是虛方法，虛方法會有調用的開銷，而 Trait 就像是直接屬於該值得方法一樣，不需要尋找 vtable，甚至可以 inline。
* 只有顯示通過 `dyn` 的調用才會有動態派發的開銷，這種才是虛方法調用。

Rust 中有兩種編寫多態代碼的方式:
1. Trait Object 
2. Generic

### Trait Object
```rust
use std::io:::Write // Error: doesn't have a size known at compile-time
```
雖然說類型大小不確定不能編譯，但寫過 C# 會很難理解，但其實接口的類型作為參數類型，本身其實是指針指向任何實現接口類型的對象引用。所以她必須是指針(引用)是不爭的事實，只是在 Rust 需要顯示寫出來。
* 一個 trait 類型的引用被稱為 trait 對象
* trait 對象指向某個值，它有生命期，它可以是可變的或者共享的

trait object 最特別的就是編譯時期 Rust 並不知道被引用的值的類型是甚麼，因此需要額外有關被引用值的類型信息，這個信息只有 Rust 自己可以幕後使用，我們無法查詢 Rust 也無法允許將 trait object 向下轉回實際的精確類型，
e.g. 將 `&mut dyn Write` 向下轉回 `Vec<u8>` 是不允許的。

##### Trait Object Layout (特徵對象的內存布局)
在內存中，trait object 是一個胖指針，由指向值的指針與指向表示該值類型的表的指針組成，占用兩個 words。
![[Pasted image 20240420163032.png|500]]
* 這個運行時的類型信息被稱為 vtalbe，vtable 只會在編譯期生成一次，然後被所有相同類型的對象共享，由 Rust 在底層自行使用，我們無法訪問。
* 與 C++ 不同的是需表指針(vptr) 被存储為類的一部分，而 Rust 使用胖指針來代替，_結構體本身不包含任何除了字段以外的東西_，這樣結構體可以實現一大堆 trait 而不需要實現包含一大堆 vptr (超讚)，故連 i32 這樣小的空間大小也能實現 trait。
* Rust 允許實現類型的指針間的轉換 e.g. `&mut File` 類型轉換成 `&mut dyn Write`，的普通引用轉換成 trait object，`Box<dyn Write>` 與 `Rc<dyn Write>` 都是一樣的，可以進行轉換且都為胖指針。
	* 在運行時 Rust 知道實際類型，只是將原本的指針，加上正確位置的 vtable，組成胖指針而已。

### Generic
我們將原本的 trait object 函數參數改為泛型函數，如下
```rust
fn say_hello(out: &mut dyn Write) { .. } // 使用 trait object

fn say_hello<W: Write>(out: &mut W) { .. } // 泛型函數
```
使用泛型函數方式 Rust 會為函數生成機器碼，會進行 monomorphization，將泛型參數轉為實際類型。

##### Trait Bound
```rust
fn run_query<M: Mapper + Serialize, R: Reducer + Serialize>(
	data: &DataSet, map: M, reduce: R) -> Results
{ ... }
```
另外還有使用 where 的寫法
```rust
fn run_query<M, R>(data: &DataSet, map: M, reduce: R) -> Result
	where M: Mapper + Serialize,
		  R: Reducer + Serialize
{ ... }
```


### 如何選擇?
##### Trait object 的好處
* 需要一個混合類型的集合
```rust
struct Salad {
	veggies: Vec<Box<dyn Vegetable>>
}
```
* 減少編譯出的二進制代碼體積，因為泛型會將所有要用到的類型都進行編譯
##### Generic 的好處
* 速度很快，因為沒有進行動態派發，甚至可以進行 inline
* 有些 trait 不支持 trait object
* 很容易添加 trait bound， 因為 Rust 不能編寫 `&mut (dyn Debug + Hash + Eq)` 這樣的類型

### 使用 Trait
* trait 能使用 pub 作為訪問修飾。
* trait 可以有默認實現，e.g. std::io::Write 中，`write_all` 就有默認實現。
* 可以為任何類型實現任意 trait，只要至少 trait 或者類型其中一個在當前 crate 定義。
* *trait 中可以使用 `Self` 關鍵字，這個 Self 表示實現了這個 trait 的類型，顧會被編譯成實際類型。* (不是 trait object)
* trait 可以包含類型關聯函數(其他語言稱靜態方法)
	* trait object 無法支持類型關聯函數，故這種 trait 無法建立 trait object

##### subtrait
子 trait 很像 C# 中的子接口，但是語意上有點不一樣，表示實現 Creature 的類型一定要實現 Visible。
```rust
trait Creature: Visible {
	fn position(&self) -> (i32, i32);
	fn facing(&self) -> Direction;
}

impl Visible for Broom {
	...
}

impl Creature for Broom {
	...
}
```
首先 impl 定義順序沒有影響，但本質上 subtrait 其實就是對於 Self 的約束，完全等同於以下寫法，
```rust
trait Creature 
	where Self: Visible
{
	...
}
```

### 完全限定方法調用
```rust
"hello".to_string();
```
這段代碼看似簡單，但實際上出現了幾個重要的角色，
1. `to_string()` 指的是 `ToString` trait 的 `to_string` 方法
2. `str` 類型實現了 `ToString` trait
**實際上方法就是一種特殊的函數**，我們有些時候會需要精確的調用方法才能避免重名問題，也就是完全限定調用。
```rust
"hello".to_string(); // 我們常用的

str::to_string("hello"); // 使用函數的方式使用方法

ToString::to_string("hello"); // 使用 trait 方法調用

<str as ToString>::to_string("hello"); // 完全限定語法
```

#### 使用時機
1. 兩個 Trait 中方法名稱相同時，又該類型實作了兩個 trait
```rust
outlaw.draw(); // 不知道是使用 Visible 還是 HasPistol 哪個的 draw 方法

Visible::draw(&outlaw); // OK

HasPistol::draw(&outlaw); // OK
```

2. 當 self 參數無法被推斷出來時
```rust
let zero = 0; // 可能是 i8, u8, ...

zero.abs(); // Error

i64::abs(zero); // OK
```

3. 傳遞函數時
```rust
let words: Vec<string> = 
	line.split_whitespace()
	.map(ToString::to_string) // 將 ToString::to_string 方法當參數傳入
	.collect();
```

4. 在 macro 中調用 trait


### 定義類型間關係的 trait
trait 也可以用於描述多個類型協同關係，e.g.
* std::iter::Iterator 將迭代器類型和生產的值類型關聯在一起
* std::ops::Mul 可以將做乘法的類型關聯起來

#### 關聯類型 (以迭代器為例)
簽名如下
```rust
pub trait Iterator {
	type Item; // 關聯類型 associated type，每個實現 Iterator 的類型都必須指明他是甚麼類型
	fn next(&mut self) -> Option<Self::Item>;
	...
}
```
標準庫中 std::env::Args 實現了 Iterator
```rust
impl Iterator for Args {
	type Item = String;
	fn next(&mut self) -> Option<Self::Item>;
}
```
我們可以自己寫一個 collect，如下
```rust
fn collect_to_vector(I: iter) -> Vec<I::Item> { // 使用關聯類型來指名 Vector 內容物類型
	let results = Vec::new();
	for value in iter {
		results.push(value);
	}
	results
}
```
我們也可以將 iter 的內容打印在 console 上，
```rust
fn dump<I>(iter: I) 
	where I: Iterator, 
		  I::Item: Debug // 我們也可以限定關聯類型的 trait
{
	for (index, value) in iter.enumerator() {
		println!("{}: {:?}", index, value);
	}
}
```

###### 其他例子
* 線程池庫中，一個 Task trait 表示一個工作單元，它可能有一個關聯的 Output 類型。
* 正則表達式中，一個 Pattern trait 表示一種搜索字符串的方式，它可能有一個關聯的 Match 類型，表示字符串中和模式匹配的所有信息。
* 關係型數據庫中，一個 DatabaseConnection trait 中他有一些關聯的 transaction、preparement 語句等等。
> 關聯類型完美適用於每一個實現都有一個特定的相關類型的情況，**我個人理解比較像是產出的類型**。

#### 泛型 trait (以運算符重載為例)
Rust 中的乘法使用 std::ops::Mul 簽名如下
```rust
pub trait Mul<RHS=Self> { // 這邊的 =Self 表示指定默認值
	type Output; // 運算結果
	fn mul(self, rhs: RHS) -> Self::Output;
}
```

### impl Trait 
我們希望返回值是某一個實現 Trait 的值，我們可以使用 trait object 來實現，如下
```rust
fn cyclical_zip(v: Vec<u8>, u: Vec<u8>) -> Box<dyn Iterator<Item = u8>> 
```
然而這樣的設計會在每次調用這個函數時，付出動態派發與堆分配的消耗，並不是一個很好的方法。
Rust 提供 impl Trait 這個性質，允許我們擦除返回值類型，只要指名他實現的 trait 或 traits，並沒有產生動態派發與堆分配，
```rust
fn cyclical_zip(v: Vec<u8>, u: Vec<u8>) -> impl Iterator<Item = u8>
```

我們可能會想像我們可以使用 impl Trait 實現簡單工廠，但實際上無法，因為 impl Trait 只是語法糖，他只是泛型的簡易版寫法，故他還是需要**在編譯時期換上實際的類型**，故簡單工廠是不可行的。


### 關聯常量
trait 中也能有關連的常量，但特別的是在 trait 中可以不用指定常量的值，可以在實際類型實作值在賦予。
trait object 無法使用關聯常量。

# Operator Overloading
---

| 類別       | trait                                                                                                                   | 運算符                                                         |
| -------- | ----------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| 一元運算符    | std::ops::Neg<br>std::ops::Not                                                                                          | `-x`<br>`!x`                                                |
| 算數運算符    | std::ops::Add<br>std::ops::Sub<br>std::ops::Mul<br>std::ops::Mul<br>std::ops::Div<br>std::ops::Rem                      | `x + y`<br>`x - y`<br>`x * y`<br>`x / y`<br>`x % y`         |
| 位運算符     | std::ops::BitAnd<br>std::ops::BitOr<br>std::ops::BitXor<br>std::ops::Shl<br>std::ops::Shr                               | `x & y`<br>`x \| y`<br>`x ^ y`<br>`x << y`<br>`x >> y`      |
| 符合賦值運算符  | std::ops::AddAssign<br>std::ops::SubAssign<br>std::ops::MulAssign<br>std::ops::DivAssign<br>std::ops::RemAssign         | `x += y`<br>`x -= y`<br>`x *= y`<br>`x /= y`<br>`x %= y`    |
| 複合賦值位運算符 | std::ops::BitAndAssign<br>std::ops::BitOrAssign<br>std::ops::BitXorAssign<br>std::ops::ShlAssign<br>std::ops::ShrAssign | `x &= y`<br>`x \|= y`<br>`x ^= y`<br>`x <<= y`<br>`x >>= y` |
| 比較       | std::cmp::PartialEq<br>std::cmp::PartialOrd                                                                             | `x == y`, `x != y`<br>`x < y`, `x > y`, `x <= y`, `x >= y`  |
| 索引       | std::ops::Index<br>std::ops::IndexMul                                                                                   | `x[y]`, `&x[y]`<br>`x[y] = z`, `&mut x[y]`                  |


# Utility Tarits
---
實用 Trait 是標準庫中能更顯著影響編寫 Rust 代碼方式的 Trait，需要熟練使用才會是真正的 Rustic，分為三大類:
1. Language extension trait(語言擴展): 包括運算符重載、Deref, DerefMut，以及轉換用的 From 和 Into
2. Marker trait(標記): 沒有任何方法，而是單純作為參數約束使用，包含 Sized, Copy。
3. Public vocabulary trait(公開詞彙): 沒有編譯器集成，我們也可以自己定義，但是常用於常見問題與常規的解決方案，大多數 crate 都不會獨立建立，而是統一使用這些約定好的 trait，如 Default, 引用借用 AsRef, AsMut, Borrow, BorrowMut，可能失敗的轉換 TryFrom, TryInto，以及 ToWoned。


| trait              | 解釋                                               |
| ------------------ | ------------------------------------------------ |
| Drop               | 解構器，當一個值被drop時會自動執行的代碼                           |
| Sized              | 標記一個類型有編譯時期已知的固定大小                               |
| Clone              | 支持克隆的類型                                          |
| Copy               | 標記一個類型可以通過按位拷貝來進克隆新值                             |
| Deref 與 DerefMut   | 為智能指針準備的 Trait                                   |
| Default            | 一個有意義的默認值的類型                                     |
| AsRef 與 AsMut      | 轉換 Trait，用於從一個類型的值借用另一個類型的引用                     |
| Borrow 與 BorrowMut | 轉換 Trait，類似於 AsRef 與 AsMut 但是而外保證一致的 Hash、順序與相等性 |
| From 與 Into        | 轉換 Trait，將一個類型的值轉換為另外一個類型的值                      |
| TryFrom 與 TryInto  | 轉換 Trait，用於可能會失敗的類型轉換                            |
| ToOwned            | 轉換 Trait，將一個引用轉換為一個所有權的值                         |

### Drop
為 std::ops::Drop，表示析構函數，簽名如下
```rust
trait Drop { 
	fn drop(&mut self); 
}
```
通常我們並不需要手動實現 std::ops::Drop，除非我們想定義一個擁有 Rust 不知道的資源類型，以 Rust 標準庫中的 FileDesc 為例子:
```rust
struct FileDesc {
	fd: c_int,
}

impl Drop for FileDesc {
	fn drop(&mut self) {
		let _  = unsafe { libc::close(self.fs) };
	}
}
```
* 一個類型若實現 Drop 就不能再實現 Copy。
* 在 prelude 中有一個函數為 `drop` 定義非常簡單，如下
```rust
fn drop<T>(_x: T) {} // 取得所有權之後，給他一個作用域，讓他直接被 drop 調
```
### Sized
為 std::marker::Sized。
固定大小類型 (sized type) 是指那些*所有實例值都占用相同的內存大小*的類型。
* Rust 為所有適合的類型都自動實現它，你不能自己實現它。
* 唯一的用途就是作為 Trait bound。

##### unsized type
Rust 中有少量的不固定大小類型:
* 切片: 字符串切片 `str`, 數組切片 `[T]` (都沒有 &)
* `dyn` 類型: trait object
這些表示值的大小都無法確定或者**精確地說無法在編譯時候確定**，只能透過指針來處理它們，而指向它們的指針總是胖指針: 一個指向值的指針 + 切片的長度或者指向 vtable 的指針，而這些附加在指針的信息會在運行時動態的被添加。

```rust
struct S<T: ?Sized> { // 表示接受 Size 與 Unsize 類型
	b: Box<T>
}
```
* 若 T 為 `i32` 則 b 為普通指針，而若為 `dyn Write` 則 b 為胖指針。
* 標準庫中偶爾看到 `?Sized` 的約束，幾乎是**意味者這個類型只能被指針指向**。

### Clone
為 std::clone::Clone，定義如下，
```rust
pub trait Clone: Sized {
	fn clone(&self) -> Self;

	fn clone_from(&mut self, source: &Self) {
		*self = source.clone();
	}
}
```
* 因為 Clone 返回值為 Self，故必須限制 Self 為 Sized。
* 拷貝意味者拷貝所有內容，故開銷較大
	* Rc 與 Arc 是個例外，指針加引用技術後返回新的指針。
* clone_from 用於值已經被建立，只是需要抽換內容時使用，會更有效率
```rust
// s 與 t 都是 string
s = t.clone(); // 這樣會導致 t 先 clone 後，s 先 drop 掉舊的值，再將克隆的值移動到 s

// 使用 clone_from 優化
s.clone_from(&t); // 這樣若 s 原本緩衝區就夠大，不會導致 s 釋放緩衝區。
```
* 標準庫類型幾乎都實現 Clone，除了一些不能拷貝的，如 std::sync::Mutex，另外 std::fs::File 其實可以拷貝，但是可能會失敗又 clone 返回值不是 Result 所以沒有實現，但是提供了 try_clone() 方法可以調用。

### Copy
為 std::marker::Copy。
對決大部分類型在 Rust 中都會發生移動，但是[[Ch4 - Ownership and Moves#Copy Types The Exception to Moves|唯一個例外就是 Copy]]。
他是一個具有特殊涵義的標記 trait，所以 Rust 只允許可以通過逐個字節進行淺拷貝自身的類型實現 Copy，如果一個類型擁有其他資源，如堆中緩衝區或者操作系統句柄，那他無法實現 Copy。
* 任何實現 Drop 的類型無法實現 Copy，因為需要 Drop 才能清理代碼的類型，他也肯定需要特殊的方式才能拷貝，不能是 Copy。
* 可以使用 `#[derive(Copy)]` 來實現 Copy。
	* 通常可以看到 `#[derive(Copy, Clone)]` 一起使用，畢竟都能逐個字節拷貝了，多個 clone 方法也沒什麼。
* 使類型實現 Copy 時需要考慮性能問題，因為隱式的拷貝會有性能損失。


### Derf 與 DerefMut
為 std::ops::Deref 與 std::ops::DerefMut，用來指定運算符 `*` 和 `.` 的行為。

像是 `Box<T>` 與 `Rc<T>` 這樣的指針類型都實現了這個 trait ，因此它們的行為和 Rust 內建的指針類型一樣。
定義如下，
```rust
pub trait Deref {
	type Target: ?Sized;
	fn deref(&self) -> &Self::Target;
}

pub trait DerefMut {
	type Target: ?Sized;
	fn deref_mut(&mut self) -> &mut Self::Target;
}
```
`Rc<T>` 有實現以上兩種 Trait，對 `Rc<T>` 進行解引用為以下操作
```rust
let r = Rc::new(String::from("hello"));

// 直接解引用
*r; 
// 本質為
*(r.deref());
// 將方法接收者傳遞寫得更仔細為
*((&r).deref());
// 使用限定調用
*(<&r as Deref>::deref(&r));
```

#### deref coercions
這兩個 Trait 還有另外一個功能，因為返回值是 Target 的引用，使用 deref 方法或者 deref_mut 方法能達成類型轉換，故 Rust 會自動插入 deref 與 deref_mut 讓方法調用可以類型匹配，這被稱之為 deref coercions。

1. 用於方法調用**接收者參數匹配** (不需要顯示將類型添加引用，因為方法本身就會自動添加)
2. 用於函數/方法**參數匹配** (需要顯示將類型添加引用)

> 注意這邊說的是將類型 `*` 直接解引用，而不是將類型的引用解引用 (也就是 `*(&s)`)，引用的解引用是 Rust 已經寫好的，返回值就是引用的值本身。


###### 案例: 從 `Rc<String>` 調用 split_at 方法
split_at 方法是定義在 str 上，並接接收者為 `&self`，可以在 `Rc<String>` 上進行調用是進行了以下 deref coercions
1. 方法的自動引用解引用，將 `Rc<String>` 轉為 `&Rc<String>`
2. `Rc<String>` 實現 `Deref<Target = String>`，故將 `&Rc<String>` 轉為 `&String`
3. `String` 實現 `Deref<Target = str>`，故將 `&String` 轉為 `&str`

###### 案例: 實現 Deref
```rust
pub struct Selector<T> {
    pub elem: Vec<T>,
    pub current: usize,
}

impl<T> Deref for Selector<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.elem[self.current]
    }
}

impl<T> DerefMut for Selector<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.elem[self.current]
    }
}

fn main() {
  let mut s = Selector {
        elem: vec!['a', 'b', 'z'],
        current: 2
    };

    assert_eq!(*s, 'z'); // 這邊 *s 其實是 *(s.deref())

    *s = 'w';

    assert_eq!(*s, 'w');
}
```

#### 不要為了類型轉換而實作 Deref
Deref 與 DerefMut 本意是為了讓智能指針 (Box, Rc, Arc) 與經常以引用方式使用並充當另外一種類型準備的，而不應該僅僅為了讓某個類型能自動使用 Target 類型的方法而實現這兩個 trait。

**重點: deref coercion 不會為了滿足 trait bound 的情況下使用**，也就是 trait bound 下不會進行 deref coercion，需要自己進行類型轉換。 
```rust

fn show_it(thing: &str) {
	println!("{}", thing);
}

fn show_it2<T: Display>(thing: T) {
	println!("{}", thing);
}

fn main() {
    let s = Selector {
        elem: vec!["good", "bad", "normal"],
        current: 2
    };

    show_it(&s); // Ok
    show_it2(&s); // Error，明明 &Selector -> &str，又 &str 有實現 Display，但是他是不會進行 deref coercion 的
    show_it2(&*s); // OK ，但需要自己手動解引用
}
```


### Default
為 std::default::Default，簽名如下，
```rust
pub trait Default {
	fn default() -> Self;
}
```
有些類型會有一些有意義的默認值，例如:
1. Vec, String, HashMap, BinaryHeap 等都是空集合
2. 數值類型默認值都是 0。
3. Option 默認值為 None

* 若一個類型 `T` 實現 Default 而標準庫會自動為 `Rc<T>`, `Arc<T>`, `Box<T>`, `Cell<T>`, `RefCell<T>`, `Cow<T>`, `Mutex<T>`, `RwLock<T>` 實現 Defualt，也就是指向 `T` 默認值的指針或持有她的鎖等。
* tuple 字段若都實現 Defualt，則會自動實現 Default
* Rust 不會為 struct 隱式實現 Default，但是字段都實現 Default 可以使用 `#[derive(Default)]` 來使 struct 實現 Default。

### AsRef 與 AsMut
為 std::convert::AsRef 與 std::convert::AsMut，簽名如下，
```rust
pub trait AsRef<T: ?Sized> {
	fn as_ref(&self) -> &T;
}

pub trait AsMut<T: ?Sized> {
	fn as_mut(&mut self) -> &mut T;
}
```
若一個類型實現了 `AsRef<T>`，意味者你可以高效的借用一個 `&T`，例如:
1. `Vec<T>` 實現了 `AsRef<[T]>`，表示你可以從 `Vec<u8>` 借用一個 `[T]`
2. `String` 實現了 `AsRef<[u8]>`，表示你可以從 `String` 借用一個 `[u8]`。

##### 使用 AsRef 作為參數
很多函數接收參數使用 AsRef 來讓函數更靈活，如 std::fs::File::open 
```rust
fn open<P: AsRef<Path>>(path: P) -> Result<File>;
```
open 真正想要的是 `&Path`，但是這樣設計就可以讓參數接收所有實作 `AsRef<Path>` 的任意類型，包括 `String`, `str` 與 `OsString` 等。

* `T` 類型實現 `AsRef<T>` 時，也會同時為 `&T` 實現 `AsRef<T>`，因為在 trait bound 下不會進行 deref coercion。
```rust
fn main() {
	// 這邊傳入的是 &str 字符串切片，並不是 str，但因為 open 使用
	// trait bound 是不會進行 deref coercion，但因為上述性質 &str
	// 也實現 AsRef，故可以通過。
	let dot_emacs = std::fs::File::open("/home/jimb/.emacs")?;
}
```

### Borrow 與 BorrowMut
為 std::borrow::Borrow 與 std::borrow::BorrowMut，與 AsRef 和 AsMut 幾乎相似，一個類型實現了 `Borrow<T>` 可以高效的借用 `&T`，但 Borrow 有更多的限制，當只有 Hash 值與比較表達式的結果都與 `T` 相同時，這個類型才該實現 Borrow (Rust 並沒有強制限制)，這讓 Borrow 用於 HashMap 與樹中的 Key 很有用。定義如下:
```rust
pub trait Borror<Borrowed: ?Sized> {
	fn borrow(&self) -> &Borrowed;
}
```

String 實現 `AsRef<str>`, `AsRef<[u8]>`, `AsRef<Path>` 但這三者的 Hash 值並不一樣，只有 `&str` 與 `String` Hash 值相同，故 String 只實現 `Borrow<str>`。

##### HashMap get 方法
```rust
impl<K, V> HashMap<K, V> 
	where K: Hash + Eq
{
	// Version1
	fn get(&self, key: K) -> Option<&V> { ... } // 這樣我們調用 get 都要向其傳遞所有權，然後被他 drop，太浪費
	// Version 2
	fn get(&self, key: &K) -> Option<&V> { ... } // 若 Key 是 String，我們只能傳遞 &String 而不能傳遞 &str
	// Version 3
	fn get<Q: ?Sized>(&self, key: &Q) -> Option<&V>  
		where K: Borrow<Q>, // 我們希望可以傳入 &str，又 Key 可以借用成為 &str
			  Q: Eq + Hash  // 這個 Q 又能進營比較與 Hash，來搭配 Borrow。
	{ ... } 
}
```
換句話說 Q 的比較與 Hash 值均與 Key 一模一樣，又 Key 也可以借用出 &Q，這樣傳入 &Q 就是一個可以接受的 Key 類型。

* 標準庫類型都實作了自身類型的 Borrow e.g. `String` 實現了 `Borrow<String>`
	* 保證了自身類型的引用一定可以做為 HashMap 的 key 參數。

### From 與 Into
為 std::convert::From 與 std::convert::Into *代表消耗一個值返回另外一個值的轉換*。定義如下，
```rust
pub trait Into<T: Sized> {
	fn into(self) -> T;
}

pub trait From<T: Sized> {
	fn from(other: T) -> Self;
}
```
* 表準庫類型實現了對自身的轉換，每個 `T` 類型都實現了 `From<T>` 與 `Into<T>`。

##### 更靈活的轉換
```rust
fn ping<A>(address: A) -> std::io::Result<bool> 
	where A: Into<Ipv4Addr>
{
	let ipv4_address = address.into();
	...
}

fn main() {
	// 以下類型都實現 Into<Ipv4Addr> 與 From<Ipv4Addr>
	println!("{:?}", ping(Ipv4Addr::new(23, 21, 68, 141)));
	println!("{:?}", ping([66, 146, 219, 98]));
	println!("{:?}", ping(0xd076eb94_u32));

	let addr1 = Ipv4Addr::from([66, 146, 219, 98]);
	let addr2 = Ipv4Addr::from(0xd076eb94_u32);

}
```



### TryFrom 與 TryInto
Into 與 From 是不會失敗的轉換，而這兩個是可以失敗的轉換，定義如下，
```rust
pub trait TryFrom: Sized { 
	type Error; 
	fn try_from(value: T) -> Result<Self, Self::Error>; 
} 

pub trait TryInto: Sized { 
	type Error; 
	fn try_into(self) -> Result<Self, Self::Error>; 
}
```

### ToOwned
給定一個引用，我們希望拷貝他的值並獲取所有權，我們會想到使用 Clone，但是 Clone 的定義無法做到返回值是 unsized type，如 str，他的定義如下，
```rust
pub trait ToOwne {
	type Owned: Borrow<Self>
	fn to_owned(&self) -> Self::Owned;
}
```


### Cow 
太難 skip 