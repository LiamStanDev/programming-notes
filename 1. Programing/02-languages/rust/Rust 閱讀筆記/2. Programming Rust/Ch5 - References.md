All the pointer types we’ve seen so far—the simple Box heap pointer, and the pointers internal to String and Vec values—are owning pointers: when the owner is dropped, the referent goes with it. Rust also has non-owning pointer types called references, which have no effect on their referents’ lifetimes.
到目前為止所有見過的指針 Box、String 與 Vec 等，都是持有值的所有權，所以 owner 被 drop，它們指向的也會被 drop 掉，Rust 也有非持有所有權的指針，稱為 references (引用)，**引用並不會影響它指向的值得生命期**。

In fact, it’s rather the opposite: references must never outlive their referents. To emphasize this, Rust refers to creating a reference to some value as borrowing the value: what you have borrowed, you must eventually return to its owner.
事實上，正是恰恰相反，**引用永遠不能存活大於其指向的值**。為了強調這個概念，Rust 使用了借用的概念，任何借用最終都會返還給所有者。

# References to Values
---
A reference lets you access a value without affecting its ownership. References come in two kinds:
引用使我們可以不取得所有權下訪問值，引用有兩種形式:

1. A **shared reference** lets you read but not modify its referent. However, you can have as many shared references to a particular value at a time as you like. Shared references are Copy.
分享引用讓你可以讀取數據但不能修改它的值，一個值可以同時擁有多個 reference。**分享引用是 Copy 類型**。

2. A **mutable reference** to a value, you may both read and modify the value. However, you may not have any other references of any sort to that value active at the same time. Mutable references are not Copy.
可變引用使你可以讀取與修改值，但只要可變引用存在，同一時間是不能存在其他的引用。**可變引用不是 Copy 類型**。

You can think of the distinction between shared and mutable references as a way to enforce a multiple readers or single writer rule at compile time.
你可以想到 shared 與 mutable 引用在於落實編譯時期的**多個讀或者一個寫原則**。

In fact, this rule doesn’t apply only to references; it covers the borrowed value’s owner as well. As long as there are shared/mutable references to a value, not even its owner can modify it; the value is locked down. 
事實上這個原則不僅僅應用在引用上。它也涵蓋被借用的值的所有者，只要這些引用存在就算連 owner 也不能修改值。

```rust
fn show(table: &Table) {
    for (artist, works) in table {  // arties: &String, works: &Vec<String>
        println!("works by {}", artist);

        for work in works { // work: &String
            println!(" {}", work);
        }
    }
}

fn sort_work(table: &mut Table) { 
    for (_artist, works) in table { // _arties: &String, &mut Vec<String>
        works.sort();
    }
}
```

# Working with References
---
### Assigning References
把一個引用賦值給變量會讓這個變量指向新的地址。
```rust
let y = 20;
let mut r = &x; // 這邊 r 聲明可變，表示我們可以改變引用的指向
r = &y; // 將 r 指向 y
```

### References to References
Rust 允許引用的引用。
```rust
struct Point { x: i32, y: i32 } 

let point = Point { x: 1000, y: 729 }; 
let r: &Point = &point; 
let rr: &&Point &r; 
let rrr: &&&Point = &rr;
```

### Comparing References
直接比較引用會觸發解引用轉換，並不是真的比較地址，如下
```rust
let x = 10;
let y = 10;

let rx = &x;
let ry = &y;

let rrx = &rx;
let rry = &ry;

assert!(rrx <= rry); // true
assert!(rrx == rry); // true
```
想要比較兩個指針的地址應該使用 `std::prt::eq`
```rust
assert!(!std::ptr::eq(rrx, rry));
```
比較運算符兩側需要是相同類型
```rust
assert!(rrx <= ry); // 類型不同，編譯不通過
```

#### Slice and trait object
![[Ch2 - Data Layout#Dynamically Sized Types (DSTs)]]


### Rules for Mutation and Sharing

##### Shared access is read-only access
Values borrowed by shared references are read-only. Across the lifetime of a shared reference, neither its referent, nor anything reachable from that referent, can be changed by anything. There exist no live mutable references to anything in that structure, its owner is held read-only, and so on. It’s really frozen.
共享引用借用的值是只讀的。在共享引用的生命期中，不管是它引用的值，還是任何通過這個值可以訪問的東西，一概都不能被修改。如果結構體的所有者是只讀的，不允許指向結構體中任何內容的可變引用(無法建立可變借用於不可變變量)。

##### Mutable access is exclusive access
A value borrowed by a mutable reference is reachable exclusively via that reference. Across the lifetime of a mutable reference, there is no other usable path to its referent or to any value reachable from there. The only references whose lifetimes may overlap with a mutable reference are those you borrow from the mutable reference itself.
可變借用的值只能通過這個引用進行訪問，在可變引用生命期中，沒有任何其他方法訪問被引用的值或只通過這個值可以訪問到的值。

![[Pasted image 20240415211834.png]]
* 對於共享引用，路徑上只可讀。
* 對於可變引用，路徑是無法被訪問的 (除非通過可變引用自身)。

###### reborrowing (再借用)
* 可以從一個共享引用借用一個共享引用。
```rust
let mut w = (107, 109);
let r = &w;
let r0 = &r.0; // Ok: 通過 r 進行再借用
let m1 = &r.1; // Error: 不能從共享引用中借用不可變引用，因為整個路徑都為 read-only
println!("{}", r0); // r0 的生命期到這
```

* 可以從可變引用借用可變引用
```rust
let mut v = (136, 139); 
let m = &mut v; 
let m0 = &mut m.0; // Ok：從可變引用借用可變引用
*m0 = 137; // 透過 m0 修改值
let r1 = &m.1; // Ok：從可變引用借用共享引用
println!("{}", r1); // r1在这里使用
```
m0 生命期中，所有 m0 可以訪問的路徑上 m 喪失所功能，所以 m 不能修改 m0 所限制的路徑，但是 r1 之所以可以是因為它並不在 m0 的路徑上，又因為 r1 的路徑上都變為只讀，所以也不能透過 m.1 來修改內容。
```rust
let mut v = (136, 139);
let m = &mut v;
let m0 = &mut m.0; 
*m0 = 137; 
m.1 = 10; // 因為不在 m0 的路徑上，所以可以修改
m.0 = 20; // Error
println!("{}", m0);
```