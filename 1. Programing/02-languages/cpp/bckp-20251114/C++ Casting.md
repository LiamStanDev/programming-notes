C++ 提供了四種類型轉換，用來取代C語言的類型轉換。

### static_cast
用於非指針與引用類型的轉換場景。
```cpp
void fun(int x) {}

int main() {
	double x = 22;
	// fun(x); // Error 精度損失 
	fun(static_cast<int>(x));
}
```
### reinterpret_cast
用於指針的類型轉換。
```cpp
void fun(double* x) {}

int main() {
	int* x = new int(3);
	// fun(static_cast<double*> x); // Error
	// C-style
	fun((double*)((void*)x)); // 先轉成void* 再轉
	// C++
	fun(reinterpret_cast<double*>x); // Ok
}
```

### const_cast
將const pointer 與 const reference的const去掉，常用於將const指針傳遞給非const 指針參數。
```cpp
void fun(double* d) {}

int main() {
	const double* d = new double(22.4);
	// fun(d); // Error
	fun(const_cast<double*>(d));
}
```

### dynamic_cast
用於將基類指針轉換成子類指針
```cpp
class People {
public:
	virtual ~People() {}
};

class Man : public People {};

void fun(Man* m) {}

int main() {
	People* m = new Man();
	// fun(m); // Error
	fun(dynamic_cast<Man*>(m));// Ok
}
```
#### 為什麼我要多添加一個virtual destructor？
1. 因為只有存在virtual table，才可能使基類指針重新指向子類。
2. 只要是繼承就應該要使用virtual destructor
