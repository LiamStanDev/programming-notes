### main 函數參數
```cpp
int main(int argc, char *argv[], char *envp[])
{
		std::cout << "Has " << argc << " arguments." << std::endl;
		for (int i = 0; i < argc; i++)
		{
				std::cout << i  << ": " << argv[i] << "\n";
		}
		std::cout << std::endl;

		for (int i = 0; envp[i] != 0; i++) 
		{
				std::cout << envp[i] << std::endl;
		}
}
```
* argc: 輸入參數個數。
* argv: 為輸入參數，第一個為執行路徑
* envp: 為環境變量，最後一個為 0

#### 設置環境變量
```cpp
// 函數簽名
int setenv(const char *name, const char *value, int overwirte);
```
* name: 變量名
* value: 變量值
* overwirte: 
	* 0: 表示不存在新增，存在不複寫。
	* 非零: 表示複寫
* 返回值:
	* 0: 表示成功。
	* -1: 表示失敗(幾乎不可能失敗)。
> 只影響當前進程，不是整個 shell