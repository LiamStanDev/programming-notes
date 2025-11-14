![[20230707_16h49m02s_grim.png]]
## 基本介紹
---
### ostream
> 表示輸出流，為所有輸出流的基類

有以下幾個ostream
1. std::cout
2. std::cerr
3. std::clog
4. std::ofsream

### istream
> 表示輸入流，用來向控制台(鍵盤)或者文件讀取取數據

有以下幾個istream
1. std::cin
2. std::ifstream

### streamstrea (補充)
> 一個用流操作做的字符串，用於方便的操作字符串 
* 頭文件：`#include<sstream>`

有如下作用：
1. 數據格式化轉換
```cpp
std::stringsream ss;
// integer to string
int num = 10;
ss << num;
std::string s = ss.str();

// string to double
ss << "22,6";
double d;
ss >> d;
```


## 文件寫入
---
### 寫入模式
- `std::ios::out` : 為默認值用於覆蓋寫入，流程：刪除文件→創建文件→寫入
- `std::ios::trunc` : 覆蓋寫入，流程：清空文件→寫入
- `std::ios::app` : 追加寫入
#### 二進制
- `ios::binary` : 二進制文件
> 默認為字符模式
```cpp
// 二進制文件追加
std::ostream fout(”hello.txt”, ios::app | ios::binary)
```

### 操作
```cpp
#include <ofstream>
int main() {
	// 若不存在會建立, 相當於fout.open("hello.txt", std::ios::out)
	std::ofstream fout("hello.txt"); 
	if(fout.is_open()) {
		fout << "Hello\n" << "I'am Liam\n" 
			<< "Today is a good day" << endl;
		fout.close();
	}
}
```

### 開啟失敗原因
1. 目錄不存在
2. 磁盤空間已滿
3. 沒有權限(Linux下很常見)

## 文件讀取
---
### 讀取模式 (只有一種)
* `std::ios::in` : 讀取

### 操作
#### 按字讀取
```cpp
#include <fstream>
int main() {
	std::ifream fin("hello.txt");
	if (fin.is_open()) {
		std::string buffer;
		while (fin >> buffer) { 
			std::cout << buffer << std::endl;
		}
	}
}
```
> 注意：按照空格讀取但是buffer不會包含空格

#### 按行讀取
```cpp
#include <fstream>
#include <string> // 為了使用getline
int main() {
	std::ifstream fin("hello.txt");
	if (fin.is_open()) {
		std::string buffer;	
		while (std::getline(fin, buffer)) {
			std::cout << buffer << std::endl;
		}
	}
}
```
> 注意：
> 1. 按照 `\n` 讀取但是buffer不會包含 `\n`
> 2. windows中換行為 `\r\n`

### 開啟失敗原因
1. 目錄不存在
2. 文件不存在
3. 沒有權限(Linux下很常見)

## 寫入二進制文件
---
* 只能使用 `write` 方法
	* `write` 要接收`const char*` 不是 `void*`
* 要設定 `std::ios::binary`
### 操作：將結構體存放到文件
```cpp
#include <fstream>
struct A {
	string name;
	char sex[5];
	double weight;
}
int main() {
	A a;
	ofstream fout("A.bat", std::ios::binary);
	if (fout.is_open()) {
		fout.write(reinterpret_cast<const char*>(&a), sizeof(A))
		fout.close();
	}
}
```


## 讀入二進制文件
---
### 通用二進制類型
1. 音訊：mp3
2. 影片：mp4
3. 圖片：jpg, png
> 所以二進制文件讀取，要依照業務需求而定

### 二進制文件 vs. 文本文件
* 文本文件：容易讀取
* 二進制文件：方便加密、壓縮

### 操作：讀回結構體
```cpp
#include <fstream>
int main() {
	A a;
	ifstream fin("hello.bat", std::ios::binary);
	if (fin.is_open()) {
		while (fin.read(reinterpret_cast<const char*>&a, sizeof(A))) {
			std::cout << a.name << std::endl;
		}
	fout.close();
	}
}
```

## 文件指針
---
文件讀取與寫入就是操作系統提供一個文件指針也就是句柄，隨者文件指針的移動來取得或者寫入文件。
### 獲取文件指針
* ofstream
```cpp
ofstream fout("hello.txt");
fout.tellg();
```
* ifstream
```cpp
ifstream fin("hello.txt");
fout.tellp();
```
> 兩者沒有不一樣，只是名稱不同...

### 移動文件指針
#### 相對移動
```cpp
fout.seekg(30); // 等同: fout.seekg(30, std::ios::cur);
fin.seekp(100);
```
> 若設定為`std::ios::binary` 每次移動就是一個bit，沒有設定就是一個byte

#### 絕對移動
```cpp
fin.seekp(128) // 若不是binary，向右移動128個字節

fin.seekp(30, std::ios::beg)  // 從頭開始，向右128字節 
fout.seekg(-5, std::ios::end) // 從最後開始向左移動5字節

fin.seekp(std::ios::beg); // 回到頭
```

## 文件緩衝
---
因為外存寫入與讀取速度很慢，所以會使用內存中的緩衝區
1. 寫入：待緩衝區寫滿，在一次性的讓磁盤寫入。
2. 讀取：將內容放入緩衝區，待讀取完了緩衝區內容，才再次在從外存讀取數據放入緩衝區。
* 減少IO次數
* 每次打開一個文件，OS就會分配一個緩衝區，每個緩衝區對應一個流。
* 讀取緩衝區不用在意，但是**寫入緩衝區需要在意**
- [u] 有些情況希望及時寫入，e.g. 避免系統掛掉數據庫尚未寫入，即時日誌等

### 無法及時寫入案例
```cpp
#include <unistd.h> 
#include <fstream> 
using namespace std; 
int main() { 
	std::ofstream fout("temp.txt");
	// ofstream << unitbuf;
	if (fout.is_open()) {
		for (int i = 0; i < 1000; i++) {
			fout << "i=" << i 
			<< ": " << "I'm liam. I'm " 
			"littlllllllllllllllllllllllllllllllllllllllllllle bird\n";
			// fout.flush();
			usleep(100000); // 1/10 seconds
		}
		fout.close();
	}
}
```

```shell
# 可以使用tail -f file_name來觀察文件寫入情況
tail -f temp.txt
```

### 刷新緩衝區的方法
* `std::endl` : 功能為刷新緩衝區＋換行
* `std::unibuf` : 設定緩衝區為單位緩衝區，設定該流永久性單位緩衝
* `flush()` : 刷新緩衝區

## 流狀態
---
表示該流的當前狀態，有以下幾個狀態(0表示沒事1表示發生)：
1. `eofbit` : 文件末尾
2. `failbit` : 輸入流未能讀取預期的字符(e.g. 非致命錯誤如想讀取整數，但內容是字符串，或者文件到末尾，或IO失敗)
3. `badbit` : 嚴重的錯誤(e.g. 對輸入流進行寫入操作、磁盤空間不足等）

### 流狀態方法
- `good()` 方法：若eofbit, failbit, badbit均為零則返回true
- `clear()` : 清理流狀態
- `setstate()` : 重置流狀態
```cpp
while(fin >> buffer) {} // 其實是封裝過後的 
// 原始寫法 
while (true) { 
	fin >> buffer; 
	if(fin.eof())  { 
		break; 
	} 
}
```
