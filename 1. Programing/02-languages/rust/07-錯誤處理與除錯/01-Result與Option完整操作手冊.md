# Result 與 Option 完整操作手冊

> 基於 Rust 1.90+ (2025) | 錯誤處理的核心基石

## 📋 概述

`Result<T, E>` 和 `Option<T>` 是 Rust 中處理錯誤和空值的兩大核心類型。與其他語言的異常機制不同，Rust 強制開發者顯式處理錯誤，從而在編譯期消除大量潛在問題。

---

## 🎯 Result<T, E> 完整指南

### 定義與本質

```rust
pub enum Result<T, E> {
    Ok(T),   // 成功，包含值 T
    Err(E),  // 失敗，包含錯誤 E
}
```

**核心特性**:
- `T` 和 `E` 可以是任何類型，沒有限制
- 主要用於函數返回值，表示可能失敗的操作
- 與 `?` 運算子配合，實現優雅的錯誤傳播
- 標準庫中大量用於 I/O、解析、網路等可能失敗的操作

### 基本使用示例

```rust
fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err("除數不能為零".to_string())
    } else {
        Ok(a / b)
    }
}

fn main() {
    match divide(10, 2) {
        Ok(result) => println!("結果: {}", result),
        Err(e) => eprintln!("錯誤: {}", e),
    }
}
```

---

### 查詢變體 (Querying)

#### 1. 基本判斷

```rust
let x: Result<i32, &str> = Ok(10);
assert!(x.is_ok());
assert!(!x.is_err());

let y: Result<i32, &str> = Err("error");
assert!(!y.is_ok());
assert!(y.is_err());
```

**方法簽名**:
```rust
pub const fn is_ok(&self) -> bool
pub const fn is_err(&self) -> bool
```

#### 2. 條件判斷 (Rust 1.70+)

對內部值套用條件函數進行判斷:

```rust
let x: Result<i32, &str> = Ok(10);
assert!(x.is_ok_and(|v| v > 5));  // Ok 且值 > 5
assert!(!x.is_ok_and(|v| v > 20));

let y: Result<i32, &str> = Err("invalid");
assert!(y.is_err_and(|e| e.contains("invalid")));
```

**方法簽名**:
```rust
pub fn is_ok_and(self, f: impl FnOnce(T) -> bool) -> bool
pub fn is_err_and(self, f: impl FnOnce(E) -> bool) -> bool
```

**使用場景**:
```rust
// 檢查 I/O 操作是否成功且讀取到足夠數據
let result = file.read_to_end(&mut buffer);
if result.is_ok_and(|bytes| bytes > 100) {
    println!("讀取了足夠的數據");
}
```

---

### 提取內部值 (Extracting)

#### 1. Panic 系列 (不安全，僅用於確定成功的場景)

```rust
// expect: 自訂 panic 訊息
let x: Result<i32, &str> = Ok(10);
assert_eq!(x.expect("應該成功"), 10);

let y: Result<i32, &str> = Err("失敗");
// y.expect("操作失敗"); // panic: 操作失敗: "失敗"

// unwrap: 通用 panic 訊息
let x = Ok(10);
assert_eq!(x.unwrap(), 10);
```

**方法簽名**:
```rust
pub fn expect(self, msg: &str) -> T
pub fn unwrap(self) -> T
```

**最佳實踐**:
```rust
// ❌ 不好: unwrap 沒有提供上下文
let config = load_config().unwrap();

// ✅ 好: expect 提供清晰的錯誤上下文
let config = load_config()
    .expect("無法載入配置文件 config.toml");
```

#### 2. 提供預設值

```rust
// unwrap_or: 固定預設值
let x: Result<i32, &str> = Ok(10);
assert_eq!(x.unwrap_or(0), 10);

let y: Result<i32, &str> = Err("error");
assert_eq!(y.unwrap_or(0), 0);  // 返回預設值

// unwrap_or_else: 懶求值的預設值
let x = Ok(10);
assert_eq!(x.unwrap_or_else(|_| 0), 10);

let y: Result<i32, String> = Err("error".to_string());
assert_eq!(y.unwrap_or_else(|e| {
    eprintln!("發生錯誤: {}", e);
    0  // 計算預設值
}), 0);

// unwrap_or_default: 使用類型的 Default 實現
let x: Result<Vec<i32>, &str> = Err("error");
assert_eq!(x.unwrap_or_default(), Vec::<i32>::new());
```

**性能考量**:
```rust
// ❌ 不好: 即使 Ok 也會創建 Vec
result.unwrap_or(Vec::new())

// ✅ 好: 只在 Err 時才創建 Vec
result.unwrap_or_else(|_| Vec::new())

// ✅ 更好: 使用 Default
result.unwrap_or_default()
```

---

### 轉換與適配 (Transformation)

#### 1. 轉換為 Option

```rust
// ok(): Ok(v) → Some(v), Err(_) → None
let x: Result<i32, &str> = Ok(10);
assert_eq!(x.ok(), Some(10));

let y: Result<i32, &str> = Err("error");
assert_eq!(y.ok(), None);

// err(): Err(e) → Some(e), Ok(_) → None
let x: Result<i32, &str> = Ok(10);
assert_eq!(x.err(), None);

let y: Result<i32, &str> = Err("error");
assert_eq!(y.err(), Some("error"));
```

#### 2. transpose: Result<Option<T>> ↔ Option<Result<T>>

```rust
let x: Result<Option<i32>, &str> = Ok(Some(5));
let y: Option<Result<i32, &str>> = Some(Ok(5));
assert_eq!(x.transpose(), y);

let x: Result<Option<i32>, &str> = Ok(None);
assert_eq!(x.transpose(), None);

let x: Result<Option<i32>, &str> = Err("error");
assert_eq!(x.transpose(), Some(Err("error")));
```

**實戰案例**:
```rust
// 從配置文件中讀取可選項
fn get_optional_config(key: &str) -> Result<Option<String>, ConfigError> {
    let value = read_config(key)?;  // Result<String, ConfigError>
    Ok(if value.is_empty() { None } else { Some(value) })
}

// 使用 transpose 簡化
fn get_optional_config_v2(key: &str) -> Option<Result<String, ConfigError>> {
    read_config(key)
        .map(|v| if v.is_empty() { None } else { Some(v) })
        .transpose()
}
```

---

### Ok 變體操作 (成功路徑)

#### 1. map: 轉換成功值

```rust
let x: Result<i32, &str> = Ok(10);
let y = x.map(|v| v * 2);
assert_eq!(y, Ok(20));

let x: Result<i32, &str> = Err("error");
let y = x.map(|v| v * 2);  // Err 不執行轉換
assert_eq!(y, Err("error"));
```

**方法簽名**:
```rust
pub fn map<U, F>(self, f: F) -> Result<U, E>
where
    F: FnOnce(T) -> U
```

**實戰案例**:
```rust
// 解析字符串為數字，然後加倍
fn parse_and_double(s: &str) -> Result<i32, std::num::ParseIntError> {
    s.parse::<i32>().map(|n| n * 2)
}

assert_eq!(parse_and_double("10"), Ok(20));
```

#### 2. and_then: 鏈式可失敗操作

```rust
let x: Result<i32, &str> = Ok(10);
let y = x.and_then(|v| {
    if v > 5 {
        Ok(v * 2)
    } else {
        Err("值太小")
    }
});
assert_eq!(y, Ok(20));
```

**方法簽名**:
```rust
pub fn and_then<U, F>(self, f: F) -> Result<U, E>
where
    F: FnOnce(T) -> Result<U, E>
```

**map vs and_then**:

```rust
// map: 轉換函數不會失敗
let result = Ok(10)
    .map(|x| x * 2);  // i32 -> i32

// and_then: 轉換函數可能失敗
let result = Ok(10)
    .and_then(|x| divide(x, 2));  // i32 -> Result<i32, _>
```

**鏈式調用**:
```rust
fn process_user_input(input: &str) -> Result<i32, String> {
    input.trim()
        .parse::<i32>()
        .map_err(|e| format!("解析失敗: {}", e))
        .and_then(|n| {
            if n > 0 {
                Ok(n)
            } else {
                Err("數字必須為正數".to_string())
            }
        })
        .map(|n| n * 2)
}

assert_eq!(process_user_input("10"), Ok(20));
assert!(process_user_input("-5").is_err());
```

#### 3. map_or / map_or_else: 帶預設值的轉換

```rust
// map_or: 成功則轉換，失敗返回預設值
let x: Result<i32, &str> = Ok(10);
assert_eq!(x.map_or(0, |v| v * 2), 20);

let y: Result<i32, &str> = Err("error");
assert_eq!(y.map_or(0, |v| v * 2), 0);

// map_or_else: 失敗時執行函數產生預設值
let x: Result<i32, &str> = Err("error");
let result = x.map_or_else(
    |e| {
        eprintln!("錯誤: {}", e);
        0
    },
    |v| v * 2
);
assert_eq!(result, 0);
```

**方法簽名**:
```rust
pub fn map_or<U, F>(self, default: U, f: F) -> U
where
    F: FnOnce(T) -> U

pub fn map_or_else<U, D, F>(self, default: D, f: F) -> U
where
    D: FnOnce(E) -> U,
    F: FnOnce(T) -> U
```

#### 4. inspect: 觀察成功值 (Rust 1.76+)

```rust
let x: Result<i32, &str> = Ok(10);
let y = x.inspect(|v| println!("成功值: {}", v));
assert_eq!(y, Ok(10));  // 返回原始值

// 用於除錯
fn process() -> Result<i32, String> {
    read_value()?
        .inspect(|v| eprintln!("讀取到: {}", v))
        .map(|v| v * 2)
        .inspect(|v| eprintln!("轉換後: {}", v))
}
```

---

### Err 變體操作 (錯誤路徑)

#### 1. map_err: 轉換錯誤類型

```rust
let x: Result<i32, &str> = Err("error");
let y = x.map_err(|e| format!("錯誤: {}", e));
assert_eq!(y, Err("錯誤: error".to_string()));
```

**實戰案例**:
```rust
use std::fs;
use std::io;

#[derive(Debug)]
enum AppError {
    Io(String),
    Parse(String),
}

fn read_number(path: &str) -> Result<i32, AppError> {
    fs::read_to_string(path)
        .map_err(|e| AppError::Io(format!("讀取檔案失敗: {}", e)))?
        .trim()
        .parse()
        .map_err(|e| AppError::Parse(format!("解析失敗: {}", e)))
}
```

#### 2. or / or_else: 提供替代結果

```rust
// or: 失敗時使用固定替代結果
let x: Result<i32, &str> = Err("error");
let y = x.or(Ok(100));
assert_eq!(y, Ok(100));

// or_else: 失敗時執行函數產生替代結果
let x: Result<i32, &str> = Err("primary failed");
let y = x.or_else(|_| {
    // 嘗試備用方案
    Ok(fallback_value())
});
```

**重試模式**:
```rust
fn fetch_with_retry(url: &str) -> Result<String, reqwest::Error> {
    fetch_from_primary(url)
        .or_else(|_| fetch_from_secondary(url))
        .or_else(|_| fetch_from_tertiary(url))
}
```

#### 3. inspect_err: 觀察錯誤值 (Rust 1.76+)

```rust
let x: Result<i32, String> = Err("失敗".to_string());
let y = x.inspect_err(|e| eprintln!("錯誤: {}", e));
assert_eq!(y, Err("失敗".to_string()));

// 記錄錯誤但不中斷流程
fn process() -> Result<i32, String> {
    risky_operation()
        .inspect_err(|e| log::error!("操作失敗: {}", e))
}
```

---

### 邏輯組合操作

#### 1. and: 成功時繼續

```rust
let x: Result<i32, &str> = Ok(10);
let y: Result<&str, &str> = Ok("hello");
assert_eq!(x.and(y), Ok("hello"));  // 返回第二個 Result

let x: Result<i32, &str> = Err("error1");
let y: Result<&str, &str> = Ok("hello");
assert_eq!(x.and(y), Err("error1"));  // 保持第一個錯誤
```

**方法簽名**:
```rust
pub fn and<U>(self, res: Result<U, E>) -> Result<U, E>
```

**實戰案例**:
```rust
// 兩個操作都必須成功
let result = validate_input(input).and(validate_permission(user));
```

---

## 🎯 Option<T> 完整指南

### 定義與本質

```rust
pub enum Option<T> {
    Some(T),  // 有值
    None,     // 無值
}
```

**核心特性**:
- 用於表示「可能存在的值」，替代其他語言的 `null`
- 編譯器強制處理 `None` 情況，消除空指針錯誤
- 與 `Result` 類似的 API 設計，易於學習

### 基本使用示例

```rust
fn find_user(id: u32) -> Option<String> {
    if id == 1 {
        Some("Alice".to_string())
    } else {
        None
    }
}

fn main() {
    match find_user(1) {
        Some(name) => println!("找到用戶: {}", name),
        None => println!("用戶不存在"),
    }
}
```

---

### 查詢變體

```rust
let x = Some(10);
assert!(x.is_some());
assert!(!x.is_none());

let y: Option<i32> = None;
assert!(!y.is_some());
assert!(y.is_none());

// is_some_and: 有值且滿足條件 (Rust 1.70+)
let x = Some(10);
assert!(x.is_some_and(|v| v > 5));
assert!(!x.is_some_and(|v| v > 20));
```

---

### 提取內部值

```rust
// expect / unwrap: 與 Result 相同
let x = Some(10);
assert_eq!(x.expect("應該有值"), 10);
assert_eq!(x.unwrap(), 10);

// unwrap_or: 提供預設值
let x = Some(10);
assert_eq!(x.unwrap_or(0), 10);

let y: Option<i32> = None;
assert_eq!(y.unwrap_or(0), 0);

// unwrap_or_else: 懶求值的預設值
let x: Option<String> = None;
let result = x.unwrap_or_else(|| {
    eprintln!("使用預設值");
    String::from("default")
});

// unwrap_or_default
let x: Option<Vec<i32>> = None;
assert_eq!(x.unwrap_or_default(), vec![]);
```

---

### 轉換操作

#### 1. 轉換為 Result

```rust
// ok_or: None → Err(固定錯誤)
let x = Some(10);
assert_eq!(x.ok_or("錯誤"), Ok(10));

let y: Option<i32> = None;
assert_eq!(y.ok_or("錯誤"), Err("錯誤"));

// ok_or_else: None → Err(計算的錯誤)
let x: Option<i32> = None;
let result = x.ok_or_else(|| {
    format!("找不到值")
});
assert_eq!(result, Err("找不到值".to_string()));
```

**實戰案例**:
```rust
// 將查找失敗轉換為明確的錯誤
fn get_user(id: u32) -> Result<User, UserError> {
    find_user_in_db(id)  // Option<User>
        .ok_or_else(|| UserError::NotFound(id))
}
```

#### 2. transpose: Option<Result<T>> ↔ Result<Option<T>>

```rust
let x: Option<Result<i32, &str>> = Some(Ok(5));
let y: Result<Option<i32>, &str> = Ok(Some(5));
assert_eq!(x.transpose(), y);

let x: Option<Result<i32, &str>> = Some(Err("error"));
let y: Result<Option<i32>, &str> = Err("error");
assert_eq!(x.transpose(), y);

let x: Option<Result<i32, &str>> = None;
assert_eq!(x.transpose(), Ok(None));
```

---

### 映射操作

#### 1. map: 轉換值

```rust
let x = Some(10);
let y = x.map(|v| v * 2);
assert_eq!(y, Some(20));

let x: Option<i32> = None;
let y = x.map(|v| v * 2);
assert_eq!(y, None);
```

#### 2. map_or / map_or_else

```rust
let x = Some(10);
assert_eq!(x.map_or(0, |v| v * 2), 20);

let y: Option<i32> = None;
assert_eq!(y.map_or(0, |v| v * 2), 0);

// map_or_else
let result = None.map_or_else(
    || {
        eprintln!("沒有值");
        0
    },
    |v| v * 2
);
```

#### 3. and_then: 鏈式操作

```rust
fn divide(a: i32, b: i32) -> Option<i32> {
    if b == 0 { None } else { Some(a / b) }
}

let x = Some(10);
let y = x.and_then(|v| divide(v, 2));
assert_eq!(y, Some(5));

let z = x.and_then(|v| divide(v, 0));
assert_eq!(z, None);
```

#### 4. filter: 條件過濾

```rust
let x = Some(10);
let y = x.filter(|v| *v > 5);
assert_eq!(y, Some(10));

let z = x.filter(|v| *v > 20);
assert_eq!(z, None);
```

**實戰案例**:
```rust
// 查找年齡大於 18 的用戶
fn find_adult_user(id: u32) -> Option<User> {
    find_user(id).filter(|user| user.age >= 18)
}
```

---

### 組合操作

```rust
// or: 提供替代值
let x = Some(10);
let y = None;
assert_eq!(x.or(y), Some(10));

let x: Option<i32> = None;
let y = Some(20);
assert_eq!(x.or(y), Some(20));

// or_else: 懶求值的替代值
let x: Option<i32> = None;
let result = x.or_else(|| {
    Some(compute_default())
});

// xor: 異或 (只有一個為 Some)
let x = Some(10);
let y = None;
assert_eq!(x.xor(y), Some(10));

let x = Some(10);
let y = Some(20);
assert_eq!(x.xor(y), None);  // 兩個都是 Some

// zip: 組合兩個 Option
let x = Some(10);
let y = Some(20);
assert_eq!(x.zip(y), Some((10, 20)));

let x = Some(10);
let y: Option<i32> = None;
assert_eq!(x.zip(y), None);

// unzip: 拆分元組
let x = Some((10, 20));
assert_eq!(x.unzip(), (Some(10), Some(20)));
```

---

## 🔧 ? 運算子與 Try Trait

### ? 運算子原理

`?` 運算子是 Rust 錯誤處理的語法糖，用於簡化錯誤傳播:

```rust
// 使用 ?
fn read_file() -> Result<String, io::Error> {
    let content = fs::read_to_string("file.txt")?;
    Ok(content)
}

// 等價於
fn read_file_verbose() -> Result<String, io::Error> {
    let content = match fs::read_to_string("file.txt") {
        Ok(v) => v,
        Err(e) => return Err(e),
    };
    Ok(content)
}
```

### Try Trait (不穩定)

`?` 運算子基於 `Try` trait 實現:

```rust
// 簡化版定義 (實際更複雜)
pub trait Try {
    type Output;
    type Residual;

    fn from_output(output: Self::Output) -> Self;
    fn branch(self) -> ControlFlow<Self::Residual, Self::Output>;
}
```

**同時支持 Result 和 Option**:

```rust
fn find_and_double(id: u32) -> Option<i32> {
    let user = find_user(id)?;  // Option<User>
    Some(user.age * 2)
}

fn read_and_parse() -> Result<i32, Box<dyn Error>> {
    let content = fs::read_to_string("file.txt")?;  // Result
    let num = content.trim().parse()?;  // Result
    Ok(num)
}
```

### 自動錯誤轉換

`?` 會自動調用 `From` trait 轉換錯誤類型:

```rust
use std::fs;
use std::io;

fn read_number() -> Result<i32, io::Error> {
    let s = fs::read_to_string("number.txt")?;  // io::Error
    let n = s.trim().parse()?;  // ParseIntError → 編譯錯誤!
    Ok(n)
}

// 需要統一錯誤類型
fn read_number_fixed() -> Result<i32, Box<dyn Error>> {
    let s = fs::read_to_string("number.txt")?;
    let n = s.trim().parse()?;  // 自動轉換為 Box<dyn Error>
    Ok(n)
}
```

---

## 🏗️ 實戰模式與最佳實踐

### 模式 1: 多層錯誤處理

```rust
fn process_user_data(user_id: u32) -> Result<Report, AppError> {
    let user = fetch_user(user_id)?
        .inspect(|u| log::info!("找到用戶: {}", u.name));
    
    let data = fetch_user_data(user.id)?
        .inspect(|d| log::debug!("資料大小: {} bytes", d.len()));
    
    let report = generate_report(&user, &data)
        .inspect_err(|e| log::error!("生成報告失敗: {}", e))?;
    
    Ok(report)
}
```

### 模式 2: 優雅的預設值處理

```rust
// ❌ 不好: 多次調用
let config = load_config().unwrap_or(Config::default());
let timeout = config.timeout.unwrap_or(30);
let retry = config.retry.unwrap_or(3);

// ✅ 好: 一次性處理
let config = load_config().unwrap_or_default();
let timeout = config.timeout.unwrap_or(30);
let retry = config.retry.unwrap_or(3);

// ✅ 更好: Builder 模式
let config = ConfigBuilder::new()
    .from_file("config.toml")
    .unwrap_or_default()
    .with_timeout(30)
    .with_retry(3)
    .build();
```

### 模式 3: Result 與 Option 轉換

```rust
// Option → Result
fn get_user(id: u32) -> Result<User, UserError> {
    users.get(&id)
        .cloned()
        .ok_or(UserError::NotFound(id))
}

// Result → Option (忽略錯誤)
fn try_get_cached(key: &str) -> Option<String> {
    cache.get(key)
        .ok()  // 將 Result<String, _> 轉為 Option<String>
}
```

### 模式 4: 鏈式組合

```rust
fn process_input(input: &str) -> Result<i32, String> {
    input.trim()
        .parse::<i32>()
        .map_err(|e| format!("解析失敗: {}", e))?
        .checked_mul(2)
        .ok_or_else(|| "溢出".to_string())?
        .checked_add(100)
        .ok_or_else(|| "溢出".to_string())
}
```

### 模式 5: 早期返回 vs 鏈式調用

```rust
// 早期返回: 適合複雜邏輯
fn process_early_return(input: &str) -> Result<i32, String> {
    let num = input.trim().parse::<i32>()
        .map_err(|e| format!("解析失敗: {}", e))?;
    
    if num < 0 {
        return Err("數字不能為負".to_string());
    }
    
    if num > 1000 {
        return Err("數字過大".to_string());
    }
    
    Ok(num * 2)
}

// 鏈式調用: 適合簡單轉換
fn process_chaining(input: &str) -> Result<i32, String> {
    input.trim()
        .parse::<i32>()
        .map_err(|e| format!("解析失敗: {}", e))
        .and_then(|n| {
            if n >= 0 && n <= 1000 {
                Ok(n * 2)
            } else {
                Err("數字範圍錯誤".to_string())
            }
        })
}
```

---

## 📊 性能考量

### unwrap_or vs unwrap_or_else

```rust
// ❌ 慢: 每次都創建 Vec (即使 Ok)
let v = result.unwrap_or(Vec::new());

// ✅ 快: 只在 Err 時才創建
let v = result.unwrap_or_else(|_| Vec::new());

// ✅ 最快: 使用 Default
let v = result.unwrap_or_default();
```

### map vs and_then 選擇

```rust
// map: 無額外開銷
let doubled = Some(10).map(|x| x * 2);

// and_then: 包裹額外的 Result/Option
let result = Some(10).and_then(|x| Some(x * 2));
//                              ^^^^^^^^^^^^^ 額外的包裹
```

---

## 🎓 進階主題

### Rust 2024 Edition 新特性

#### 1. 改進的 ? 錯誤訊息 (Rust 1.75+)

```rust
// 現在編譯器會提供更清晰的錯誤訊息
fn foo() -> Result<i32, io::Error> {
    let s = fs::read_to_string("file.txt")?;
    let n = s.parse()?;  // 錯誤: ParseIntError 無法轉換為 io::Error
    //      ^^^^^^^^^ 編譯器會建議使用 map_err
    Ok(n)
}
```

#### 2. Option::inspect 和 Result::inspect (Rust 1.76+)

已在前面章節介紹。

### 自訂 Try Trait (未來)

```rust
// 可能在未來穩定
impl Try for MyCustomType {
    type Output = Success;
    type Residual = Failure;
    
    // 實現細節...
}
```

---

## 📚 完整範例: 文件處理器

```rust
use std::fs;
use std::io;
use std::path::Path;

#[derive(Debug)]
enum ProcessError {
    Io(io::Error),
    Parse(String),
    Validation(String),
}

impl From<io::Error> for ProcessError {
    fn from(e: io::Error) -> Self {
        ProcessError::Io(e)
    }
}

fn process_file(path: impl AsRef<Path>) -> Result<Vec<i32>, ProcessError> {
    // 讀取文件
    let content = fs::read_to_string(path)?
        .inspect(|s| log::debug!("讀取了 {} 字節", s.len()));
    
    // 解析每一行
    let numbers: Result<Vec<i32>, _> = content
        .lines()
        .map(|line| {
            line.trim()
                .parse::<i32>()
                .map_err(|e| ProcessError::Parse(
                    format!("無法解析 '{}': {}", line, e)
                ))
        })
        .collect();
    
    let numbers = numbers?;
    
    // 驗證
    if numbers.is_empty() {
        return Err(ProcessError::Validation(
            "文件為空".to_string()
        ));
    }
    
    // 過濾有效值
    let valid_numbers: Vec<i32> = numbers
        .into_iter()
        .filter(|n| *n > 0)
        .collect();
    
    if valid_numbers.is_empty() {
        return Err(ProcessError::Validation(
            "沒有有效的正數".to_string()
        ));
    }
    
    Ok(valid_numbers)
}

fn main() {
    match process_file("numbers.txt") {
        Ok(numbers) => {
            println!("處理了 {} 個數字", numbers.len());
            println!("總和: {}", numbers.iter().sum::<i32>());
        }
        Err(e) => {
            eprintln!("錯誤: {:?}", e);
        }
    }
}
```

---

## 🔍 常見陷阱

### 陷阱 1: unwrap 濫用

```rust
// ❌ 危險: 生產環境可能 panic
let config = load_config().unwrap();

// ✅ 安全: 使用 expect 提供上下文
let config = load_config()
    .expect("配置文件缺失，請檢查 config.toml");

// ✅ 更好: 處理錯誤
let config = load_config().unwrap_or_else(|e| {
    eprintln!("載入配置失敗: {}", e);
    Config::default()
});
```

### 陷阱 2: 過度嵌套

```rust
// ❌ 難讀
match result1 {
    Ok(v1) => match result2 {
        Ok(v2) => match result3 {
            Ok(v3) => Some(v1 + v2 + v3),
            Err(_) => None,
        },
        Err(_) => None,
    },
    Err(_) => None,
}

// ✅ 清晰
let sum = result1
    .and_then(|v1| result2.map(|v2| (v1, v2)))
    .and_then(|(v1, v2)| result3.map(|v3| v1 + v2 + v3))
    .ok();
```

### 陷阱 3: 忽略 Err 的上下文

```rust
// ❌ 損失錯誤信息
let data = fetch_data().ok();

// ✅ 記錄錯誤
let data = fetch_data()
    .inspect_err(|e| log::error!("獲取數據失敗: {}", e))
    .ok();
```

---

## 📖 參考資料

1. [The Rust Programming Language - Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html)
2. [Rust By Example - Error Handling](https://doc.rust-lang.org/rust-by-example/error.html)
3. [std::result::Result Documentation](https://doc.rust-lang.org/std/result/enum.Result.html)
4. [std::option::Option Documentation](https://doc.rust-lang.org/std/option/enum.Option.html)
5. [RFC 0243 - Trait-based exception handling](https://rust-lang.github.io/rfcs/0243-trait-based-exception-handling.html)
6. [Error Handling in Rust (Blog Post)](https://blog.burntsushi.net/rust-error-handling/)

---

*最後更新: 2025-01-17*  
*Rust 版本: 1.90+*
