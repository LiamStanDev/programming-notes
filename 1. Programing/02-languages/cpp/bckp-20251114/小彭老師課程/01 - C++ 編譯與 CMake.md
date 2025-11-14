# 什麼編譯器
---
### C++ 代碼本質
```cpp
#include <cstdio>

int main() {
	printf("hello, world!\n");
	return 0;
}
```
* `<cstdio>` 與 `<stdio.h>` 不同的是 cstdio 是包裝在 std 命名空間下的，也就是說可以使用 `std::printf` 
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

# CMake
---
### 構建系統 Makefile
關於 Makefile 的細節這邊就不說了，可以看 [[1. Programing/02-languages/cpp/bckp-20251114/Makefile]]。
而 Makefile 有一些缺點：
1. 只能運行在 Unix/Linux 環境下，windows 下需要配置很多東西
2. 需要自行指定依賴關係
3. 語法過於簡單，比較難做判斷語句
4. 不同編譯器的 flag 規則不同，都要自行編寫

### 構建系統的構建系統 CMake
CMake 改進了所有 Make 所造成的問題，雖然最後還是會生成一個　Makefile (Unix/Linux) 或者 .vcproj (Windows)。

### 編譯可執行文件
```cmake
# 設定最低版本
cmake_minimum_required(VERSION 3.12)
# 設定項目名稱
project(Test02)

# 設定執行檔
add_executable(${PROJECT_NAME} main.cpp ${SOURCES})
```

#### 執行
```shell
cmake -B build # 該命令會建立一個 build 文件夾，並在裡面生成 Makefile

# 與以下相同
mkdir build && cd build && cmake ..
```

##### 指定編譯器
方式一：寫在 CMakeLists.txt 中 (不推薦)，因為別人編譯可能不通過
```cmake
set(CMAKE_C_COMPILER /home/liam/.local/gcc/bin/gcc)
set(CMAKE_CXX_COMPILER /home/liam/.local/gcc/bin/g++)
```

方式二：cmake 的命令行參數
```shell
cmake -DCMAKE_C_COMPILER="/bin/gcc" -DCMAKE_CXX_COMPILER="/bin/g++" -B build
```

方式三：全局變數
```shell
export CC="/bin/gcc"
export CXX="/bin/g++"
```

### 編譯庫文件
```cmake
cmake_minimum_required(VERSION 3.12)

project(hellocmake LANGUAGES CXX)

add_library(hello SHARED hello.cpp) # 會自動添加 lib 前綴, 選項有 STATIC 與 SHARED
add_executable(a.out main.cpp)
target_link_libraries(a.out PUBLIC hello)
```
用 lld 分析 `a.out` 結果如下，
```text
0000000000401126 <main>:
  401126:       55                      push   %rbp
  401127:       48 89 e5                mov    %rsp,%rbp
  40112a:       e8 01 ff ff ff          call   401030 <_Z5hellov@plt>
  40112f:       b8 00 00 00 00          mov    $0x0,%eax
  401134:       5d                      pop    %rbp
  401135:       c3                      ret
```
跳轉到 `@plt` 插樁函數位置。

### 子模塊
子模括需要解決幾個問題：
1. 頭文件默認搜索當前目錄，需要設置頭文件位置
2. 子模塊有自己編譯的規則
3. (optional) clangd 在做 LSP 的時候需要知道頭文件位置

根目錄下 CMakeLists.txt 改為
```cmake
cmake_minimum_required(VERSION 3.12)

project(hellocmake LANGUAGES CXX)

set(CMAKE_EXPORT_COMPILE_COMMANDS 1) # 會生成 compile_commands.json 文件，用於給 clangd 使用

add_subdirectory(hello) # 添加子模塊目錄

add_executable(a.out main.cpp)
target_link_libraries(a.out PUBLIC hello)
# 這邊去掉了 target_include_directories 放到子模組中
```

子目錄下的 CMakeLists.txt 添加
```cmake
add_library(hello SHARED hello.cpp)
# 子模組添加 target_include_directories 並用 . 表示當前目錄的所有頭文件，
# 只要引用的文件中使用到 hello 庫，e.g. target_link_libraries(a.out PUBLIC hello)
# 就會自動添加頭文件
target_include_directories(hello PUBLIC .)
```

##### PUBLIC vs PRIVATE
PUBLIC 表示當 Link 該庫的時候，要不要將內容傳播給使用者，如上面的例子 hello.o 被 link 了 hello.h，這樣使用 target_link_libraries 的 a.out 也能使用 hello.o 傳播給他的 hello.h 頭文件。PRIVATE 則是相反。

> 簡單來說就是是否讓引用者能看到，也就是父模塊可見性

##### 其他可能會用到的 cmake 函數
```cmake
target_include_directories(myapp PUBLIC /usr/includes/eigen3) # 添加頭文件搜索目錄
target_link_libraries(myapp PUBLIC hello) # 添加鏈接庫
target_add_definitions(myapp PUBLIC MY_MACRO=1) # 添加宏定義
target_compile_options(myapp PUBLIC -opeionmp) # 添加編譯器命令行選項
target_sources(myapp PUBLIC hello.cpp other.cpp) # 添加編譯源文件

# 以下沒有 target 版本的會用在所有目標文件中 (不推薦)
include_directories(/usr/includes/eigen3) # 添加頭文件搜索目錄
link_libraries(hello) # 添加鏈接庫
add_definitions(MY_MACRO=1) # 添加宏定義
compile_options(-opeionmp) # 添加編譯器命令行選項
```

##### 第三方庫
1. 直接 git clone 下來後作為子模塊編譯
2. 系統預安裝庫 e.g. 使用 dnf 下載的
```cmake
find_package(fmt REQUIRED)
target_link_libraries(myexec PUBLIC fmt::fmt)
```
> 為什麼是 fmt::fmt，前面是包名稱，後面是庫 or 組件名稱

### 編寫 Makefile 來執行 cmake
這邊用來在項目根目錄中更方便的編譯與執行
```python
# 編譯
default_target : env
	@cmake -B build
	@cd build && make && ./a.out
.PHONY : default_target

# 配置 clangd 需要的文件
env : 
	@ln -sf build/compile_commands.json .
.PHONY : env

# 反彙編
disasm : 
	@cd build && objdump -d a.out | less
.PHONY : disasm
```

# 實戰

### 我喜歡的寫法
```python
cmake_minimum_required(VERSION 3.12)
project(cuda_test LANGUAGES CXX)

set(CMAKE_BUILD_TYPE Debug)
set(CMAKE_EXPORT_COMPILE_COMMANDS true)
# set(CMAKE_INSTALL_PREFIX "/opt/myapp") 

# Compiler (This should avoid, it's not for other to build)
# you can use set environment variable (CC, CXX).
set(CMAKE_C_COMPILER /home/liam/.local/gcc/bin/gcc)
set(CMAKE_CXX_COMPILER /home/liam/.local/gcc/bin/g++)
set(CMAKE_CUDA_HOST_COMPILER /usr/local/cuda/bin/nvcc)

# RPath
set(CMAKE_BUILD_RPATH "/home/liam/.local/gcc/lib64;/home/liam/.local/gcc/lib") # 用於 Debug 時
set(CMAKE_INSTALL_RPATH "/opt/lib") # 用於 Release 時候

# C++ Standard
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED true)

# Variables
set(INC_DIR /usr/local/cuda/include)
set(LIB_DIR /usr/local/cuda/lib64)
set(LIBS cudart cublas)

# Executable
add_executable(${PROJECT_NAME} main.cc)

# Includs and link
target_include_directories(${PROJECT_NAME} PRIVATE ${INC_DIR})
target_link_directories(${PROJECT_NAME} PRIVATE ${LIB_DIR})
target_link_libraries(${PROJECT_NAME} PRIVATE ${LIBS})

# Message
message(STATUS "CMAKE_C_COMPILER: ${CMAKE_C_COMPILER}")
message(STATUS "CMAKE_CXX_COMPILER: ${CMAKE_CXX_COMPILER}")
message(STATUS "CMAKE_CUDA_HOST_COMPILER: ${CMAKE_CUDA_HOST_COMPILER}")
message(STATUS "INC_DIR: ${INC_DIR}")
message(STATUS "LIB_DIR: ${LIB_DIR}")
message(STATUS "LIBS: ${LIBS}")
```
##### INSTALL_PREFIX
表示運行 `make install` 會安裝的路徑，默認位置：
1. 可執行文件： `/usr/local/bin`
2. 頭文件：`/usr/local/lib`
3. 共享庫：`/usr/local/include`

##### RPATH
表示 runtime path，可以指定相對路徑與絕對路徑，與 -L 指定路徑不一樣， -L 指定的是編譯時的路徑，而實際運行時還是會尋找 [[#靜態鏈接 vs. 與動態鏈接庫|詳細資訊請看這邊]] ，而 RPATH 會寫入 ELF 文件，具有最高的優先權。
* `BUILD_RPATH`: 表示在 build (`make`) 後會用的 rpath
* `INSTALL_RPATH`: 表示在 install (`make install`) 後會用的 rpath

### Qt 項目
```cmake
#====================設定項目信息====================
# 設定最低版本
cmake_minimum_required(VERSION 3.12)
# 設定項目名稱
project(Test02)

#====================設定基礎環境====================
set(CMAKE_CXX_STANDARD 17)
set (CMAKE_PREFIX_PATH "/opt/homebrew/Cellar/qt/6.3.1")
set(CMAKE_AUTOMOC ON) # 自動使用宏
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON) # 自動使用UI
set(CMAKE_AUTOUIC_SEARCH_PATHS ./ui) # 指定ui的位置不然要與src放在一起

#====================搜尋項目的庫====================
# 搜尋要用的庫，但先要將其放入環境變量裡
# 教學網址：https://blog.csdn.net/wsepom/article/details/122076768
find_package(Qt6 COMPONENTS Core Gui Widgets REQUIRED)

#===================設定項目目錄結構==================
# 加入head, src, ui
set(INC_DIR ./include)
set(SRC_DIR ./src)
set(UI_DIR ./ui)
set(RESOURCE_DIR ./resource)

# 下面已經不建議使用了
# 若只有INC_DIR有head file，那就只要寫下面的第一個就好
#include_directories(${INC_DIR})
#include_directories(${SRC_DIR})
#include_directories(${UI_DIR})

# 將文件裡面的內容都遍歷進去
file(GLOB_RECURSE SOURCES
        "${UI_DIR}/*.ui"
        "${RESOURCE_DIR}/*.qrc"
        "${INC_DIR}/*.h"
        "${SRC_DIR}/*.cpp")

#====================設定執行檔====================
add_executable(${PROJECT_NAME} main.cpp ${SOURCES})

#====================設定頭文件====================
#target_include_directories(Test02 PRIVATE ${UI_DIR}) #若裡面有head file要加
#target_include_directories(Test02 PRIVATE ${SRC_DIR}) #若裡面有head file要加
target_include_directories(Test02 PRIVATE ${INC_DIR})

#====================設定連接庫====================
# Link libraries
target_link_libraries(Test02 Qt::Core Qt::Gui Qt::Widgets)
```

### CUDA cublas 項目
```cmake
cmake_minimum_required(VERSION 3.12)
project(cuda_test)

set(CMAKE_BUILD_TYPE Debug)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_EXPORT_COMPILE_COMMANDS 1)

find_package(CUDAToolkit REQUIRED)

add_executable(${PROJECT_NAME} main.cc)

target_include_directories(${PROJECT_NAME} PUBLIC ${CUDA_TOOLKIT_INCLUDE} )
target_link_libraries(${PROJECT_NAME} PUBLIC CUDA::cudart CUDA::cublas)
```

