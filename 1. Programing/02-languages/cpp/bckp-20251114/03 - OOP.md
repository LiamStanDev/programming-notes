# 對象模型
---
### 成員類型
* 數據成員: `nonstatic`, `static`
* 函數成員: `nonstatic`, `static`, `virtual`
### 對象大小
* 所有**非靜態成員**數據大小
* 內存對齊的填充
* 為了支持 virtual 函數成員多一個指向 virtual table 的指針。
### 彙編放置位置
* 成員函數與普通函數: .text
* 靜態成員變量: .data (與全局變量地址相鄰)
* 類中沒有非靜態成員 (空的類)，還是會佔用一字節佔位。
- 對象的地址為第一個非靜態成員變量的地址。
* 存在對象指針表，儲存函數、靜態成員變量與普通成員變量。(因為對象組成分布在內存不同區域)
* 空指針可以訪問靜態成員變量與函數 (因為靜態成員不會使用 this 指針)
```cpp
class Person
{
public:
	static int population;
	static void show() { std::cout << population << std::endl; }
}

int Person::population = 10;

int main()
{
	Person *p = nullptr;
	// 以下不會報錯
	int res = p->popluation;
	p->show();
}
```


# 三大構造函數
---
### 案例: 完整的類
```cpp
#include <algorithm>
#include <iostream>
#include <iterator>
#include <string>

class People
{
public:
  People() : People("", 0) {}
  
  People(std::string name, int age) : m_name(name), m_age(age) {}

  virtual ~People()
  {
    delete m_ptr;
    m_ptr = nullptr;
  }

  People(const People &other)
      : m_name(other.m_name), m_age(other.m_age), m_ptr(nullptr)
  {
    std::copy(std::cbegin(other.m_memo), std::cend(other.m_memo),
              std::begin(m_memo));
    if (other.m_ptr != nullptr)
    {
      m_ptr = new int(*other.m_ptr);
    }
  }

  People &operator=(const People &other)
  {
    if (this != &other)
    {
      m_name = other.m_name;
      m_age = other.m_age;
      std::copy(std::cbegin(other.m_memo), std::cend(other.m_memo),
                std::begin(m_memo));

      delete m_ptr;
      m_ptr = nullptr;
      if (other.m_ptr != nullptr)
      {
        m_ptr = new int(*other.m_ptr);
      }
    }

    return *this;
  }

  void show() const
  {
    std::cout << "name: " << m_name << ", age: " << m_age << std::endl;
    std::cout << "My memo: " << m_memo << std::endl;
    std::cout << (long)m_memo << std::endl;
    std::cout << m_memo[0] << std::endl;
    std::cout << (long)m_ptr << std::endl;
  }

private:
  std::string m_name;
  int m_age;
  char m_memo[301]{};
  int *m_ptr{};
};

int main()
{
	// stack
  People p1;
  People p2("Liam", 26);
  p1.show();
  p2.show();

	// heap
  People *p6 = new People("Andy", 24);
  p6->show();
  delete p6;
  p6 = nullptr;

	// copy constructor
  People p8(p2);
  p8.show();

  // assign operator
  p8 = p3;
  p8.show();

  return 0;
}
```

### 構造函數
- 若沒有寫構造函數，編譯器會自動提供一個空的構造函數
- 構造函數之間是不能相互調用
- 使用new創建對象，會調用構造函數
- 構造函數不要做太多的工作，獨立出一個函數讓構造函數調用

