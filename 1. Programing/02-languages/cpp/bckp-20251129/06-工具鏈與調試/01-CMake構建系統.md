# CMake 構建系統

CMake 是跨平台的構建系統生成工具,現代 C++ 專案的首選構建工具。

---

## 1. 現代 CMake 基礎

### 1.1 最小 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(HFTSystem VERSION 1.0.0 LANGUAGES CXX)

# 設定 C++ 標準
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 添加執行檔
add_executable(hft_engine main.cpp)
```

### 1.2 Target-Based 現代寫法

```cmake
# 建立函式庫 target
add_library(market_data STATIC
    src/market_data.cpp
    src/order_book.cpp
)

# 設定 include 目錄 (PUBLIC: 傳遞給依賴者)
target_include_directories(market_data
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/src
)

# 連結函式庫
target_link_libraries(market_data
    PUBLIC
        pthread
    PRIVATE
        fmt::fmt
)

# 編譯選項
target_compile_options(market_data
    PRIVATE
        $<$<CXX_COMPILER_ID:GNU>:-Wall -Wextra -O3 -march=native>
        $<$<CXX_COMPILER_ID:Clang>:-Wall -Wextra -O3 -march=native>
)
```

---

## 2. 專案結構最佳實踐

### 2.1 推薦目錄結構

```
hft_system/
├── CMakeLists.txt
├── cmake/
│   ├── CompilerWarnings.cmake
│   └── FindSomething.cmake
├── include/
│   └── hft/
│       ├── market_data.hpp
│       └── trading_engine.hpp
├── src/
│   ├── market_data.cpp
│   └── trading_engine.cpp
├── tests/
│   ├── CMakeLists.txt
│   └── test_market_data.cpp
├── benchmarks/
│   ├── CMakeLists.txt
│   └── bench_order_book.cpp
└── external/
    └── (第三方庫)
```

### 2.2 頂層 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(HFTSystem
    VERSION 1.0.0
    DESCRIPTION "High Frequency Trading System"
    LANGUAGES CXX
)

# 選項
option(BUILD_TESTS "Build tests" ON)
option(BUILD_BENCHMARKS "Build benchmarks" ON)
option(ENABLE_LTO "Enable Link-Time Optimization" ON)

# C++ 標準
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 輸出目錄
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# 包含 cmake 模組
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# 子目錄
add_subdirectory(src)

if(BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()

if(BUILD_BENCHMARKS)
    add_subdirectory(benchmarks)
endif()
```

---

## 3. 第三方庫整合

### 3.1 使用 find_package

```cmake
# 尋找已安裝的套件
find_package(Boost 1.75 REQUIRED COMPONENTS system thread)
find_package(OpenSSL REQUIRED)

target_link_libraries(hft_engine
    PRIVATE
        Boost::system
        Boost::thread
        OpenSSL::SSL
)
```

### 3.2 使用 FetchContent (推薦)

```cmake
include(FetchContent)

# 獲取 fmt 庫
FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 9.1.0
)

# 獲取 Google Benchmark
FetchContent_Declare(
    benchmark
    GIT_REPOSITORY https://github.com/google/benchmark.git
    GIT_TAG v1.8.0
)

FetchContent_MakeAvailable(fmt benchmark)

# 使用
target_link_libraries(hft_engine PRIVATE fmt::fmt)
target_link_libraries(bench_app PRIVATE benchmark::benchmark)
```

### 3.3 使用 ExternalProject

```cmake
include(ExternalProject)

ExternalProject_Add(
    jemalloc
    URL https://github.com/jemalloc/jemalloc/releases/download/5.3.0/jemalloc-5.3.0.tar.bz2
    CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=<INSTALL_DIR>
    BUILD_COMMAND make
    INSTALL_COMMAND make install
)
```

---

## 4. 編譯選項與優化

### 4.1 編譯器警告

```cmake
# cmake/CompilerWarnings.cmake
function(set_project_warnings target)
    set(MSVC_WARNINGS
        /W4
        /permissive-
    )
    
    set(GCC_CLANG_WARNINGS
        -Wall
        -Wextra
        -Wpedantic
        -Wshadow
        -Wnon-virtual-dtor
        -Wold-style-cast
        -Wcast-align
        -Wunused
        -Woverloaded-virtual
        -Wconversion
        -Wsign-conversion
        -Wformat=2
    )
    
    if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
        target_compile_options(${target} PRIVATE ${GCC_CLANG_WARNINGS})
    elseif(MSVC)
        target_compile_options(${target} PRIVATE ${MSVC_WARNINGS})
    endif()
endfunction()
```

### 4.2 優化選項

```cmake
# Release 模式優化
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    target_compile_options(hft_engine PRIVATE
        -O3
        -march=native
        -mtune=native
        -flto
        -ffast-math
        -funroll-loops
    )
    
    # LTO
    if(ENABLE_LTO)
        set_target_properties(hft_engine PROPERTIES
            INTERPROCEDURAL_OPTIMIZATION TRUE
        )
    endif()
endif()
```

---

## 5. 建置類型配置

```cmake
# 定義建置類型
set(CMAKE_CONFIGURATION_TYPES "Debug;Release;RelWithDebInfo;MinSizeRel" CACHE STRING "" FORCE)

# Debug 配置
set(CMAKE_CXX_FLAGS_DEBUG "-g -O0 -DDEBUG")

# Release 配置
set(CMAKE_CXX_FLAGS_RELEASE "-O3 -DNDEBUG -march=native")

# RelWithDebInfo 配置 (推薦用於 profiling)
set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O2 -g -DNDEBUG")
```

---

## 6. 實戰範例: 完整 HFT 專案

```cmake
# 頂層 CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(HFTSystem VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 選項
option(BUILD_TESTS "Build tests" ON)
option(BUILD_BENCHMARKS "Build benchmarks" ON)
option(ENABLE_SANITIZERS "Enable sanitizers" OFF)
option(ENABLE_LTO "Enable LTO" ON)

# 第三方庫
include(FetchContent)

FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 9.1.0
)
FetchContent_Declare(spdlog
    GIT_REPOSITORY https://github.com/gabime/spdlog.git
    GIT_TAG v1.11.0
)
FetchContent_MakeAvailable(fmt spdlog)

# 主要函式庫
add_library(hft_core STATIC
    src/market_data/feed_handler.cpp
    src/market_data/order_book.cpp
    src/trading/engine.cpp
    src/trading/strategy.cpp
    src/network/udp_receiver.cpp
    src/utils/logger.cpp
)

target_include_directories(hft_core
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/src
)

target_link_libraries(hft_core
    PUBLIC
        fmt::fmt
        spdlog::spdlog
    PRIVATE
        pthread
)

# 執行檔
add_executable(hft_main src/main.cpp)
target_link_libraries(hft_main PRIVATE hft_core)

# Sanitizers
if(ENABLE_SANITIZERS)
    target_compile_options(hft_core PRIVATE
        -fsanitize=address,undefined
        -fno-omit-frame-pointer
    )
    target_link_options(hft_core PRIVATE
        -fsanitize=address,undefined
    )
endif()

# 測試
if(BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()
```

---

## 參考資料

1. **官方文件**
   - [CMake Documentation](https://cmake.org/documentation/)
   - [Modern CMake](https://cliutils.gitlab.io/modern-cmake/)

2. **書籍**
   - "Professional CMake: A Practical Guide" (Craig Scott, 2021)
