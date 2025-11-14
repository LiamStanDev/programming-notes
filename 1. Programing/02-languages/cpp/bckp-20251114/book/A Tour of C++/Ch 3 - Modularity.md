### Separate Compilation

### Modules

### Namespaces


### Error Handling
#### C++ 提供的錯誤處裡功能
* 類型系統: 使用更高級別的抽象來簡化程式設計並限制錯誤 (這邊指的是透過良好的封裝)
* 例外處理機制: 使用 `try` 、`catch` 與 `throw` 來處理執行階段錯誤
* RAII: 使用建構函數獲取資源、並使用解構函示來釋放資源，確保資源釋放得到保證。
##### 使用例外處理機制
```cpp
#include <stdexception>

double& Vector::operator[](int i) {
	if (i < 0; size() <= i) {
		throw std::out_of_range{"Vector::operator[]}
	}
}

void f(Vector& v) {
	try {
		v[v.size()] = 7;
	} catch(std::out_of_range& err) {
		std::cerr << err.what() << "\n"; // 注意這個 waht 只會打印 error message，並不會有調用棧的資訊
	}
}
```

* 若函數不會拋出錯誤，應該標記上 `noexcept`
* 若標記 `noexcept` 函數拋出錯誤，會直接 terminate
#### 錯誤處理的重要概念 Invariant
類設計的核心概念，表示在 object 存在期間都可以滿足合理條件，以下三件事情可以保證:
1. RAII: 透過建構函數獲取資源，使用解構函數來釋放資源，保證初始化與釋放的合理狀態。
2. Precondition: 函數中在被調用的時候要先確認條件是否合理

#### 選擇錯誤處理方式
##### 1. Error Code (Return Error Indicator)
* 失敗是正常與可預期的，比如說打開一個路徑不存在的文件。
* 直接呼叫者預期可以合理的處理該錯誤。
##### 2. Exception
* 罕見錯誤，以至於寫程式時容易會忘記檢查的錯誤，使用 Exception 拋出
* 直接呼叫者無法立刻處理該錯誤，使用 Exception 拋出，讓錯誤可以向上傳播。
##### 3. Terminating the program
* 無法復原的錯誤，如 memory exhaustion (一般的程序無法處理，但不一定適用於所有情況)
* 錯誤復原的方式須透過重新啟動與重新開機等方式
#### 補充: 各公司對 C++ exception 的態度
###### Google
明確規定不使用 C++ 異常機制。
1. 性能開銷: C++ 異常處理會在某些平台上引入運行時開銷 (但書中說開銷很小)
2. 代碼複雜性: 異常處理會導致代碼邏輯變得複雜，錯誤傳播難以追蹤
3. 跨語言兼容性: C++ 異常會與其他語言交互時引發問題
4. Debug 困難
解決方式: 
1. 透過錯誤返回數值: `Status` 或者 `StatusOr<T>`，來表示操作的成功與失敗
2. 宏與斷言: 對於嚴重錯誤使用 `assert`
###### Facebook
支持使用 C++ 異常處理機制
###### Microsoft
支持使用 C++ 異常處理機制

> 我的想法異常是 C++ 標準，能夠與語言特性如 RAII 和智能指針自然結合，並且可以解偶異常處理邏輯與主要程式碼混合，增加可讀性，另外現代編譯器已經實現諸多優化，只要操作中沒有拋出異常性能開銷可以忽略。

### Function Arguments and Return Values
Function 之間傳遞資訊可以透過: 
1. Global varaibles: 非常不推薦，error prone
2. pointer and reference
3. shared state in a class object
> 這邊在提醒一下 pointer 存放的是地址，reference 指的是 object 本身(memory)請參見 [[Ch 1 - The Basic#Type, Variables, and Arithmetic]]

在傳遞資訊的時候我們必須考慮:
**1. Is object is copy or shared?**
**2. If an object is shared, is it mutable?**
**3. Is an object moved, leaving an "empty object" behind?**


#### Argument Passing

| 方式                      | 是否共享 | 是否可變 |
| ----------------------- | ---- | ---- |
| Pass by value (default) | 否    | 不用考慮 |
| Pass by reference       | 是    | 是    |
| Pass by const reference | 是    | 否    |


#### Return Value