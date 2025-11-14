# Intro
###### Functional Programming
using functions as values by passing them in arguments, returning them from other functions, assigning them to variables for later execution.

# Closure
---
* closures are **anonymous functions** you can **save in a variable** or **pass as arguments** to other functions.
* closures **can capture values** from the scope in which they’re defined.
##### Capturing References or Moving Ownership
There're tree ways to capture values from their environment: 
1. borrowing immutabley: 會從 closure 內部使用的方式自行判斷，是否是 immutable reference
2. borrowing mutably: 若 closure 內部有進行修改，則會使用 mutable reference
3. taking ownership: 需要顯示的使用 `move` 關鍵字，會將使用 moving ownership 的方式捕獲變量
	* 這種方式常用於建立一個新的 thread，因為 thread 的生命週期不確定。

##### Fn Trait
* The way a closure captures and handles values from the environment affects which traits the closure implements.
* Closures will automatically implement one, two, or all three of these `Fn` traits.
###### 三種 Fn Trait
1. `FnOnce`: Applies to closures that **can be called once**. 特點: 捕獲所有權，故只能調用一次。
2. `FnMut`: Applies to closures that **don’t move** captured values, but might mutate the captured values. 特點: 可重複調用 + 可變。
3. `Fn`: Applies to closures that **don’t move** captured values and that **don’t mutate** captured values. 特點: 可重複調用 + 不可變。

* 嚴格程度: `FnOnce` > `FnMut` > `Fn`。

* 使用 `move` 關鍵字不代表他一定實現 `FnOnce`，重點是內部是如何使用捕獲變量。

> 註: `fn(i32) -> i32` 表示函數指針為類型，而 `Fn`、`FnMut` 與 `FnOnce` 為 Trait 不是類型。


##### Return closure
返回的 closure 使用引用作為輸入參數，因為 closure 有延遲調用的功能，要確保引用還有效，故要標記生命週期。
```rust
fn make_a_cloner<'a>(s_ref: &'a str) -> impl Fn() -> String + 'a {
		move || s_ref.to_string() // 返回 closure，這邊將 s_ref 這個引用 move 進去，而不是 &s_ref
}
```
* `impl Fn() -> String + 'a` 表示這個 Fn Trait 只能生命週期小於 `'a ` 。

若只有一個引用輸入參數，也可以使用 lifetime elision (生命週期消除規則) 來簡化
```rust
fn make_a_cloner(s_ref: &str) -> impl Fn() -> String + '_ {
		move || s_ref.to_string() // 返回 closure，這邊將 s_ref 這個引用 move 進去，而不是 &s_ref
}
```
* `+ '_ ` 還是需要標記，因為能讓調用者知道這個 closure 是具備生命週期，而不是想調用就調用。

# Iterator
---
##### Iterator Trait
```rust
pub trait Iterator {
		type Item;

		fn next(&mut self) -> Option<Self::Item>;
}
```
* All iterators implement `Iterator` trait which is defined in the standard library.
* `Item` type is used in the return type of the `next` method
* The `next` method, which returns one item of the iterator at a time wrapped in `Some` and, when iteration is over, returns `None`.


##### From collection to iterator
Most of collections are using the same naming style for creating iterator:
* `iter()` : immutable reference of collection data.
* `iter_mut()` : mutable reference of collection data.
* `into_iter()` : ownership of collection data.

##### Iterator adaptor
Iterator adaptors are methods defined on the `Iterator` trait that don't consume the iterator. They produce different iterators by changing some aspect of the original iterator.

```rust
let v1: Vec<i32> = vec![1,2,3];

let v2:Vec<_> = v1.iter().map(|x| x + 1).collect(); // map is adaptor, collect is consummer
```


