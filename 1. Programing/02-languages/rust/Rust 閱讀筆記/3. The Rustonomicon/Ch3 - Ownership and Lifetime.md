# Reference
---
有兩種類型的引用:
* 共享引用: `&`
* 可變引用: `&mut`
他們遵循以下規則:
1. 一個引用的生命期不能超過他所引用對象的生命期
2. 一個可變引用不能有別名

#### 甚麼是別名 (Alias)?
> 別名定義: variable and pointers alias if they refer to overlapping regions of memory
> **變量與指針指向相同的內存空間。**

# Lifetime
---


