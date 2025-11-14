枚舉是一個強大的自定義數據類型，在 Rust 中非常強大，相較於 C++ 與 C# 用於限制取值範圍，Rust 更像 C 語言中的 Union，但 Union 是不安全危險的，而 Rust 則是類型安全的。
_枚舉用於一個值有多種可能的情況，使用他的代價就是需要使用模式匹配來安全訪問_。

# Enum
---
### C 風格枚舉
```rust
enum Ordering {
	Less,
	Equal,
	Greater,
}

fn compare(n: i32, m: i32) -> Ordering {
	if n < m {
		Ordering::Less
	} else if n > m {
		Ordering::Greater
	} else {
		Ordering::Equal
	}
}
```
在 C 風格枚舉會被儲存成整數，默認會從 0 開始，有時候可以告訴 Rust 他們是甚麼值
```rust
enum HttpStatus {
	Ok = 200,
	NotFound = 404,
	NotModified = 304,
	...
}
```

* C 風格的枚舉可以使用 `as` 轉換成整數，但反向不行。
* 枚舉能實現 trait
```rust
#[derive(Copy, Clone, Debug, PartialEq, Eq)]
pub enum TimeUnit {
    Seconds,
    Minuts,
    Hours,
    Days,
    Months,
    Year,
}
```
* 枚舉也能擁有方法
```rust
impl TimeUnit {
    pub fn plural(self) -> &'static str {
        match self {
            Self::Seconds => "seconds",
            Self::Days => "days",
            Self::Hours => "hours",
            ...
        }
    }
}
```

### 帶數據的枚舉
##### _tuple variant_
```rust
enum RoughTime {
	InThePast(TimeUnit, u32),
	JustNow,
	InTheFuture(TimeUnit, u32),
}

let t = RoughTime::InThePast(TimeUnit::Years, 4 * 20 + 7);
```
##### _struct variant_
```rust
enum Shape {
	Sphere { center: Point3d, radius: f32 },
	Cuboid { corner1: Point3d, corner2: Point3d },
}

let s = Shape::Sphere { center: ORIGIN, 1.0 };
```

Rust 共有三種 variants:
1. Unit-like variants: 也就是 C 風格 enum
2. Tuple-like variants
3. Struct-like variants
一個枚舉可能同時存在三種不同的 variant。

### Enum Layout (枚舉的內存布局)
在內存中存放方式類似 C 的聯合體，都是暫用同一塊區域，而區域大小使用枚舉中占用最大的大小。

#### 使用枚舉
枚舉在樹形的結構非常有用，
###### 案例一: Json
Json 中像樹型一層欠套一層，我們可以使用枚舉來表達，
```rust
pub enum Json {
    Null,
    Boolean(bool),
    Number(f64),
    String(String),
    Array(Vec<Json>),
    Object(Box<HashMap<String, Json>>),
}
```
* HashMap 使用 Box 是因為想要讓空見更緊湊
	* Null 占用 1 byte
	* Boolean 占用 1 byte
	* Number 占用 1 word (64 bits = 8 bytes)
	* String 占用 3 words
	* Array 占用 3 words
	* Object 占用 1 word ，若使用 HashMap 為 8 words
	* 共占用 4 words (包含對齊)

###### 案例二: 二叉數
```rust
pub enum BinaryTree<T> {
    Empty,
    NotEmpty(Box<TreeNode<T>>),
}

pub struct TreeNode<T> {
    pub elem: T,
    pub left: Box<BinaryTree<T>>,
    pub right: Box<BinaryTree<T>>
}
```
要習慣 Rust 的數據布局最好的方式就是畫圖:
* 箭頭: 表示為 Box 或者其他的智能指針
* 矩形: 表示為結構體或者 tuple

這樣設計 BinaryTree 只會占用 1 word，因為 NotEmpty 指占用一個指針。 

# Pattern
---
我們得到了 Enum 這個強大得的工具，但隨之而來的是代價，也就是枚舉的訪問必須使用 Pattern (模式) 這種安全的方式。

* 被匹配的值裡面的任何內容都會被*拷貝或者移動到新變量中*

> 模式與表達式是天然的相反概念，表達式建立值，模式消耗值，或者說**表達式把值裝進結構中，而模式把值取出結構中**
##### 為甚麼要用模式?
因為 enum 與 C 語言的 Union 一樣，我們更甚者 Rust 還無法保證布局與 C 一樣，在我們不知道實際會是哪個 variant 時，我們就取得值是不安全操作內存的方式，而模式是藉由匹配來判定是否滿足某個 variant，故這是一種安全的訪問方式。

| 模式類型   | 示例                                                             | 註釋                                     |
| ------ | -------------------------------------------------------------- | -------------------------------------- |
| 字面量    | 100<br>"name"                                                  | 匹配一個精確值                                |
| 範圍     | 0..=100<br>'a'..='k                                            | 匹配範圍內的所有值                              |
| 通配符    | _                                                              | 匹配任何值並忽略                               |
| 變量     | name<br>mut count                                              | 與_相同，匹配所有值，但把值移動或拷貝進局部變量               |
| ref 模式 | ref field<br>ref mut field                                     | 借用匹配的值                                 |
| 引用模式   | &value<br>&(k,v)                                               | 指匹配引用                                  |
| 綁定模式   | val @ 0..=99<br>ref circle @ Shape::Circle { .. }              | 匹配右側的值，使用左側為變量名                        |
| 枚舉模式   | Some(value)<br>None<br>Pet::Orca                               |                                        |
| 數組模式   | [a,b,c,d,e]<br>[heading, carom, correction]                    |                                        |
| 切片模式   | [first, second]<br>[first, _, third]<br>[first, .., nth]<br>[] |                                        |
| 結構體模式  | Color(r,g,b)<br>Point { x, y }<br>Account { id, name, ... }    |                                        |
| 多重模式   | 'a' \| 'A'                                                     | 只能作用於可反駁的模式 (match, if let, while let) |
| 守衛表達式  | x if x * x <= 2                                                | 只能用在 match                             |

##### Literals, Variables, and Wildcards in Pattern
看上表

##### Tuple and Struct Patterns
看上表

##### Array and Slice Patterns
```rust
// 數組模式
fn hsl_to_rgb(hsl: [u8; 3]) -> [u8;3] {
    match hsl {
        [_, _, 0] => [0,0,0],
        [_, _, 255] => [255, 255, 255],
        ...
    }
}

// 切片模式
fn greet_people(names: &[&str]) {
    match names {
        [] => {
            println!("Hello, nobody.")
        }
        [a] => {
            println!("Hello, {}.", a)
        }
        [a, b] => {
            println!("Hello, {} and {}.", a, b)
        }
        [a, .., b] => {
            println!("Hello, everyone from {} to {}.", a, b)
        }
    }
}
```

##### Reference Patterns (重要)
因為模式匹配後會發生移動 (除非 Copy trait 類型)，這時候不希望消耗它的所有權，我們可以使用引用模式， Rust 支持兩種引用模式:
1. `ref` 模式: 包含 `ref` 與 `ref mut`
2. `&` 模式: 包含 `&` 與 `&mut`
###### ref 模式
之前的匹配產生了以下問題
```rust
match account {
	Account { name, language, .. } => {
		ui.greet(&name, &language);
		ui.show_settings(&account); // Error! 這邊 account 所有權已經被消耗掉了，移動給 name 與 language
	}
}
```
我們可以使用 ref 模式來借用內部的引用
```rust
match account {
	Account { ref name, ref language, .. } => { // 這時候 name 與 language 就是引用了
		ui.greet(name, language);
		ui.show_settings(&account); // Ok
	}
}
```

###### & 模式
與 ref 模式正好相反，若我們匹配的是引用，則在模式中使用 `&` 會變成值。
```rust
match friend.borrow_car() { // 這邊返回的是 &Car
	Some(&Car { engine, .. }) => // error: can't move out of borrow 
	... 
	None => {} 
}
```
![[Pasted image 20240420111826.png|600]]
如上圖所示，x, y, z 是 Copy 類型，所以他們發生拷貝，但如果不是則會發生移動，但因為本身 &Point 就是借用，所以不行獲得所有權，所以還得搭配 ref 來進一步獲取內部引用。
> 這樣做的好處是**先去掉一層引用，然後再使用 ref 在取得內部引用**。

##### 綁定模式
`x @ pattern` 用 pattern 來匹配，然後將成功匹配的值移動或拷貝進去 x。
```rust
match chars.next() { 
	Some(digit @ '0'..='9') => read_number(digit, chars),
	...
}
```

### 模式可以用在哪?
##### 不可反駁模式 (irrefutable pattern)
表示一定能成功的匹配，如同 JS 的解構(destructuring)與 Python 的解包(unpacking)。
```rust
// 解構一個結構體
let Track { album, track_number, title, .. } = song; 

// 解構一個 tuple 參數
fn distance_to((x, y): (f64, f64)) -> f64 { .. } 

// 迭代 hashmap 鍵值對
for (id, document) in &cache_map { 
	println!("Document #{}: {}", id, document.title); 
}

// 解引用閉包參數
let sum = numbers.fold(0, |a, &num| a + num);
```

##### 可反駁模式 (refutable pattern)
表示存在可能不匹配的模式，常用於 Option 或者 Result。
1. match 表達式
2. if let 表達式
3. while let 表達式
```rust
if let Some(document) = cache_map.get(&id) {
	return send_cached_response(document);
}

while let Err(err) = present_cheesy_anti_robot_task() {
	log_robot_attempt(err);
	// 讓用戶重試
}
```


### 實現排序二叉樹
```rust
pub struct TreeNode<T> {
    pub elem: T,
    left: Box<BinaryTree<T>>,
    right: Box<BinaryTree<T>>,
}

pub enum BinaryTree<T> {
    Empty,
    NotEmpty(Box<TreeNode<T>>),
}

impl<T: Ord> BinaryTree<T> {
    pub fn add(&mut self, value: T) {
        match *self {
            BinaryTree::Empty => {
                *self = BinaryTree::NotEmpty(Box::new(TreeNode {
                    elem: value,
                    left: Box::new(BinaryTree::Empty),
                    right: Box::new(BinaryTree::Empty),
                }));
            },
            BinaryTree::NotEmpty(ref mut boxed_node) => {
                if value <= boxed_node.elem  {
                    boxed_node.left.add(value);
                } else {
                    boxed_node.right.add(value);
                }
            }
        }
    }
}
```