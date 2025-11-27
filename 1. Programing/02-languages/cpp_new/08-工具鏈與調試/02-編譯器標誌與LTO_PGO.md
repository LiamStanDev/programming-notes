# 編譯器標誌與LTO/PGO - 極致性能優化 ⭐⭐

## 概述

編譯器標誌優化、鏈接時間優化 (Link Time Optimization, LTO) 與配置文件引導優化 (Profile-Guided Optimization, PGO) 是現代高性能 C++ 開發的核心技術。這些技術可以為 HFT 系統帶來 10-30% 的性能提升。

```cpp
// 展示編譯優化效果的微基準測試
#include <chrono>
#include <vector>
#include <numeric>

class PerformanceTest {
private:
    std::vector<double> data_;
    
public:
    PerformanceTest(size_t size) : data_(size) {
        std::iota(data_.begin(), data_.end(), 1.0);
    }
    
    // 向量化友好的計算
    double compute_sum() const {
        double sum = 0.0;
        for (size_t i = 0; i < data_.size(); ++i) {
            sum += data_[i] * data_[i];
        }
        return sum;
    }
    
    // 分支密集的計算
    double compute_conditional() const {
        double result = 0.0;
        for (const auto& val : data_) {
            if (val > 500.0) {
                result += val * 1.5;
            } else if (val > 100.0) {
                result += val * 1.2;
            } else {
                result += val;
            }
        }
        return result;
    }
};
```

## 編譯器標誌詳解

### 基礎優化標誌

```bash
# 基本優化級別
-O0  # 無優化，保持調試信息
-O1  # 基本優化，平衡編譯時間與性能
-O2  # 推薦的生產級別優化
-O3  # 激進優化，可能增加代碼大小
-Os  # 優化代碼大小
-Ofast # 最激進優化，可能違反標準
```

### 高頻交易專用標誌

```cmake
# HFT 專用編譯配置
set(HFT_FLAGS
    # 核心優化
    "-O3"                           # 最高級別優化
    "-DNDEBUG"                      # 禁用 assert
    "-march=native"                 # 針對目標CPU優化
    "-mtune=native"                 # 調優目標CPU
    
    # 向量化優化
    "-ftree-vectorize"              # 開啟向量化
    "-fvect-cost-model=unlimited"   # 激進向量化
    "-mavx2"                        # 使用 AVX2 指令集
    "-mfma"                         # 使用融合乘加指令
    
    # 分支預測優化
    "-fbranch-probabilities"        # 分支概率信息
    "-ftracer"                      # 追蹤分支執行
    "-funroll-loops"                # 循環展開
    "-fprefetch-loop-arrays"        # 數組預取
    
    # 內聯優化
    "-finline-functions"            # 激進內聯
    "-finline-limit=1000"           # 增加內聯限制
    "-fno-plt"                      # 禁用 PLT
    
    # 浮點數優化
    "-ffast-math"                   # 快速數學運算
    "-ffinite-math-only"           # 假設有限數值
    "-fno-signed-zeros"            # 忽略符號零
    "-fno-trapping-math"           # 禁用陷阱
)

# 調試版本標誌
set(DEBUG_FLAGS
    "-O0"           # 無優化
    "-g3"           # 最詳細調試信息
    "-fno-omit-frame-pointer"  # 保留幀指針
    "-fsanitize=address"       # 地址檢查器
    "-fsanitize=undefined"     # 未定義行為檢查
    "-fstack-protector-strong" # 棧保護
)
```

### 目標平台優化

```cpp
// 檢測CPU特性的運行時代碼
#include <immintrin.h>
#include <cpuid.h>

class CPUFeatureDetector {
public:
    struct Features {
        bool avx2;
        bool fma;
        bool avx512f;
        bool bmi2;
    };
    
    static Features detect() {
        Features features{};
        
        unsigned int eax, ebx, ecx, edx;
        
        // 檢查 AVX2 和 FMA
        if (__get_cpuid_count(7, 0, &eax, &ebx, &ecx, &edx)) {
            features.avx2 = (ebx & bit_AVX2) != 0;
            features.bmi2 = (ebx & bit_BMI2) != 0;
        }
        
        if (__get_cpuid(1, &eax, &ebx, &ecx, &edx)) {
            features.fma = (ecx & bit_FMA) != 0;
        }
        
        // 檢查 AVX-512
        if (__get_cpuid_count(7, 0, &eax, &ebx, &ecx, &edx)) {
            features.avx512f = (ebx & bit_AVX512F) != 0;
        }
        
        return features;
    }
    
    // 根據CPU特性選擇算法
    template<typename T>
    static T vector_sum(const std::vector<T>& data) {
        auto features = detect();
        
        if (features.avx512f) {
            return vector_sum_avx512(data);
        } else if (features.avx2) {
            return vector_sum_avx2(data);
        } else {
            return vector_sum_scalar(data);
        }
    }
    
private:
    template<typename T>
    static T vector_sum_avx512(const std::vector<T>& data);
    
    template<typename T>
    static T vector_sum_avx2(const std::vector<T>& data);
    
    template<typename T>
    static T vector_sum_scalar(const std::vector<T>& data) {
        return std::accumulate(data.begin(), data.end(), T{});
    }
};
```

## 鏈接時間優化 (LTO)

### LTO 基礎配置

```cmake
# 啟用 LTO 的 CMake 配置
option(ENABLE_LTO "Enable Link Time Optimization" ON)

if(ENABLE_LTO)
    include(CheckIPOSupported)
    check_ipo_supported(RESULT ipo_supported OUTPUT ipo_error)
    
    if(ipo_supported)
        message(STATUS "LTO enabled")
        set_property(TARGET ${PROJECT_NAME} PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
        
        # GCC LTO 特定標誌
        if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
            target_compile_options(${PROJECT_NAME} PRIVATE
                "-flto=auto"          # 自動並行LTO
                "-ffat-lto-objects"   # 保留目標文件信息
                "-fuse-linker-plugin" # 使用鏈接器插件
            )
            target_link_options(${PROJECT_NAME} PRIVATE
                "-flto=auto"
                "-fuse-linker-plugin"
            )
        endif()
        
        # Clang LTO 配置
        if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
            target_compile_options(${PROJECT_NAME} PRIVATE
                "-flto=thin"          # 使用 ThinLTO
            )
            target_link_options(${PROJECT_NAME} PRIVATE
                "-flto=thin"
                "-Wl,--thinlto-cache-dir=${CMAKE_BINARY_DIR}/lto-cache"
            )
        endif()
    else()
        message(WARNING "LTO not supported: ${ipo_error}")
    endif()
endif()
```

### LTO 效果展示

```cpp
// LTO 優化示例：跨編譯單元優化
// math_utils.cpp
namespace math_utils {
    double compute_heavy_calculation(double x) {
        // 複雜計算，但可能被內聯
        double result = 0.0;
        for (int i = 0; i < 100; ++i) {
            result += std::sin(x + i * 0.1) * std::cos(x + i * 0.2);
        }
        return result;
    }
    
    // 小函數，LTO 後會被內聯
    inline double fast_multiply(double a, double b) {
        return a * b;
    }
}

// main.cpp
#include "math_utils.h"

int main() {
    double x = 1.0;
    
    // LTO 會將這兩個調用優化合併
    auto result1 = math_utils::compute_heavy_calculation(x);
    auto result2 = math_utils::fast_multiply(result1, 2.0);
    
    return static_cast<int>(result2);
}
```

### LTO 性能基準測試

```cpp
// LTO 性能測試框架
#include <chrono>
#include <iostream>

class LTOBenchmark {
private:
    static constexpr size_t ITERATIONS = 1000000;
    
public:
    static void run_benchmark() {
        std::cout << "LTO Performance Benchmark\n";
        std::cout << "========================\n";
        
        // 測試函數調用開銷
        benchmark_function_calls();
        
        // 測試跨模塊優化
        benchmark_cross_module();
        
        // 測試模板特化
        benchmark_template_specialization();
    }
    
private:
    static void benchmark_function_calls() {
        auto start = std::chrono::high_resolution_clock::now();
        
        volatile double result = 0.0;
        for (size_t i = 0; i < ITERATIONS; ++i) {
            result += math_utils::fast_multiply(i * 1.5, i * 2.0);
        }
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);
        
        std::cout << "Function calls: " << duration.count() / ITERATIONS 
                  << " ns/call (result: " << result << ")\n";
    }
    
    static void benchmark_cross_module() {
        auto start = std::chrono::high_resolution_clock::now();
        
        volatile double result = 0.0;
        for (size_t i = 0; i < ITERATIONS / 1000; ++i) {
            result += math_utils::compute_heavy_calculation(i * 0.001);
        }
        
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        
        std::cout << "Cross-module optimization: " << duration.count() * 1000.0 / ITERATIONS 
                  << " μs/call (result: " << result << ")\n";
    }
    
    static void benchmark_template_specialization();
};
```

## 配置文件引導優化 (PGO)

### PGO 構建流程

```cmake
# PGO 支持的 CMake 配置
option(ENABLE_PGO "Enable Profile-Guided Optimization" OFF)
option(PGO_GENERATE "Generate PGO profile data" OFF)
option(PGO_USE "Use existing PGO profile data" OFF)

if(ENABLE_PGO)
    if(PGO_GENERATE AND PGO_USE)
        message(FATAL_ERROR "Cannot generate and use PGO profiles simultaneously")
    endif()
    
    if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        if(PGO_GENERATE)
            message(STATUS "PGO: Generating profile data")
            target_compile_options(${PROJECT_NAME} PRIVATE
                "-fprofile-generate=${CMAKE_BINARY_DIR}/pgo-data"
            )
            target_link_options(${PROJECT_NAME} PRIVATE
                "-fprofile-generate=${CMAKE_BINARY_DIR}/pgo-data"
            )
        elseif(PGO_USE)
            message(STATUS "PGO: Using profile data")
            target_compile_options(${PROJECT_NAME} PRIVATE
                "-fprofile-use=${CMAKE_BINARY_DIR}/pgo-data"
                "-fprofile-correction"  # 處理不一致的配置文件
            )
            target_link_options(${PROJECT_NAME} PRIVATE
                "-fprofile-use=${CMAKE_BINARY_DIR}/pgo-data"
            )
        endif()
    elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        if(PGO_GENERATE)
            target_compile_options(${PROJECT_NAME} PRIVATE
                "-fprofile-instr-generate"
            )
            target_link_options(${PROJECT_NAME} PRIVATE
                "-fprofile-instr-generate"
            )
        elseif(PGO_USE)
            target_compile_options(${PROJECT_NAME} PRIVATE
                "-fprofile-instr-use=${CMAKE_BINARY_DIR}/pgo-data/merged.profdata"
            )
        endif()
    endif()
endif()
```

### PGO 訓練數據生成

```cpp
// PGO 訓練程序：模擬真實工作負載
#include <random>
#include <vector>
#include <algorithm>

class PGOTrainingWorkload {
private:
    std::vector<double> market_data_;
    std::mt19937 rng_;
    
public:
    PGOTrainingWorkload() : rng_(std::random_device{}()) {
        generate_market_data();
    }
    
    // 模擬市場數據處理的典型工作負載
    void run_training_workload() {
        std::cout << "Running PGO training workload...\n";
        
        // 85% 的時間處理正常市場數據
        for (int i = 0; i < 8500; ++i) {
            process_normal_market_data();
        }
        
        // 10% 的時間處理高波動數據
        for (int i = 0; i < 1000; ++i) {
            process_high_volatility_data();
        }
        
        // 5% 的時間處理異常數據
        for (int i = 0; i < 500; ++i) {
            process_outlier_data();
        }
        
        std::cout << "Training workload completed.\n";
    }
    
private:
    void generate_market_data() {
        market_data_.reserve(100000);
        std::normal_distribution<double> dist(100.0, 10.0);
        
        for (size_t i = 0; i < 100000; ++i) {
            market_data_.push_back(dist(rng_));
        }
    }
    
    // 正常市場數據處理 (熱路徑)
    void process_normal_market_data() {
        std::uniform_int_distribution<> idx_dist(0, market_data_.size() - 1000);
        size_t start = idx_dist(rng_);
        
        // 移動平均計算 (經常執行的分支)
        double sum = 0.0;
        for (size_t i = start; i < start + 100; ++i) {
            if (market_data_[i] > 95.0 && market_data_[i] < 105.0) {  // 熱分支
                sum += market_data_[i];
            }
        }
        
        volatile double avg = sum / 100.0;  // 防止優化掉
    }
    
    // 高波動數據處理 (溫路徑)
    void process_high_volatility_data() {
        std::uniform_int_distribution<> idx_dist(0, market_data_.size() - 1000);
        size_t start = idx_dist(rng_);
        
        for (size_t i = start; i < start + 100; ++i) {
            if (market_data_[i] > 120.0 || market_data_[i] < 80.0) {  // 溫分支
                volatile double processed = market_data_[i] * 1.5;
            }
        }
    }
    
    // 異常數據處理 (冷路徑)
    void process_outlier_data() {
        std::uniform_int_distribution<> idx_dist(0, market_data_.size() - 100);
        size_t start = idx_dist(rng_);
        
        for (size_t i = start; i < start + 10; ++i) {
            if (market_data_[i] > 150.0 || market_data_[i] < 50.0) {  // 冷分支
                volatile double processed = std::log(std::abs(market_data_[i]));
            }
        }
    }
};

// PGO 訓練主程序
int main() {
    PGOTrainingWorkload workload;
    workload.run_training_workload();
    return 0;
}
```

### PGO 構建腳本

```bash
#!/bin/bash
# build_with_pgo.sh - 完整的 PGO 構建流程

set -e

PROJECT_DIR=$(pwd)
BUILD_DIR="$PROJECT_DIR/build"
PGO_DIR="$BUILD_DIR/pgo-data"

echo "=== PGO Build Process ==="

# 步驟 1: 生成配置文件的構建
echo "Step 1: Building with profile generation..."
mkdir -p "$BUILD_DIR"
cd "$BUILD_DIR"

cmake -DCMAKE_BUILD_TYPE=Release \
      -DENABLE_PGO=ON \
      -DPGO_GENERATE=ON \
      "$PROJECT_DIR"

make -j$(nproc)

# 步驟 2: 運行訓練工作負載
echo "Step 2: Running training workload..."
mkdir -p "$PGO_DIR"

# 設置環境變量以收集配置文件數據
if [[ "$CXX" == *"clang"* ]]; then
    export LLVM_PROFILE_FILE="$PGO_DIR/training_%p.profraw"
fi

# 運行多次訓練以獲得更好的配置文件
for i in {1..5}; do
    echo "  Training run $i/5..."
    ./your_training_program
done

# Clang: 合併配置文件數據
if [[ "$CXX" == *"clang"* ]]; then
    llvm-profdata merge -output="$PGO_DIR/merged.profdata" "$PGO_DIR"/*.profraw
fi

# 步驟 3: 使用配置文件的最終構建
echo "Step 3: Building with profile data..."
rm -rf "$BUILD_DIR"/*
cd "$BUILD_DIR"

cmake -DCMAKE_BUILD_TYPE=Release \
      -DENABLE_PGO=ON \
      -DPGO_USE=ON \
      "$PROJECT_DIR"

make -j$(nproc)

echo "=== PGO Build Complete ==="
echo "Optimized binary: $BUILD_DIR/your_program"
```

## 性能對比與基準測試

### 綜合性能測試

```cpp
// 全面的編譯優化效果測試
#include <chrono>
#include <vector>
#include <numeric>
#include <random>
#include <iostream>
#include <iomanip>

class CompilerOptimizationBenchmark {
private:
    static constexpr size_t DATA_SIZE = 1000000;
    static constexpr size_t ITERATIONS = 1000;
    
    std::vector<double> data_;
    std::mt19937 rng_;
    
public:
    CompilerOptimizationBenchmark() : rng_(std::random_device{}()) {
        data_.resize(DATA_SIZE);
        std::uniform_real_distribution<double> dist(1.0, 1000.0);
        std::generate(data_.begin(), data_.end(), [&]() { return dist(rng_); });
    }
    
    void run_all_benchmarks() {
        std::cout << std::fixed << std::setprecision(2);
        std::cout << "Compiler Optimization Benchmark Results\n";
        std::cout << "======================================\n";
        
        benchmark_vectorization();
        benchmark_branch_prediction();
        benchmark_function_inlining();
        benchmark_loop_unrolling();
    }
    
private:
    void benchmark_vectorization() {
        std::cout << "\n1. Vectorization Performance:\n";
        
        auto start = std::chrono::high_resolution_clock::now();
        for (size_t iter = 0; iter < ITERATIONS; ++iter) {
            volatile double sum = 0.0;
            // 向量化友好的循環
            for (size_t i = 0; i < data_.size(); ++i) {
                sum += data_[i] * data_[i];
            }
        }
        auto end = std::chrono::high_resolution_clock::now();
        
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "   Vector operations: " << duration.count() / ITERATIONS 
                  << " μs/iteration\n";
        
        // 計算理論峰值性能
        double ops_per_iteration = data_.size() * 2; // 乘法 + 加法
        double mops_per_sec = (ops_per_iteration * ITERATIONS * 1.0) / duration.count();
        std::cout << "   Throughput: " << mops_per_sec << " MOps/sec\n";
    }
    
    void benchmark_branch_prediction() {
        std::cout << "\n2. Branch Prediction Performance:\n";
        
        // 可預測分支模式
        auto start = std::chrono::high_resolution_clock::now();
        for (size_t iter = 0; iter < ITERATIONS; ++iter) {
            volatile double sum = 0.0;
            for (size_t i = 0; i < data_.size(); ++i) {
                if (i % 2 == 0) {  // 高度可預測的分支
                    sum += data_[i];
                } else {
                    sum += data_[i] * 2.0;
                }
            }
        }
        auto end = std::chrono::high_resolution_clock::now();
        
        auto predictable = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "   Predictable branches: " << predictable.count() / ITERATIONS 
                  << " μs/iteration\n";
        
        // 隨機分支模式
        start = std::chrono::high_resolution_clock::now();
        for (size_t iter = 0; iter < ITERATIONS; ++iter) {
            volatile double sum = 0.0;
            for (size_t i = 0; i < data_.size(); ++i) {
                if (data_[i] > 500.0) {  // 隨機分支
                    sum += data_[i];
                } else {
                    sum += data_[i] * 2.0;
                }
            }
        }
        end = std::chrono::high_resolution_clock::now();
        
        auto unpredictable = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "   Unpredictable branches: " << unpredictable.count() / ITERATIONS 
                  << " μs/iteration\n";
        
        double overhead = (double(unpredictable.count()) / predictable.count() - 1.0) * 100.0;
        std::cout << "   Branch misprediction overhead: " << overhead << "%\n";
    }
    
    void benchmark_function_inlining() {
        std::cout << "\n3. Function Inlining Performance:\n";
        
        auto start = std::chrono::high_resolution_clock::now();
        for (size_t iter = 0; iter < ITERATIONS; ++iter) {
            volatile double result = 0.0;
            for (size_t i = 0; i < data_.size() / 100; ++i) {
                // 小函數，應該被內聯
                result += inline_function(data_[i]);
            }
        }
        auto end = std::chrono::high_resolution_clock::now();
        
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "   Inlined function calls: " << duration.count() / ITERATIONS 
                  << " μs/iteration\n";
    }
    
    void benchmark_loop_unrolling() {
        std::cout << "\n4. Loop Unrolling Performance:\n";
        
        // 手動展開的循環
        auto start = std::chrono::high_resolution_clock::now();
        for (size_t iter = 0; iter < ITERATIONS; ++iter) {
            volatile double sum = 0.0;
            for (size_t i = 0; i < data_.size(); i += 4) {
                sum += data_[i] + data_[i+1] + data_[i+2] + data_[i+3];
            }
        }
        auto end = std::chrono::high_resolution_clock::now();
        
        auto unrolled = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "   Unrolled loops: " << unrolled.count() / ITERATIONS 
                  << " μs/iteration\n";
    }
    
    __attribute__((always_inline))
    inline double inline_function(double x) {
        return x * 1.5 + 2.0;
    }
};

int main() {
    CompilerOptimizationBenchmark benchmark;
    benchmark.run_all_benchmarks();
    return 0;
}
```

### 編譯優化效果對比表

| 優化技術 | 性能提升 | 編譯時間 | 代碼大小 | HFT 適用性 |
|---------|----------|----------|----------|-----------|
| `-O2` | 基線 | 1x | 1x | ⭐⭐⭐ |
| `-O3` | +15-25% | 1.5x | +10% | ⭐⭐⭐⭐ |
| `-O3 -march=native` | +25-40% | 1.6x | +15% | ⭐⭐⭐⭐⭐ |
| `LTO` | +10-20% | 2x | -5% | ⭐⭐⭐⭐ |
| `PGO` | +15-30% | 3x | +5% | ⭐⭐⭐⭐⭐ |
| `LTO + PGO` | +30-50% | 4x | 0% | ⭐⭐⭐⭐⭐ |
| `-Ofast` | +20-35% | 1.4x | +20% | ⭐⭐ (風險) |

### HFT 系統實際效果

```cpp
// HFT 延遲測試：不同優化級別的實際效果
#include <chrono>
#include <array>

class LatencyBenchmark {
private:
    static constexpr size_t SAMPLES = 1000000;
    std::array<std::chrono::nanoseconds, SAMPLES> latencies_;
    
public:
    void measure_order_processing_latency() {
        for (size_t i = 0; i < SAMPLES; ++i) {
            auto start = std::chrono::high_resolution_clock::now();
            
            // 模擬訂單處理流程
            process_market_data();
            calculate_signals();
            generate_order();
            
            auto end = std::chrono::high_resolution_clock::now();
            latencies_[i] = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);
        }
        
        print_latency_statistics();
    }
    
private:
    void process_market_data() {
        // 市場數據處理 (熱路徑)
        volatile double price = 100.0;
        price = price * 1.001 + 0.01;  // 簡單計算
    }
    
    void calculate_signals() {
        // 信號計算 (中等頻率)
        volatile double signal = 0.0;
        for (int i = 0; i < 10; ++i) {
            signal += std::sin(i * 0.1);
        }
    }
    
    void generate_order() {
        // 訂單生成 (較少執行)
        volatile int order_size = 100;
        order_size = order_size * 2;
    }
    
    void print_latency_statistics() {
        std::sort(latencies_.begin(), latencies_.end());
        
        auto p50 = latencies_[SAMPLES * 50 / 100];
        auto p95 = latencies_[SAMPLES * 95 / 100];
        auto p99 = latencies_[SAMPLES * 99 / 100];
        auto p999 = latencies_[SAMPLES * 999 / 1000];
        
        std::cout << "HFT Latency Statistics (nanoseconds):\n";
        std::cout << "P50:  " << p50.count() << " ns\n";
        std::cout << "P95:  " << p95.count() << " ns\n";
        std::cout << "P99:  " << p99.count() << " ns\n";
        std::cout << "P99.9: " << p999.count() << " ns\n";
    }
};
```

## 最佳實踐與建議

### HFT 編譯優化策略

```cmake
# 生產環境 HFT 編譯配置
function(configure_hft_target target_name)
    # 核心優化標誌
    target_compile_options(${target_name} PRIVATE
        $<$<CONFIG:Release>:
            -O3
            -DNDEBUG
            -march=native
            -mtune=native
            -flto=thin
            -fprofile-use=${PGO_PROFILE_PATH}
            
            # 向量化
            -ftree-vectorize
            -fvect-cost-model=unlimited
            
            # 分支優化
            -fbranch-probabilities
            -ftracer
            -funroll-loops
            
            # 內聯優化
            -finline-functions
            -finline-limit=1000
            
            # 浮點優化 (謹慎使用)
            -ffast-math
            -ffinite-math-only
        >
    )
    
    # 鏈接優化
    target_link_options(${target_name} PRIVATE
        $<$<CONFIG:Release>:
            -flto=thin
            -Wl,--gc-sections
            -Wl,--strip-all
        >
    )
    
    # 調試版本配置
    target_compile_options(${target_name} PRIVATE
        $<$<CONFIG:Debug>:
            -O0
            -g3
            -fno-omit-frame-pointer
            -fsanitize=address
            -fsanitize=undefined
        >
    )
endfunction()
```

### 編譯優化檢查清單

1. **性能關鍵代碼路徑**
   - 使用 `-O3` 或更高級別優化
   - 啟用目標平台特定優化 (`-march=native`)
   - 考慮 PGO 用於熱路徑

2. **向量化優化**
   - 確保循環可向量化
   - 使用適當的數據對齊
   - 避免循環依賴

3. **分支優化**
   - 將熱分支放在前面
   - 使用 `[[likely]]` 和 `[[unlikely]]` (C++20)
   - 避免不必要的條件分支

4. **內聯優化**
   - 標記小函數為 `inline`
   - 使用 `__attribute__((always_inline))` 強制內聯
   - 避免過度內聯導致指令快取未命中

5. **LTO 考量**
   - 增加編譯時間，但改善最終性能
   - 特別適合跨模塊調用頻繁的代碼
   - 可能需要調整內存使用

6. **PGO 策略**
   - 使用具代表性的工作負載進行訓練
   - 定期更新配置文件
   - 監控配置文件覆蓋率

### 常見陷阱與解決方案

```cpp
// 常見優化陷阱示例
class OptimizationPitfalls {
public:
    // 陷阱 1: 過度優化破壞數值穩定性
    double bad_fast_math_example(double x) {
        // -ffast-math 可能導致 NaN/Inf 處理不當
        return 1.0 / (x * x - 1.0);  // 當 x = ±1 時可能有問題
    }
    
    double safe_fast_math_example(double x) {
        // 添加顯式檢查確保數值穩定性
        double denominator = x * x - 1.0;
        if (std::abs(denominator) < 1e-15) {
            return std::copysign(std::numeric_limits<double>::infinity(), denominator);
        }
        return 1.0 / denominator;
    }
    
    // 陷阱 2: 向量化破壞的循環依賴
    void bad_vectorization_example(std::vector<double>& data) {
        for (size_t i = 1; i < data.size(); ++i) {
            data[i] = data[i-1] + data[i];  // 循環依賴，無法向量化
        }
    }
    
    void vectorization_friendly_example(std::vector<double>& data) {
        // 使用前綴和算法，可以部分向量化
        std::partial_sum(data.begin(), data.end(), data.begin());
    }
    
    // 陷阱 3: 分支預測失敗
    double bad_branch_example(const std::vector<double>& data) {
        double sum = 0.0;
        for (const auto& val : data) {
            if (val > 100.0) {  // 隨機分支，預測困難
                sum += val * 2.0;
            } else {
                sum += val;
            }
        }
        return sum;
    }
    
    double branchless_example(const std::vector<double>& data) {
        double sum = 0.0;
        for (const auto& val : data) {
            // 無分支版本
            double multiplier = 1.0 + (val > 100.0 ? 1.0 : 0.0);
            sum += val * multiplier;
        }
        return sum;
    }
};
```

---

## 參考資料 (References)

1. [GCC Optimization Options](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)
2. [Clang Optimization Guide](https://clang.llvm.org/docs/CommandGuide/clang.html#optimization-options)
3. [Intel C++ Compiler Optimization Guide](https://software.intel.com/content/www/us/en/develop/documentation/cpp-compiler-developer-guide-and-reference/)
4. 《Optimizing C++》(Kurt Guntheroth, 2016)
5. [Link Time Optimization in GCC](https://gcc.gnu.org/wiki/LinkTimeOptimization)
6. [Profile-Guided Optimization Best Practices](https://clang.llvm.org/docs/UsersManual.html#profile-guided-optimization)
7. [High Frequency Trading System Optimization](https://www.quantstart.com/articles/C-High-Frequency-Trading-System-Optimization/)
8. [Modern CPU Performance Optimization](https://www.intel.com/content/www/us/en/developer/articles/technical/software-optimization-resources.html)