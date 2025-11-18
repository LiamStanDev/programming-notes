# Rust 快速入門

本文檔專為**已有其他語言經驗**的開發者設計，快速上手 Rust 核心概念。涵蓋所有基礎知識點並附帶實用範例，可以作為學習速查表或參考手冊。

**目標讀者**: 熟悉 C/C++、Java、Python、Go 等語言，想要快速掌握 Rust 的開發者。

**學習路線**: 建議按順序閱讀，每個概念都有清晰定義和代碼示例。完成本文後即可開始實際項目開發。

---

## 1. 所有權系統 (Ownership System)

### 1.1 所有權 (Ownership)

**定義**: Rust 的核心記憶體管理機制，規定每個值都有唯一的擁有者 (owner)，當擁有者離開作用域時，值會被自動釋放。

**三大規則**:
1. 每個值在任何時候都只能有一個擁有者
2. 當擁有者離開作用域時，值會被丟棄 (dropped)
3. 值的所有權可以被轉移 (move)

```rust
fn main() {
    let s1 = String::from("hello"); // s1 擁有字符串
    let s2 = s1;                     // 所有權轉移給 s2，s1 不再有效
    // println!("{}", s1);           // ❌ 編譯錯誤: s1 已失效
    println!("{}", s2);              // ✅ s2 是有效的擁有者
}
```

**為什麼需要**: 避免懸垂指針 (dangling pointer)、雙重釋放 (double free)、記憶體洩漏，在編譯期保證記憶體安全。

---

### 1.2 移動語義 (Move Semantics)

**定義**: 當值被賦值給另一個變量或傳遞給函數時，所有權會被轉移 (move)，原變量失效。

```rust
fn take_ownership(s: String) {
    println!("{}", s);
} // s 在這裡被 drop

fn main() {
    let s = String::from("hello");
    take_ownership(s);  // s 的所有權移動到函數中
    // println!("{}", s); // ❌ 錯誤: s 已經被移動
}
```

**適用類型**: 
- 不實現 `Copy` trait 的類型 (如 `String`, `Vec<T>`, `Box<T>`)
- 複雜的堆分配類型

---

### 1.3 複製語義 (Copy Semantics)

**定義**: 對於實現了 `Copy` trait 的簡單類型，賦值時會進行按位複製，原變量仍然有效。

```rust
fn main() {
    let x = 5;      // i32 實現了 Copy
    let y = x;      // x 被複製給 y
    println!("{} {}", x, y); // ✅ 兩者都有效
}
```

**實現 Copy 的類型**:
- 所有整數類型 (`i32`, `u64`, 等)
- 布爾類型 (`bool`)
- 浮點類型 (`f32`, `f64`)
- 字符類型 (`char`)
- 元組 (當所有成員都是 `Copy` 時)
- 不可變引用 `&T` (可變引用 `&mut T` 不是 Copy)

**限制**: 如果類型實現了 `Drop` trait，則不能實現 `Copy`。

---

### 1.4 借用 (Borrowing)

**定義**: 通過引用 (reference) 訪問值而不獲取所有權的機制。

#### 不可變借用 (Immutable Borrow) - `&T`

```rust
fn calculate_length(s: &String) -> usize {
    s.len()
} // s 離開作用域，但不會 drop，因為它只是引用

fn main() {
    let s1 = String::from("hello");
    let len = calculate_length(&s1); // 借用 s1
    println!("'{}' 的長度是 {}", s1, len); // ✅ s1 仍然有效
}
```

**規則**: 可以同時存在多個不可變引用。

#### 可變借用 (Mutable Borrow) - `&mut T`

```rust
fn append_world(s: &mut String) {
    s.push_str(", world");
}

fn main() {
    let mut s = String::from("hello");
    append_world(&mut s);
    println!("{}", s); // "hello, world"
}
```

**規則**: 在特定作用域中，對某個數據只能有**一個**可變引用，或**多個**不可變引用，但不能同時存在。

---

### 1.5 借用檢查器 (Borrow Checker)

**定義**: Rust 編譯器的組件，在編譯期檢查借用規則，確保沒有數據競爭 (data race)。

**核心規則**:
1. 在任意給定時間，**要麼**只能有一個可變引用，**要麼**只能有多個不可變引用
2. 引用必須總是有效的 (不能懸垂)

```rust
fn main() {
    let mut s = String::from("hello");
    
    let r1 = &s;     // ✅ 不可變借用
    let r2 = &s;     // ✅ 可以有多個不可變借用
    // let r3 = &mut s; // ❌ 錯誤: 不能在有不可變借用時創建可變借用
    
    println!("{} {}", r1, r2);
    // r1 和 r2 在這之後不再使用
    
    let r3 = &mut s; // ✅ 現在可以創建可變借用
    r3.push_str("!");
}
```

**為什麼需要**: 防止數據競爭，在編譯期保證線程安全。

---

### 1.6 生命期 (Lifetime)

**定義**: 引用保持有效的作用域範圍，用於確保引用不會比它所指向的數據活得更長。

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let string1 = String::from("long string");
    let string2 = String::from("short");
    
    let result = longest(&string1, &string2);
    println!("最長的字符串是: {}", result);
}
```

**語法**: `'a` 表示生命期參數，告訴編譯器返回值的生命期與輸入參數的生命期關係。

**常見生命期**:
- `'static`: 整個程序運行期間都有效
- `'a`, `'b`: 泛型生命期參數

**生命期省略規則**: 編譯器可以在某些情況下自動推斷生命期，無需顯式標註。

---

## 2. 類型系統 (Type System)

### 2.1 標量類型 (Scalar Types)

**定義**: 表示單個值的類型。

| 類型 | 說明 | 範例 |
|------|------|------|
| **整數** | `i8`, `i16`, `i32`, `i64`, `i128`, `isize` (有符號)<br>`u8`, `u16`, `u32`, `u64`, `u128`, `usize` (無符號) | `let x: i32 = 42;` |
| **浮點數** | `f32` (單精度), `f64` (雙精度，默認) | `let y: f64 = 3.14;` |
| **布爾** | `bool` | `let flag: bool = true;` |
| **字符** | `char` (4 bytes, Unicode) | `let c: char = '😊';` |

---

### 2.2 複合類型 (Compound Types)

#### 元組 (Tuple)

**定義**: 將多個不同類型的值組合成一個複合類型，長度固定。

```rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);
    let (x, y, z) = tup;  // 解構
    let five_hundred = tup.0; // 索引訪問
}
```

#### 數組 (Array)

**定義**: 相同類型元素的固定長度集合，在棧上分配。

```rust
fn main() {
    let arr: [i32; 5] = [1, 2, 3, 4, 5];
    let first = arr[0];
    
    let zeros = [0; 100]; // 創建包含 100 個 0 的數組
}
```

**vs 向量 (Vector)**: 數組長度固定，向量長度可變且在堆上分配。

---

### 2.3 字符串類型

#### `String` vs `&str`

| 特性 | `String` | `&str` |
|------|----------|--------|
| **所有權** | 擁有數據 | 借用數據 |
| **可變性** | 可變 | 不可變 |
| **分配位置** | 堆 | 棧 (指向堆或靜態數據) |
| **使用場景** | 需要修改或擁有字符串 | 只讀、函數參數、字符串切片 |

```rust
fn main() {
    let s1: String = String::from("hello"); // 擁有的字符串
    let s2: &str = "world";                 // 字符串切片 (字面量)
    let s3: &str = &s1[0..2];               // 從 String 借用的切片
    
    let mut s4 = String::from("Hello");
    s4.push_str(", world!"); // ✅ String 可變
    // s2.push_str("!"); // ❌ &str 不可變
}
```

---

### 2.4 智能指針 (Smart Pointers)

#### `Box<T>` - 堆分配

**定義**: 在堆上分配數據，擁有所有權，離開作用域時自動釋放。

```rust
fn main() {
    let b = Box::new(5); // 在堆上分配整數
    println!("b = {}", b);
} // b 離開作用域，堆記憶體被釋放
```

**使用場景**:
- 編譯期無法確定大小的類型 (如遞歸類型)
- 大量數據的所有權轉移 (避免棧複製)
- Trait 對象

---

#### `Rc<T>` - 引用計數

**定義**: 允許多個所有者共享數據的智能指針，通過引用計數追蹤所有者數量。

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(5);
    let b = Rc::clone(&a); // 增加引用計數
    let c = Rc::clone(&a);
    
    println!("引用計數: {}", Rc::strong_count(&a)); // 3
} // a, b, c 離開作用域，計數歸零，記憶體被釋放
```

**限制**: 只能用於單線程，數據是不可變的。

---

#### `Arc<T>` - 原子引用計數

**定義**: 線程安全的 `Rc<T>`，使用原子操作維護引用計數。

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3]);
    
    let data_clone = Arc::clone(&data);
    let handle = thread::spawn(move || {
        println!("{:?}", data_clone);
    });
    
    println!("{:?}", data);
    handle.join().unwrap();
}
```

**使用場景**: 多線程共享只讀數據。

---

#### `RefCell<T>` - 內部可變性

**定義**: 提供運行時借用檢查的容器，允許在不可變引用下修改數據。

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(5);
    
    *data.borrow_mut() += 1; // 運行時可變借用
    println!("{}", data.borrow()); // 6
}
```

**vs 編譯期借用檢查**: 
- `RefCell<T>` 在運行時檢查借用規則
- 違反規則會 panic，而非編譯錯誤

**常見組合**: `Rc<RefCell<T>>` - 多所有者 + 可變數據 (單線程)

---

#### `Mutex<T>` / `RwLock<T>` - 線程安全的內部可變性

**`Mutex<T>` (互斥鎖)**:

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);
    
    {
        let mut num = m.lock().unwrap(); // 獲取鎖
        *num += 1;
    } // 鎖自動釋放
    
    println!("{:?}", m);
}
```

**`RwLock<T>` (讀寫鎖)**:

```rust
use std::sync::RwLock;

fn main() {
    let lock = RwLock::new(5);
    
    // 多個讀者
    {
        let r1 = lock.read().unwrap();
        let r2 = lock.read().unwrap();
        println!("{} {}", r1, r2);
    }
    
    // 單個寫者
    {
        let mut w = lock.write().unwrap();
        *w += 1;
    }
}
```

**常見組合**: `Arc<Mutex<T>>` - 多線程共享可變數據

---

### 2.5 枚舉 (Enum) 與模式匹配

#### 枚舉定義

**定義**: 可以是多種不同變體之一的類型。

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

fn main() {
    let home = IpAddr::V4(127, 0, 0, 1);
    let loopback = IpAddr::V6(String::from("::1"));
}
```

---

#### `Option<T>` - 可選值

**定義**: 表示值可能存在或不存在。

```rust
enum Option<T> {
    Some(T),
    None,
}
```

**使用**:

```rust
fn divide(a: i32, b: i32) -> Option<i32> {
    if b == 0 {
        None
    } else {
        Some(a / b)
    }
}

fn main() {
    match divide(10, 2) {
        Some(result) => println!("結果: {}", result),
        None => println!("除數為零"),
    }
}
```

**為什麼沒有 null**: Rust 沒有 `null`，使用 `Option<T>` 顯式處理空值，避免空指針錯誤。

---

#### `Result<T, E>` - 錯誤處理

**定義**: 表示操作可能成功 (`Ok`) 或失敗 (`Err`)。

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

**使用**:

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");
    
    let f = match f {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => panic!("文件不存在"),
            other_error => panic!("打開文件失敗: {:?}", other_error),
        },
    };
}
```

**`?` 運算子**: 簡化錯誤傳播。

```rust
use std::io;
use std::fs::File;

fn read_file() -> Result<String, io::Error> {
    let mut s = String::new();
    File::open("hello.txt")?.read_to_string(&mut s)?;
    Ok(s)
}
```

---

#### 模式匹配 (Pattern Matching)

**`match` 表達式**:

```rust
fn main() {
    let number = 7;
    
    match number {
        1 => println!("一"),
        2 | 3 | 5 | 7 | 11 => println!("質數"),
        13..=19 => println!("青少年"),
        _ => println!("其他"),
    }
}
```

**`if let` 語法糖**:

```rust
fn main() {
    let some_value = Some(3);
    
    if let Some(3) = some_value {
        println!("是三");
    }
}
```

**解構**:

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };
    
    match p {
        Point { x: 0, y } => println!("在 y 軸上: {}", y),
        Point { x, y: 0 } => println!("在 x 軸上: {}", x),
        Point { x, y } => println!("其他位置: ({}, {})", x, y),
    }
}
```

---

## 3. Trait 系統

### 3.1 Trait 定義

**定義**: Rust 的接口機制，定義類型必須實現的行為。

```rust
trait Summary {
    fn summarize(&self) -> String;
    
    // 默認實現
    fn author(&self) -> String {
        String::from("Unknown")
    }
}

struct Article {
    title: String,
    content: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}: {}", self.title, &self.content[..50])
    }
}
```

---

### 3.2 Trait 約束 (Trait Bounds)

**定義**: 限制泛型參數必須實現特定的 trait。

```rust
fn print_summary<T: Summary>(item: &T) {
    println!("{}", item.summarize());
}

// 多個 trait 約束
fn notify<T: Summary + Clone>(item: &T) {
    // ...
}

// where 子句 (可讀性更好)
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
    // ...
}
```

---

### 3.3 常見 Trait

#### `Clone` - 顯式複製

```rust
#[derive(Clone)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1.clone(); // 顯式複製
}
```

---

#### `Copy` - 隱式複製

**定義**: 實現 `Copy` 的類型在賦值時會自動按位複製。

```rust
#[derive(Copy, Clone)] // Copy 需要 Clone
struct Point {
    x: i32,
    y: i32,
}
```

**限制**: 類型及其所有成員都必須是 `Copy` 的。

---

#### `Debug` - 調試格式化

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{:?}", p);  // Point { x: 1, y: 2 }
    println!("{:#?}", p); // 美化輸出
}
```

---

#### `Display` - 用戶友好格式化

```rust
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{}", p); // (1, 2)
}
```

---

#### `Default` - 默認值

```rust
#[derive(Default)]
struct Config {
    timeout: u32,
    retries: u8,
}

fn main() {
    let config = Config::default();
    println!("{} {}", config.timeout, config.retries); // 0 0
}
```

---

#### `PartialEq` / `Eq` - 相等比較

```rust
#[derive(PartialEq, Eq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 1, y: 2 };
    assert_eq!(p1, p2);
}
```

- `PartialEq`: 部分相等 (允許 NaN 等特殊情況)
- `Eq`: 完全相等 (自反性、對稱性、傳遞性)

---

#### `PartialOrd` / `Ord` - 排序比較

```rust
#[derive(PartialEq, Eq, PartialOrd, Ord)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let mut points = vec![
        Point { x: 2, y: 3 },
        Point { x: 1, y: 2 },
    ];
    points.sort(); // 需要 Ord
}
```

---

#### `Iterator` - 迭代器

```rust
struct Counter {
    count: u32,
}

impl Iterator for Counter {
    type Item = u32;
    
    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;
        if self.count <= 5 {
            Some(self.count)
        } else {
            None
        }
    }
}
```

---

#### `From` / `Into` - 類型轉換

```rust
struct Number {
    value: i32,
}

impl From<i32> for Number {
    fn from(item: i32) -> Self {
        Number { value: item }
    }
}

fn main() {
    let num = Number::from(30);
    let num: Number = 30.into(); // Into 自動實現
}
```

---

#### `Send` / `Sync` - 線程安全標記

**`Send`**: 可以安全地在線程間轉移所有權。

```rust
// 大多數類型都是 Send
// 例外: Rc<T>, 裸指針
```

**`Sync`**: 可以安全地在線程間共享引用。

```rust
// T 是 Sync 意味著 &T 是 Send
// 例外: Cell<T>, RefCell<T>
```

這兩個 trait 是**自動實現**的 marker traits。

---

### 3.4 Trait 對象 (Trait Objects)

**定義**: 使用 `dyn Trait` 實現動態分派 (dynamic dispatch)。

```rust
trait Draw {
    fn draw(&self);
}

struct Circle;
struct Square;

impl Draw for Circle {
    fn draw(&self) {
        println!("繪製圓形");
    }
}

impl Draw for Square {
    fn draw(&self) {
        println!("繪製方形");
    }
}

fn main() {
    let shapes: Vec<Box<dyn Draw>> = vec![
        Box::new(Circle),
        Box::new(Square),
    ];
    
    for shape in shapes {
        shape.draw(); // 運行時決定調用哪個實現
    }
}
```

**vs 泛型**: 
- 泛型使用靜態分派 (編譯期)，性能更好
- Trait 對象使用動態分派 (運行期)，更靈活

---

## 4. 泛型 (Generics)

### 4.1 泛型函數

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

---

### 4.2 泛型結構體

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

// 只為特定類型實現方法
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

---

### 4.3 泛型枚舉

```rust
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

---

### 4.4 關聯類型 (Associated Types)

**定義**: Trait 中定義的類型佔位符，由實現者指定具體類型。

```rust
trait Iterator {
    type Item; // 關聯類型
    
    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter {
    count: u32,
}

impl Iterator for Counter {
    type Item = u32; // 指定關聯類型
    
    fn next(&mut self) -> Option<Self::Item> {
        // ...
    }
}
```

**vs 泛型參數**: 關聯類型每個實現只能有一種，泛型參數可以有多種實現。

---

## 5. 閉包 (Closures)

### 5.1 閉包語法

**定義**: 可以捕獲環境變量的匿名函數。

```rust
fn main() {
    let x = 4;
    
    // 完整形式
    let add_one_v1 = |num: i32| -> i32 { num + 1 };
    
    // 類型推導
    let add_one_v2 = |num| num + 1;
    
    // 捕獲環境變量
    let equal_to_x = |z| z == x; // 捕獲 x
    
    println!("{}", equal_to_x(4)); // true
}
```

---

### 5.2 閉包捕獲模式

Rust 會自動選擇最少限制的捕獲方式:

1. **不可變借用 (`Fn`)**: 
```rust
let list = vec![1, 2, 3];
let only_borrows = || println!("{:?}", list);
only_borrows();
println!("{:?}", list); // ✅ list 仍可用
```

2. **可變借用 (`FnMut`)**:
```rust
let mut list = vec![1, 2, 3];
let mut borrows_mutably = || list.push(7);
borrows_mutably();
```

3. **獲取所有權 (`FnOnce`)**:
```rust
let list = vec![1, 2, 3];
let consume = move || {
    println!("{:?}", list);
    // list 的所有權被移動到閉包中
};
consume();
// println!("{:?}", list); // ❌ list 已被移動
```

**`move` 關鍵字**: 強制閉包獲取環境變量的所有權。

---

## 6. 併發 (Concurrency)

### 6.1 線程 (Threads)

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("子線程: {}", i);
            thread::sleep(Duration::from_millis(1));
        }
    });
    
    for i in 1..5 {
        println!("主線程: {}", i);
        thread::sleep(Duration::from_millis(1));
    }
    
    handle.join().unwrap(); // 等待子線程結束
}
```

---

### 6.2 通道 (Channels)

**定義**: 線程間傳遞消息的機制。

```rust
use std::sync::mpsc; // multiple producer, single consumer
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    thread::spawn(move || {
        let val = String::from("hello");
        tx.send(val).unwrap();
    });
    
    let received = rx.recv().unwrap();
    println!("收到: {}", received);
}
```

---

### 6.3 共享狀態 (Shared State)

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];
    
    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("結果: {}", *counter.lock().unwrap());
}
```

---

## 7. 異步編程 (Async Programming)

### 7.1 `async` / `await`

**定義**: Rust 的異步語法，基於 Future。

```rust
async fn fetch_data() -> String {
    // 模擬異步操作
    String::from("數據")
}

async fn process() {
    let data = fetch_data().await; // 等待 Future 完成
    println!("{}", data);
}
```

---

### 7.2 `Future` Trait

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
    type Output;
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

**`Poll` 枚舉**:
- `Poll::Ready(value)`: Future 已完成
- `Poll::Pending`: Future 未完成，等待喚醒

---

### 7.3 Runtime (Tokio 範例)

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let task1 = tokio::spawn(async {
        sleep(Duration::from_millis(100)).await;
        println!("任務 1");
    });
    
    let task2 = tokio::spawn(async {
        sleep(Duration::from_millis(50)).await;
        println!("任務 2");
    });
    
    task1.await.unwrap();
    task2.await.unwrap();
}
```

---

## 8. 錯誤處理

### 8.1 `panic!` - 不可恢復錯誤

```rust
fn main() {
    panic!("程序崩潰");
}
```

**使用場景**: 程序遇到無法恢復的錯誤狀態。

---

### 8.2 `Result<T, E>` - 可恢復錯誤

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("username.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

---

### 8.3 自定義錯誤類型

```rust
use std::fmt;

#[derive(Debug)]
enum MyError {
    IoError(std::io::Error),
    ParseError(std::num::ParseIntError),
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            MyError::IoError(e) => write!(f, "IO 錯誤: {}", e),
            MyError::ParseError(e) => write!(f, "解析錯誤: {}", e),
        }
    }
}

impl std::error::Error for MyError {}
```

---

## 9. 模塊系統 (Module System)

### 9.1 模塊定義

```rust
// src/lib.rs
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    front_of_house::hosting::add_to_waitlist();
}
```

---

### 9.2 文件模塊

```
src/
  ├── lib.rs
  ├── front_of_house.rs
  └── front_of_house/
      └── hosting.rs
```

```rust
// src/lib.rs
mod front_of_house;

pub use front_of_house::hosting;
```

---

### 9.3 `use` 關鍵字

```rust
use std::collections::HashMap;
use std::io::{self, Write}; // 引入多個項目
use std::fmt::Result;
use std::io::Result as IoResult; // 別名

fn main() {
    let mut map = HashMap::new();
    map.insert(1, 2);
}
```

---

### 9.4 可見性 (Visibility)

- `pub`: 公開
- (默認): 私有
- `pub(crate)`: crate 內公開
- `pub(super)`: 父模塊公開
- `pub(in path)`: 指定路徑內公開

```rust
mod outer {
    pub mod inner {
        pub(crate) fn crate_visible() {}
        pub(super) fn parent_visible() {}
        fn private() {}
    }
}
```

---

## 10. Unsafe Rust

### 10.1 Unsafe 能力

在 `unsafe` 塊中可以:

1. **解引用裸指針**
2. **調用 unsafe 函數或方法**
3. **訪問或修改可變靜態變量**
4. **實現 unsafe trait**
5. **訪問 union 的字段**

```rust
fn main() {
    let mut num = 5;
    
    let r1 = &num as *const i32;      // 不可變裸指針
    let r2 = &mut num as *mut i32;    // 可變裸指針
    
    unsafe {
        println!("r1: {}", *r1);
        *r2 = 10;
        println!("r2: {}", *r2);
    }
}
```

---

### 10.2 FFI (Foreign Function Interface)

**調用 C 函數**:

```rust
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe {
        println!("C 的 abs(-3) = {}", abs(-3));
    }
}
```

---

## 11. 宏 (Macros)

### 11.1 聲明宏 (Declarative Macros)

```rust
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

---

### 11.2 過程宏 (Procedural Macros)

**派生宏 (Derive Macro)**:

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}
```

**屬性宏 (Attribute Macro)**:

```rust
#[tokio::main]
async fn main() {
    // ...
}
```

**函數宏 (Function-like Macro)**:

```rust
println!("Hello, {}!", "world");
```

---

## 12. 測試 (Testing)

### 12.1 單元測試

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_add() {
        assert_eq!(2 + 2, 4);
    }
    
    #[test]
    #[should_panic]
    fn test_panic() {
        panic!("這應該 panic");
    }
    
    #[test]
    fn test_result() -> Result<(), String> {
        if 2 + 2 == 4 {
            Ok(())
        } else {
            Err(String::from("失敗"))
        }
    }
}
```

---

### 12.2 集成測試

```
project/
  ├── src/
  │   └── lib.rs
  └── tests/
      └── integration_test.rs
```

```rust
// tests/integration_test.rs
use my_crate;

#[test]
fn test_integration() {
    assert_eq!(my_crate::add(2, 2), 4);
}
```

---

## 13. 常用屬性 (Attributes)

| 屬性 | 說明 |
|------|------|
| `#[derive(...)]` | 自動派生 trait 實現 |
| `#[cfg(test)]` | 條件編譯 (測試) |
| `#[allow(dead_code)]` | 抑制編譯器警告 |
| `#[deprecated]` | 標記為廢棄 |
| `#[must_use]` | 返回值必須使用 |
| `#[inline]` | 建議內聯 |
| `#[no_std]` | 不使用標準庫 |

```rust
#[derive(Debug, Clone)]
#[allow(dead_code)]
struct Config {
    #[deprecated(since = "1.0.0", note = "使用 new_field")]
    old_field: i32,
    new_field: i32,
}

#[must_use]
fn important_function() -> i32 {
    42
}
```

---

## 14. Cargo 基礎

### 14.1 常用命令

```bash
cargo new project_name      # 創建新項目
cargo build                 # 構建項目
cargo build --release       # 發布構建
cargo run                   # 運行項目
cargo test                  # 運行測試
cargo check                 # 快速檢查代碼
cargo doc --open            # 生成並打開文檔
cargo fmt                   # 格式化代碼
cargo clippy                # Lint 檢查
```

---

### 14.2 `Cargo.toml` 基礎

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.35", features = ["full"] }

[dev-dependencies]
criterion = "0.5"

[profile.release]
opt-level = 3
lto = true
```

---

## 15. 內存佈局

### 15.1 棧 vs 堆

| 特性 | 棧 (Stack) | 堆 (Heap) |
|------|-----------|----------|
| **速度** | 快 | 慢 |
| **大小** | 固定，編譯期已知 | 可變，運行期確定 |
| **分配** | 自動 | 手動 (`Box`, `Vec`, 等) |
| **生命期** | LIFO，作用域結束自動釋放 | 引用計數或所有權管理 |
| **類型示例** | `i32`, `[i32; 5]` | `String`, `Vec<T>`, `Box<T>` |

---

### 15.2 大小已知 vs 未知

**大小已知 (Sized)**:
- 大多數類型 (`i32`, `String`, `Vec<T>`)
- 可以在棧上分配

**大小未知 (Unsized / DST - Dynamically Sized Types)**:
- `str` (字符串切片)
- `[T]` (數組切片)
- `dyn Trait` (Trait 對象)
- 必須通過指針訪問: `&str`, `Box<[T]>`, `Box<dyn Trait>`

---

## 16. 零成本抽象 (Zero-Cost Abstraction)

**定義**: Rust 的抽象不會引入運行時開銷，編譯器會優化到與手寫底層代碼相同的性能。

**示例**:

```rust
// 高層抽象
let sum: i32 = (1..=100).filter(|x| x % 2 == 0).sum();

// 編譯後性能等同於手寫循環
let mut sum = 0;
for i in 1..=100 {
    if i % 2 == 0 {
        sum += i;
    }
}
```

**關鍵原則**: "你不使用的功能不會付出代價，你使用的功能無法更高效實現"。

---

## 總結

這份速查表涵蓋了 Rust 的所有核心概念:

1. **所有權系統**: Rust 最獨特的記憶體管理機制
2. **類型系統**: 強大的靜態類型與泛型
3. **Trait 系統**: Rust 的多態與代碼復用機制
4. **錯誤處理**: `Result` 和 `Option` 的函數式錯誤處理
5. **併發**: 安全的多線程與異步編程
6. **Unsafe**: 需要時可以突破安全限制
7. **宏**: 元編程能力
8. **模塊**: 代碼組織

掌握這些概念後，你就可以深入學習後續的進階主題。建議在閱讀其他章節時，隨時回來查閱這份速查表以鞏固基礎。

---

## 參考資料

1. [The Rust Programming Language (官方書)](https://doc.rust-lang.org/book/)
2. [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
3. [The Rustonomicon (Unsafe Rust)](https://doc.rust-lang.org/nomicon/)
4. [Rust Reference](https://doc.rust-lang.org/reference/)
5. [Rust Standard Library Documentation](https://doc.rust-lang.org/std/)
