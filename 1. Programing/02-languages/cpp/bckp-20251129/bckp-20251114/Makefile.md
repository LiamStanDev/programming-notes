 # 基本
---
### 格式
```make
target : prerequisties
		command
```
- target : 目標文件，也可以是一個標籤
- prerequisties: 為生成target所要依賴的文件
- command: make需要執行的命令，前面加上`@` 可以隱藏執行的命令打印到terminal
```makefile
debug :
	@echo hello
```
    
### 規則
1. 文件名稱必須為`Makefile` 或者 `makefile`
2. 若只輸入`make` 會執行文件中第一個target
3. target 依賴的 prerequisties 若不存在，就會繼續找尋生成 prerequisties 的 target
4. 若再次執行 make 指令時，會先看 prerequisties 是否有改動，有改動才會再次編譯。
    → 不會重複編譯未改動的文件
    
### 偽目標
偽目標不是一個文件而是一個標籤，我們要顯示的指名這個”目標”才能讓其生效
- 偽目標名稱不能與文件名重名，否則不會執行
- 避免重名可以使用`.PHONY` 來顯示指名一個目標為偽目標
```makefile
.PHONY : clean
clean : 
		@rm -rf ./build
```

> 建議標籤都寫上`.PHONY`

# 變量
---
### 變量定義
```makefile
cpp := src/main.cpp
obj := objs/main.o
```
- 在`:=` 之後的內容會直接當成字符串，立即賦值

```makefile
cpp := src/main.cpp
obj := objs/main.o

$(obj) : $(cpp) 
	@g++ -c $(cpp) -o $(obj)
```

### 預定義變量
- `$@` : target 的完整名稱
- `$<` : 第一個依賴文件名稱
- `$^` : 所有依賴文件，以空格分開，不包含重複的依賴
```makefile
cpp := src/main.cpp
obj := objs/main.o

$(obj) : $(cpp) 
	@g++ -c $^ -o $@

debug : 
	@echo $(cpp)
	@echo $(obj)

clean : 
	@rm -rf ./objs/*.o
	

.PHONY : debug clean
```

### 不同的等號
1. `=`
- 不會立即求值 (傳引用)
- 可以賦值給變量
```makefile
HOST_ARCH = aarch64
TARGET_ARCH = $(HOST_ARCH)

HOST_ARCH = amd64

debug :
	@echo $(TARGET_ARCH)

.PHONY : debug
```

```makefile
結果為：amd64
```

2. `:=`
- 立即求值 (傳值 copy)
- 賦給變量之後，就無法在給變量重新賦值
    ```makefile
    HOST_ARCH = aarch64
    TARGET_ARCH := $(HOST_ARCH)
    
    HOST_ARCH = amd64
    
    debug :
    	@echo $(TARGET_ARCH)
    
    .PHONY : debug
    ```
    
    ```makefile
    結果為：aarch64
    ```
    
    <aside> 💡 基本上優先使用`:=` ，如有需求的話才會使用 `=`
    
    </aside>
    
3. `?=`
    - 默認賦值
    - 若已經定義，就不會做任何事情
4. `+=`
    - 在後面添加，並自動添加空格
```makefile
src := ./src
CXXFLAG := -m64 -Wall -c
CXXFLAG += $(src)

debug : 
	@echo $(CXXFLAG)

.PHONY : debug
```

```makefile
結果為：-m64 -Wall -c ./src
```
    

## 函數
---
### 基本語法
```makefile
$(fn, arguments) or ${fn, arguments}
```

### 常用函數
1. `shell`
- 使用shell命令
```makefile
cpp_srcs := $(shell find ./src -name "*.cpp")
# cpp_srcs := $(wildcard ./src/*.cpp)

debug : 
	@echo $(cpp_srcs)

.PHONY : debug
```
    
2. `subst` : substitute string
- 字符串替換
- 用法：`$(subst <from>,<to>,<test>)`
- 把`<text>`中的`<from>`字符串換成`<to>`
    ```makefile
    cpp_srcs := $(shell find ./src -name "*.cpp")
    # 將./src/換成./objs/
    cpp_objs := $(subst ./src/,./objs/,$(cpp_srcs))
    # 將.cpp換成.o 
    cpp_objs := $(subst .cpp,.o,$(cpp_objs))
    debug : 
    	@echo $(cpp_srcs)
    	@echo $(cpp_objs)
    
    .PHONY : debug
    ```
    
3. `patsubst` : pattern substitute string
- 用法：`$(patsubst <pattern>,<replacement>,<text>)`
- 使用`%`表示任意長度的字符串
- 從`<text>`取出`<pattern>`替換成`<replacement>`
```makefile
cpp_srcs := $(shell find ./src -name "*.cpp")
cpp_objs := $(patsubst %.cpp,%.o,$(cpp_srcs))

debug : 
	@echo $(cpp_srcs)
	@echo $(cpp_objs)

.PHONY : debug
```
    
4. `foreach`
- 用法：`$(foreach <var>,<list>,<text>)`
- 把字符串`<list>`中的元素逐一取出來，執行`<text>`包含的表達式
- 返回：`<text>`所返回的每個字符串所組成的由空個分開的字符串
```makefile
# 我們希望能弄成
# gcc -o *.o -c *.cpp -I/usr/include -I/usr/include/openmpi
include_paths := /usr/include /usr/include/openmpi 

# foreach 方式
# I_flag := $(foreach item,$(include_paths),-I$(item))

# 更簡潔方式
I_flag := $(include_paths:%=-I%d)

debug : 
	@echo $(I_flag)

.PHONY : debug
```
    
5. `dir`
- 功能：取得文件的父級目錄
```makefile
objs/%.o : src/%.cpp
	@mkdir -p $(dir $@) # 取得objs
	@g++ -c $^ -o $@ $(cxx_flag)
```
    
6. `notdir`
- 功能：去掉所有文件的路徑，只留下文件名稱
- `$(notdir <names…>)`

7. `filter`
- 功能：過濾文件
- `$(filter <pattern>,<names…>)`

8. `basename`
- 功能：去掉文件後綴
- `$(basename <names…>)`
    
```makefile
libs := $(shell find /usr/lib -name "lib*")
libs := $(notdir $(libs))
# 取得靜態庫庫名
a_libs := $(filter %.a,$(libs)) # 過濾.a
a_libs := $(basename $(a_libs)) # 取得名子
a_libs := $(subst lib,,$(a_libs)) # 去掉lib開頭
# 取得動態庫庫名
so_libs := $(filter %.so,$(libs)) # 過濾.so
so_libs := $(basename $(so_libs)) # 取得名子
so_libs := $(subst lib,,$(so_libs)) # 去掉lib開頭

debug : 
	@echo $(a_libs)
	@echo $(so_libs)

.PHONY : debug
```
    

## 實戰
---
### 簡單的編譯、鏈接與運行

```cpp
CPP_SRCS := $(shell find src -name "*.cpp")
CPP_OBJS := $(patsubst src/%.cpp,build/objs/%.o,$(CPP_SRCS))

build/objs/%.o : src/%.cpp
	@mkdir -p $(dir $@)
	@g++ -c $< -o $@

build/bin/exec : $(CPP_OBJS)
	@mkdir -p $(dir $@)
	@g++ $^ -o $@

run : build/bin/exec
	@./$<

debug : 
	@echo $(CPP_SRCS)
	@echo $(CPP_OBJS)

clean : 
	@rm -v -rf objs

.PHONY : debug clean run
```

### 編譯選項

- `-m64` : 編譯成64位
- `-std=` : 指定編譯標準 e.g. `-std=c++11`, `-std=c++14`
- `-g` : 包含調適訊息
- `-w` : 不顯示警告
- `-O` : 優化等級, e.g. `-O3`
- `-I` : 加**頭文件路徑前**
- `fPIC` : Position-independent Code，產生沒有絕對地址，全部使用相對地址，代碼可以被加載到內存的任意位置，且可以正確被運行。這是**共享庫所要求的**。

### 鏈接選項

- `-l` : 加在**庫名稱前**，靜態庫、動態庫都要使用，需要去掉lib開頭
- `-L` : 加在**庫路徑前**
- `-Wl,<options>` : 將逗號分個的`<option>`傳遞給鏈接器
- `-rpath=` : “運行”的時候，去找的目錄，運行的時候要找`.so`文件，會從這個選項裡面指定的地方去找。

### 約定俗成的變量名稱
- `CC` : 指定c的編譯器
- `CXX` : 指定c++的編譯器
- `CFLAGS` : c編譯器的編譯選項
- `CXXFLAGS` : c++編譯器的編譯選項
- `CPPFLAG` : 預處理選項
- `LDFLAGS` : 鏈接器選項

### 編譯帶頭文件的程序
```makefile
CPP_SRCS := $(shell find src -name "*.cpp")
CPP_OBJS := $(patsubst src/%.cpp,build/objs/%.o,$(CPP_SRCS))
PROJECT_NAME := mini
INCLUDE_PATHS := include

I_FLAGS := $(INCLUDE_PATHS:%=-I%) 
CXX_FLAGS := -g -O3 $(I_FLAGS)

build/objs/%.o : src/%.cpp
	@mkdir -p $(dir $@)
	@g++ -c $< -o $@ $(CXX_FLAGS)

build/bin/$(PROJECT_NAME) : $(CPP_OBJS)
	@mkdir -p $(dir $@)
	@g++ $^ -o $@

clean : 
	@rm -rf build

run : build/bin/$(PROJECT_NAME)
	@./$<

debug : 
	@echo $(CPP_SRCS)
	@echo $(CPP_OBJS)
	@echo $(INCLUDE_PATHS)
	@echo $(I_FLAGS)
	@echo $(CXX_FLAGS)

.PHONY : debug clean run
```

### 編譯靜態庫
```makefile
LIB_SRCS := $(shell find src -name "*.cpp")
LIB_SRCS := $(filter-out src/main.cpp,$(LIB_SRCS))
LIB_OBJS := $(patsubst src/%.cpp,build/objs/%.o,$(LIB_SRCS))
PROJECT_NAME := mini

# compile
INCLUDE_PATHS := include
I_FLAGS := $(INCLUDE_PATHS:%=-I%)
CXXFLAGS := -g -O3 -std=c++14 $(I_FLAGS)
# linkage
LIB_PATH := build/lib
LINKING_LIBS := mini
l_OPTIONS := $(LINKING_LIBS:%=-l%)
L_OPTIONS := $(LIB_PATH:%=-L%)
LDFLAGS := $(l_OPTIONS) $(L_OPTIONS)

# =============== compile static lib =================
build/objs/%.o : src/%.cpp
	@mkdir -p $(dir $@)
	@g++ -o $@ -c $<  $(CXXFLAGS)

build/lib/lib$(PROJECT_NAME).a : $(LIB_OBJS)
	@mkdir -p $(dir $@)
	@ar -r $@ $^

static : build/lib/lib$(PROJECT_NAME).a

# =============== linkage ===========================
build/objs/main.o : src/main.cpp
	@mkdir -p $(dir $@)
	@g++ -o $@ -c $^ $(CXXFLAGS)

build/bin/exec : build/objs/main.o
	@mkdir -p $(dir $@)
	@g++ -o $@ $^ $(CXXFLAGS) $(LDFLAGS)

debug :
	@echo $(LIB_SRCS)
	@echo $(LIB_OBJS)
	@echo $(I_FLAGS)

run : build/bin/exec
	@./$<

clean : 
	@rm -rf build

.PHONY : debug clean static run
```