# Iterator and IntoIterator
---
### Iterator
Iterator 為一個可以進行跌代的值。
```rust
trait Iterator {
	type Item;
	fn next(&mut self) -> Option<Self::Item>;
	// lots of default methods ...
}
```

### IntoIterator
若一個值可以以自然的方式進行跌代，該類型可以實作 `IntoIterator`
```rust
trait IntoIterator
where Self::IntoIter: Iterator<Item=Self::Item>
{
	type Item;
	type IntoIter: Iterator;
	fn into_iter(self) -> Self::IntoIter; // 注意這邊是 self
}
```
* where 約束的是 `into_iter` 返回值，必須為 Iterator，且產出值為 Item

#### 對 Vec 進行 for 迴圈本質
```rust
let v = vec!["antimony", "arsenic", "aluminum"];

// 使用 for 迴圈
for elem in &v {
	println!("{}", elem);
}

// 本質
let mut iter = (&v).into_iter();
while let Some(elem) = iter.next() {
	println!("{}", elem);
}
```

# 產生 Iterator
---
#### 透過集合的方法
大多數的集合都提供如下兩個產生 Iterator 的方法
1. `iter()`
2. `iter_mut()`

#### 透過 IntoIterator 的 `into_iter` 方法
* **集合的共享引用**(e.g. `&Vec<String>`) 使用 `into_iter`，返回 `Iterator<&Self::Item>`，**不會消耗集合所有權** 
	* 同理 `for elem in &collection {...}`
	* 與對集合使用 `iter()` 相同
* **集合的可變引用** (e.g. `&mut Vec<String>` ) 使用 `into_iter`，返回 `Iterator<&mut Self::Item>`，**不會消耗集合所有權**
	* 同理 `for elem in &mut collection {...}`
	* 與對集合使用 `iter_mut()` 相同
* **集合的值** (e.g. `Vec<String>`) 使用 `into_iter`，返回 `Iterator<Self::Item>`，**會消耗集合所有權**
	* 同理 `for elem in collection {...}`
> 注意並不是所有集合都有提供三種 IntoIterator 實作，如 BTreeSet 因為可變性會導致 Hash 值被修改。

#### 透過 from_fn 與 succcessors
##### from_fn
給 `std::iter::from_fn` 返回值為 `Option<T>` 的 closure，他會為他產出 Iterator，該 Iterator 會透過 closure 產出值，值到返回 None
* closure 為 `FnMut() -> Option<T>`
```rust
// 產生 1000 個 [0,1] 的隨機數
let lengths: Vec<f64> = from_fn(|| Some((random::<f64>() - random::<f64>()).abs()))
	.take(1000)
	.collect();
```

##### successors
`std::iter::successors` 與 `from_fn` 非常相同，只是他會用前值作為 closure 的參數繼續產生值。
* closure 為 `FnMut(&T) -> Option<T>`
```rust
// 產生平方序列 [2, 4, 6, ...]
let series: Vec<f64> = successors(Some(2.0), |&v| Some(v * v)).take(10).collect();
```

#### 透過 drain 方法
很多集合有提供 drain 方法，它會將集合中**抽取出一個範圍**，並返回 `Drain<T>` (為 Iterator)，當 `Drain<T>` 被 drop 掉，原集合就可以被使用 (因為 `drain` 接收 `&mut self`)。
```rust
let mut outer = "Earth".to_string();
let inner = outer.drain(1..4); // 產生 Iterator，此時 outer 正在被可變借用
let inner = String::from_iter(inner); // from_iter 會接收所有權

assert_eq!(outer, "Eh");
assert_eq!(inter, "art");
```


# Pipeline
---
### Iterator Adapters
* map
* filter
* filer_map
* flat_map
* flatten
* take
* tak_while
* skip
* skip_while
* peekable
* fuse
* rev
* inspect
* chain
* eumerate
* zip
* by_ref
* cloned
* copied
* cycle


### Iterator Comsumers
* count
* sum
* product
* max
* min
* max_by
* min_by
* max_by_key
* min_by_key
* any
* all
* position
* rposition
* fold
* rfold
* try_fold
* try_rfold
* nth
* nth_back
* last
* find
* rfind
* find_map
* coolect
* partition
* for_each
* try_for_each

# Implement Interator
---
* for 迴圈時會先使用 IntoIterator 的 into_iter 方法，但類型本身已經是 Iterator 該怎麼做? 
	* Rust 會為每一個 Iterator 類型自動實作以下的空白實作 (Blank Implement)
```rust
impl<I: Iterator> IntoIterator for I {
	type Item = I:Item;
	type IntoIter = I;

	fn into_iter(self) -> Self::IntoIter {
		self
	}
}	
```