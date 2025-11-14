# Ch 1 - C++ 語法
---
### 甚麼可以放入 if 條件中?
##### (1) 布林值
```cpp
if (true) { ... }
if (a > b) { ... }
```

##### (2) 指針
指針可以隱式轉換成 bool
* 非空指針為 `true`
* 空指針為 `false`
```cpp
int* ptr = nullptr; 
if (ptr) { ... } // 条件为 false，因为 ptr 是 nullptr
```

##### (3) 賦值表達式
賦值操作會返回被賦的值。
```cpp
int *p = get_pointer();
if (p = another_pointer) { ... }
```
> 這邊要注意若你的賦值為引用綁定，返回是一個別名而不是指針或者數值，不能隱式轉換成布林值
```cpp
if (Smiley* p = dynamic_cast<Smiley*>(ps)) // 合法
if (Smiley& r = dynamic_cast<Smiley&>(*ps)) // 不合法
```


##### (1) 布林值
# Ch 4 - Classes
---
> [!PDF|yellow] [[A Tour of C++ 2nd.pdf#page=65&selection=33,0,43,0&color=yellow|A Tour of C++ 2nd, p.52]]
> Vector obeys the same rules for naming, scope, allocation, lifetime, etc. (§1.5), as does a built-in type, such as int and char

我們應該在設計 Class 時候要讓使用者像在使用 built-in type 一樣操作。透過 RAII 技術可以讓 object 可以安全的初始化與釋放。


> [!PDF|red] [[A Tour of C++ 2nd.pdf#page=65&selection=89,0,131,65&color=red|A Tour of C++ 2nd, p.52]]
> The constructor allocates the elements and initializes the Vector members appropriately. The destructor deallocates the elements. This handle-to-data model is very commonly used to manage data that can vary in size during the lifetime of an object. The technique of acquiring resources in a constructor and releasing them in a destructor, known as Resource Acquisition Is Initialization or RAII, allows us to eliminate ‘‘naked new operations,’’ that is, to avoid allocations in general code and keep them buried inside the implementation of well-behaved abstractions. Similarly, ‘‘naked delete operations’’ should be avoided. Avoiding naked new and naked delete makes code far less error-prone and far easier to keep free of resource leaks (§13.2)

我們應該避免使用裸 `new`/`delete` 對資源進行操作，而是做一個良好的封裝。

> [!PDF|yellow] [[A Tour of C++ 2nd.pdf#page=66&selection=152,0,176,63&color=yellow|A Tour of C++ 2nd, p.53]]
> A static_cast does not check the value it is converting; the programmer is trusted to use it correctly. This is not always a good assumption, so if in doubt, check the value. Explicit type conversions (often called casts to remind you that they are used to prop up something broken) are best avoided. Try to use unchecked casts only for the lowest level of a system. They are error-prone. Other casts are reinterpret_cast for treating an object as simply a sequence of bytes and const_cast for ‘‘casting away const.’’ Judicious use of the type system and well-designed libraries allow us to eliminate unchecked casts in higher-level software.

C++ 提供的類型轉換有三種:
* `static_cast`: 顯示類型轉換，但它並不會檢查轉換的數值是否合理
* `reinterpret_cast`: 將對象視為一串 byte sequence 然後進行轉換，這種轉換不會進行任何類型檢查
* `const_cast`: 用於從原本被標記 `const` 的對象上移除 `const` 限定符，會破壞原本保證的不變性。
以上三種轉換盡量在必要的時候才使用，他們用於轉換是否成功要取決於 value 本身，故轉換本身要靠程序員來保證，書中建議先檢查數值在進行轉換。

> [!PDF|red] [[A Tour of C++ 2nd.pdf#page=72&selection=117,0,121,80&color=red|A Tour of C++ 2nd, p.59]]
> A virtual destructor is essential for an abstract class because an object of a derived class is usually manipulated through the interface provided by its abstract base class. In particular, it may be deleted through a pointer to a base class. Then, the virtual function call mechanism ensures that the proper destructor is called. That destructor then implicitly invokes the destructors of its bases and members

virtual descructor 必須存在，以保證在基類指針釋放的時候能找到正確的子類析構函數。

> [!PDF|yellow] [[A Tour of C++ 2nd.pdf#page=74&selection=89,0,90,53&color=yellow|A Tour of C++ 2nd, p.61]]
> Objects are constructed ‘‘bottom up’’ (base first) by constructors and destroyed ‘‘top down’’ (derived first) by destructors.

> [!PDF|yellow] [[A Tour of C++ 2nd.pdf#page=74&selection=172,0,183,1&color=yellow|A Tour of C++ 2nd, p.61]]
> > We use dynamic_cast to a pointer type when a pointer to an object of a different derived class is a valid argument. We then test whether the result is nullptr. 

`dynamic_cast` 提供一種安全的方式來將基類指針轉為子類指針，若無法轉換會返回 `nullptr`。
`dynamic_cast` 也能將基類 object 轉為子類引用，若無法轉換會拋出 `std::bad_cast` 異常。


# Ch 5 - Essential Operations
---

```cpp
class {
 public:
	X(Sometype); // ordinary constructor
	X(); // default constructor
	X(const X&); // copy constructor
	X(X&&); // move constructor
	X &operator=(const X&); // copy assignment
	X &operator=(X&&); // move assignment
	~X(); // destructor
}
```

> [!PDF|yellow] [[A Tour of C++ 2nd.pdf#page=79&selection=59,0,74,15&color=yellow|A Tour of C++ 2nd, p.66]]
> There are five situations in which an object can be copied or moved: • As the source of an assignment • As an object initializer • As a function argument • As a function return value • As an exception

> [!PDF|red] [[A Tour of C++ 2nd.pdf#page=79&selection=124,0,135,1&color=red|A Tour of C++ 2nd, p.66]]
> When a class has a pointer member, it is usually a good idea to be explicit about copy and move operations. The reason is that a pointer may point to something that the class needs to delete, in which case the default memberwise copy would be wrong. Alternatively, it might point to something that the class must not delete.

*有 pointer 的 class 通常需要顯示的去寫 copy 與 move 操作*，因為 pointer 通常指向一個需要被 `delete` 資源

黃金準則為: **"要馬全部都顯示定義，不然全部默認"**

> [!PDF|red] [[A Tour of C++ 2nd.pdf#page=84&selection=159,39,165,51&color=red|A Tour of C++ 2nd, p.71]]
> So an rvalue is – to a first approximation – a value that you can’t assign to, such as an integer returned by a function call. Thus, an rvalue reference is a reference to something that nobody else can assign to, so we can safely ‘‘steal’’ its value

rvalue 粗淺的來說是不能被 assign 的值。

> [!PDF|yellow] [[A Tour of C++ 2nd.pdf#page=86&selection=124,0,128,15&color=yellow|A Tour of C++ 2nd, p.73]]
> Also, memory is not the only resource. A resource is anything that has to be acquired and (explicitly or implicitly) released after use. Examples are memory, locks, sockets, file handles, and thread handles.

資源可以分為:
* memory
* locks
* sockets
* file handles
* thread handles
等等。

> [!PDF|yellow] [[A Tour of C++ 2nd.pdf#page=86&selection=136,66,138,5&color=yellow|A Tour of C++ 2nd, p.73]]
> Leaks must be avoided in any long-running system, but excessive resource retention can be almost as bad as a leak.

這段說明對於長期運行的系統我們要避免內存洩漏，另外我們要盡可能短的佔用資源。


> [!PDF|red] [[A Tour of C++ 2nd.pdf#page=87&selection=13,27,14,82&color=red|A Tour of C++ 2nd, p.74]]
> Resources can be moved from scope to scope using move semantics or ‘‘smart pointers,’’ and shared ownership can be represented by ‘‘shared pointers’’

