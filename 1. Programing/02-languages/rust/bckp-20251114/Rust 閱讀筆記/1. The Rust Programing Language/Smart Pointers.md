# Intro
---
* A pointer is a general concept for **a variable that contains an address** in memory.
* The most common kind of pointer in Rust is a **reference**.
	* reference 與 raw pointer 無異，只是 reference 需要遵守 pointer memory safty principal (borrow checker).

##### What is smart pointer ?
Smart pointer are data structure that act like a pointer but also **have additional metadata and capability**.
* References only borrow data, smart pointers **own** the data they point to.
* Smart pointers implement the **Deref** and **Drop** traits.
* Both `String` and `Vec` are smart pointers.


# Box Point Data on the Heap
---
* Boxes allow you to store data on the **heap** rather than the stack.
* Don’t have performance overhead.

###### Situation for using box
* Having a type with **dynamic size** whose size can't be known at compile time.
* When Having large data but ensure the data **won't be copied**.
* **Trait object**.

```rust

let x: [Box<(usize, usize)>; 4] // x has 32 bytes on stack.
```




# Deref and Drop Trait
---
### Deref Trait
#### Deref
###### Signature
```rust
pub trait Deref {
    type Target: ?Sized; // ? 表示可以是 Sized or Unsized.

    fn deref(&self) -> &Self::Target;
}
```
* Used for immutable dereferencing operations, like `*v`.

#### DerefMut
###### Signature
```rust
pub trait DerefMut: Deref {
    
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```
* Used for mutable dereferencing operations, like in `*v = 1;`

#### Example: Implement Custom Smart Pointer
Implement a smart pointer which implement `Deref` and `DerefMut`.

```rust
use std::ops::{Deref, DerefMut};

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(val: T) -> MyBox<T> {
        MyBox(val)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl<T> DerefMut for MyBox<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}

fn main() {
    let mut x = MyBox::new(22);

    println!("value: {}", *x); // 實際上: *(x.deref())
    *x = 55; // 實際上: *(x.deref_mut()) = 55
    println!("value: {}", *x);
}
```

#### Implicit Deref Coercion
* Converts a reference to a type that implements the `Deref` trait into a **reference to another type**.
* Performs on **arguments** to functions and methods (the only situation)
###### Example: String
`String` 為 Smart Pointer，有 implement Deref Trait，定義如下:
```rust
impl Deref for String {
		type Target = str;
		fn deref(&self) -> &Self::Target;
}
```
* 對 string 解引用，會得到 `str` 如下:
```rust
let s = String::from("Hello"); 

let b: &str = s.deref();
```
* 隱式強制解引用
```rust
fn hello(name: &str) {
    println!("Hello, {}!", name);
}

fn main() {
    let m = MyBox::new(String::from("Rust"));
    
		// &MyBox<String> -> &String -> &str
    hello(&m); // hello((*m.deref()).deref());
}
```

> 也是這個原因所有 `str` 的方法都可用於 `String`。

### Drop Trait
###### Signature
```rust
pub trait Drop {
		fn drop(&mut self);
}
```

* `p.drop()` is not allowed, but `std::mem::drop(p)` can.

# Rc
---
##### Reference Counting
* keeps track of the **number of references** to a value.
* **multiple owners** by cloning.
* allocate data on the **heap**.
* **single-threaded** scenarios.
* can only **immutalbe reference** to owned data.

##### Increase the reference count
```rust
let a = Rc::new(String::from("Hello"));
let b = Rc::clone(&a);

let c = &*a; // *a has only read permission.
```



# Interior Mutability Pattern
---
##### Interior Mutability Pattern
Allows you to **mutate** data even when there are **immutable references** to that data.
> 沒有標記 mut 的變量，其持有的數值與其子數值都無法改變，而內部可變性在外部不可變的情況下讓內部可變。
##### RefCell
* Represents single ownership over the data it holds.
* Single-threaded scenarios
* **mutate** the value inside the `RefCell<T>` even when the `RefCell<T>` is **immutable**

```rust
// Wrong
{
		let x = 5;
		let y = &mut x; // Error! can't create immutable reference for immutable value.
}
// Right
{
		let x = RefCell::new(5);
		let mut y = x.borrow_mut(); // 內部可變性雖然可以建立可變引用，但是需要標註 mut 才能進行修改
		*y = 20;
		drop(y); // The RefCell has no NLL support. It's still not to obey borrow rule.
		println!("{}", *x.borrow());
}
```




# Reference Cycles
---
###### Reference Cycle
memory leaks because the reference count of each item in the cycle will never reach 0, and the values will never be dropped.

##### Prevent reference cycle by Weak
```rust
fn main() {
    let a = Rc::new(String::from("Hello"));

    let b = Rc::clone(&a);

    println!("Strong count: {}", Rc::strong_count(&b));

    let c = Rc::downgrade(&a); // c: Weak<_>

    println!("Strong count: {}", Rc::strong_count(&b));

    let d = match c.upgrade() { // c.upgrade return Option<Rc<T>>
        Some(val) => val,
        None => panic!(),
    };

    println!("Strong count: {}", Rc::strong_count(&b));
    println!("val: {}", *d);
}
```