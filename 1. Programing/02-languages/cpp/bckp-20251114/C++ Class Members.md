# 類的成員
---
1. 構造函數
	1. 普通構造函數
	2. 拷貝構造函數
	3. 析構函數
	4. 移動構造函數
2. 成員變量
	1. 普通成員變量
	2. 靜態成員變量
	3. 成員常量
3. 成員函數
	1. 普通成員函數
	2. 靜態成員函數
	3. 常量成員函數
4. 賦值函數
	1. 拷貝賦值函數
	2. 移動賦值函數
5. 嵌套類
6. 運算符重載

# Main Constructors
---
## Constructor (構造函數)
```cpp
class People {
...
public:
	People() {
		init();
	}
	People(const std::string& name) : m_name(name) {}
private:
	init() {...}
}
```
- 若沒有寫構造函數，編譯器會自動提供一個空的構造函數
- **構造函數之間是不能相互調用**
- 使用new創建對象，會調用構造函數
- 構造函數不要做太多的工作，獨立出一個函數讓構造函數調用 (init)

## Destructor (析構函數)
```cpp
class People {
...
public:
	virtual ~People() {}
}
```
- 若沒有寫析構函數，編譯器會自動提供一個空的析構函數
- **動態分配的內存，要在析構函數釋放**
- 使用delete釋放對象，會調用析構函數
* *virtual* 可以解決基類指針指向子對象，子類無法析構的內存洩漏

## Copy Constructor (拷貝構造函數)
```cpp
class People {
public:
	People(const People& other) {
		m_name = other.m_name; 
		m_age = other.m_age;  
		m_ptr = new int; // 先分配新的內存 
		memcpy(m_ptr, other.m_ptr, sizeof(int)); // 進行賦值
	}
}
```

- 若沒有寫拷貝構造函數，編譯器會自動提供一個拷貝構造函數，把”值”拷貝（注意指針）⇒ **編譯器提供的構造函數是淺拷貝** 
> **數組不要當成指針**，他是該class中的一部分，拷貝構造函數會將原對象的array中的值(每個數值)，拷貝到新對象的array中，**只需要注意動態分配內存**
* 函數使用*值傳遞會調用拷貝構造函數*
* **返回值若使用值傳遞，可能會調用拷貝構造函數**
> 因為存在Return value optimization (RVO)，編譯器會直接將返回值構造到接收變量上 ^bda101
#### 取消RVO
```shell
# 添加編譯器選項 -fno-elide-constructors
g++ ... -o ... -fno-elide-constructors
```
## Move Constructor (移動構造函數)
```cpp
class People {
	People (People&& other) {
		if(m_ptr != nullptr) {
			delete m_ptr; // 清空自己資源
		}
		m_ptr = other.m_data; // 交換指針
		other.m_data = nullptr; // 置空對方
	}
}
```
* 為使用`std::move` 先決條件之一，條件如下：
	1. 移動構造
	2. 移動賦值
* 使用函數參數使用*右值引用傳遞*會調用移動構造函數
* 返回值使用右值引用，可能會調用移動構造函數
	> 也是因為[[C++ Class Members#^bda101|RVO]] 
* 若沒有持有資源（堆區、文件句柄等），移動語意沒有意義

# Assign Operator
---
## Copy Assign (拷貝賦值函數)
```cpp
class People {
	People& operator=(const People& other) {
		if(this == &other) {
			return *this;
		}
		if (m_ptr != nullptr) {
			delete m_ptr;
			m_ptr == nullptr
		} 
		m_ptr = new int;
		memcpy(m_ptr, other.m_ptr, sizeof(int));
		}
	}
}
```
#### 為什麼返回值要使用引用（左值引用）？
因為我們希望返回值是自己本身且不是指針，若不是引用則會在構造一個新的對象，那返回的就不是本身，若返回指針就無法進行鏈式調用。

## Move Assign (移動賦值函數)
```cpp
class People {
	People& operator=(People&& other) {
		if (this == &other) {
			return *this
		}
		if (m_ptr != nullptr) {
			delete m_ptr;
			m_ptr = nullptr;
		}
		m_ptr = other.m_ptr;
		other.m_ptr = nullptr;
		return *this;
	}
}
```

