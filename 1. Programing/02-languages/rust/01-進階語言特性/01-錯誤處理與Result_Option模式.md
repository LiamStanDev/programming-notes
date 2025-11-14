# 01 - 錯誤處理與 Result/Option 模式

## 概述

Rust 的錯誤處理系統基於型別系統,使用 `Result<T, E>` 和 `Option<T>` 兩個核心枚舉來處理可能的錯誤和空值情況。這種設計迫使開發者顯式處理錯誤,避免了傳統語言中的 null pointer exceptions 和未處理異常。

### 核心理念

- **顯式優於隱式**: 錯誤必須被顯式處理或傳播
- **型別安全**: 編譯器保證所有可能的錯誤路徑都被考慮
- **零成本抽象**: 編譯後性能與手動錯誤檢查相當

---

## `Result<T, E>` 完整操作手冊

`Result<T, E>` 是一個泛型枚舉,表示操作可能成功 (`Ok(T)`) 或失敗 (`Err(E)`)。

```rust
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

### 1. 查詢狀態

#### `is_ok()` 和 `is_err()`

檢查 Result 是成功還是失敗。

```rust
fn main() {
    let success: Result<i32, &str> = Ok(42);
    let failure: Result<i32, &str> = Err("error occurred");

    assert!(success.is_ok());
    assert!(failure.is_err());
}
```

#### `is_ok_and()` 和 `is_err_and()` (Rust 1.70+)

結合條件檢查,避免多次匹配。

```rust
fn main() {
    let result: Result<i32, i32> = Ok(42);
    
    // 檢查是否成功且值滿足條件
    assert!(result.is_ok_and(|&x| x > 40));
    
    let error: Result<i32, i32> = Err(404);
    // 檢查是否失敗且錯誤碼匹配
    assert!(error.is_err_and(|&e| e == 404));
}
```

### 2. 提取值

#### `unwrap()` - 危險操作

直接提取值,失敗時 panic。**僅用於原型開發或確定不會失敗的場景。**

```rust
fn main() {
    let result: Result<i32, &str> = Ok(42);
    assert_eq!(result.unwrap(), 42);
    
    // 以下會 panic
    // let failure: Result<i32, &str> = Err("failed");
    // failure.unwrap(); // thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: "failed"'
}
```

#### `expect()` - 帶錯誤訊息的 unwrap

提供自定義 panic 訊息,便於調試。

```rust
fn main() {
    let config = std::fs::read_to_string("config.toml")
        .expect("配置文件必須存在"); // 更清晰的錯誤訊息
}
```

#### `unwrap_or()` - 提供默認值

失敗時返回指定的默認值。

```rust
fn main() {
    let result: Result<i32, &str> = Err("error");
    assert_eq!(result.unwrap_or(0), 0);
    
    let success: Result<i32, &str> = Ok(42);
    assert_eq!(success.unwrap_or(0), 42);
}
```

#### `unwrap_or_else()` - 延遲計算默認值

使用閉包計算默認值,僅在失敗時執行。

```rust
fn main() {
    let error: Result<i32, String> = Err("not found".to_string());
    
    let value = error.unwrap_or_else(|e| {
        eprintln!("Error occurred: {}", e);
        0 // 默認值
    });
    
    assert_eq!(value, 0);
}
```

#### `unwrap_or_default()` - 使用型別默認值

要求 `T` 實現 `Default` trait。

```rust
fn main() {
    let result: Result<Vec<i32>, &str> = Err("error");
    assert_eq!(result.unwrap_or_default(), Vec::<i32>::new());
    
    let result: Result<i32, &str> = Err("error");
    assert_eq!(result.unwrap_or_default(), 0); // i32 默認值為 0
}
```

### 3. 轉換操作

#### `ok()` - 轉換為 Option

丟棄錯誤訊息,僅保留成功值。

```rust
fn main() {
    let result: Result<i32, &str> = Ok(42);
    assert_eq!(result.ok(), Some(42));
    
    let error: Result<i32, &str> = Err("failed");
    assert_eq!(error.ok(), None);
}
```

#### `err()` - 提取錯誤

丟棄成功值,僅保留錯誤。

```rust
fn main() {
    let result: Result<i32, &str> = Err("error message");
    assert_eq!(result.err(), Some("error message"));
    
    let success: Result<i32, &str> = Ok(42);
    assert_eq!(success.err(), None);
}
```

#### `transpose()` - 轉置嵌套結構

將 `Result<Option<T>, E>` 轉為 `Option<Result<T, E>>`。

```rust
fn main() {
    let result: Result<Option<i32>, &str> = Ok(Some(42));
    let transposed: Option<Result<i32, &str>> = result.transpose();
    assert_eq!(transposed, Some(Ok(42)));
    
    let none_result: Result<Option<i32>, &str> = Ok(None);
    assert_eq!(none_result.transpose(), None);
}
```

### 4. Ok 值操作

#### `map()` - 轉換成功值

僅在成功時應用函數,錯誤直接傳遞。

```rust
fn main() {
    let result: Result<i32, &str> = Ok(21);
    let doubled = result.map(|x| x * 2);
    assert_eq!(doubled, Ok(42));
    
    let error: Result<i32, &str> = Err("error");
    let mapped = error.map(|x| x * 2);
    assert_eq!(mapped, Err("error"));
}
```

#### `and_then()` - 鏈式操作 (flatMap)

用於**連接多個可能失敗的操作**,避免嵌套的 `Result<Result<T, E>, E>`。

```rust
fn parse_number(s: &str) -> Result<i32, &str> {
    s.parse::<i32>().map_err(|_| "解析失敗")
}

fn double_if_positive(n: i32) -> Result<i32, &str> {
    if n > 0 {
        Ok(n * 2)
    } else {
        Err("數字必須為正數")
    }
}

fn main() {
    let result = parse_number("21")
        .and_then(double_if_positive);
    assert_eq!(result, Ok(42));
    
    let result = parse_number("-5")
        .and_then(double_if_positive);
    assert_eq!(result, Err("數字必須為正數"));
}
```

#### `map_or()` - 提供默認值的 map

```rust
fn main() {
    let result: Result<i32, &str> = Ok(21);
    assert_eq!(result.map_or(0, |x| x * 2), 42);
    
    let error: Result<i32, &str> = Err("error");
    assert_eq!(error.map_or(0, |x| x * 2), 0);
}
```

#### `map_or_else()` - 處理兩種情況

第一個閉包處理錯誤,第二個處理成功。

```rust
fn main() {
    let result: Result<i32, String> = Err("not found".to_string());
    
    let value = result.map_or_else(
        |e| {
            eprintln!("Error: {}", e);
            0
        },
        |x| x * 2
    );
    
    assert_eq!(value, 0);
}
```

#### `inspect()` - 副作用操作 (Rust 1.76+)

用於調試,不改變 Result。

```rust
fn main() {
    let result: Result<i32, &str> = Ok(42);
    
    result
        .inspect(|x| println!("成功值: {}", x))
        .map(|x| x * 2);
}
```

### 5. Err 值操作

#### `map_err()` - 轉換錯誤類型

用於統一錯誤類型或添加上下文。

```rust
fn main() {
    let result: Result<i32, i32> = Err(404);
    let mapped = result.map_err(|code| format!("錯誤碼: {}", code));
    assert_eq!(mapped, Err("錯誤碼: 404".to_string()));
}
```

#### `or()` - 提供替代 Result

失敗時使用另一個 Result。

```rust
fn main() {
    let primary: Result<i32, &str> = Err("primary failed");
    let fallback: Result<i32, &str> = Ok(42);
    
    assert_eq!(primary.or(fallback), Ok(42));
}
```

#### `or_else()` - 延遲計算替代值

```rust
fn main() {
    let result: Result<i32, &str> = Err("error");
    
    let fallback = result.or_else(|e| {
        eprintln!("原錯誤: {}", e);
        Ok(0) // 提供默認值
    });
    
    assert_eq!(fallback, Ok(0));
}
```

#### `inspect_err()` - 檢查錯誤 (Rust 1.76+)

```rust
fn main() {
    let result: Result<i32, &str> = Err("failed");
    
    result
        .inspect_err(|e| eprintln!("錯誤: {}", e))
        .ok();
}
```

### 6. 邏輯組合

#### `and()` - 兩者都成功

```rust
fn main() {
    let a: Result<i32, &str> = Ok(1);
    let b: Result<i32, &str> = Ok(2);
    assert_eq!(a.and(b), Ok(2)); // 返回第二個值
    
    let c: Result<i32, &str> = Err("error");
    assert_eq!(a.and(c), Err("error"));
}
```

---

## `Option<T>` 完整操作手冊

`Option<T>` 用於表示值可能不存在的情況,避免 null pointer。

```rust
pub enum Option<T> {
    Some(T),
    None,
}
```

### 1. 查詢狀態

#### `is_some()` 和 `is_none()`

```rust
fn main() {
    let some = Some(42);
    let none: Option<i32> = None;
    
    assert!(some.is_some());
    assert!(none.is_none());
}
```

#### `is_some_and()` (Rust 1.70+)

```rust
fn main() {
    let number = Some(42);
    assert!(number.is_some_and(|x| x > 40));
}
```

### 2. 提取值

方法與 Result 類似:
- `unwrap()`, `expect()`, `unwrap_or()`, `unwrap_or_else()`, `unwrap_or_default()`

```rust
fn main() {
    let some = Some(42);
    assert_eq!(some.unwrap(), 42);
    assert_eq!(some.unwrap_or(0), 42);
    
    let none: Option<i32> = None;
    assert_eq!(none.unwrap_or(0), 0);
}
```

### 3. 轉換操作

#### `ok_or()` - 轉換為 Result

將 None 轉為指定錯誤。

```rust
fn main() {
    let some = Some(42);
    assert_eq!(some.ok_or("沒有值"), Ok(42));
    
    let none: Option<i32> = None;
    assert_eq!(none.ok_or("沒有值"), Err("沒有值"));
}
```

#### `ok_or_else()` - 延遲錯誤計算

```rust
fn main() {
    let none: Option<i32> = None;
    let result = none.ok_or_else(|| format!("錯誤: 值不存在"));
    assert_eq!(result, Err("錯誤: 值不存在".to_string()));
}
```

#### `transpose()` - 轉置 `Option<Result<T, E>>`

```rust
fn main() {
    let option: Option<Result<i32, &str>> = Some(Ok(42));
    let result: Result<Option<i32>, &str> = option.transpose();
    assert_eq!(result, Ok(Some(42)));
}
```

### 4. 映射與過濾

#### `map()`

```rust
fn main() {
    let some = Some(21);
    assert_eq!(some.map(|x| x * 2), Some(42));
    
    let none: Option<i32> = None;
    assert_eq!(none.map(|x| x * 2), None);
}
```

#### `filter()` - 條件過濾

```rust
fn main() {
    let number = Some(42);
    assert_eq!(number.filter(|&x| x > 40), Some(42));
    assert_eq!(number.filter(|&x| x < 40), None);
}
```

#### `and_then()` - 鏈式操作

```rust
fn divide(numerator: i32, denominator: i32) -> Option<i32> {
    if denominator == 0 {
        None
    } else {
        Some(numerator / denominator)
    }
}

fn main() {
    let result = Some(10)
        .and_then(|x| divide(x, 2))
        .and_then(|x| divide(x, 5));
    
    assert_eq!(result, Some(1));
}
```

### 5. 組合操作

#### `or()` - 提供替代值

```rust
fn main() {
    let none: Option<i32> = None;
    let some = Some(42);
    
    assert_eq!(none.or(some), Some(42));
    assert_eq!(some.or(None), Some(42));
}
```

#### `xor()` - 互斥或

```rust
fn main() {
    assert_eq!(Some(1).xor(Some(2)), None); // 兩者都有
    assert_eq!(Some(1).xor(None), Some(1)); // 僅一個有
    assert_eq!(None.xor(Some(2)), Some(2));
    assert_eq!(None.xor(None), None);
}
```

#### `zip()` - 組合兩個 Option

```rust
fn main() {
    let a = Some(1);
    let b = Some(2);
    assert_eq!(a.zip(b), Some((1, 2)));
    
    let c: Option<i32> = None;
    assert_eq!(a.zip(c), None);
}
```

#### `unzip()` - 解構 Option 元組

```rust
fn main() {
    let pair = Some((1, "hello"));
    assert_eq!(pair.unzip(), (Some(1), Some("hello")));
    
    let none: Option<(i32, &str)> = None;
    assert_eq!(none.unzip(), (None, None));
}
```

---

## `?` 運算子與 `Try` Trait

### `?` 運算子原理

`?` 是語法糖,用於簡化錯誤傳播。

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_file_content(path: &str) -> io::Result<String> {
    let mut file = File::open(path)?; // 等價於下方代碼
    let mut content = String::new();
    file.read_to_string(&mut content)?;
    Ok(content)
}

// ? 運算子的展開形式
fn read_file_content_explicit(path: &str) -> io::Result<String> {
    let mut file = match File::open(path) {
        Ok(f) => f,
        Err(e) => return Err(e.into()), // 自動轉換錯誤類型
    };
    
    let mut content = String::new();
    match file.read_to_string(&mut content) {
        Ok(_) => Ok(content),
        Err(e) => Err(e.into()),
    }
}
```

### `?` 用於 Option

```rust
fn get_first_char(s: &str) -> Option<char> {
    s.chars().next()
}

fn process(input: &str) -> Option<char> {
    let first = get_first_char(input)?; // None 會提前返回
    Some(first.to_ascii_uppercase())
}

fn main() {
    assert_eq!(process("hello"), Some('H'));
    assert_eq!(process(""), None);
}
```

### Try Trait (Unstable)

`?` 運算子基於 `Try` trait 實現,目前仍在 nightly 階段。

---

## 自定義錯誤類型

### 實現 `std::error::Error` Trait

```rust
use std::fmt;

#[derive(Debug)]
enum MyError {
    NotFound,
    PermissionDenied,
    InvalidInput(String),
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            MyError::NotFound => write!(f, "資源未找到"),
            MyError::PermissionDenied => write!(f, "權限被拒絕"),
            MyError::InvalidInput(msg) => write!(f, "無效輸入: {}", msg),
        }
    }
}

impl std::error::Error for MyError {}

fn do_something() -> Result<(), MyError> {
    Err(MyError::NotFound)
}

fn main() {
    match do_something() {
        Ok(_) => println!("成功"),
        Err(e) => eprintln!("錯誤: {}", e),
    }
}
```

---

## `thiserror` 與 `anyhow` 使用場景

### `thiserror` - 庫開發者使用

用於定義結構化的錯誤類型,適合庫作者。

```toml
[dependencies]
thiserror = "1.0"
```

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DatabaseError {
    #[error("連接失敗: {0}")]
    ConnectionFailed(String),
    
    #[error("查詢錯誤")]
    QueryError(#[from] sqlx::Error),
    
    #[error("記錄未找到: id={id}")]
    NotFound { id: i64 },
}

fn query_user(id: i64) -> Result<String, DatabaseError> {
    if id == 0 {
        return Err(DatabaseError::NotFound { id });
    }
    Ok("User".to_string())
}
```

### `anyhow` - 應用程式使用

用於應用程式快速開發,提供靈活的錯誤處理。

```toml
[dependencies]
anyhow = "1.0"
```

```rust
use anyhow::{Context, Result};
use std::fs;

fn read_config() -> Result<String> {
    let content = fs::read_to_string("config.toml")
        .context("無法讀取配置文件")?;
    
    // 添加上下文訊息
    let parsed = content.parse::<i32>()
        .context("配置文件格式錯誤")?;
    
    Ok(content)
}

fn main() -> Result<()> {
    let config = read_config()?;
    println!("配置: {}", config);
    Ok(())
}
```

### 選擇建議

| 使用場景 | 推薦工具 | 理由 |
|---------|---------|------|
| 庫開發 | `thiserror` | 提供明確的錯誤類型給使用者 |
| 應用程式 | `anyhow` | 快速開發,靈活處理多種錯誤 |
| 性能敏感 | 手動實現 | 避免額外依賴和運行時開銷 |

---

## 錯誤傳播與上下文添加

### 使用 `.context()` 添加上下文

```rust
use anyhow::{Context, Result};
use std::fs;

fn process_file(path: &str) -> Result<()> {
    let content = fs::read_to_string(path)
        .with_context(|| format!("讀取文件失敗: {}", path))?;
    
    let number: i32 = content.trim().parse()
        .context("文件內容必須是數字")?;
    
    println!("數字: {}", number);
    Ok(())
}
```

### 自定義上下文包裝

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct ContextError {
    message: String,
    source: Box<dyn Error>,
}

impl fmt::Display for ContextError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}: {}", self.message, self.source)
    }
}

impl Error for ContextError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        Some(self.source.as_ref())
    }
}
```

---

## Rust 2024 Edition 錯誤處理新特性

### 1. `?` 在 `main` 中使用 (已穩定)

```rust
use std::fs;
use std::io;

fn main() -> io::Result<()> {
    let content = fs::read_to_string("file.txt")?;
    println!("{}", content);
    Ok(())
}
```

### 2. Try blocks (Unstable)

```rust
#![feature(try_blocks)]

fn main() {
    let result: Result<i32, &str> = try {
        let x = Some(10).ok_or("沒有值")?;
        let y = Some(20).ok_or("沒有值")?;
        x + y
    };
    
    println!("{:?}", result);
}
```

### 3. 錯誤鏈追蹤改進

Rust 1.76+ 改進了錯誤鏈的 Debug 輸出。

```rust
use std::error::Error;

fn print_error_chain(e: &dyn Error) {
    eprintln!("錯誤: {}", e);
    let mut source = e.source();
    while let Some(e) = source {
        eprintln!("  原因: {}", e);
        source = e.source();
    }
}
```

---

## 常見模式與最佳實踐

### 1. Early Return Pattern

```rust
fn validate_user(age: i32, name: &str) -> Result<(), String> {
    if age < 18 {
        return Err("年齡必須大於 18".to_string());
    }
    
    if name.is_empty() {
        return Err("名稱不能為空".to_string());
    }
    
    Ok(())
}
```

### 2. 組合多個 Option

```rust
fn get_config() -> Option<String> {
    std::env::var("CONFIG_PATH")
        .ok()
        .or_else(|| Some("default.toml".to_string()))
}
```

### 3. 錯誤轉換

```rust
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    ParseError(ParseIntError),
    Custom(String),
}

impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> Self {
        AppError::ParseError(e)
    }
}

fn parse_number(s: &str) -> Result<i32, AppError> {
    let num = s.parse::<i32>()?; // 自動轉換
    Ok(num)
}
```

### 4. 避免過度使用 `unwrap()`

```rust
// ❌ 不好
fn bad_example() {
    let value = some_function().unwrap();
}

// ✅ 好
fn good_example() -> Result<(), Box<dyn std::error::Error>> {
    let value = some_function()?;
    Ok(())
}
```

---

## 性能考量

### Result 和 Option 的零成本抽象

Rust 的 `Result` 和 `Option` 在優化後與手動錯誤檢查性能相同。

```rust
// 編譯器會優化為相同的機器碼
pub fn with_result(x: i32) -> Result<i32, &'static str> {
    if x > 0 {
        Ok(x * 2)
    } else {
        Err("負數")
    }
}

pub fn manual_check(x: i32) -> (i32, bool) {
    if x > 0 {
        (x * 2, true)
    } else {
        (0, false)
    }
}
```

### 避免不必要的分配

```rust
// ❌ 每次都分配字符串
fn bad() -> Result<i32, String> {
    Err(format!("錯誤碼: {}", 404))
}

// ✅ 使用 &'static str
fn good() -> Result<i32, &'static str> {
    Err("錯誤")
}

// ✅ 或使用枚舉
enum Error {
    NotFound,
    InvalidInput,
}
```

---

## 實戰案例: 文件處理

```rust
use anyhow::{Context, Result};
use std::fs;
use std::path::Path;

fn process_file(path: &Path) -> Result<Vec<i32>> {
    let content = fs::read_to_string(path)
        .with_context(|| format!("無法讀取文件: {:?}", path))?;
    
    let numbers: Result<Vec<i32>> = content
        .lines()
        .enumerate()
        .map(|(i, line)| {
            line.trim()
                .parse::<i32>()
                .with_context(|| format!("第 {} 行格式錯誤", i + 1))
        })
        .collect();
    
    numbers
}

fn main() -> Result<()> {
    let path = Path::new("numbers.txt");
    
    match process_file(path) {
        Ok(numbers) => {
            println!("成功讀取 {} 個數字", numbers.len());
            println!("總和: {}", numbers.iter().sum::<i32>());
        }
        Err(e) => {
            eprintln!("處理失敗: {:?}", e);
        }
    }
    
    Ok(())
}
```

---

## 參考資料

1. [The Rust Programming Language - Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html)
2. [Rust by Example - Error Handling](https://doc.rust-lang.org/rust-by-example/error.html)
3. [std::result - Rust Documentation](https://doc.rust-lang.org/std/result/)
4. [std::option - Rust Documentation](https://doc.rust-lang.org/std/option/)
5. [thiserror - crates.io](https://crates.io/crates/thiserror)
6. [anyhow - crates.io](https://crates.io/crates/anyhow)
7. [Rust Error Handling - Best Practices](https://www.lpalmieri.com/posts/error-handling-rust/)
