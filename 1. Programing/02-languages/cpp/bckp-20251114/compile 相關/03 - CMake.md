# 初探
---
CMake 是一個開源、跨平台的自動化建構工具。
> CMake 並不是包管理工具

#### 優點
* 操作透明
* 專注現代 C++
* 跨平台: Windows, Linux, macOS
* 支持主流 IDE
#### 缺點
* 還在成長
* CMake 本身為一個語言，具有學習成本

#### C/C++ 編譯流程
1. 預處理 (`-E` 參數): 進行宏替換等工作
2. 編譯 (`-S` 參數): 透過 gcc/msvc/clang 將 C++ 文件轉為匯編語言
3. 匯編 (`-C` 參數): 生成二進制文件，linux (`.o`) windows (`.obj`)
4. 鏈接: 將多個二進制文件鏈接成一個可執行文件

#### 在 CMake 下的編譯流程
1. `cmake -B build`: 稱成構建文件，`-B` 用來指定 build 輸出目錄
2. `cmake --build build`: 執行構建文件，`--build` 指定 build 目錄
> `cmake -B build -G Ninja -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++`

# 語法
---
### 1. 打印
```cmake
# 單行
message("hello")
message(hello) # 可以不加上雙引號

# 多行
# 方式一
message("hello
world
")
# 方式二
message([[hello
world
]])

# 打印變量值
message(${CMAKE_VERSION})
```
### 2. 變量操作
變量分為兩種:
1. CMake 提供
2. 自訂義變量

#### set 方法
```cmake
set(<variable> <value> [PARENT_SCOPE])
```
* 範例
```cmake
# 設定單個值
# 1.
set(Var1 XXX)
message(${Var1})

# 2. 
set([[My Var]] ZZZ)  # 雙引號也可以
message(${My\ Var})

# 設定多個值
set(LISTVALUE a1 a2)
message(${LISTVALUE}) # 輸出為: a1a2，打印的時候不會有間隔

# 環境變數
# 打印
message($ENV{PATH})
# 設定
set(ENV{CXX} "g++")


# 刪除變量
unset(Var1)
unset(ENV{CXX})
```


#### list 方法
```cmake
lsit(APPEND <list> [<element>...]) # 列表中添加元素
list(REMOVE_ITEM <list> <value> [value...]) # 刪除元素
list(LENGTH <list> <out-var>) # 獲取列表元素個數
list(FIND <list> <value> <out-var>) # 查找元素索引並返回
list(INSERT <list> <index> [<element>...]) # 在指定位置插入
list(REVERSE <list>) # 反轉
list(SORT <list> [...]) # 排序
```
* 範例
```cmake
# 創建 list
set(LISTVALUE a1 a2 a3) # 方式一
list(APPEND port p1 p2 p3) # 方式二

# 獲取長度
list(LENGTH LISTVALUE len)
message(${len})
```
### 3. 流程控制
#### if 
```cmake
if(<condition>)
	<commands>
elseif(<condition>)
	<commands>
else()
	<commands>
endif
```
* 範例
```cmake
set(VARBOOL TRUE)

if (NOT VARBOOL OR VARBOOL)
	message(TRUE)
else()
	message(FALSE)
endif()

if(1 LESS 2)
	message("1 LESS 2")
endif()
```

#### loop
```cmake
# 第一種
foreach(<loop_var> RANGE <max>)
	<commands>
endforeach()

# 第二種
foreach(<loop_var> RANGE <min> <max> [<step>])
	<commands>
endforeach()

# 第三種
foreach(<loop_var> IN [LISTS <lists>] [ITEMS <items>])
	<commands>
endforeach()

# 第四種
foreach(<loop_var> IN ZIP_LISTS L1 L2)
	<commands>
endforeach()
```

### 4. 函數
```cmake
funcion(<name> [<argument>...])
	<commands>
endfunction()	
```
* 範例
```cmake
# 定義
function(MyFunc FirstArg)
	message("My Func: ${CMAKE_CURRENT_FUNCTION}")
	message("FirstArg ${FrstArg}")
endfunction()

# 調用
set(Arg "first value")
MyFunc(${Arg})
```

# 構建方式
---
### 1. 直接寫入源代碼路徑
* 項目結構
```text
.
├── animal
│  ├── dog.cc
│  └── dog.hpp
├── CMakeLists.txt
└── main.cc
```

* CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 3.20)
project(Animal CXX)
add_executable(${PROJECT_NAME} main.cc animal/dog.cc) # 直接寫路徑
```
### 2. 調用 cmake 腳本
* 項目結構
```cmake
.
├── animal
│  ├── animal.cmake
│  ├── cat.cc
│  ├── cat.hpp
│  ├── dog.cc
│  └── dog.hpp
├── CMakeLists.txt
└── main.cc
```
* animal/animal.cmake
```cmake
set(animal_sources animal/dog.cc animal/cat.cc)
```
* CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 3.20)
project(Animal CXX)
include(animal/animal.cmake)
message(${animal_sources})
add_executable(${PROJECT_NAME} main.cc ${animal_sources})
```
### 3. CMakeLists 嵌套: 子項目編譯成庫
* 項目結構
```text
.
├── animal
│  ├── cat.cc
│  ├── cat.hpp
│  ├── CMakeLists.txt
│  ├── dog.cc
│  └── dog.hpp
├── CMakeLists.txt
└── main.cc
```

* animal/CMakeLists.txt
```cmake
add_library(AnimalLib STATIC cat.cc dog.cc)
target_link_libraries(AnimalLib PUBLIC .)
```


* CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 3.20)
project(Animal CXX)
add_subdirectory(animal)
add_executable(${PROJECT_NAME} main.cc ${animal_sources})
target_link_libraries(${PROJECT_NAME} PUBLIC AnimalLib)

message("-> Build directory: ${PROJECT_BINARY_DIR}")
# target_include_directories(${PROJECT_NAME} PUBLIC ${PROJECT_BINARY_DIR} ${PROJECT_SOURCE_DIR}/animal) # 這樣 main.cc 就可以直接使用 hpp 文件(但已經包含在 animal/CMakeLists.txt 中)
```
> 會生成 libAnimallib.a 之後再進行鏈接

### 4. CMakeLists 嵌套: 子項目編譯成 Object
這是一種比較新的方式，Object Library 是一個特殊的庫類型，他將目標文件編譯成一個庫，但不會生成最終的鏈結文件，意味著在後續 `add_library` 或 `add_executable` 命令中，將 Object Library 作為源文文件進行鏈接。
> cmake 需 >= 3.12

* animal/CMakeLists.txt
```cmake
add_library(AnimalLib OBJECT cat.cc dog.cc)
target_link_libraries(AnimalLib PUBLIC .)
```

* CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 3.20)
project(Animal CXX)
add_subdirectory(animal)
add_executable(${PROJECT_NAME} main.cc ${animal_sources})
target_link_libraries(${PROJECT_NAME} PUBLIC AnimalLib)
```
> 生成多個 .o 文件，最後再進行鏈接

# 靜態庫與動態庫
---
### Public vs. Private
* **PUBLIC**: 本目標需要使用，且依賴該目標的其他目標也需要
* **INTERFACE**: 本目標不需要使用，且依賴該目標的其他目標也需要
* **PRIVATE**: 本目標需要使用，且依賴該目標的其他目標不需要


### 1. 生成靜態庫與動態庫
* **靜態庫**
在鏈接階段，會將 .o 文件與引用到的庫一起鏈接到打包可執行文件中。(Linux: `lib<name>.a`, Windows: `lib<name>.lib`)
> 本質上只是透過 ar 命令將多個 .o 文件打包在一起形成 archive 文件，ld 會從裡面取出 .o 進行鏈接。
```cmake
cmake_minimum_required(VERSION 3.20)
project(Animal CXX)

message("src dir: ${PROJECT_SOURCE_DIR}/src")
file(GLOB SRC ${PROJECT_SOURCE_DIR}/src/*.cc) # 查找所有 src 下 .cc 文件
include_directories(${PROJECT_SOURCE_DIR}/include)

set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/a) # 設定 LIBRARY 輸出位置
add_library(animal STATIC ${SRC})
```
* **動態庫**
在鏈接階段不會鏈接到目標代碼中，而是運行時動態載入的。 (Linux: `lib<name>.so`, Windows: `lib<name>.dll`)
```cmake
cmake_minimum_required(VERSION 3.20)
project(Animal CXX)

message("src dir: ${PROJECT_SOURCE_DIR}/src")
file(GLOB SRC ${PROJECT_SOURCE_DIR}/src/*.cc) # 查找所有 src 下 .cc 文件
include_directories(${PROJECT_SOURCE_DIR}/include)

set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/a) # 設定 LIBRARY 輸出位置
add_library(animal SHARED ${SRC})
```

### 2. 調用靜態庫與動態庫
* **靜態庫**
	  1. 引入頭文件
	  2. 鏈接靜態庫
	  3. 生成可執行文件
```cmake
cmake_minimum_required(VERSION 3.20)
project(Animal CXX)

set(CMAKE_EXPORT_COMPILE_COMMANDS 1)

add_executable(${PROJECT_NAME} main.cc)
target_include_directories(${PROJECT_NAME} PUBLIC ${PROJECT_SOURCE_DIR}/include)
target_link_directories(${PROJECT_NAME} PUBLIC ${PROJECT_SOURCE_DIR}/a)
target_link_libraries(${PROJECT_NAME} PUBLIC animal)
```

* **動態庫**
	1. 引入頭文件
	2. 聲明庫目錄
	3. 生成可執行文件
	4. 鏈接動態庫(運行時)
```cmake
cmake_minimum_required(VERSION 3.20)
project(Animal CXX)

set(CMAKE_EXPORT_COMPILE_COMMANDS 1)

add_executable(${PROJECT_NAME} main.cc)
target_include_directories(${PROJECT_NAME} PUBLIC ${PROJECT_SOURCE_DIR}/include)
target_link_directories(${PROJECT_NAME} PUBLIC ${PROJECT_SOURCE_DIR}/a)
target_link_libraries(${PROJECT_NAME} PUBLIC animal)
```



# 條件編譯
---
通過傳入不同參數而影響編譯過程。
1. 用 option 定義變量
2. 根據變量是 ON/OFF 來執行不同的編譯過程
3. 執行 cmake 命令時加上 `-D<VAR_NAME>=ON/OFF` 來進行控制
* 語法
```cmake
option(<variable> "<help_text>" [value])
```
* 例子
```cmake
option(USE_CATTWO "Use cat two" ON)
if(USE_CATTWO)
	set(SRC cat.cc dog.cc cattwo.cc)
else()
	set(SRC cat.cc dog.cc)
endif()
```

# 實戰
---
項目架構如下:
```text
your_project/
├── CMakeLists.txt       # 主 CMake 文件
├── include/             # 頭文件目錄
│   └── your_header.h   # 你的頭文件
├── src/                 # 源碼目錄
│   ├── main.cc         # 主要源文件
│   └── other_file.cc   # 其他源文件
├── tests/               # 測試文件目錄
│   └── test_main.cc    # 測試用例
├── build/               # 編譯生成的目錄（cmake -B build）
└── README.md            # 項目說明文件（可選）

```

主要 CMakeLists.txt 文件如下:
```python
cmake_minimum_required(VERSION 3.20)
project(cuda_test LANGUAGES CXX)

# General settings
set(CMAKE_BUILD_TYPE Debug)
set(CMAKE_EXPORT_COMPILE_COMMANDS 1)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# Compiler warnings
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(WARNINGS -Wall -Wextra -pedantic -Werror -Wstrict-overflow)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    set(WARNINGS /W4 /WX)
endif()

# Options
option(BUILD_SHARED_LIBS "Build shared libraries" ON)
set(INC_DIR "" CACHE PATH "Include directory for external libraries")
set(LIB_DIR "" CACHE PATH "Library directory for external libraries")
set(LIBS "" CACHE STRING "Libraries to link against")

# Executable or library
if(BUILD_SHARED_LIBS)
    add_library(${PROJECT_NAME} SHARED main.cc)
else()
    add_library(${PROJECT_NAME} STATIC main.cc)
endif()

# Includes and linking
if(INC_DIR)
    target_include_directories(${PROJECT_NAME} PRIVATE ${INC_DIR})
endif()

if(LIB_DIR)
    target_link_directories(${PROJECT_NAME} PRIVATE ${LIB_DIR})
endif()

if(LIBS)
    target_link_libraries(${PROJECT_NAME} PRIVATE ${LIBS})
endif()

target_compile_options(${PROJECT_NAME} PRIVATE ${WARNINGS})

# Testing
enable_testing()
add_executable(test_${PROJECT_NAME} test_main.cc)
target_link_libraries(test_${PROJECT_NAME} PRIVATE ${PROJECT_NAME})
add_test(NAME TestExample COMMAND test_${PROJECT_NAME})

# Installation
install(TARGETS ${PROJECT_NAME}
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)
install(DIRECTORY include/ DESTINATION include)

# Messages
message(STATUS "CMAKE_BUILD_TYPE: ${CMAKE_BUILD_TYPE}")
message(STATUS "CMAKE_CXX_STANDARD: ${CMAKE_CXX_STANDARD}")
message(STATUS "CMAKE_C_COMPILER: ${CMAKE_C_COMPILER}")
message(STATUS "CMAKE_CXX_COMPILER: ${CMAKE_CXX_COMPILER}")
message(STATUS "INC_DIR: ${INC_DIR}")
message(STATUS "LIB_DIR: ${LIB_DIR}")
message(STATUS "LIBS: ${LIBS}")
```

* 構建方式
```shell
# 建立構建文件
cmake -B build \
	-DCMAKE_BUILD_TYPE=Debug \ 
	-DINC_DIR=/usr/local/cuda/include \ 
	-DLIB_DIR=/usr/local/cuda/lib64 \ 
	-DLIBS="cudart;cublas"
# 執行編譯
cmake --build build
# 執行測試
ctest
# 安裝
cmake --install . --prefix /path/to/install
```

* 構建方式: 使用 Makefile
```python
# 定義變數
BUILD_DIR := build
TARGET := demo-test
CMAKE_GENERATOR := Ninja
CMAKE_C_COMPILER := gcc
CMAKE_CXX_COMPILER := g++

# PHONY 目標，避免與文件同名導致的問題
.PHONY: clean build run objdump debug install test

# 清理所有生成的文件
clean:
	@rm -rf $(BUILD_DIR)
	@echo "Cleaned build directory."

# 配置和編譯項目
build:
	@cmake -B $(BUILD_DIR) -G $(CMAKE_GENERATOR) \
		-DCMAKE_C_COMPILER=$(CMAKE_C_COMPILER) \
		-DCMAKE_CXX_COMPILER=$(CMAKE_CXX_COMPILER)
	@cmake --build $(BUILD_DIR)
	@echo "Build completed."

# 運行生成的可執行文件
run: build
	@$(BUILD_DIR)/$(TARGET)

# 使用 objdump 反匯編
objdump: build
	@objdump -S $(BUILD_DIR)/$(TARGET) | less

# 使用 gdb 調試可執行文件
debug: build
	@gdb --args $(BUILD_DIR)/$(TARGET)

# 安裝生成的可執行文件
install: build
	@cmake --install $(BUILD_DIR) --prefix /usr/local
	@echo "Installed $(TARGET) to /usr/local."

# 清理中間文件
clean-obj:
	@find $(BUILD_DIR) -type f -name '*.o' -delete
	@echo "Cleaned object files."

# 運行測試（假設 CTest 已啟用）
test: build
	@cd $(BUILD_DIR) && ctest --output-on-failure

# 編譯和運行測試
test-run: test

# 打印構建系統的信息
info:
	@cmake -B $(BUILD_DIR) -G $(CMAKE_GENERATOR) -LAH | less
```

# 其他技巧
---
### 遞迴的取得路徑下所有匹配文件，並會自動更新
```cmake
file(GLOB_RECURSE NIX_SRC CONFIGURE_DEPENDS *.cpp)
```
* `CONFIGURE_DEPENDS` 會使每次編譯的時候都會檢查文件樹是否有更新

### 依照 Debug 與 Release 區分 compile option
```cmake
target_compile_options(
    nix # project name
    PRIVATE
        $<$<CONFIG:Debug>:${DEBUG_COMPILE_OPTIONS}>
        $<$<CONFIG:Release>:${RELEASE_COMPILE_OPTIONS}>
)
```