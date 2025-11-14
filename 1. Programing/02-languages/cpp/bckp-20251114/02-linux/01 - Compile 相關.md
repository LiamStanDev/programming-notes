### Compiler (gcc)
#### 編譯選項
```shell
-o: 表示指定輸出文件名稱
-g: 添加 Debug 功能
-O0: (默認)不做優化
-O1: 對 Compile 進行優化，會縮小二進制文件大小
-O2: 提高執行效率
-O3: 更高的優化
-c: 只 Compile 但不 Link
-Wall: 啟用所有警告
```

#### gcc 介紹
* 組成:
	* gcc
	* g++
	* libgcc: c 運行時庫
	* libstdc++: c++ 運行時庫
* 包含軟體:
	* ar: 建立動態庫
	* as: 彙編器
	* gdb: Dubger
	* ld: 鏈接器
	* make

### 靜態庫與動態庫
#### 靜態庫
* 在 Compile 時，會合併進入目標二進制文件中。

#### 動態庫
##### 製作動態庫
```shell
g++ -fPIC -shared -o libpulic.so pulic.cpp
```
* `-fPIC`: (必須)表示生成獨立的庫，使得其可以在內存中任何位置被載入，不受地址空間限制，可以被多個進程共享。
##### 鏈結動態庫
```shell
g++ -o main main.cpp -L/home/liam/shared -lpublic
```
* 系統默認尋找的動態連接庫位置為 `/usr/lib` 下，一般安裝程序的動態庫會放在這，找不到會尋找 `LD_LIBRARY_PATH` 環境變量中。需要將動態庫路徑添加到這個環境變量中。

### CMake
#### 工程目錄
```shell
src: 源文件
include: 頭文件
lib: 靜態庫與動態庫
build: 構建文件
CMakeLists.txt
README
```

#### 實戰
```cmake
#====================設定項目信息====================
# 設定最低版本
cmake_minimum_required(VERSION 3.21)
# 項目名稱
project(Test) 

#====================設定環境變數====================
# 設定 C++ 標準 
set(CMAKE_CXX_STANDARD 20)
# 設定項目目錄結構
set(INC_DIR include)
# 設定輸出目錄
set(SRC_DIR src)
# 添加動態庫路徑
set(CMAKE_PREFIX_PATH 
				"/path/to/qt/installation1" 
				"/path/to/qt/installation2" 
				"/path/to/qt/installation3"
)

#====================設定執行檔====================
add_executable(${PROJECT_NAME} main.cpp)

#====================設定頭文件==================== 
target_include_directories(${PROJECT_NAME} PRIVATE ${INC_DIR})

#====================設定源文件==================== 
# 以下也可以改成逐個文件添加
file(GLOB SRC_FILES "${SRC_DIR}/*.cpp")
target_sources(${PROJECT_NAME} PRIVATE
		${SRC_FILES}
)

#====================設定連接庫====================
# 查找包，需放入 CMAKE_PREFIX_PATH 中
find_package(Qt6 COMPONENTS Core Gui Widgets REQUIRED)
target_link_libraries(${PROJECT_NAME} Qt::Core Qt::Gui Qt::Widgets)
```

### 執行
```shell
cd build
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
cmake --build . # or make
```
* `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` 會生成 compile_commands.json 文件，請移到項目跟目錄，使得 clangd 能找到頭文件。