# 什麼是編譯器
---
### C++ 代碼本質
```cpp
#include <cstdio>

int main() {
	printf("hello, world!\n");
	return 0;
}
```
> `<cstdio>` 與 `<stdio.h>` 不同的是 cstdio 是包裝在 std 命名空間下的，也就是說可以使用 `std::printf` 

之後執行 `g++ main.cpp -o a.out` 編譯成為可執行文件，我們可以使用將二進制程式碼轉為彙編代碼，如下
```shell
# -D 表示將所有 section 反彙編
# -d 表示只反彙編 .text section
objdump -D a.out | less 
```
查詢 main 函數後結果如下
```text
0000000000401126 <main>:
  401126:       55                      push   %rbp
  401127:       48 89 e5                mov    %rsp,%rbp
  40112a:       bf 10 20 40 00          mov    $0x402010,%edi
  40112f:       e8 fc fe ff ff          call   401030 <puts@plt> # 調用了 puts
  401134:       b8 00 00 00 00          mov    $0x0,%eax
  401139:       5d                      pop    %rbp
  40113a:       c3                      ret
```
我們發現它調用了 puts@plt，查詢如下
```text
0000000000401030 <puts@plt>:
  401030:       ff 25 ca 2f 00 00       jmp    *0x2fca(%rip)        # 404000 
```
其中看到它進行了跳轉，而跳轉為一個相對地址，我們先說明 PLT (Procedure Linkage Table)， PLT 表用來調用共享庫中的函數，這些函數的實際地址在執行時由動態鏈接器解析。我們可以使用 `objdump -D /lib/libc.so6` 發現裡面有 `puts` 函數。
再來我們可以使用以下命令來找到一個二進制文件的動態連接庫有哪些，
```shell
ldd a.out
```
結果如下
```text
        linux-vdso.so.1 (0x00007f18f8594000)
        libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007f18f8200000)
        libm.so.6 => /lib64/libm.so.6 (0x00007f18f8495000)
        libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007f18f8467000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f18f8013000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f18f8596000)
```

### 編譯
我們可以將源文件逐個編譯對象文件
```shell
g++ -o hello.o -c hello.cpp 
g++ -o main.o -c main.cpp 
g++ -o a.out main.o hello.o 
```

### 鏈接
#### 靜態鏈接 vs. 與動態鏈接庫
因為有些功能會被多個可執行文件或其他的庫使用，我們希望它可以獨立出來，可分為：
1. 靜態庫: 可執行文件可以獨立執行，但編譯後體積較大
2. 動態庫: 在可執行文件中生成"插樁函數"，當可執行文件會在執行時，會載入指定目錄中的 .dll 文件放置內存中空閒位置，稱為重定向，可以減少編譯體積。
	1. Windows 下會先查找可執行文件下同目錄，其次是 `%PATH%`
	2. Linux 順序如下
		1. ELF 格式中的 `RPATH` (Run Path，須在編譯命令上添加)
		2. `LD_LIBRARY_PATH` 環境變量
		3. 系統默認路徑 `/usr/lib`
> 注意 `.o` 不是鏈接庫而是目標文件，為 ELF 文件格式，`.a` 與 `.so` 為 ELF 文件集合
#### 編譯庫
##### 編譯成靜態庫
```shell
# 先編譯成目標文件
g++ -c file1.cpp file2.cpp file3.cpp
# 在進行打包
ar rvs libmylibrary_static.a file1.o file2.o file3.o
```
* `ar` 是一個 archive 工具，r 表示 replacement，v 表示 verbose，s 表示 symbol table

##### 編譯成動態庫
```shell
# 先編譯成獨立位置的目標文件
g++ -shared -o libmylibrary.so file1.o file2.o file3.o
# 進行打包
g++ -shared -o libmylibrary.so file1.o file2.o file3.o
```
* `-fPIC` 選項是為了生成位置獨立的代碼（Position Independent Code）
	
#### 鏈接
鏈接就都長一樣，只差別在名稱。
```shell
g++ -o myprogram  main.o -L/path/to/libraries -lmylibrary
```


#### 一個相對完整的編譯
對應的 g++ 命令如下
```shell
g++ -std=c++20 -g -o cuda_test main.cc -I/usr/local/cuda/include -L/usr/local/cuda/lib64 -lcudart -lcublas
```
- `-std=c++20`: 指定使用 C++20 標準。
- `-g`: 添加調試信息。
- `-o cuda_test`: 指定輸出的可執行文件名為 `cuda_test`。
- `main.cc`: 指定編譯的源碼文件。
- `-I/usr/local/cuda/include`: 指定頭文件搜索路徑，這裡假設 CUDA 的頭文件在 `/usr/local/cuda/include`。
- `-L/usr/local/cuda/lib64`: 指定庫文件搜索路徑，這裡假設 CUDA 的庫文件在 `/usr/local/cuda/lib64`。
- `-lcudart -lcublas`: 指定需要鏈接的庫文件，`-lcudart` 是 CUDA runtime 库，`-lcublas` 是 CUDA 的 BLAS 库。

### 頭文件
因為 C++ 很要求要求上下文，不然 C++ 並不知道那個東西是變量、類或者函數等，所以都需要在使用之前先聲明，但每個地方要用都需要先聲明，故統一出一個文件名為頭文件，這樣就不用重複寫，只ㄒ需要引入相同的頭文件就行。

* `#include` 尖括號與雙引號的差別：
	* 尖括號：會默認搜尋 `/usr/includes`，而不會引入當前目錄的頭文件。
	* 雙引號：會依照當前目錄，或者依據雙引號內容指定目錄進行搜索。
* 頭文件遞歸引用問題: 也就是很多文件都互相引用相同的頭文件
	* 解決方法：`#pragma once` (目前編譯器都支持這個預編譯指令)
* 若引入的頭文件是 c 語言的，若開發者希望 c++ 開發者也能使用，會在頭文件中增加如下
```c
#ifdef __cplusplus
extern "C" {
#endif

// 一大堆 c 語言內容

#ifdef __cplusplus
}
#endif
```
這樣 c++ 編譯器編譯時候就會帶上。


### 常見的編譯器設定
##### 基本警告選項：
* -Wall：啟用大多數常見且有用的警告。
* -Wextra：啟用額外的警告，這些警告通常與潛在的編碼問題有關。
##### 特定警告選項：
* -Wconversion：警告隱式類型轉換可能導致數據丟失或其他問題。
* -Wfloat-equal：警告浮點數比較（因為浮點數比較可能不可靠）。
* -Wformat：檢查 printf 和 scanf 格式字符串是否正確。
* -Wuninitialized：警告使用未初始化的變量。
* -Wunused：警告未使用的變量、函數和參數。
* -Wcast-align：警告類型轉換可能導致內存對齊問題。
* -Wpointer-arith：警告不安全的指針運算。
* -Wstrict-aliasing：檢查可能違反嚴格別名規則的代碼。
* -Wswitch：警告 switch 語句中未處理的 case。
* -Wswitch-enum：警告 switch 語句中未處理的枚舉值。
* -Wredundant-decls：警告重複的聲明。
* -Wsign-compare：警告有符號和無符號值之間的比較。
* -Wmissing-braces：警告初始值設定項中缺少大括號。
* -Wparentheses：警告可疑的括號用法。
* -Wunknown-pragmas：警告未知的 #pragma 指令。
##### 更嚴格的警告選項：
* -pedantic：要求編譯器完全遵循 C++ 標準，並警告標準之外的所有代碼。
* -Werror：將所有警告視為錯誤，這樣任何警告都會阻止編譯成功。
* -Wstrict-overflow：警告編譯器假設的算術溢出行為。
* -Wstrict-prototypes：在 C 中警告函數原型中缺少參數類型（對於 C++ 不適用）。
##### 地址和未定義行為檢查：
* -fsanitize=address：啟用地址錯誤檢查（例如內存越界訪問）。需要添加 libasan
* -fsanitize=undefined：啟用未定義行為檢查（例如整數溢出、非法類型轉換）。 需要添加 libubsan