# 類型轉換相關
---
### Deref & DerefMut
`std::opt::Deref` 與 `std::opt::DerefMut` 表示該類型**可以使用解引用運算子**，使其可以與內置的引用一樣操作，定義如下
```rust
trait Deref {
	type Target: ?Sized; // 表示該類型(Self)內部擁有的或引用的值
	fn deref(&self) -> &Self::Target;
}

trait DerefMut {
	fn deref_mut(&mut self) -> &mut Self::Target;
}
```


### AsRef & AsMut

### Borrow & BorrowMut

### From & Into (TryFrom & TryInto)

# 類型性質相關

# 其他
