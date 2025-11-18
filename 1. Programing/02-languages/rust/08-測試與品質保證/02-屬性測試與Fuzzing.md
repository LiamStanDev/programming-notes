# 屬性測試與 Fuzzing

> 基於 Rust 1.90+ (2025) | 自動化發現邊界情況與潛在問題

## 📋 概述

傳統測試需要開發者手動編寫每個測試案例,容易遺漏邊界情況。**屬性測試** (Property-based Testing) 和 **Fuzzing** 通過自動生成大量測試輸入,幫助發現隱藏的 bug 和邊界情況。

---

## 🎯 什麼是屬性測試?

### 傳統測試 vs 屬性測試

```rust
// 傳統測試: 基於範例
#[test]
fn test_reverse_traditional() {
    assert_eq!(reverse("abc"), "cba");
    assert_eq!(reverse("hello"), "olleh");
    assert_eq!(reverse(""), "");
}

// 屬性測試: 基於不變量
#[test]
fn test_reverse_property() {
    // 屬性: 反轉兩次應該得到原字符串
    proptest!(|(s: String)| {
        let reversed = reverse(&s);
        let double_reversed = reverse(&reversed);
        prop_assert_eq!(s, double_reversed);
    });
}
```

**核心概念**:
- **範例測試**: 驗證特定輸入的輸出
- **屬性測試**: 驗證所有輸入都滿足某種性質

### 常見的測試屬性

```rust
// 1. 可逆性 (Reversibility)
reverse(reverse(x)) == x

// 2. 等冪性 (Idempotence)
sort(sort(x)) == sort(x)

// 3. 交換律 (Commutativity)
add(a, b) == add(b, a)

// 4. 結合律 (Associativity)
add(a, add(b, c)) == add(add(a, b), c)

// 5. 不變量保持 (Invariant Preservation)
length(append(x, y)) == length(x) + length(y)
```

---

## 🚀 使用 proptest

### 安裝

```toml
[dev-dependencies]
proptest = "1.4"
```

### 基本用法

```rust
use proptest::prelude::*;

fn add(a: i32, b: i32) -> i32 {
    a + b
}

proptest! {
    #[test]
    fn test_add_commutative(a: i32, b: i32) {
        // 加法交換律: a + b = b + a
        prop_assert_eq!(add(a, b), add(b, a));
    }
    
    #[test]
    fn test_add_associative(a: i32, b: i32, c: i32) {
        // 加法結合律: (a + b) + c = a + (b + c)
        prop_assert_eq!(
            add(add(a, b), c),
            add(a, add(b, c))
        );
    }
    
    #[test]
    fn test_add_identity(a: i32) {
        // 加法單位元: a + 0 = a
        prop_assert_eq!(add(a, 0), a);
    }
}
```

### 自定義生成器

```rust
use proptest::prelude::*;

// 生成 1-100 之間的數字
proptest! {
    #[test]
    fn test_range(n in 1..=100) {
        prop_assert!(n >= 1 && n <= 100);
    }
}

// 生成特定長度的字符串
proptest! {
    #[test]
    fn test_string_length(s in "[a-z]{5,10}") {
        prop_assert!(s.len() >= 5 && s.len() <= 10);
        prop_assert!(s.chars().all(|c| c.is_ascii_lowercase()));
    }
}

// 生成向量
proptest! {
    #[test]
    fn test_vec(v in prop::collection::vec(0..100, 0..50)) {
        prop_assert!(v.len() <= 50);
        prop_assert!(v.iter().all(|&x| x < 100));
    }
}
```

### 組合生成器

```rust
use proptest::prelude::*;

#[derive(Debug, Clone)]
struct User {
    name: String,
    age: u32,
    email: String,
}

fn user_strategy() -> impl Strategy<Value = User> {
    (
        "[a-z]{3,10}",           // name
        18u32..=100u32,          // age
        "[a-z]{3,10}@[a-z]{3,10}\\.com"  // email
    ).prop_map(|(name, age, email)| User { name, age, email })
}

proptest! {
    #[test]
    fn test_user_validation(user in user_strategy()) {
        prop_assert!(user.age >= 18);
        prop_assert!(user.name.len() >= 3);
        prop_assert!(user.email.contains('@'));
    }
}
```

---

## 🎨 實戰範例

### 範例 1: 測試排序函數

```rust
use proptest::prelude::*;

fn my_sort(mut v: Vec<i32>) -> Vec<i32> {
    v.sort();
    v
}

proptest! {
    #[test]
    fn test_sort_preserves_length(v: Vec<i32>) {
        let sorted = my_sort(v.clone());
        prop_assert_eq!(sorted.len(), v.len());
    }
    
    #[test]
    fn test_sort_is_sorted(v: Vec<i32>) {
        let sorted = my_sort(v);
        for i in 0..sorted.len().saturating_sub(1) {
            prop_assert!(sorted[i] <= sorted[i + 1]);
        }
    }
    
    #[test]
    fn test_sort_preserves_elements(v: Vec<i32>) {
        let sorted = my_sort(v.clone());
        for &elem in &v {
            let count_original = v.iter().filter(|&&x| x == elem).count();
            let count_sorted = sorted.iter().filter(|&&x| x == elem).count();
            prop_assert_eq!(count_original, count_sorted);
        }
    }
    
    #[test]
    fn test_sort_is_idempotent(v: Vec<i32>) {
        let sorted_once = my_sort(v.clone());
        let sorted_twice = my_sort(sorted_once.clone());
        prop_assert_eq!(sorted_once, sorted_twice);
    }
}
```

### 範例 2: 測試編碼/解碼

```rust
use proptest::prelude::*;
use base64::{Engine, engine::general_purpose};

fn encode(data: &[u8]) -> String {
    general_purpose::STANDARD.encode(data)
}

fn decode(s: &str) -> Result<Vec<u8>, base64::DecodeError> {
    general_purpose::STANDARD.decode(s)
}

proptest! {
    #[test]
    fn test_encode_decode_roundtrip(data: Vec<u8>) {
        let encoded = encode(&data);
        let decoded = decode(&encoded).unwrap();
        prop_assert_eq!(data, decoded);
    }
    
    #[test]
    fn test_encode_always_ascii(data: Vec<u8>) {
        let encoded = encode(&data);
        prop_assert!(encoded.is_ascii());
    }
}
```

### 範例 3: 測試解析器

```rust
use proptest::prelude::*;

#[derive(Debug, PartialEq)]
struct Config {
    host: String,
    port: u16,
}

fn parse_config(s: &str) -> Result<Config, String> {
    let parts: Vec<&str> = s.split(':').collect();
    if parts.len() != 2 {
        return Err("invalid format".to_string());
    }
    
    let host = parts[0].to_string();
    let port = parts[1].parse()
        .map_err(|_| "invalid port".to_string())?;
    
    Ok(Config { host, port })
}

fn format_config(config: &Config) -> String {
    format!("{}:{}", config.host, config.port)
}

proptest! {
    #[test]
    fn test_parse_format_roundtrip(
        host in "[a-z]{1,20}",
        port in 1..=65535u16
    ) {
        let config = Config {
            host: host.clone(),
            port,
        };
        
        let formatted = format_config(&config);
        let parsed = parse_config(&formatted).unwrap();
        
        prop_assert_eq!(config, parsed);
    }
}
```

---

## 🔧 進階 proptest 技巧

### 收縮 (Shrinking)

當測試失敗時,proptest 會嘗試找到最小的失敗案例:

```rust
use proptest::prelude::*;

fn buggy_function(n: i32) -> i32 {
    if n > 100 {
        panic!("Bug!");  // 有 bug
    }
    n * 2
}

proptest! {
    #[test]
    fn test_buggy(n in 0..1000) {
        let result = buggy_function(n);
        prop_assert_eq!(result, n * 2);
    }
}

// 失敗時,proptest 會收縮到最小失敗案例: n = 101
```

### 自定義收縮策略

```rust
use proptest::prelude::*;
use proptest::strategy::{Strategy, ValueTree};

// 自定義生成器,只生成偶數
fn even_numbers() -> impl Strategy<Value = i32> {
    (0..1000).prop_map(|x| x * 2)
}

proptest! {
    #[test]
    fn test_even(n in even_numbers()) {
        prop_assert_eq!(n % 2, 0);
    }
}
```

### 條件生成

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_positive_division(
        a in 1..1000,
        b in 1..1000
    ) {
        let result = a / b;
        prop_assert!(result >= 0);
    }
}

// 生成滿足條件的值
proptest! {
    #[test]
    fn test_sorted_pair(a in 0..100, b in 0..100) {
        let (min, max) = if a <= b { (a, b) } else { (b, a) };
        prop_assert!(min <= max);
    }
}
```

### 回歸測試

```rust
use proptest::prelude::*;
use proptest::test_runner::Config;

proptest! {
    // 使用固定種子進行回歸測試
    #![proptest_config(Config {
        cases: 1000,      // 運行 1000 個案例
        max_shrink_iters: 10000,
        .. Config::default()
    })]
    
    #[test]
    fn test_with_config(x in 0..100) {
        prop_assert!(x < 100);
    }
}
```

---

## 🎲 Fuzzing

### 什麼是 Fuzzing?

Fuzzing 是一種自動化測試技術,通過生成隨機或變異的輸入來測試程序,尋找崩潰、記憶體錯誤或其他異常行為。

### 使用 cargo-fuzz

#### 安裝

```bash
$ cargo install cargo-fuzz
```

#### 創建 Fuzz Target

```bash
# 初始化 fuzzing
$ cargo fuzz init

# 創建 fuzz target
$ cargo fuzz add fuzz_parser
```

**fuzz/fuzz_targets/fuzz_parser.rs**:
```rust
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    // 將隨機字節轉換為字符串
    if let Ok(s) = std::str::from_utf8(data) {
        // 測試解析器
        let _ = my_crate::parse(s);
    }
});
```

#### 運行 Fuzzer

```bash
$ cargo fuzz run fuzz_parser

# 限制運行時間
$ cargo fuzz run fuzz_parser -- -max_total_time=60

# 限制內存
$ cargo fuzz run fuzz_parser -- -rss_limit_mb=2048
```

### 實戰範例: Fuzzing JSON 解析器

```rust
// fuzz/fuzz_targets/fuzz_json.rs
#![no_main]
use libfuzzer_sys::fuzz_target;
use serde_json;

fuzz_target!(|data: &[u8]| {
    // 嘗試解析為 JSON
    let _ = serde_json::from_slice::<serde_json::Value>(data);
});
```

**運行**:
```bash
$ cargo fuzz run fuzz_json
```

### 結構化 Fuzzing

```rust
#![no_main]
use libfuzzer_sys::fuzz_target;
use arbitrary::Arbitrary;

#[derive(Debug, Arbitrary)]
struct FuzzInput {
    operation: Operation,
    value: i32,
}

#[derive(Debug, Arbitrary)]
enum Operation {
    Add,
    Subtract,
    Multiply,
    Divide,
}

fuzz_target!(|input: FuzzInput| {
    let result = match input.operation {
        Operation::Add => 100i32.saturating_add(input.value),
        Operation::Subtract => 100i32.saturating_sub(input.value),
        Operation::Multiply => 100i32.saturating_mul(input.value),
        Operation::Divide => {
            if input.value != 0 {
                100 / input.value
            } else {
                return;
            }
        }
    };
    
    // 檢查結果是否有效
    assert!(result.abs() < i32::MAX);
});
```

---

## 📊 完整範例: 測試自定義數據結構

```rust
use proptest::prelude::*;

#[derive(Debug, Clone, PartialEq)]
struct BoundedVec<T> {
    data: Vec<T>,
    max_size: usize,
}

impl<T> BoundedVec<T> {
    fn new(max_size: usize) -> Self {
        Self {
            data: Vec::new(),
            max_size,
        }
    }
    
    fn push(&mut self, value: T) -> Result<(), &'static str> {
        if self.data.len() >= self.max_size {
            Err("capacity exceeded")
        } else {
            self.data.push(value);
            Ok(())
        }
    }
    
    fn len(&self) -> usize {
        self.data.len()
    }
    
    fn is_empty(&self) -> bool {
        self.data.is_empty()
    }
    
    fn pop(&mut self) -> Option<T> {
        self.data.pop()
    }
}

// 生成器
fn bounded_vec_strategy() -> impl Strategy<Value = BoundedVec<i32>> {
    (1..=100usize).prop_flat_map(|max_size| {
        prop::collection::vec(any::<i32>(), 0..=max_size)
            .prop_map(move |data| BoundedVec { data, max_size })
    })
}

proptest! {
    // 屬性 1: 長度不超過最大值
    #[test]
    fn test_bounded_vec_respects_max_size(mut bv in bounded_vec_strategy()) {
        prop_assert!(bv.len() <= bv.max_size);
        
        // 嘗試添加更多元素
        while bv.len() < bv.max_size {
            bv.push(42).unwrap();
        }
        
        // 現在應該拒絕新元素
        prop_assert!(bv.push(42).is_err());
    }
    
    // 屬性 2: push 和 pop 是可逆的
    #[test]
    fn test_push_pop_roundtrip(
        mut bv in bounded_vec_strategy(),
        value: i32
    ) {
        let original_len = bv.len();
        
        if bv.len() < bv.max_size {
            bv.push(value).unwrap();
            let popped = bv.pop().unwrap();
            
            prop_assert_eq!(popped, value);
            prop_assert_eq!(bv.len(), original_len);
        }
    }
    
    // 屬性 3: is_empty 的正確性
    #[test]
    fn test_is_empty_consistency(bv in bounded_vec_strategy()) {
        prop_assert_eq!(bv.is_empty(), bv.len() == 0);
    }
    
    // 屬性 4: pop 減少長度
    #[test]
    fn test_pop_decreases_length(mut bv in bounded_vec_strategy()) {
        if !bv.is_empty() {
            let original_len = bv.len();
            bv.pop();
            prop_assert_eq!(bv.len(), original_len - 1);
        } else {
            prop_assert!(bv.pop().is_none());
        }
    }
}

// 單元測試仍然有用
#[cfg(test)]
mod unit_tests {
    use super::*;
    
    #[test]
    fn test_empty_bounded_vec() {
        let bv: BoundedVec<i32> = BoundedVec::new(10);
        assert!(bv.is_empty());
        assert_eq!(bv.len(), 0);
    }
    
    #[test]
    fn test_push_to_capacity() {
        let mut bv = BoundedVec::new(3);
        assert!(bv.push(1).is_ok());
        assert!(bv.push(2).is_ok());
        assert!(bv.push(3).is_ok());
        assert!(bv.push(4).is_err());
    }
}
```

---

## 🎯 屬性測試 vs 單元測試

### 何時使用屬性測試?

```rust
// ✅ 適合屬性測試的情況:

// 1. 數學屬性
proptest! {
    #[test]
    fn test_multiplication_commutative(a: i32, b: i32) {
        prop_assert_eq!(a * b, b * a);
    }
}

// 2. 編碼/解碼往返
proptest! {
    #[test]
    fn test_serialize_deserialize(data: MyStruct) {
        let serialized = serialize(&data);
        let deserialized = deserialize(&serialized).unwrap();
        prop_assert_eq!(data, deserialized);
    }
}

// 3. 不變量保持
proptest! {
    #[test]
    fn test_sort_preserves_length(v: Vec<i32>) {
        let sorted = sort(v.clone());
        prop_assert_eq!(sorted.len(), v.len());
    }
}
```

### 何時使用單元測試?

```rust
// ✅ 適合單元測試的情況:

// 1. 特定業務邏輯
#[test]
fn test_discount_calculation() {
    let price = 100.0;
    let discount = calculate_discount(price, "VIP");
    assert_eq!(discount, 20.0);  // VIP 享有 20% 折扣
}

// 2. 邊界條件
#[test]
fn test_division_by_zero() {
    assert!(divide(10, 0).is_err());
}

// 3. 錯誤訊息
#[test]
fn test_error_message() {
    let err = validate_email("invalid").unwrap_err();
    assert_eq!(err, "invalid email format");
}
```

---

## 🔍 常見陷阱

### 陷阱 1: 過度依賴隨機性

```rust
// ❌ 不好: 測試可能不穩定
proptest! {
    #[test]
    fn test_flaky(n: i32) {
        use std::time::SystemTime;
        let now = SystemTime::now();
        // 依賴時間的測試不穩定
    }
}

// ✅ 好: 測試確定性的屬性
proptest! {
    #[test]
    fn test_deterministic(n: i32) {
        prop_assert_eq!(abs(abs(n)), abs(n));
    }
}
```

### 陷阱 2: 忽略生成器的範圍

```rust
// ❌ 危險: 可能溢出
proptest! {
    #[test]
    fn test_overflow(a: i32, b: i32) {
        let sum = a + b;  // 可能 panic
    }
}

// ✅ 安全: 限制範圍或使用 saturating
proptest! {
    #[test]
    fn test_safe(a in 0..1000, b in 0..1000) {
        let sum = a.saturating_add(b);
        prop_assert!(sum >= a && sum >= b);
    }
}
```

### 陷阱 3: 測試運行時間過長

```rust
// ❌ 慢: 默認運行 256 次
proptest! {
    #[test]
    fn test_slow(v: Vec<i32>) {
        // 複雜的操作
    }
}

// ✅ 調整配置
proptest! {
    #![proptest_config(ProptestConfig::with_cases(100))]
    
    #[test]
    fn test_adjusted(v: Vec<i32>) {
        // 只運行 100 次
    }
}
```

---

## 🎓 最佳實踐

### 1. 組合使用不同測試類型

```rust
// 單元測試: 特定案例
#[test]
fn test_edge_case() {
    assert_eq!(parse(""), None);
}

// 屬性測試: 一般性質
proptest! {
    #[test]
    fn test_parse_format_roundtrip(s: String) {
        if let Some(parsed) = parse(&s) {
            let formatted = format(&parsed);
            prop_assert_eq!(parse(&formatted), Some(parsed));
        }
    }
}

// Fuzzing: 發現崩潰
// (在 fuzz/fuzz_targets/ 中)
```

### 2. 從失敗中學習

```rust
// 當屬性測試失敗時,添加為回歸測試
#[test]
fn test_regression_found_by_proptest() {
    // proptest 發現的失敗案例
    assert_eq!(parse("特殊輸入"), Some(預期結果));
}
```

### 3. 文檔化測試屬性

```rust
proptest! {
    /// 測試屬性: 排序後的數組應該是有序的
    /// 這個測試驗證了 sort 函數的基本正確性
    #[test]
    fn test_sort_is_sorted(v: Vec<i32>) {
        let sorted = sort(v);
        for window in sorted.windows(2) {
            prop_assert!(window[0] <= window[1]);
        }
    }
}
```

---

## 📖 參考資料

1. [proptest Documentation](https://docs.rs/proptest/)
2. [proptest Book](https://altsysrq.github.io/proptest-book/)
3. [cargo-fuzz Documentation](https://rust-fuzz.github.io/book/cargo-fuzz.html)
4. [The Fuzzing Book](https://www.fuzzingbook.org/)
5. [QuickCheck (Haskell)](https://hackage.haskell.org/package/QuickCheck) - 屬性測試的起源
6. [Hypothesis (Python)](https://hypothesis.readthedocs.io/) - 另一個屬性測試框架

---

*最後更新: 2025-01-17*  
*Rust 版本: 1.90+*
