# Language extension traits
---
#### 介紹
表示作為 Rust 語言擴充功能的 Trait，如運算符重載。

### Drop
### Deref & DerfMut
### From & Into

# Marker traits
---
#### 介紹
最多是用來作為泛型參數的限制，表示無法透過其他形式表達的限制，如無法透過運算來描述這種限制。

### Sized
### Copy
# Public vocabulary trais
---
#### 介紹
用來定義標準庫中不同 crates 對接的規範，本身沒有任何編譯器的整合，只是作為實現標準，讓 crate 間容易交互。

### Default
### AsRef & AsMut
### Borrow & BorrowMut
### TryFrom & TryInto

### ToOwned
### Cow
### Clone