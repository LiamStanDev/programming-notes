# Arguments
---
### 函數參數原則
1. 不需要修改傳入參數
	1. 基本數據類型與很小的結構體: 使用值傳遞
	2. 數組: 使用const pointer
	3. 很大的結構體: 使用const pointer or const referece
	4. 類：使用const referecne
2. 需要修改傳入參數
	1. 基本數據類型：使用指針
	2. 數組 : 使用指針
	3. 結構體：使用引用或指針
	4. 類：使用引用

### 右值引用參數
當希望資源被移動到該函數，也就是轉移所有權的時候。
#### 案例：push_back()
```cpp
template <class T, class Alloc = allocator<T>> 
class vector {
public:
	void push_back(T&& value) {
		...
	}
}


int main() {
	std::string str = "Hello";
	std::vector<int> v;
	v.push_back(std::move(str));
}
```


### 外部函數作為參數
該情景發生在傳入函數指針，Functor, 與lambda表達式。
#### 只能傳函數指針
```cpp
void fun(int x, double y) {};

void apply_fun(void (*f)(int, int))  { f(11, 23);}
```

#### 使用std::function
可以傳入以上所有種類的函數。
```cpp
void apply_fun(const std::function<int(double)>& f) { f(11, 23); }
```

