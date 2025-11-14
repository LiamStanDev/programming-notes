# 右值引用
---
右值引用就是讓一個**臨時對象（rvalue）續命**，延長其生命週期與直到右值引用變量結束，使其可以像其他變量一樣操作。
### 右值概念
右值表示表達式結束之後就不再存在的臨時對象，分為
1. 純右值（臨時變量）：
	1. 函數非左值引用的返回值
	2. 匿名對象
	3. 運算符表達式結果
	4. 字面值：`1`, `3.14`, `"hello"`
```cpp
struct AA { int m_a; }; 
AA get_temp() { return AA{10}; }

int main() {
	AA&& aa = get_temp(); // 其實返回值優化就是做這個
}
```
2. 將亡值（xvalue）
	表示移動後的對象，也就是 `std::move()` 的返回值
```cpp
// 將亡值
// e.g. 1
std::vector<int>&& foo() { // 需要使用右值引用 
	std::vector<int> temp = {1, 2, 3}; 
	return std::move(temp); // std::move後的返回值為xvalue
}

// e.g. 2
std::vector<int> v1 = {1, 2, 3}; 
std::vector<int> v2 = std::move(v1); // std::move(v1) 為 xvalue

```
> 示例2中的 `std::vector<int>`有提供移動構造函數

### std::move
用來將**左值轉換成右值**，用於將其持有的資源轉移給另一個變量。
#### 注意事項
1. 返回值不要使用`std::move`
	因為編譯器有做[[C++ Class Members#^bda101|RVO]]，RVO本身就會將返回值直接構造在接收變量上，不用多此一舉導致而外開銷。
2.	不要返回引用，包括左值引用與右值引用，請交給編譯器進行優化
3. C++11所有容器都實現移動語意，可以使用 `std::move`

# 移動語意
---
一個類要實現移動語意，要實移動構造函數與移動賦值函數。
![[C++ Class Members#Move Constructor (移動構造函數)]]
![[C++ Class Members#Move Assign (移動賦值函數)]]
### 調用移動構造函數時機
```cpp
AA a1;
AA a2(std::move(a1)); // 顯示調用
AA a3 = AA(std::move(a2)); // 顯示調用
AA a4 = std::move(a3); // 隱式調用
```
#### 不會使用時機
```cpp
AA a1; // 構建
auto f = []() {
    AA aa;
    return aa;
};
AA a2 = f(); 
```
> **為何沒有調用移動構造？**
> 因為RVO導致直接構造在a2上。
### 調用移動賦值函數時機
```cpp
AA a1; // 構建
AA a2; // 構建
a2 = std::move(a1); // 賦值
auto f = []() {
    AA aa;
    return aa;
};
a2 = f(); 
```
> 只要定義移動賦值函數，返回值的臨時變量會被移動給接收變量。


# 完美轉發 (Perfect Forwarding)
---
某一個函數作為轉發函數去調用別的重載函數時，因為該函數在接收參數時會將參數當作左值（資訊損失），導致不能轉發給右值做為參數的重載函數。
### 此困境條件
1. 有一個重載函數分別重載右值引用與左值引用作為參數。
2. 需要一個轉發函數調用該重載函數。

### 困境
```cpp
// 左值引用重載函數
void fun(int& x) {
	std::cout << "rvalue: " << x << std::endl;
}
// 右值引用重載函數
void fun(int&& x) {
	std::cout << "rvalue: " << x << std::endl;
}

// 轉發函數
void fun_pass(int x) {
	fun(x); // 在這邊都會被當成右值
}

int main() {
	int x = 10;
	fun_pass(x); // lvalue
	fun_pass(9); // rvalue
}
```

```shell
# 結果
lvalue: 10
lvalue: 9
```

### 改進
#### 那重載轉發函數？
```cpp
void fun_pass(int& x) {
	fun(x); // 調用左值重載
}
void fun_pass(int&& x) {
	fun(std::move(x)); // 調用右值重載
}
```
> 右值引用作為接收參數，使其可以用於分辨接收者是誰，但是同樣會損失右值資訊。

#### 使用完美轉發
完美轉發為一個模板，免去重載轉發函數，細節如下：
1. 轉發函數參數必須為 `T&&` ，若使用 `int&&` 則無法接收左值
2. 需要使用 `std::forward<T>` 轉發
```cpp
template <typename T>
void fun_pass(T&& x) {
	fun(std::forward<T>(i));
}
```

#### 是否可以使用const referece 接收左右值再轉發？
ans: 不行，可以轉發但是都會變成左值，如下
```cpp
void fun_pass(const int& x) {
	fun(std::forward<int>(const_cast<int&>(i)));
}
```

