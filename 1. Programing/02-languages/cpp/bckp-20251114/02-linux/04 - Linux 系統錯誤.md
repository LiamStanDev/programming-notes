### 簡介
在 C++ 程序中，調用了系統庫函數，可以通過函數的返回值來判斷調用是否成功。但其實還存在一個全局變量 `errno` 存放調用的錯誤 flag。
在程式中可以透過 `errno` 值來具體判斷錯誤原因，e.g. accept 返回值是 -1 表示失敗，但失敗原因可能是非阻塞導致。[[05 - Non-blocking IO#非阻塞 accept]]

* 頭文件 `<errno.h>`
* 配合 `strerror()` 與 `perror()` 可查看詳細錯誤訊息。
* libc 提供 133 個錯誤 flags。(從 1 ~ 133)
> 注意: 只有系統調用發生錯誤才會設置 errno，但執行成功並不會清空 errno，故 **errno 需要與系統調用返回值一起看**。 


### 使用
#### strerror() 函數
用來將錯誤代碼轉換為描述錯誤的字符串。
##### 簽名
```cs
char *strerror(int errnum); // 非線程安全
int strerror_r(int errnum, char *buf, size_t buflen); // 線程安全 
```

##### 查看錯誤
```cpp
int main()
{
  int iret = mkdir("/tmp/aaa", 0755);
  std::cout << "iret=" << iret << std::endl;
  std::cout << errno << ": " << strerror(errno) << std::endl;

  return 0;
}
```
> 執行第二次就會報錯。

#### perror() 函數
打印最近一次的錯誤訊息在 console 上，並沒有甚麼用，因為後臺運行程式需要 log 在一個文件中。


