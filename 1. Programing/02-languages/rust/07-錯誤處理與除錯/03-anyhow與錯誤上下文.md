# anyhow 與錯誤上下文

> 基於 Rust 1.90+ (2025) | 簡化應用層錯誤處理

## 📋 概述

`anyhow` 是一個專為**應用程式**設計的錯誤處理庫,提供靈活的錯誤類型和豐富的上下文添加功能。與 `thiserror` 不同,`anyhow` 專注於簡化錯誤處理,而非定義精確的錯誤類型。

---

## 🎯 thiserror vs anyhow

### 使用場景對比

| 特性 | thiserror | anyhow |
|------|-----------|--------|
| **目標** | 定義庫的錯誤類型 | 簡化應用的錯誤處理 |
| **錯誤類型** | 明確的 enum | 動態的 `anyhow::Error` |
| **上下文** | 手動添加 | `.context()` 簡化 |
| **適用於** | 庫 (libraries) | 應用 (applications) |
| **類型安全** | 強類型 | 弱類型 |
| **API 契約** | 明確的錯誤變體 | 靈活的錯誤訊息 |

### 選擇建議

```rust
// ✅ 庫代碼: 使用 thiserror
// 提供明確的錯誤類型給調用者
pub fn library_function() -> Result<Data, MyLibError> { }

// ✅ 應用代碼: 使用 anyhow
// 簡化錯誤處理,專注於業務邏輯
fn application_logic() -> anyhow::Result<()> { }

// ✅ 混合使用: 庫用 thiserror,應用用 anyhow
fn app() -> anyhow::Result<()> {
    let data = library_function()?;  // 自動轉換為 anyhow::Error
    Ok(())
}
```

---

## 🚀 anyhow 基礎

### 安裝

```toml
[dependencies]
anyhow = "1.0"
```

### 基本用法

```rust
use anyhow::{Result, Context};
use std::fs;

// anyhow::Result<T> 等價於 Result<T, anyhow::Error>
fn read_config() -> Result<String> {
    let content = fs::read_to_string("config.toml")?;
    Ok(content)
}

fn main() -> Result<()> {
    let config = read_config()?;
    println!("Config: {}", config);
    Ok(())
}
```

**核心優勢**:
- `anyhow::Error` 可以接受任何實現了 `std::error::Error` 的類型
- 自動轉換,無需手動 `map_err`
- 支持在 `main` 函數中使用

---

## 🔧 添加上下文

### context() 方法

```rust
use anyhow::{Result, Context};
use std::fs;

fn read_file(path: &str) -> Result<String> {
    fs::read_to_string(path)
        .context(format!("failed to read file: {}", path))?;
    Ok(content)
}

// 或使用固定字符串
fn read_config() -> Result<String> {
    fs::read_to_string("config.toml")
        .context("failed to read config file")?;
    Ok(content)
}
```

**等價於**:
```rust
fs::read_to_string(path)
    .map_err(|e| anyhow::Error::new(e)
        .context(format!("failed to read file: {}", path)))?;
```

### with_context() 懶求值

```rust
use anyhow::{Result, Context};

fn expensive_context() -> String {
    // 昂貴的計算
    format!("computed: {}", compute_something())
}

// ❌ 不好: 即使成功也會計算
operation().context(expensive_context())?;

// ✅ 好: 只在失敗時才計算
operation().with_context(|| expensive_context())?;

// ✅ 更好: 直接使用閉包
operation().with_context(|| {
    format!("operation failed at {}", now())
})?;
```

### 鏈式添加上下文

```rust
use anyhow::{Result, Context};
use std::fs;

fn load_user_config(user_id: u64) -> Result<Config> {
    let path = format!("/home/user{}/config.toml", user_id);
    
    fs::read_to_string(&path)
        .with_context(|| format!("failed to read config file: {}", path))?
        .parse()
        .with_context(|| format!("failed to parse config for user {}", user_id))?
}

// 錯誤輸出:
// Error: failed to parse config for user 123
// 
// Caused by:
//     0: failed to read config file: /home/user123/config.toml
//     1: No such file or directory (os error 2)
```

---

## 🎨 創建自定義錯誤

### 使用 anyhow! 宏

```rust
use anyhow::{anyhow, Result};

fn validate_age(age: i32) -> Result<()> {
    if age < 0 {
        return Err(anyhow!("age cannot be negative: {}", age));
    }
    if age > 150 {
        return Err(anyhow!("age too large: {}", age));
    }
    Ok(())
}

fn main() -> Result<()> {
    validate_age(-5)?;  // Error: age cannot be negative: -5
    Ok(())
}
```

### 使用 bail! 提前返回

```rust
use anyhow::{bail, Result};

fn process_data(data: &[u8]) -> Result<()> {
    if data.is_empty() {
        bail!("data cannot be empty");
    }
    
    if data.len() > 1024 {
        bail!("data too large: {} bytes", data.len());
    }
    
    // 處理數據...
    Ok(())
}

// 等價於
fn process_data_verbose(data: &[u8]) -> Result<()> {
    if data.is_empty() {
        return Err(anyhow!("data cannot be empty"));
    }
    Ok(())
}
```

### 使用 ensure! 條件檢查

```rust
use anyhow::{ensure, Result};

fn divide(a: i32, b: i32) -> Result<i32> {
    ensure!(b != 0, "division by zero");
    Ok(a / b)
}

fn validate_username(username: &str) -> Result<()> {
    ensure!(!username.is_empty(), "username cannot be empty");
    ensure!(username.len() >= 3, "username too short: {}", username.len());
    ensure!(username.len() <= 20, "username too long: {}", username.len());
    Ok(())
}

// 等價於
fn validate_username_verbose(username: &str) -> Result<()> {
    if username.is_empty() {
        return Err(anyhow!("username cannot be empty"));
    }
    Ok(())
}
```

---

## 🔗 錯誤降級 (Downcast)

### 檢查特定錯誤類型

```rust
use anyhow::{Error, Result};
use std::io;

fn handle_error(err: Error) {
    // 降級為 io::Error
    if let Some(io_err) = err.downcast_ref::<io::Error>() {
        match io_err.kind() {
            io::ErrorKind::NotFound => {
                eprintln!("File not found");
            }
            io::ErrorKind::PermissionDenied => {
                eprintln!("Permission denied");
            }
            _ => {
                eprintln!("IO error: {}", io_err);
            }
        }
        return;
    }
    
    // 處理其他錯誤
    eprintln!("Unknown error: {}", err);
}

fn main() -> Result<()> {
    match std::fs::read("file.txt") {
        Ok(_) => Ok(()),
        Err(e) => {
            let err = Error::from(e);
            handle_error(err);
            Ok(())
        }
    }
}
```

### downcast() 消耗錯誤

```rust
use anyhow::Error;
use std::io;

fn process_error(err: Error) {
    // 嘗試降級為 io::Error
    match err.downcast::<io::Error>() {
        Ok(io_err) => {
            eprintln!("IO error: {:?}", io_err.kind());
        }
        Err(other_err) => {
            eprintln!("Other error: {}", other_err);
        }
    }
}
```

### 實戰範例: 區分錯誤類型

```rust
use anyhow::{Error, Result};
use thiserror::Error as ThisError;

#[derive(ThisError, Debug)]
enum BusinessError {
    #[error("user not found: {user_id}")]
    UserNotFound { user_id: u64 },
    
    #[error("permission denied")]
    PermissionDenied,
}

fn handle_request(user_id: u64) -> Result<()> {
    // 業務邏輯...
    Err(BusinessError::UserNotFound { user_id }.into())
}

fn main() {
    match handle_request(123) {
        Ok(_) => println!("Success"),
        Err(err) => {
            // 檢查是否為業務錯誤
            if let Some(business_err) = err.downcast_ref::<BusinessError>() {
                match business_err {
                    BusinessError::UserNotFound { user_id } => {
                        eprintln!("User {} not found, creating new user...", user_id);
                        // 特殊處理
                    }
                    BusinessError::PermissionDenied => {
                        eprintln!("Access denied");
                    }
                }
            } else {
                // 其他錯誤
                eprintln!("Internal error: {}", err);
            }
        }
    }
}
```

---

## 🎯 實戰模式

### 模式 1: 應用層錯誤處理

```rust
use anyhow::{Context, Result};
use std::fs;
use std::path::Path;

struct AppConfig {
    database_url: String,
    port: u16,
}

fn load_config(path: impl AsRef<Path>) -> Result<AppConfig> {
    let path = path.as_ref();
    
    let content = fs::read_to_string(path)
        .with_context(|| format!("failed to read config file: {}", path.display()))?;
    
    let config: AppConfig = toml::from_str(&content)
        .with_context(|| format!("failed to parse config file: {}", path.display()))?;
    
    validate_config(&config)
        .context("config validation failed")?;
    
    Ok(config)
}

fn validate_config(config: &AppConfig) -> Result<()> {
    ensure!(!config.database_url.is_empty(), "database_url is required");
    ensure!(config.port > 0, "port must be greater than 0");
    ensure!(config.port < 65535, "port must be less than 65535");
    Ok(())
}

fn main() -> Result<()> {
    let config = load_config("config.toml")?;
    println!("Loaded config: {:?}", config);
    Ok(())
}
```

### 模式 2: 多個操作的錯誤累積

```rust
use anyhow::{Context, Result};

fn setup_application() -> Result<()> {
    create_directories()
        .context("failed to create application directories")?;
    
    initialize_database()
        .context("failed to initialize database")?;
    
    start_server()
        .context("failed to start server")?;
    
    Ok(())
}

fn create_directories() -> Result<()> {
    std::fs::create_dir_all("data/cache")
        .context("failed to create cache directory")?;
    
    std::fs::create_dir_all("data/logs")
        .context("failed to create logs directory")?;
    
    Ok(())
}

// 錯誤輸出:
// Error: failed to initialize database
//
// Caused by:
//     0: connection failed
//     1: tcp connection refused
```

### 模式 3: 與第三方庫集成

```rust
use anyhow::{Context, Result};
use reqwest;
use serde::Deserialize;

#[derive(Deserialize)]
struct User {
    id: u64,
    name: String,
}

async fn fetch_user(user_id: u64) -> Result<User> {
    let url = format!("https://api.example.com/users/{}", user_id);
    
    let response = reqwest::get(&url)
        .await
        .with_context(|| format!("failed to fetch user from {}", url))?;
    
    let user = response.json::<User>()
        .await
        .context("failed to parse user response")?;
    
    Ok(user)
}

#[tokio::main]
async fn main() -> Result<()> {
    let user = fetch_user(123)
        .await
        .context("failed to load user data")?;
    
    println!("User: {} (ID: {})", user.name, user.id);
    Ok(())
}
```

### 模式 4: 優雅的錯誤報告

```rust
use anyhow::{Context, Result};
use std::io::{self, Write};

fn print_error_chain(err: &anyhow::Error) {
    let mut stderr = io::stderr();
    
    writeln!(stderr, "Error: {}", err).ok();
    
    for (i, cause) in err.chain().skip(1).enumerate() {
        writeln!(stderr, "  {}: {}", i, cause).ok();
    }
}

fn main() {
    if let Err(err) = run_app() {
        print_error_chain(&err);
        std::process::exit(1);
    }
}

fn run_app() -> Result<()> {
    load_config()
        .context("application startup failed")?;
    Ok(())
}
```

---

## 🔍 進階技巧

### 技巧 1: 自定義錯誤類型與 anyhow 混用

```rust
use anyhow::{Context, Result};
use thiserror::Error;

// 庫層: 使用 thiserror 定義明確的錯誤
#[derive(Error, Debug)]
pub enum DatabaseError {
    #[error("connection failed: {0}")]
    ConnectionFailed(String),
    
    #[error("query failed: {0}")]
    QueryFailed(String),
}

pub fn connect_database(url: &str) -> Result<Database, DatabaseError> {
    // 庫代碼...
}

// 應用層: 使用 anyhow 簡化錯誤處理
fn setup_app() -> Result<()> {
    let db = connect_database("postgres://localhost")
        .context("failed to connect to database")?;  // 自動轉換
    
    Ok(())
}
```

### 技巧 2: 錯誤包裝器

```rust
use anyhow::{Error, Result};

struct RequestContext {
    user_id: u64,
    request_id: String,
}

impl RequestContext {
    fn wrap_error<E>(&self, err: E, message: &str) -> Error 
    where
        E: std::error::Error + Send + Sync + 'static,
    {
        Error::new(err)
            .context(format!(
                "{} (user_id: {}, request_id: {})",
                message, self.user_id, self.request_id
            ))
    }
}

fn handle_request(ctx: &RequestContext) -> Result<()> {
    std::fs::read("data.txt")
        .map_err(|e| ctx.wrap_error(e, "failed to read data"))?;
    
    Ok(())
}
```

### 技巧 3: 條件錯誤處理

```rust
use anyhow::{bail, ensure, Result};

fn process_file(path: &str, strict: bool) -> Result<()> {
    let content = std::fs::read_to_string(path)?;
    
    // 嚴格模式下檢查文件大小
    if strict {
        ensure!(
            content.len() <= 1024 * 1024,
            "file too large: {} bytes (max 1MB in strict mode)",
            content.len()
        );
    } else if content.len() > 10 * 1024 * 1024 {
        bail!("file too large: {} bytes (max 10MB)", content.len());
    }
    
    Ok(())
}
```

### 技巧 4: 錯誤恢復

```rust
use anyhow::{Context, Result};

fn fetch_data_with_fallback(primary_url: &str, fallback_url: &str) -> Result<String> {
    // 嘗試主要來源
    match fetch_data(primary_url) {
        Ok(data) => Ok(data),
        Err(e) => {
            eprintln!("Primary fetch failed: {}", e);
            eprintln!("Trying fallback...");
            
            // 嘗試備用來源
            fetch_data(fallback_url)
                .context("both primary and fallback sources failed")
        }
    }
}

fn fetch_data(url: &str) -> Result<String> {
    // 實現...
    Ok(String::new())
}
```

---

## 📊 完整範例: HTTP API 客戶端

```rust
use anyhow::{anyhow, bail, ensure, Context, Result};
use reqwest;
use serde::{Deserialize, Serialize};
use std::time::Duration;

#[derive(Debug, Serialize, Deserialize)]
struct ApiResponse<T> {
    success: bool,
    data: Option<T>,
    error: Option<String>,
}

#[derive(Debug, Deserialize)]
struct User {
    id: u64,
    username: String,
    email: String,
}

struct ApiClient {
    base_url: String,
    client: reqwest::Client,
}

impl ApiClient {
    fn new(base_url: String) -> Result<Self> {
        ensure!(!base_url.is_empty(), "base_url cannot be empty");
        ensure!(
            base_url.starts_with("http://") || base_url.starts_with("https://"),
            "base_url must start with http:// or https://"
        );
        
        let client = reqwest::Client::builder()
            .timeout(Duration::from_secs(30))
            .build()
            .context("failed to create HTTP client")?;
        
        Ok(Self { base_url, client })
    }
    
    async fn get_user(&self, user_id: u64) -> Result<User> {
        let url = format!("{}/users/{}", self.base_url, user_id);
        
        let response = self.client
            .get(&url)
            .send()
            .await
            .with_context(|| format!("failed to send request to {}", url))?;
        
        if !response.status().is_success() {
            bail!(
                "request failed with status {}: {}",
                response.status(),
                url
            );
        }
        
        let api_response: ApiResponse<User> = response
            .json()
            .await
            .context("failed to parse API response")?;
        
        if !api_response.success {
            let error_msg = api_response.error
                .unwrap_or_else(|| "unknown error".to_string());
            bail!("API error: {}", error_msg);
        }
        
        api_response.data
            .ok_or_else(|| anyhow!("API returned success but no data"))
    }
    
    async fn create_user(&self, username: &str, email: &str) -> Result<User> {
        ensure!(!username.is_empty(), "username cannot be empty");
        ensure!(email.contains('@'), "invalid email format");
        
        let url = format!("{}/users", self.base_url);
        
        #[derive(Serialize)]
        struct CreateUserRequest<'a> {
            username: &'a str,
            email: &'a str,
        }
        
        let request_body = CreateUserRequest { username, email };
        
        let response = self.client
            .post(&url)
            .json(&request_body)
            .send()
            .await
            .with_context(|| format!("failed to create user via {}", url))?;
        
        if !response.status().is_success() {
            let status = response.status();
            let body = response.text().await.unwrap_or_default();
            bail!(
                "failed to create user (status {}): {}",
                status,
                body
            );
        }
        
        let api_response: ApiResponse<User> = response
            .json()
            .await
            .context("failed to parse create user response")?;
        
        api_response.data
            .ok_or_else(|| anyhow!("user created but no data returned"))
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    // 初始化客戶端
    let client = ApiClient::new("https://api.example.com".to_string())
        .context("failed to initialize API client")?;
    
    // 獲取用戶
    let user = client.get_user(1)
        .await
        .context("failed to fetch user")?;
    
    println!("User: {} ({})", user.username, user.email);
    
    // 創建用戶
    let new_user = client.create_user("alice", "alice@example.com")
        .await
        .context("failed to create new user")?;
    
    println!("Created user: {} (ID: {})", new_user.username, new_user.id);
    
    Ok(())
}
```

---

## 🎓 最佳實踐

### 1. 在 main 函數中使用 anyhow::Result

```rust
use anyhow::Result;

// ✅ 好: 自動打印錯誤鏈
fn main() -> Result<()> {
    run_app()?;
    Ok(())
}

// ❌ 不好: 需要手動處理錯誤
fn main() {
    if let Err(e) = run_app() {
        eprintln!("Error: {}", e);
        std::process::exit(1);
    }
}
```

### 2. 添加有意義的上下文

```rust
// ❌ 不好: 缺少上下文
fs::read_to_string(path)?;

// ✅ 好: 添加上下文
fs::read_to_string(path)
    .with_context(|| format!("failed to read file: {}", path))?;

// ✅ 更好: 添加業務上下文
fs::read_to_string(config_path)
    .with_context(|| format!(
        "failed to load user {} configuration from {}",
        user_id, config_path
    ))?;
```

### 3. 使用 bail! 和 ensure! 簡化代碼

```rust
// ❌ 冗長
if value < 0 {
    return Err(anyhow!("value cannot be negative"));
}

// ✅ 簡潔
ensure!(value >= 0, "value cannot be negative");

// ❌ 冗長
if !condition {
    return Err(anyhow!("condition not met"));
}

// ✅ 簡潔
bail!("condition not met");
```

### 4. 庫用 thiserror,應用用 anyhow

```rust
// lib.rs - 庫代碼
use thiserror::Error;

#[derive(Error, Debug)]
pub enum LibError {
    #[error("invalid input: {0}")]
    InvalidInput(String),
}

pub fn lib_function() -> Result<(), LibError> {
    // ...
}

// main.rs - 應用代碼
use anyhow::{Context, Result};

fn main() -> Result<()> {
    lib_function()
        .context("library call failed")?;
    Ok(())
}
```

---

## 🔍 常見陷阱

### 陷阱 1: 過度使用 anyhow

```rust
// ❌ 不好: 庫應該使用 thiserror
pub fn public_api() -> anyhow::Result<Data> {
    // 調用者無法匹配具體錯誤類型
}

// ✅ 好: 庫使用具體錯誤類型
pub fn public_api() -> Result<Data, LibError> {
    // 調用者可以處理特定錯誤
}
```

### 陷阱 2: 忘記添加上下文

```rust
// ❌ 不好: 錯誤信息不清晰
let config = load_config()?;

// ✅ 好: 提供上下文
let config = load_config()
    .context("failed to load application config")?;
```

### 陷阱 3: context vs with_context 的性能差異

```rust
// ❌ 慢: 即使成功也會分配字符串
operation().context(format!("failed at {}", now()))?;

// ✅ 快: 只在失敗時才分配
operation().with_context(|| format!("failed at {}", now()))?;
```

---

## 📖 參考資料

1. [anyhow Documentation](https://docs.rs/anyhow/)
2. [anyhow GitHub Repository](https://github.com/dtolnay/anyhow)
3. [Error Handling Isn't All About Errors](https://sabrinajewson.org/blog/errors)
4. [Rust Error Handling - thiserror vs anyhow](https://nick.groenen.me/posts/rust-error-handling/)
5. [Error Handling in Rust](https://blog.burntsushi.net/rust-error-handling/)

---

*最後更新: 2025-01-17*  
*Rust 版本: 1.90+*
