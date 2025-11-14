# fn 與 Fn
---
### 函數變量與類型
在 Rust 中函數可以做為變量一樣進行傳遞，類型為 `fn` (依據參數與返回值而不同)，函數變量本質是函數指針，與 C/C++ 中的概念相同。
```rust
let process(socket: TcpStream) -> i64 { ... }

fn main() {
	// 將函數作為變量
	let p: fn(TcpStream) -> i64 = process;
}

// 函數接收函數變量
fn dispatch(f: fn(TcpStream) -> i64) { ... }
```

### Closure
Closure 中文為閉包，與函數相同只是閉包可以**捕獲外部變量**進行操作，所以除了函數指針以外還須要有空間存放捕獲的數據，可以直接用 Class 中的字段的去理解，所以類型與函數的 `fn` 並不相同。閉包的類型為 `Fn`
```rust
fn count_selected<F>(selections: &Vec<City>, f: F) -> usize 
	where F: Fn(&City) -> bool
{ ... }
```

#### Closure 很快
在其他語言中會將 Closure 放在 heap 中，並且會進行分配與垃圾回收，所以建立、呼叫與回收都會消耗性能，且 closure 還會導致無法進行內聯優化，Rust 中不會將它放在 heap 中，編譯器也會對空間優化與函數進行內聯，空間布局如下:
![[Pasted image 20241113223643.png|800]]
> 第一種只占用兩個索引空間，第二種會獲取所有權，第三種完全不占用空間。

# 三種 Closure Traits
---
### 內部操作方式決定 Trait 種類
Closure 可以捕獲外部變數，在 Rust 的安全機制下捕獲方式必須明確，一共有三種捕獲方式，也分別對應不同的 Closure Trait:
1. 共享引用: `Fn`
2. 獨佔引用: `FnMut`
3. 獲得所有權: `FnOnce`
而編譯器決定 **Closure 實作的 Trait 是藉由 Closure 內部是怎麼操作捕獲變數**。
```rust
struct People {
	name: String,
	age: i32
}

impl People {
	fn show_detail(&self) { ... }
	fn chage_name(&mut self, new_name: String) { ... }
}

fn main() {
	// Example 1
	let p1 = People::new( {...} );
	let f1: Fn() = || { p1.showDetail(); }  // 閉包只需捕獲 &p1 (共享引用)

	// Example 2
	let mut p2 = People::new( {...} );
	let mut f2: FnMut() = || { p1.change_name("Liam"); } // 閉包捕獲 &mut p2 (獨佔引用)

	// Example 3
	let p3 = People::new( {...} );
	let f2: FnOnce() = || { drop(p3); } // 閉包捕獲 p3 (所有權): 因為 drop(self)

	// Example 4
	let p4 = People::new( {...} );
	let f4: Fn() = move || { p1.showDetail(); }  // 閉包只需捕獲 &p1 (共享引用)，就算添加 move 也不會改變類型
}
```

> Q: 那 `move` 用在哪?
> A: 當我們使用 thread::spawn 時候，因為線程有獨立的生命期，故閉包需要將捕獲變量的所有權也就是需要 `FnOnce`，此時因為編譯器會依據內部實現決定閉包類型，故使用 move 告訴編譯器，就算裡面只使用引用但還是會移入。
> **決定閉包類型的是內部的使用而不是 `move` 關鍵字**

```rust
// Example 1
let p1 = People::new( {...} );
let f1: Fn() = move || p1.show_detaul(); // 一樣是 Fn Trait
f1();
f1(); // error: 不能調用第二次，因為 p1 已經被移動

// Example 2
fn aa<F>(f: F)
where
    F: Fn(),
{
    f()
}

fn bb<F>(f: F)
where
    F: FnMut(),
{
    f()
}

let p1 = People::new( {...} );
let f1: Fn() = move || p1.show_detaul(); // 一樣是 Fn Trait
aa(f1); // Ok 沒問題!
aa(f1); // error: 不能調用第二次，因為 p1 已經被移動
```
> 注意 Example 2，若 f1 真的因為 `move` 而變成 `FnOnce`，第一次調用 `aa` 會出現編譯錯誤，因為 `FnOnce` > `Fn`。


### 包含關係
* `Fn` 屬於 closure + 函數(`fn`)族，可以無限次調用。
* `FnMut` 屬於 closure 族，clousre 本身被宣告 `mut` 就能夠無限次調用。
* `FnOnce` 屬於 closure 族，只能被調用一次。
`Fn` 可以滿足 `FnMut` 的使用條件(標記 `mut` 就能無限調用)，又 FnMut 可以滿足 `FnOnce` 的使用條件(至少能被調用一次)。
> `Fn` 為 `FnMut` 的子 Trait，`FnMut` 為 `FnOnce` 的子 Trait

### 每個 Closure 都是不同類型
儘管同樣是實作 `Fn(&Request) -> Response` Trait 的 closure，也會是完全不同的類型，Rust 會告訴你 `no two closures, even if identical, have the same type`，可以理解為因為捕獲外部變量，導致每個 closure 實際占用的空間大小不同，概念同實作相同 Trait 的不同類型。

以下案例:
我們自己建立一個 router，他可以配置路由對應的 closure
```rust
use std::collections::HashMap;

struct Request {
    method: String,
    url: String,
    headers: HashMap<String, String>,
    body: Vec<u8>,
}

struct Response {
    code: u32,
    headers: HashMap<String, String>,
    body: Vec<u8>,
}

struct BasicRouter<C>
where
    C: Fn(&Request) -> Response,
{
    // map for url to method
    routes: HashMap<String, C>,
}

impl<C> BasicRouter<C>
where
    C: Fn(&Request) -> Response,
{
    fn new() -> BasicRouter<C> {
        BasicRouter {
            routes: HashMap::new(),
        }
    }

    fn add_route(&mut self, url: &str, callback: C) {
        self.routes.insert(url.to_owned(), callback);
    }
}

fn main() {
    let mut router = BasicRouter::new();
    let msg = "Hello";
    // 第一次使用沒問題
    router.add_route("/", |_| Response {
        code: 200,
        headers: HashMap::new(),
        body: msg.into(),
    });
	// 第二次使用出錯了
    router.add_route("/", |_| Response { // Error: no two closures, even if identical, have the same type
        code: 200,
        headers: HashMap::new(),
        body: msg.into(),
    });
}
```

#### 解決方式: 使用 dyn + Box
```rust
struct Request {
    method: String,
    url: String,
    headers: HashMap<String, String>,
    body: Vec<u8>,
}

struct Response {
    code: u32,
    headers: HashMap<String, String>,
    body: Vec<u8>,
}

type BoxedCallback = Box<dyn Fn(&Request) -> Response>;
struct BasicRouter {
    // map for url to method
    routes: HashMap<String, BoxedCallback>,
}

impl BasicRouter {
    fn new() -> BasicRouter {
        BasicRouter {
            routes: HashMap::new(),
        }
    }

    fn add_route<C>(&mut self, url: &str, callback: C)
    where
        C: Fn(&Request) -> Response + 'static,
    {
        self.routes.insert(url.to_owned(), Box::new(callback));
    }
}
```

##### 為甚麼 Trait bound 需要加上 `'static`
ref: [闭包碰到特征对象-01 - Rust语言圣经(Rust Course)](https://course.rs/compiler/fight-with-compiler/lifetime/closure-with-static.html)
`'static` 在這邊表示閉包的生命期要為靜態生命期。

**Trait Object 隱式具有 `'static` 生命期**，故 `Box<dyn Fn(&Request) -> Response>` 需具備 'static 生命期，但 `Fn` 生命期由捕獲變數而定，故需額外指定 Fn Trait 為靜態生命期。