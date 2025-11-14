# Ownership
---
Every value has a single owner that determines its lifetime. When the owner is freed—dropped, in Rust terminology—the owned value is dropped too.
**每一個 value 都有且僅有一個所有者，它決定了該值的生命期**，只要所有者被 drop，其所有的值同樣會被 drop。

```rust
fn print_padonvan() {
		let mut padovan = vec![1, 1, 1]; // allocated here
		for i in 3..10 {
				let next = padonvan[i - 2];
				padonvan.push(next);
		}
			println!("P(1..10) = {:?})", padonvan);
} // drop here
```


##### Box
Rust’s Box type serves as another example of ownership. A `Box<T>` is a pointer to a value of type `T` stored on the heap. Since a Box owns the space it points to, when the Box is dropped, it frees the space too.
Rust 的 Box 代表另外一種所有權，`Box<T>` 是一個智能指針指向類型 `T`，並將值建立在 heap 中，因為 Box 擁有指向空間的所有權，所以 Box 被 `drop` 該值也會被 `drop`。

##### Struct 
```rust
#[derive(Debug)]
struct Person { name: String, birth: i32 }

let mut composers = Vec::new();
composers.push(Person { name: "Palestrina".to_string(), birth: 1525});
composers.push(Person { name: "Dowland".to_string(), birth: 1563});
composers.push(Person { name: "Lully".to_string(), birth: 1632});

for composer in &composers {
		println!("{}, born {}", composer.name, composer.birth);
}
```

There are many ownership relationships here, but each one is pretty straightforward: composers owns a vector; the vector owns its elements, each of which is a Person structure; each structure owns its fields; and the string field owns its text.
這邊有很多所有權關係，但是每一個都很直觀: `composers` 擁有一個 `vector`; `vector` 擁有它其中的每一個 `Person` 結構體; 每一個結構體擁有它的字段; `string` 擁有它的文字。

It follows that the owners and their owned values form trees: your owner is your parent, and the values you own are your children. And at the ultimate root of each tree is a variable; when that variable goes out of scope, the entire tree goes with it.
**所有者與其擁有的值透過樹型進行組織**，你的 owner 就是 parent，而值就是 children，而每一棵樹的根節點就是 varaible，當 variable 離開作用域，整棵樹都會 drop。

Every value in a Rust program is a member of some tree, rooted in some variable.
**Rust 中的每個 value 都是樹中的一個成員，它的其根是某個變數**。

That said, the concept of ownership as we’ve explained it so far is still much too rigid to be useful. Rust extends this simple idea in several ways:
所有權的觀念太過死板，以至於我們僅使用所有權會導致難以用，所以 Rust 作了以下的擴展:
* You  can  move  values  from  one  owner  to  another. This allows you to  build,  rearrange, and tear down the tree.
	你可以移動值到另外一個所有者，使得你可以建構、調整與分解 tree。
* Very simple types like integers, floating-point numbers, and characters are excused from the ownership rules. These are called Copy types.
	簡單的類型如整數、浮點數與字符可以避免所有權規則，因為他們實現了 Copy 特性。
* The standard library provides the reference-counted pointer types Rc and Arc, which allow values to have multiple owners, under some restrictions.
	標準庫提供引用計數指針 `Rc` 與 `Arc`，在某些限制下使值可以擁有多個所有者。
* You can “borrow a reference” to a value; references are non-owning pointers, with limited lifetimes.
	可以對值借用一個引用，因為引用是不會獲取所有權且有限生命期的指針。

Each of these strategies contributes flexibility to the ownership model, while still upholding Rust’s promises.
這些策略使我們可以更彈性的使用所有權模型，且同樣使 Rust 能保證其內存安全。

# Moves
---
In Rust, for most types, operations like assigning a value to a variable, passing it to a function, or returning it from a function don’t copy the value: they move it. The source relinquishes ownership of the value to the destination and becomes uninitialized; the destination now controls the value’s lifetime. 
在 Rust 中大部分類型進行賦值、傳參或返回並不會進行值拷貝，而是透過移動。源所有者放棄對於值的所有權，並轉移給目標，讓源所有者未初始化，然後值的生命期交給新的所有者控制。


Moving values around like this may sound inefficient, but there are two things to keep in mind. First, the moves _always apply to the value proper_, not the heap storage they own. For vectors and strings, the value proper is the threeword header alone; the potentially large element arrays and text buffers sit where they are in the heap. Second, the Rust compiler’s code generation is good at “seeing through” all these moves; in practice, the machine code often stores the value directly where it belongs.
移動乍看之下會顯得效率很差，但是有兩件事需要知道。首先，**移動永遠用於值本身，而不作用於它擁有的 heap 空間**，以 string 或者 vector 移動的是 3 words，而內部的大量的元素還是在原本位置。再者， Rust 編譯後的機器碼通常會直接將值存儲在它們應該存儲的地方，而不會產生多餘的操作。


#### Moves and Indexed Content
```rust
let mut v = Vec::new();

for i in 101..106 {
		v.push(i.to_string());
}

let htird = v[2];
let fifth = v[4];
```
For this to work, Rust would somehow need to remember that the third and fifth elements of the vector have become uninitialized, and track that information until the vector is dropped.
若要讓以上的代碼能夠進行，Rust 必須要去記得 `v[2]` 與 `v[4]` 已經被移動了，現在是未初始化不能使用，所以需要再 vector 裡面添加而外的空間去紀錄移出的信息，這樣並不是一個很好的設計且有而外的開銷。

Rust suggests using a reference, in case you want to access the element without moving it.
Rust 建議使用引用，因為引用可以取得該值但又不會移動它。

##### what if you really do want to move an element out of a vector?
1. Method 1: find a method that does.
```rust
let mut v = Vec::new(); 
for i in 101 .. 106 { 
		v.push(i.to_string()); 
}

let fifth = v.pop().expect("vector empty!"); // 彈出最後一個

let second = v.swap_remove(1); // 移出指定 index 的所有權，然後將最後一個放到指定的 index

let third = std::mem::replace(&mut v[2], "substitute".to_string()); // 取代
```
2. Method 2: iterator
```rust
let mut v = Vec::new();

for i in 101..106 {
		v.push(i.to_string());
}

// shared ref
for n in &v {} // n: &string

// mutable ref
for n in &mut v {} // n: &mut string

// onwership
for n in v {} // n: string
```
3. Method 3: use `Option` etc.
```rust
let mut v = Vec::new();

for i in 101..106 {
		v.push(Some(i.to_string());
}

// Method 1: std::mem::replace()
let first = std::mem::replace(&mut v[0], None);
// Method 2: take()
let first = v[0].take();
```


# Copy Types: The Exception to Moves
---
Moves keep ownership of such types clear and assignment cheap. But for simpler types like integers or characters, this sort of careful handling really isn’t necessary.
Move 使此類型的所有權清晰與低消耗的分配。但對於簡單類型如字符，這種處裡是沒有任何的必要。

Only types for which a simple bit-for-bit copy suffices can be Copy. As we’ve already explained, String is not a Copy type, because it owns a heap-allocated buffer. For similar reasons, Box is not Copy; it owns its heap-allocated referent.
只有簡單且可以逐位複製就足夠進行拷貝的類型才是 Copy 類型，我們已經解釋 String 不是 Copy 類型，因為它持有在 heap 分配的 buffer。同樣的原因 Box 也不能 Copy。

The File type, representing an operating system file handle, is not Copy; duplicating such a value would entail asking the operating system for another file handle. Similarly, the MutexGuard type, representing a locked mutex, isn’t Copy: this type isn’t meaningful to copy at all, as only one thread may hold a mutex at a time.
文件類型代表是 OS 文件句柄，它不是 Copy 類型，複製此類型隱含向操作系統申請新的文件句柄。同樣地，MutexGuard 類型表示一個鎖上的 mutex，也同樣不能被複製，因為要確保一個只有一個 Thread 能取得鎖 (MutexGuard)。

As a rule of thumb, any type that needs to do something special when a value is dropped cannot be Copy: a Vec needs to free its elements, a File needs to close its file handle, a MutexGuard needs to unlock its mutex, and so on. Bit-for-bit duplication of such types would leave it unclear which value was now responsible for the original’s resources.
作為一個簡單的準則，**任何類型需要在 drop 時做特別的事情是不能被 Copy**。Vec 需要清理的持有的 buffer、File 需要關閉文件句柄、MutexGuard 需要解鎖它的 mutex 等等。此類型進行逐位複製將會導致不清楚的哪個值負責原始資源。

#### Let User-define struct can be copied
```rust
#[derive(Copy, Clone)]
struct Label { number: u32 }
```
* Clone: 表示 Label 可以使用 `clone()` 方法
* Copy: 表示 Label 可以不受 move 限制，可以進行 bit-for-bit 複製。

###### Why aren’t user-defined types automatically Copy?
o making a type Copy represents a serious commitment on the part of the implementer: if it’s necessary to change it to non-Copy later, much of the code that uses it will probably need to be adapted.
默認使自訂義類型不實現 Copy，是因為要讓使用者想清楚確定該類型是否能被拷貝，因為原本不能被 Copy 的類型，後來改為能 Copy 程式碼基本不用改變，但是能被 Copy 的類型轉為不能被 Copy，程式碼需要大幅度調整。

# Rc and Arc: Shared Ownership
---

Rust’s memory and thread-safety guarantees depend on ensuring that no value is ever simultaneously shared and mutable. Rust assumes the referent of an Rc pointer might in general be shared, so it must not be mutable.
**Rust 的內存安全與線程安全基於確保沒任何一個值同時可 shared 與 mutable**。 Rc 智能指針是 shared 所以它是無法被修改。