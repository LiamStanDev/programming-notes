# 三種結構體
---

Rust 的結構體把多個類型的值組成單個值，可以將他們當作一個整體進行處裡。
共有三種類型的結構體:
1. name-field
2. tuple-like
3. unit-like


### Name-Field Structs
```rust
struct GrayscaleMap {
	pixels: Vec<u8>,
	size: (usize, usize),
}
```
* 結構體名稱: 大駝峰
* 字段、方法: 蛇行
* 結構體本身為 `pub`，字段也是不一定可見，需要字段也為 `pub` 才可見。

##### 字段不可見如何創建結構體?
Ans: 使用結構體的關聯函數。

像是 String, Vec 這些都無法使用 struct expression 創建，如下:
```rust
let image = GrayscaleMap { 
	pixels: vec![0; width * height], 
	size: (width, height) 
};
```
這是因為他們兩個字段是私有的，我們可以透過他的關聯函數創建，
```rust
let s = String::from("Hello");
let v = Vec::<String>::new();
```


### Tuple-Like Structs
```rust
struct Bounds(usize, usize); // private

pub struct Bounds(pub usize, usize); // public
```

###### New type
```rust
struct Ascii(Vec<u8>);
```
> 後續章節會在說明，他的含意。


### Unit-Like Structs
```rust
struct Onesuch;
```
* 不占用內存空間


# 結構體布局 (Struct Layout)
---
```rust
struct GrayscaleMap {
	pixels: Vec<u8>,
	size: (usize, usize),
}
```
* Rust 並不會保證在內存中的順序如同字段順序一樣排列，但你可以使用 `#[repr(C)]` 來保證與 C, C++ 的布局兼容性。
* Rust 保證在結構體內存塊中放入儲存字段的值。
> Python 會將 pixels 與 size 放在堆空間，然後讓 GrayscaleMap 的字段值向他們。
> Rust 會把 pixels 與 size 放到 GrayscaleMap 值裡面，有 pixels 持有的 vector 的 buffer 放在堆上 (也就是 vector 的其他資訊放在 GrayscaleMap 上)。

# impl 塊
---
* 在 impl 定義的函數稱為關聯函數，定義在外的反而稱為自由函數。
* 方法需要使用 self, &self, &mut self 作為方法第一個參數。
* 只能顯示使用 self 才能獲取使用字段。
* Rust 能知道方法的接收參數是甚麼進行動轉換
```rust
let mut v = Vec::new();
v.push("H");
// 以下兩種方式跟上面相同
(&mut v).push("H");
Vec::push(&mut v, "H");
```

### 方法
#### 為甚麼要分開數據成員與方法?
* 很容易看完數據成員，不用瀏覽幾百行還可能有遺漏
* 可以用同一種方式定義三種結構體與 enum, i32 等的方法 (故 Rust 不使用對象稱之而稱所有為值，Rust 可以為所有值添加方法)。
* impl 可以很容易實現 trait


#### Box, Rc, Arc 傳遞 Self

##### 自動借用
我們可以使用 Box, Rc, Arc 等作為接收者如下，
```rust
struct Queue {
    older: Vec<char>,
    younger: Vec<char>,
}

impl Queue {
    fn add_older(mut self: Box<Queue>, data: char) -> Box<Self> {
        self.older.push(data);
        self
    }
}
```
但這樣只能使用 Box 這個智能指針進行調用
```rust
fn main() {
	// 使用 Box
    let mut q = Box::new(Queue {
        older: Vec::new(),
        younger: Vec::new(),
    });

    q = q.add_older('c'); // Ok
    
	// 不使用 Box
    let mut p = Queue {
        older: Vec::new(),
        younger: Vec::new(),
    };

    p.add_older('c'); // Error
}
```
這個情況下沒有甚麼必要，因 **Rust 能自動的從 `Box`, `Rc` 與 `Arc` 中借用出 `&self` 與 `&mut self` 引用**，所以使用 `&self` 與 `&mut self` 是一般性正確的方法簽名。
```rust
impl Queue {
    fn add_older(&mut self, data: char) {
        self.older.push(data);
    }
}

fn main() {
	// 使用 Box
    let mut q = Box::new(Queue {
        older: Vec::new(),
        younger: Vec::new(),
    });

    q.add_older('c'); // Ok

	// 不使用 Box
    let mut p = Queue {
        older: Vec::new(),
        younger: Vec::new(),
    };

    p.add_older('c'); // Ok
}
```
> 使用 `&self` 與 `&mut self` 更具一般性。

##### 管理指針的所有權
想像我們有一個 Node 它裡面有數值 val 與子節點 children，我們希望讓子節點有方法添加進父節點，但是節點本身所有權不只被父節點管理。
```rust
struct Node {
	val: String,
	children: Vec<Rc::<Node>>, // 子節點可以有多人持有
}

impl Node {
	fn append_to(self, parent: &mut Node) {
		parent.children.push(Rc::new(self)); 
	}
}
```
這樣寫沒有意義，因為我們將子節點的所有權傳給了父節點，子節點無法共享。
所以我們應該讓子節點傳入的是 Rc 指針，故改寫為，
```rust
impl Node {
	fn append_to(self: Rc<Node>, parent: &mut Node) {
		parent.children.push(Rc::clone(&self)); 
	}
}
```
這樣原本的所有者還能使用子節點，運行方式如下:
```rust
fn main() {
	let mut root = Node {
		val: "root".to_string(),
		children: Vec::new(),
	};

	let owned = Rc::new(Node {
		val: "owned",
		children: Vec::new(),
	});

	owned.append_to(&mut parent);
}
```
###### 總結
接收者有幾種設計思路:
* 方法只讀不消耗所有權: `&self`
* 方法可變不消耗所有權: `&mut self`
* 方法消耗所有權: `self`
* 方法獲取所有權並增加引用計數: `self: Rc<Self>`，且在 `Rc::clone()` 完成後傳遞
* 方法獲取引用計數本身: `self: Rc<Self>`，並直接傳遞尚未 `clone` 的 `Rc`

### 類型關聯函數
與類型綁定的函數，函數中餐數沒有接收者，與自由函數的差別只在於語意。

### 類型關聯常量
在 impl 塊中也能建立類型關聯的常量，
```rust
pub struct Vector2 {
	x: f32,
	y: f32,
}

impl Vector2 {
	const ZERO: Vector2 = Vector2 { x: 0.0, y: 0.0 };
	const UNIT: Vector2 = Vector2 { x: 1.0, y: 0.0};
	const NAME: &'static str =  "Vector2";
	const ID: i32: 18;
}
```

# 泛型結構體
---
```rust
// 泛型結構體
pub struct Queue<T> {
    older: Vec<T>,
    younger: Vec<T>,
}

// 對所有類型定義
impl<T> Queue<T> {
    pub fn new() -> Self {
        Self {
            older: Vec::new(),
            younger: Vec::new(),
        }
    }

    pub fn push(&mut self, t: T) {
        self.younger.push(t);
    }

    pub fn is_empty(&self) -> bool {
        self.older.is_empty() && self.younger.is_empty()
    }
}

// 對指定類型定義
impl Queue<f64> {
    pub fn sum(&self) -> f64 {
        let mut res = 0.0;
        for num in &self.older {
            res += num;
        }
        for num in &self.younger {
            res += num;
        }
        res
    }
}
```

# 生命期參數結構體
---
結構體生命期參數表示其字段的引用指向的值至少要存活的跟結構體一樣久，也就是在**結構體被 drop 之前，裡面字段引用的值都還活者**。
```rust
pub struct Extrema<'a> {
    pub greatest: &'a i32,
    pub least: &'a i32,
}

impl<'a> Extrema<'a> {
    pub fn find_extrema(slice: &'a [i32]) -> Extrema<'a> { // 此處是可以省略生命期參數的
        let mut greatest = &slice[0];
        let mut least = &slice[0];

        for i in 1..slice.len() {
            if slice[i] < *least {
                least = &slice[i];
            }
            if slice[i] > *greatest {
                greatest = &slice[i];
            }
        }

        Extrema { greatest, least }
    }
}
```

# 派生常見的 trait
---
目前我們自定義的結構體甚麼實用的功能都沒有，我們可以使用`#[derive()]`自動實現標準庫提供的特性:
* `Copy`: 字節複製，不涉及堆上複製
* `Clone`: 整體都進行拷貝，包含堆上數據
* `Debug`: 可以使用 `"{:?}"` 進行打印
* `PartialEq`: 可以使用相等運算符 `==`
* `PartialOrd`: 可以使用比較運算符 `>`, `<`, `>=`, `<=`
只要字段都有實現這些 trait，結構體就能使用 `#[derive()]` 進行派生。
我們也可以手動使用 impl 實現這些特徵，但這邊我們沒有

##### 為甚麼 Rust 不自動實現非得我們手動派生?
因為只要自動派生，這些 API 會變得公有訪問，但有些結構體語意上並不能支持或者不允許這些操作。
```rust
#[derive(Copy, Clone, Debug, PartialEq)]
pub struct Point {
	x: f64,
	y: f64
}
```

# Interior Mutability
---
可變性與共享性同時存在是導致所有問題的根源，但我們有時候希望不要這麼死板的進行限制。
##### 例子
我們有一個蜘蛛機器人
```rust
pub struct SipderRobot {
	species: String,
	web_enabled: bool,
	log_devices: [fd::FileDesc, 8]; // 可寫入的日誌裝置有哪些
}
```
在蜘蛛機器人值被初始化後我們就讓它永遠不能改變。
有各種系統都會持有機器人 (共享同一個機器人)，如捕時、攻擊與睡眠等，
```rust
pub struct SpiderSeneces {
	robot: Rc<SpiderRobot>,
	eyes: [Camera; 32],
	motion: Accelerometer,
}
```
此時我們希望進行日誌任務，但是日誌必須使用 File，但是 File 必須為 `mut`，所有寫入的函數都必須使用 `&mut`。
但是 Rc 因為是共享，所以	Rust 不可能允許共享下可變，我們*需要一個不可變的值 (本例為 SpiderRobot 結構體) 的內部有一小部分數值可以改變*。
這就是內部可變性模式，Rust 為此提供幾種方式，我們會討論最值觀的 `Cell<T>` 與 `RefCell<T>` (都在 `std::cell` 模塊中)。

### Cell
`Cell<T>` 結構體只包含單個私有類型為 `T` 的值，它的特點就是當你就算沒有將 Cell 的所有者變量設為 `mut`，我們還是有辦法修改裡面的數據。
* `Cell::new(value)`: 建立新的 Cell，並將 value 移動進去。
* `cell.get()`: 取得 Cell 內部的值
* `cell.set(value)`: 設定 Cell 內部的值
	* 他的簽名很特別 `fn set(&self, value: T)`，並不為 `&mut self`

```rust
pub struct SipderRobot {
	...
	counter: Cell<u32>, // Counter 會被修改
	...
}

impl SipderRobot {
	fn add_one(&self) { // 不使用 &mut self
		let n = self.counter.get(); // 取得 cell 的值
		self.counter.set(n + 1); // 設定 cell 的值
	}
}
```
但是這並不能解決問題，因為 `cell.get()` 返回的是拷貝，所以只能使用實現了 `Copy` trait 的類型，而 File 並不是 `Copy` 類型。

### RefCell
`RefCell` 與 `Cell` 一樣只包含單個 T 類型的值，不同的是 RefCell 可以借用內部的值，而不是像 Cell 會拷貝出新的值。
* `RefCell::new(value)`: 創建新的 RefCell 並把 value 移動進去。
* `ref_cell.borrow()`: 返回 `Ref<T>` 本值為內部值的一個共享引用，若同時內部值已經存在可變借用，該方法會 panic
* `ref_cell.borrow_mut()`: 返回 `RefMut<T>` 本值為內部值的一個可變引用，若當下內部值已經被借用，該方法會 panic
* `ref_cell.try_borrow()`, `ref_cell.try_borrow_mut()`: 返回 Result，並不會 panic。
> 只有當你打破可變引用的獨佔性時才會觸發 panic

##### 性能
普通的引用錯誤會在編譯時期得到編譯錯誤，而 RefCell 不會在編譯時期錯誤而是遞延到運行時報錯，故他會在運行時存在性能損失。

##### 解決 SpiderRobot 問題
```rust
pub struct SpiderRobot {
	...
    log_file: RefCell<File>
    ...
}

impl SpiderRobot {
	pub fn log(&self, message: &str) {
		let mut file = self.log_file.borror_mut(); 
		writeln!(file, "{}", message).unwrap();
	}
}
```
* 現在 file 的類型為 `RefMut` 結構體。
* file 不是單純的可變引用，而是結構體，所以我們要修改內部的值要使用 `mut` 標記變量。

##### Cell 與 RefCell 的限制
這兩者都不是多線程安全的，之後會介紹 `Mutex<T>`，原子量與全局變量會在介紹多線程的內部可變性。

