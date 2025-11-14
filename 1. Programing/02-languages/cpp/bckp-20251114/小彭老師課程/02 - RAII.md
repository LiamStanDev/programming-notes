# 語言的演進
---
### C 語言
```c
#include <cstdio>
#include <cstdlib>

int main() {
  size_t nv = 4;
  int *v = (int *)malloc(nv * sizeof(int));
  v[0] = 4;
  v[1] = 3;
  v[2] = 2;
  v[3] = 1;

  int sum = 0;
  for (size_t i = 0; i < nv; i++) {
    sum += v[i];
  }

  printf("%d\n", sum);
  free(v);

  return 0;
}
```

### STL 容器庫
```cpp
#include <iostream>
#include <vector>

int main() {
  std::vector<int> v(4);
  v[0] = 4;
  v[1] = 3;
  v[2] = 2;
  v[3] = 1;
  int sum = 0;
  for (size_t i = 0; i < v.size(); i++) {
    sum += v[i];
  }

  std::cout << sum << std::endl;
  return 0;
}
```

### C++ 11 列表初始化

```cpp
int main() {
  std::vector<int> v = {4, 3, 2, 1};

  int sum = 0;
  // 以下也是 C++11 提出的 range-based for-loop
  for (int vi : v) {
    sum += vi;
  }

  std::cout << sum << std::endl;
  return 0;
}
```

#### 模板算法 + lambda 表達式
```cpp
int main() {
  std::vector<int> v = {4, 3, 2, 1};
  int sum = 0;

  std::for_each(v.begin(), v.end(), [&](int vi) { sum += vi; });

  std::cout << sum << std::endl;
  return 0;
}
```

##### c++ 14 後支持 lambda 參數使用 auto
```cpp
int main() {
  std::vector<int> v = {4, 3, 2, 1};
  int sum = 0;

  std::for_each(v.begin(), v.end(), [&](auto vi) { sum += vi; });

  std::cout << sum << std::endl;
  return 0;
}
```

##### C++ 17 支持 compile-time argument deduction (編譯期參數推斷)
```cpp
int main() {
  std::vector v = {4, 3, 2, 1}; // 這邊泛形不用添加，可以透過元素推斷
  int sum = 0; 

  std::for_each(v.begin(), v.end(), [&](auto vi) { sum += vi; });

  std::cout << sum << std::endl;
  return 0;
}
```

##### C++ 17 支持常數值運算
```cpp
int main() {
  std::vector v = {4, 3, 2, 1};
  int sum = 0;
  
  sum = std::reduce(v.begin(), v.end(), 0, [](int x, int y) { 
    return x + y; 
  });

  std::cout << sum << std::endl;
}
```

##### C++ 20 
* 區間 range：概念很像 LINQ
* 模塊 module
* 允許函數參數 auto
* 協程與生成器
* format 支持：可以使用像是 rust 的字符串格式化

# 構造函數
---
#### RAII (Requisition is Initialization)
資源獲取視為初始化，反之，資源釋放視為銷毀。
```cpp
int main() {
  std::vector<int> v(4); // 構造函數，獲取資源，背後 malloc 獲取資源
  v[0] = 4;
  v[1] = 3;
  v[2] = 2;
  v[3] = 1;
  int sum = 0;
  for (size_t i = 0; i < v.size(); i++) {
    sum += v[i];
  }

  std::cout << sum << std::endl;
  return 0; 
} // 解構函數，離開作用域，自動釋放，背後進行 free
```
> 重要！！！在新版 C++ 就不要再使用 `new` 與 `delete` 來手動管理資源。

##### exception-safe 異常安全
C++ 標準保證當發生異常，會調用已經創建的解構函數，所以 C++ 不需要有 finally 語句塊。

### 自定義構造函數
```cpp
struct Pig {
  std::string m_name;
  int m_weight;
  // 無參數
  Pig() : m_name("一隻豬"), m_weight(10) {}
  // 單一參數，記得使用 explicit 修飾，避免隱式的使用賦值調用構造函數
  explicit Pig(int weight)
      : m_name("一隻重達" + std::to_string(weight) + "kg 的豬"),
        m_weight(weight) {}
  // 多參數
  Pig(const std::string &name, int weight) : m_name(name), m_weight(weight) {}
};

int main() {
  Pig pig; // 無參數不用帶括號
  // Pig pig2 = 100; // error 因為使用 explicit 修飾
  Pig pig2(100);
  Pig pig3("豬豬", 10);

  return 0;
}
```

> 補充：多參數下 explicit 也有用，用於使用 `{}` 初始化列表直接使用隱式賦值調用構造函數。
```cpp
Pig pig = { "豬豬", 10 }; // 若雙參數構造函數標記 explicit 的話，無法編譯通過
```

##### 調用構造函數使用 `{}` `()` 有什麼區別 (重要)
`()` 會進行隱式類型轉換，而使用 `{}` 不會轉換。如下
```cpp
int(3.14f) // 不會出錯
int{3.14f} // 會出錯

Pig("豬豬", 3.14f); // 不會出錯
Pig{"豬豬", 3.14f}; // 會出錯，因為 float 不能轉換成 int
```
* 在 google code style 中有明確說明不要使用 `()` 調用構造函數
* 需要進行類型轉換應使用 
	1. `static_cast<int>(3.14f)` 而不是 `int(3.14f)`
	2. `reinterpret_cast<void*>i(0xb8000)` 而不是 `(void*)0xb8000`

> `{}` 比 `()` 更安全，可以避免 narror cast (窄轉換)。

##### 成員初始化
這種方式可以保證安全，不會因為沒有注意而生成為初使化的數值，因為 C 語言 struct 默認是不會進行初始化，而是該空間原本的數值。
```cpp
struct Pig {
	std::string m_name { "佩其" };
	int m_weight { 80 }; // 等價於 int m_weight {};
	Demo m_demo { "Hello", "world" };
	void *p { nullptr };  // 等價於 void *p {};
}
```
> 建議對所有基礎類型，強烈建議都先這樣進行初始化，避免塞入隨機數值。

### 拷貝構造與拷貝賦值
拷貝構造函數與拷貝賦值函數都是編譯器會默認生成的，裡面都會進行 byte-wise copy 也就是**淺拷貝**。
* 拷貝賦值函數相較於拷貝構造低效，因為拷貝賦值函數會先調用默認的構造函數，在進行賦值。
* 拷貝賦值函數用於讓使用者方便進行拷貝時才考慮使用
* 可以透過 `=delete` 進行刪除
```cpp
class C {
	C(const C &c); // 拷貝構造函數
	C &operator=(const C &c); // 拷貝賦值函數
}
```


### 移動
##### 為什麼需要移動？
我們會將類的拷貝都設定為深拷貝（需要自己寫），而有些時候我們只希望進行安全轉移數據，而不是拷貝出新的一份。
* 移動的時間複雜度 O(1) 拷貝的時間複雜度 O(n)
* 可以使用 `std::move` 來實現移動
* 原來的變量持有的指針會被清零，就不會造成 double free

> 補充 `std::swap`
> 我們可以使用該標準庫提供的函數來作到交換兩個對象。不會涉及到 temp，交換效率為 O(1)。

##### 會自動觸發 move 的情況
* return 時，這個是因為編譯器默認做了 RVO

##### 細節
* `std::move` 只是一個修飾符號，直接使用沒有意義，如下
```cpp
int main() {
std::vector<int> v;
std::move(v); // 並不會清空 v
std::vector<int> v2(std::move(v)); // 會調用移動構造函數，會清空 v
std::vector<int> v3 = std::move(v); // 會調用移動賦值函數，會清空 v
}
```

#### 移動構造與移動賦值函數
```cpp
class Vector {
	Vector(Vector &&other); // 移動構造
	Vector &operator=(Vector &&other); // 移動賦值
}
```

##### 默認實現
* 移動構造 = 拷貝構造 -> 被移動者解構 -> 被移動者默認構造
	* 最後一步調用默認構造是因為生出空的對象
* 移動賦值 = 拷貝賦值 -> 被移動者解構 -> 被移動者默認構造
	* 最後一步調用默認構造是因為生出空的對象
> 默認的移動構造與移動賦值是低效的，但不會出錯。

##### 自己實現
```cpp
Vector(Vector &&other) {
	m_size = other.m_size;
	other.m_size = 0;
	m_data = other.m_data; //　資源交換
	other.m_data = nullptr;
}

Vector &operator=(Vector &&other) {
	this->~Vector(); // 原本的先解構
	m_size = other.m_size;
	other.m_size = 0;
	m_data = other.m_data; // 資源交換
	other.m_data = nullptr;
	return *this;
}
```

### 三五法則
避免淺拷貝導致多個類共用相同的資源，解構時產生 double free。
* 一個類定義了解構函數，必須同時定義或刪除拷貝構造與拷貝賦值
* 一個類定義了拷貝構造函數，必須同時定義或刪除拷貝賦值函數。
	* 注意刪除拷貝賦值函數還需要將拷貝構造函數加上 `explict`，避免使用 = 來調用。
* 一個類定義了移動構造函數，那必須同時定義或刪除移動賦值函數。
* 一個類定義了拷貝構造函數或拷貝賦值函數，最好同時定義移動構造與移動賦值函數。
> 重點中的重點，把淺拷貝改為深拷貝


# 智能指針
---
### 解決內存管理問題 unique_ptr
* 使用 `std::make_unique_ptr<C>` 來建立等價於使用 `new C`
* 在離開作用域時候自動觸發解構函數，來實現資源釋放
* 想要提前釋放的話有兩種方式
	* `p = nullptr`
	* `p.reset()`
* 刪除拷貝構造函數，導致其無法被拷貝

##### 傳參數情況
* 情況一：函數不需要獲得所有權
	* 使用 `p.get()` 獲取原始指針進行傳參
```cpp
// 函數定義如下
void func(C *p);
```
* 情況二：函數獲取所有權
	* 使用 `std::move(p)` 進行傳參
```cpp
// 函數定義如下
void func(std::unique_ptr<C> p); // 這樣會調用移動構造函數
```

### 更智能的指針 shared_ptr
unique_ptr 解決了 double free 問題的方式禁止拷貝，但是會導致傳遞參數的時候只能透過移動，會導致使用困難。
而 shared_ptr 解決了這個問題：
* 透過犧牲效率，來換取方便使用
	* 因為維護引用計數，當初始化時計數為 1， shared_ptr 本身被拷貝時候加 1，shared_ptr 本身被解構時候減 1，直到為 0 時解構指向的對象。(對象只有一個)
* 引用計數是 automic 的，是多線程安全的，但對象本身可能不是。（效率比 unique 差的主要原因）


#### 循環引用問題
因為兩個對象都持有對方的 shared_ptr，所以引用計數永遠不會為零，導致的內存洩漏，如下
```cpp
#include <memory>

struct C {
  typedef std::shared_ptr<C> ptr;
  C::ptr m_child;
  C::ptr m_parent;
};

int main() {
  C::ptr parent = std::make_shared<C>();
  C::ptr child = std::make_shared<C>();

  parent->m_child = child;
  child->m_parent = parent;

  parent = nullptr; // parent 不會釋放，因為 child 還指向它
  child = nullptr; // child 不會釋放，因為 parent 還指向它
  return 0;
}
```

##### 解決方式一：weak_ptr
```cpp
#include <memory>

struct C {
  typedef std::shared_ptr<C> ptr;
  C::ptr m_child;
  std::weak_ptr<C> m_parent;
};

int main() {
  C::ptr parent = std::make_shared<C>();
  C::ptr child = std::make_shared<C>();

  parent->m_child = child;
  child->m_parent = parent;

  parent = nullptr; // parent 不會釋放，因為 child 還指向它
  child = nullptr; // child 不會釋放，因為 parent 還指向它
  return 0;
}
```

###### weak_ptr 使用方式
```cpp
int main() {
  std::shared_ptr<C> p = std::make_shared<C>();
  std::cout << p.use_count() << std::endl;

  std::weak_ptr<C> weak_p = p;
  std::cout << p.use_count() << std::endl;

  func(std::move(p));
  std::cout << p.use_count() << std::endl;

  if (weak_p.expired()) { // 確認是否失效
    std::cout << "已經失效" << std::endl;
  } else {
    weak_p.lock()->do_something(); // lock 取得 shared_ptr
  }

  return 0;
}
```

### 類成員中的智能指針 (難點)
##### 如何判斷要使用什麼智能指針
組合一：
1. `unique_ptr`: 當該對象僅僅只屬於我。e.g. 父窗口指向子窗口
	* 當成員中有 `unique_ptr` 該類的拷貝構造與拷貝賦值會被刪除。 
2. 原始指針: 當對象不屬於我，但給予者釋放前我必然釋放（必須很清楚），有誤判風險。 
	* e.g. 子窗口指向父窗口，因為父窗口關閉子窗口必然關閉。

組合二：
1. `shared_ptr`: 當對象有多個其他的對象共享。
2. `weak_ptr`: 當對象不屬於我，給予者釋放後我仍然可能不被釋放。
	* e.g. 指向窗口上次被點擊的元素（被點擊的元素可能已經被釋放）

安全類型:
1. `int i` 基礎類型
2. `std::vector<int> arr` STL 容器
3. `std::shared_ptr<Obj> child` 智能指針
4. `Obj *parent` 從智能指針裡 `.get()` 出來的原始指針

不安全類型:
1. `char *ptr` malloc/free 或者 new/delete 生成的原始指針
2. `GLint tex` 該類型對應某種資源(為 GPU 的資源)
3. `std::vector<Obj *> objs` STL 容器存放了不安全的對象

##### 心法
* 當你需要自定義解構函數，代表該類有管理資源，需要考慮三五法則
* 若該類都是都是安類型，五大構造都只需要默認就是安全的
無外忽兩種情況：
1. 類管理著資源：
	* 處理方式：刪除拷貝函數，統一使用智能指針管理資源
2. 類是數據結構： e.g. 自己精心製作的數據結構
	* 處理方式：絕對不要使用默認
		* 想要支持：自己要定義拷貝函數與移動函數
		* 不想支持：就刪掉

# 其他資源
---
[google coding style C++](https://google.github.io/styleguide/cppguide.html#C++_Version)


