### Ownership in match: binding model
source: https://doc.rust-lang.org/reference/patterns.html#binding-modes

Def: binding model make it easier to bind references to values, when value is matched by a non-reference pattern.
我們可以使用 `let a = &s` 來取得 s 的引用，但是在 match 時我們很難做到取得 enum 內部的引用或者可變所有權，而 binding model 就可以方便的讓我們取得內部的引用。

##### Problem
```rust
fn main() {
    let mut opt = Some(String::from("Hello world"));

    match opt {
        Some(s) => println!("Some: {}", s), // moved here，這邊取得的是不可變的所有權
        None => println!("None!"),
    };

    println!("{:?}", opt); // Error ! because opt is moved.
}
```

##### Solution 1
* 顯示使用 `ref`, `ref mut` 來取得內部引用，或者使用 `mut` 來取得內部的可變所有權
```rust
fn main() {
    let mut opt = Some(String::from("Hello world"));

    match opt {
        Some(ref s) => println!("Some: {}", s), // s: &String
        None => println!("None!"),
    };

    println!("{:?}", opt);
}
```

##### Solution 2
隱式的使外部的 reference 推進內部 (push down)。
```rust
```rust
fn main() {
    let mut opt = Some(String::from("Hello world"));

    match &opt { // &Option<String>
        Some(s) => println!("Some: {}", s), // s: &String
        None => println!("None!"),
    };

    println!("{:?}", opt);
}
```