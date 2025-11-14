# 00 - Misc
---
### Three-way comparison
為了要讓 C++ 自定義類型可以支持比較運算符號，故要為其重載 `==`, `!=`, `>`, `<`, `>=`, `<=` (共六種) 操作符太麻煩，故 C++ 引入三路比較。
* 只要重載 `==` 就不需要重載 `!=`
	1. 編譯器先尋找 `!=` 重載實現，找不到執行第二步
	2. 將 `a != b` 表達式改寫為 `!(a != b)` 
* 只要重載 `<=>` 就等同於實現 `<=`, `>=`, `<`, `>`
![[Pasted image 20250209103345.png]]
#### 返回值
在 `<compare>` 中定義以下幾個類型，其中他們都實作重載很多運算符號
- `std::strong_ordering`：如果比較結果可以確定，並且滿足完全排序，如 `int`。提供以下
	1. `std::string_ordering::less`
	2. `std::string_ordering::greater`
	3. `std::string_ordering::equal`
- `std::partial_ordering`：如果比較結果可能是「無法比較」的狀態，如`float`, `double` (因為 nan)。
	1. `std::partial_ordering::less`
	2. `std::partial_ordering::greater`
	3. `std::partial_ordering::equivalent`
	4. `std::partial_ordering::unordered` (float 中為 nan)
- `std::weak_ordering`：如果某些比較結果可能無法確定。(不知道甚麼時候用)
#### 自己實

```c++
```

# 03 - Container
---
### Iterator
C++ 有六種 Iterator 如下:
1. Input/Output Iterator: 用於 algorithm 來限制功能使用，非 container 使用
	* Input iterator: 只對 iterator 有 readonly 權限，e.g. `==`, `!=`, `->`
	* Output iterator: 只對 iterator 有 write-only 權限，e.g. `*it = val`, `it++`, `++it`, `it1 = it2`
2. Forward Iterator: 
	* e.g. `link_list`
3. Bidirectional Iterator:

> 他們並非類型而是種類，也就是說每個容器都有提供各自的 iterator 類型，而該類型屬於以上六種種類


### array

### vector
vector 為動態擴容數組，vector 中會記錄
1. first 指針
2. last 指針
3. end 指針



# 04 - Function
---
### Pointer to member function
* 舊版寫法
```c++
using FuncPtr = void (Person::*)(unsigned int);  // 需要加上類型限定符號

// 綁定
FuncPrt fptr = &Person::AddAge;
// 調用
(p.*fptr)(10); // 若 p 為 Person
(p->*fptr)(10); // 若 p 為 Person*
```
* 新寫法
```c++
#invclude <functional>

// 綁定
auto fptr = &Person::AddAge;

// 調用
std::invoke(fptr, p, 5); // 不管 p 為 pointer 還是 object 都可以
```

#### std::invoke
* 若為 pointer to member function + object: 會使用 `.*fptr()`
* 若為 pointer to member function + object pointer: 會使用 `->*fptr()`
* 其他: 會使用 `()`