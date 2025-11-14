# Pointer
---
### Const Pointer
分為兩種
1. 常量指針（最中要）: 表示指針指向的對象不能改變
2. 指針常量（被reference取代）： 表示指針的指向地址不能改變
3. 常指針常量（被const reference取代）： 表示指針指向的地址與對象均不能改變

```cpp
int x = 190;
const int* pi = &x; // 常量指針
// *pi = 22; // Error
```

### Null Pointer
用來表示指向無，會指向操作地址提空的區域，此區域是不能進行任何操作的
```cpp
int* p = 0;
int* p2 = NULL;
int* p3 = nullptr; // C++ 標準
```
#### 細節
1. 對空指針進行姐引用會導致程序崩潰
2. 尚未初始化的指針先指向 `nullptr` (Best practice)
3. delete 之後主動將該指針賦值給 `nullptr` (Best practice)
4. 函數不要返回局部變量地址 (Dangling pointer)

### Function Pointer
用來表示函數的地址，常用於將函數作為參數傳入別的函數。
```cpp
void fun(int x, double y) {};

void apply_fun(void (*f)(int, int))  { f(11, 23);}
```


# Reference
---
### 左值引用
左值引用幾乎同等於指針常量，該引用不能改變指向，並且不用解引用就能操作對象。
```cpp
int x = 10;
int& y = x;
int z = 22;
// y = z; // Error
```

### 右值引用
![[C++ Move Statement#右值引用]]

### 常量引用
常量引用表示該引用對象不能改變，同常指針常量。
使用場景：
1. 函數參數，該參數不能修改引用的對象
2. 函數參數，該參數可以傳入左值與右值 
