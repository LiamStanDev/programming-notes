# 分支預測與 CPU 流水線 (Branch Prediction & CPU Pipeline)

## 1. CPU 流水線架構

現代 CPU 為了提高指令吞吐量，採用了 **流水線 (Pipeline)** 技術。一條指令的執行被分解為多個階段，類似於工廠的裝配線。

### 1.1 典型流水線階段
雖然現代 CPU (如 Intel Core, AMD Ryzen) 的流水線非常深 (14-20+ 級)，但可以簡化為以下 5 個基本階段：

1.  **Fetch (取指)**: 從 Instruction Cache (L1i) 讀取指令。
2.  **Decode (解碼)**: 將指令翻譯成微指令 (uOps)。
3.  **Execute (執行)**: ALU 進行運算。
4.  **Memory (訪存)**: 讀寫數據 (L1d Cache)。
5.  **Write Back (寫回)**: 將結果寫回寄存器。

### 1.2 分支的代價 (Branch Penalty)
當 CPU 遇到分支指令 (如 `if`, `for`, `while`) 時，它必須決定下一條指令從哪裡取。
*   **預測正確**: 流水線繼續滿載運行，代價極小 (~1 cycle)。
*   **預測錯誤 (Misprediction)**: CPU 必須**清空 (Flush)** 整個流水線中已加載的錯誤指令，並重新從正確地址取指。
    *   **代價**: 15-20 個 CPU 週期 (約 5-7ns @ 3GHz)。
    *   **影響**: 在 HFT 中，一次預測錯誤可能相當於幾十次簡單運算的時間。

---

## 2. 分支預測器 (Branch Predictor)

CPU 內部有複雜的硬件單元負責猜測分支方向。

### 2.1 靜態 vs 動態預測
*   **靜態預測**: 基於編譯器提示或簡單規則 (例如：假設向後跳轉的循環分支總是 Taken)。
*   **動態預測**: 基於歷史執行記錄 (Branch History Buffer)。如果一個分支過去 99% 都是 True，CPU 會猜測它這次也是 True。

### 2.2 可預測 vs 不可預測
*   **可預測**: 模式固定 (如 `TTTTT...` 或 `TNTNTN...`)。
*   **不可預測**: 依賴於隨機數據 (如 `rand() % 2`)。

```cpp
// ❌ 不可預測分支 (性能殺手)
// 假設 data 是隨機整數
for (int x : data) {
    if (x > 50) { // 50% 概率，無規律
        sum += x;
    }
}

// ✅ 排序後變為可預測
std::sort(data.begin(), data.end());
for (int x : data) {
    if (x > 50) { // 前半部分全 False，後半部分全 True
        sum += x;
    }
}
// 性能提升可能達 2-5 倍
```

---

## 3. 消除分支 (Branchless Programming)

在熱路徑 (Hot Path) 中，消除不可預測的分支是優化的核心。

### 3.1 算術替代分支
利用布爾值轉換為整數 (0 或 1) 的特性。

```cpp
// ❌ 分支寫法
if (x > y) {
    result = x;
} else {
    result = y;
}

// ✅ 無分支寫法 (編譯器可能優化為 CMOV 指令)
result = (x > y) ? x : y;

// ✅ 純算術寫法 (適用於特定場景)
// 如果 condition 為真，mask 為 0xFFFFFFFF，否則為 0
int mask = -(x > y); 
result = (mask & x) | (~mask & y);
```

### 3.2 查找表 (Lookup Table)
用數組索引替代 `switch` 或多個 `if-else`。

```cpp
// ❌ 分支寫法
char to_hex(int v) {
    if (v < 10) return '0' + v;
    return 'A' + (v - 10);
}

// ✅ 查找表寫法
const char hex_table[] = "0123456789ABCDEF";
char to_hex(int v) {
    return hex_table[v & 0xF]; // 無分支，僅一次內存讀取
}
```

### 3.3 HFT 案例：訂單驗證
訂單驗證通常涉及多個條件，如果用 `&&` 連接，會產生短路分支。

```cpp
struct Order { double price; int qty; };

// ❌ 短路邏輯 (Short-circuiting) -> 多個分支
bool validate_branched(const Order& o) {
    if (o.price <= 0) return false; // Branch 1
    if (o.qty <= 0) return false;   // Branch 2
    if (o.qty > 1000) return false; // Branch 3
    return true;
}

// ✅ 位運算邏輯 (Bitwise) -> 無分支
// 使用 & 而不是 &&，強制計算所有條件，然後合併結果
bool validate_branchless(const Order& o) {
    bool p_ok = (o.price > 0);
    bool q_min = (o.qty > 0);
    bool q_max = (o.qty <= 1000);
    
    // 編譯器會生成無分支的比較指令 (SETcc) 和位運算 (AND)
    return p_ok & q_min & q_max; 
}
```

---

## 4. 編譯器提示 (Compiler Hints)

告訴編譯器哪些分支是 "Hot Path" (極大機率執行)，哪些是 "Cold Path" (錯誤處理)。

### 4.1 C++20 `[[likely]]` / `[[unlikely]]`

```cpp
void process_market_data(const Tick& tick) {
    // 99.9% 的情況下數據是有效的
    if (tick.is_valid()) [[likely]] {
        update_book(tick);
    } else [[unlikely]] {
        // 編譯器會將這段代碼移到函數末尾或冷區 (Cold Section)
        // 減少 Instruction Cache Miss
        log_error("Invalid tick");
    }
}
```

### 4.2 傳統宏 (GCC/Clang)
在 C++20 之前或需要兼容舊編譯器時使用。

```cpp
#define LIKELY(x)   __builtin_expect(!!(x), 1)
#define UNLIKELY(x) __builtin_expect(!!(x), 0)

if (LIKELY(ptr != nullptr)) {
    // ...
}
```

> **警告**: 不要濫用。如果你標記了 `likely` 但實際只有 50% 的概率，會誤導編譯器進行錯誤的優化 (如錯誤的代碼佈局)，反而降低性能。

---

## 5. 循環優化 (Loop Optimization)

循環本質上是條件跳轉。

### 5.1 循環展開 (Loop Unrolling)
減少循環控制分支的次數，並增加指令級並行 (ILP)。

```cpp
// 原始循環
for (int i = 0; i < n; ++i) {
    sum += data[i];
}

// 展開後 (手動或編譯器自動)
for (int i = 0; i < n; i += 4) {
    sum += data[i];
    sum += data[i+1];
    sum += data[i+2];
    sum += data[i+3];
}
// 優勢: 分支次數減少 4 倍，且 4 次加法可能並行執行 (如果無依賴)
```

### 5.2 數據依賴 (Data Dependency)
流水線需要等待前一條指令的結果才能執行下一條，這稱為**數據冒險 (Data Hazard)**。

```cpp
// ❌ 依賴鏈 (Dependency Chain)
// 下一次加法必須等待上一次 sum 更新完成
for (int i = 0; i < n; ++i) sum += data[i];

// ✅ 多路累加 (Breaking Dependency)
// sum1, sum2, sum3, sum4 可以並行計算
for (int i = 0; i < n; i += 4) {
    sum1 += data[i];
    sum2 += data[i+1];
    sum3 += data[i+2];
    sum4 += data[i+3];
}
sum = sum1 + sum2 + sum3 + sum4;
```

---

## 6. 實戰檢查清單

1.  **識別熱點分支**: 使用 `perf record -e branch-misses` 找出 Misprediction 高的代碼。
2.  **消除隨機分支**: 對於不可預測的邏輯，嘗試用算術運算、位運算或查找表替代。
3.  **標記冷熱路徑**: 對錯誤檢查、邊界檢查使用 `[[unlikely]]`。
4.  **優化循環**: 確保關鍵循環被展開 (使用 `#pragma GCC unroll` 或編譯器標誌)，並打破數據依賴鏈。
5.  **避免虛函數**: 虛函數調用 (Virtual Call) 也是一種間接分支，且難以預測。在 HFT 核心路徑中應使用 CRTP (Curiously Recurring Template Pattern) 替代動態多態。

## 參考資料
*   [Intel 64 and IA-32 Architectures Optimization Reference Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
*   [Agner Fog's Optimization Manuals](https://www.agner.org/optimize/)
*   [CppCon 2015: Chandler Carruth "Tuning C++: Benchmarks, and CPUs, and Compilers! Oh My!"](https://www.youtube.com/watch?v=nXaxk27zwlk)
