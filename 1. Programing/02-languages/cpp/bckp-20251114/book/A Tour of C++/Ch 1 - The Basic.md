# 1.2 Programs
![[Pasted image 20250116211728.png]]
ISO 標準的 C++ 有以下組成:
1. Core language features
2. Standard-library components


### Type, Variables, and Arithmetic
A *declaration* is a statement that introduces an entity into the program. It specifies a type for the entity:
• A *type* defines a set of possible values and a set of operations (for an object).
• An *object* is some memory that holds a value of some type.
• A *value* is a set of bits interpreted according to a type.
• A *variable* is a named object

#### Initialization
* 使用 `=` 可能會有 *narrowing conversion* 的問題 (主要是為了間容C)
* 使用 `{}` 或者 `= {}` 可以避免這個問題 (最推薦)
```cpp
int i1 = 7.8; // narrowing, but no compiler error
// 推薦寫法
int i2 {7.8}; // error! narrowing
int i3 = {7.8}; // error! narrowing (same with above one)
```

#### const and constexpr
##### const
大致上意味著「**我保證不改變這個值**」。它主要用於將指針和引用將數據傳遞給函數，不必擔心它被修改。
* 他只是一個承諾，沒有其他的物理上的含意
* const 的值可以在運行時計算
###### 用於函數返回值
表示返回值**指向**的內容是無法修改的。
> 這邊要注意 C++ 都是採值傳遞，若非指針則完全沒有意義。
##### constexpr
constexpr 大致上意味著「**在編譯時被評估**」。它主要用於指定常量，允許將數據**放置在只讀內存中**。
* 編譯器會在編譯期間進行運算，可以提高性能。
* 放在只讀空間中，有更高的安全性。
###### 用於函數返回值
1. 若參數都是 const expression，則該函數為編譯期就會計算的函數
2. 若參數是 runtime variable，也可以運行只是就沒有進行編譯其運算的功能
     -> 不用為了 constexpr 寫兩個一樣的函數

### Pointers, Arrays, and References
* **Arrays**: collection of data is a **contiguously allocated** sequence of elements of the same type

| Type      | Description                                                                                                               |
| --------- | ------------------------------------------------------------------------------------------------------------------------- |
| Arrays    | A collection of data which is a **contiguously allocated** sequence of elements of the same type.                         |
| Pointers  | A variable that stores the memory address of another variable (object).                                                   |
| Reference | A convenient alternative to a pointer that provides indirect access to a variable without needing explicit dereferencing. |

##### Null Pointer
Represent the notion of "*no object avalible*".

