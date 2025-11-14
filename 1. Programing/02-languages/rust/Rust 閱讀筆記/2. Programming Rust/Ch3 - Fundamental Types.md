### Pointer Types
#### References
It’s easiest to get started by thinking of references as Rust’s basic pointer type. The expression &x produces a reference to x; in Rust terminology, we say that it _borrows a reference to x_.
最簡單的思考 references 的方式就是當它是 Rust 中最基本的指針類型，表達式 `&x` 代表產出一個 `x` 的引用，在 Rust 的術語中稱為借用一個 `x` 的引用。

Unlike C pointers, however, Rust references are never null: there is simply no way to produce a null reference in safe Rust. And unlike C, Rust tracks the ownership and lifetimes of values, so mistakes like dangling pointers, double frees, and pointer invalidation are ruled out at compile time.
與 C 語言的指針不同，Rust 的引用不可能為 `null`，在安全 Rust 下是根本無法產生空指針。另外與 C 語言指針不同，Rust 的引用會追蹤值所有權與生命期，所以懸垂指針、二次釋放與無效指針在 compile 時期就已經被排除。

###### References come in two flavors
1. `&T` (shared reference)
An _immutable, shared reference._ You can have many shared references to a given value at a time, but they are read-only: modifying the value they point to is forbidden, as with const T* in C.
`&T` 為不可變共享引用。我們可以對於給定的值同時建立多個共享引用，但它們只能讀 (read-only)，修改它們指向的值是禁止的，同 C 語言中的 `const T*`。
* 註解: 將一個變數取 `&T` 後會將原變數的寫與所有權的 permission 取消，所以原變數也只能讀，直到所有引用都被 drop

2. `&mut T` (mutable reference)
A _mutable, exclusive reference_. You can read and modify the value it points to, as with a `T*` in C. But for as long as the reference exists, you may not have any other references of any kind to that value. In fact, the only way you may access the value at all is through the mutable reference.
`mut &T` 為可變的獨有引用。我們可以修改與讀取該引用指向的值，同 C 語言中的 `T*`。但是只要可變獨有引用存在時，不能存在其他的引用指向該值，獲取取該值的方式只能透過該可變獨有引用。
* 註解: 將一個變數取 `&mut T` 後會將原變數的讀、寫與所有權的 permission 取消，所以原變數在可變獨有引用存在時，會喪失任何功能。

Rust uses this dichotomy between shared and mutable references to enforce a _“single writer or multiple readers”_ rule.
Rust 採用在共享與可變引用的二分法去保證 "一個寫與多個讀"。

#### Boxes
The simplest way to allocate a value in the heap.
為最簡單的方式將值放入 heap。

#### Raw Pointers
Rust also has the raw pointer types `*mut T` and `*const T`. Raw pointers really are just like pointers in C++. Rust makes no effort to track what it points to. However, you may only dereference raw pointers within an unsafe block.
Rust 同樣也有提供原始指針類型 `*mut T` 與 `*const T`，但 Rust 並不會去追蹤原始指針指向的值，而且你只能在 unsafe block 中對原始指針進行解引用。


### Arrays, Vectors, and Slice
skip Arrays and Vectors
#### Slices
A slice, written `[T]` without specifying the length, is a region of an array or vector. Since a slice can be any length, slices can’t be stored directly in variables or passed as function arguments. Slices are always passed by reference.
`[T]` 為切片，切片不指定長度，是 `array` 或者 `vector` 的部分區間，因為大小不定所以無法被直接存放在變數或者傳入函數參數中，只能透過引用傳遞。

##### reference to a slice
A reference to a slice is a *fat pointer*: a two-word value comprising a pointer to the slice’s first element, and the number of elements in the slice.
切片的引用是一個胖指針，裡面包含了兩個 word 大小的值，分別為一個指向切片第一個元素的指針與 slice 中的元素個數。

```rust
let v: Vec = vec![0.0, 0.707, 1.0, 0.707]; 
let a: [f64; 4] = [0.0, -0.707, -1.0, -0.707]; 

let sv: &[f64] = &v; // 顯示表示切片引用類型
let sa: &[f64] = &a; // 顯示表示切片引用類型
```

Rust automatically converts the `&Vec` reference and the `&[f64; 4]` reference to slice references that point directly to the data. This makes slice references a good choice when you want to write a function that operates on either an array or a vector.
在指名為切片類型後，Rust 會自動的將 vector 的引用與 array 的引用轉換成切片。_若函數想要能處理 vector 與 array 的話，這會使得切片的引用是一個好的參數類型選擇_。e.g. `sort()`, `reverse()` 都是 implement 在 `[T]` 上的，但可以給 vector 與 array 使用。