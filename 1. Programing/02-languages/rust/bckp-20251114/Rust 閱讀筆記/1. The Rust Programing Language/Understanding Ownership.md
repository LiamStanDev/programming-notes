### Ownership at Runtime
* Rust allocates local variables in stack frames, which are allocated when a function is called and deallocated when the call ends.
* Local variables can hold either data or pointer.
* Pointer can be created either throungh boxes (pointers owning data on the heap) or reference (non-owning pointors).
![[Pasted image 20240322105909.png | 600]]

### Ownership at Compile-time
Rust trucks three permissions on each varaible.
1. **Read**: data can be **copied** to another location.
2. **Write**: data can be mutated **in-place**
3. **Own**: data can be **moved** or **dropped**

* A varialbe not declared as `let mut`, it has R + O without W.
##### Variable's **permission can be changed** if it is **moved** or **borrowed**.
###### move
* A move of a variable with a non-copied type (e.g. `Box<T>` or `String`)，requires R + O permission。
* The move elimiates all permission on the variable.

###### borrow
* **Immutable borrow** create an immutale reference, and also **disables the borrowed data W + O**. (important !!!)
	![[Pasted image 20240322111154.png | 600]]
	* can't not `let s2 = *s_ref` ，because `*s_ref` only R permission and `String` has data on heap (not implement copy trait)。

* **Mutable borrow** creates a mutable reference, and also **disable the borrowed data R + W + O**.
	![[Pasted image 20240322111817.png|600]]

> **Creating immutable reference need Read permission** on varaible, so if exist mutable reference can't not create immutable reference from variable.

* immutable 與 mutalbe reference 本身都只有 R + O，除非在聲明前面添加 `mut`，差別在對 variable permission 的改變與解引用後的 permission。
##### Slice
Slices are a special kind of reference that refer to a contiguous sequence of data in memory。
* slice is fat pointer with metadata (length).


### Ownership in method
* 錯誤
```rust
enum ShirtColor {
    Red,
    Blue,
}

struct Inventory {
    shirts: Vec<ShirtColor>,
}

impl Inventory {
		// self 是不可變引用
    fn most_stocked(&self) -> ShirtColor {
        let mut num_red = 0;
        let mut num_blue = 0;
				// self.shirt 本質是 (*self).shirt，此時 *self 只具備 Read，而不具備 Own
				// 所以值無法進行 move 而只能進行 Read 的 copy，但 ShirtColor 沒有實現 Copied trait 故報錯。
        for shirt in self.shirts {
            match shirt {
                ShirtColor::Red => num_red += 1,
                ShirtColor::Blue => num_blue += 1,
            }
        }

        if num_blue > num_red {
            ShirtColor::Blue
        } else {
            ShirtColor::Red
        }
    }
}
```
* 更正
```rust
impl Inventory {
		// self 是不可變引用
    fn most_stocked(&self) -> ShirtColor {
        let mut num_red = 0;
        let mut num_blue = 0;
				// &self.shirts 本質是 &(*self).shirt
        for shirt in &self.shirts {
            match shirt {
                ShirtColor::Red => num_red += 1,
                ShirtColor::Blue => num_blue += 1,
            }
        }

        if num_blue > num_red {
            ShirtColor::Blue
        } else {
            ShirtColor::Red
        }
    }
}
```