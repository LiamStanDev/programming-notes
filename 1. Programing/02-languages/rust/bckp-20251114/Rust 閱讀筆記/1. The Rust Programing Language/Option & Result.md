source: https://doc.rust-lang.org/std/option/index.html

# Option
---
### Querying the variant
* `is_some()`
* `is_none()`

### Adapters for working with reference
* `as_ref()`: `&Option<T>` -> `Option<&T>`
* `as_mut()`: `&mut Option<T>` -> `Option<&mut T>`
* `as_deref()`: 與 `as_ref` 一樣會獲得內部的引用，但會多 deref coercing 性質。
* `as_deref_mut()`: 與 `as_mut` 一樣會獲得內部的引用，但會多 deref coercing 性質。
* `as_pin_ref`: `Pin<&Option<T>>` -> `Option<Pin<&T>>`
* `as_pin_mut`: `Pin<&mut Option<T>>` -> `Option<Pin<&mut T>>`
* `copied()`: `Option<&T>` -> `Option<T>` 會建立一個新的 `Option` 

### Extracting the contained value
* `expect()`: 若為 `None` 則報錯，並顯示 `expect` 指定的內容
* `unwrap()`: 若為 `None` 則報錯
* `unwrap_or()`: 若為 `None` 返回 `unwrap_or()` 指定的內容
* `unwrap_or_defult()`: 用於有實現 `Default` trait 的類型
* `unwrap_or_else()`: 傳入 closure，若為 Some 直接返回，若為 None 會進入 close 中進一步處理。
> 註: `_or` 結尾都是返回"值"，而 `_else` 結尾會給一個 closure

### Transform
##### Option to Result
* `ok_or()`
* `ok_or_else()`
* `transpose()`: `Option<Result<_>>` -> `Result<Option<_>>`
* `filter()`: 若傳入的 closure 返回 `true` 則返回 `Some(v)` 否則 `None`
* `map()`: 若為 None 則不變，若為 `Some(v)` 會透過 closure 轉為 `Some(u)` (類型可以不一樣)。


### Trait
* Clone
```rust
impl<T> Clone for Option<T> where T: Clone
```
只要內部類型有 Clone，Option 就有 Clone

* Copy
```rust
impl<T> Copy for Option<T> where T: Copy
```
只要內部類型有 Copy，Option 就有 Copy


# Result
---
### Querying the variant
* `is_ok()`
* `is_err()`

### Adapters for working with reference
* `as_ref()`: `&Result<T,E>` -> `Result<&T,&E>`
* `as_mut()`: `&mut Result<T>` -> `Option<&mut T, &mut E>`
* `as_deref()`: 與 `as_ref` 一樣會獲得內部的引用，但會多 deref coercing 性質。
* `as_deref_mut()`: 與 `as_mut` 一樣會獲得內部的引用，但會多 deref coercing 性質。

### Extracting the contained value
* `expect()`: 若為 `Err` 則報錯，並顯示 `expect` 指定的內容
* `unwrap()`: 若為 `Err` 則報錯
* `unwrap_or()`: 若為 `Err` 返回 `unwrap_or()` 指定的內容
* `unwrap_or_defult()`: 用於有實現 `Default` trait 的類型
* `unwrap_or_else()`: 傳入 closure，若為 Some 直接返回，若為 None 會進入 close 中進一步處理。
> 註: `_or` 結尾都是返回"值"，而 `_else` 結尾會給一個 closure
### Transform
##### Option to Result
* `ok_or()`
* `ok_or_else()`
* `transpose()`: `Option<Result<_>>` -> `Result<Option<_>>`
* `filter()`: 若傳入的 closure 返回 `true` 則返回 `Some(v)` 否則 `None`
* `map()`: 若為 None 則不變，若為 `Some(v)` 會透過 closure 轉為 `Some(u)` (類型可以不一樣)。