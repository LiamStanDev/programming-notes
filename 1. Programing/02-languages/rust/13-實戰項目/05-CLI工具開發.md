# CLI 工具開發

命令列工具 (Command-Line Interface, CLI) 是系統開發的重要組成部分。Rust 以其零成本抽象、優秀的錯誤處理和易於分發的特性,成為構建高性能 CLI 工具的理想選擇。本文將介紹如何使用 Rust 開發專業級的 CLI 工具。

---

## 1. 核心 CLI 庫生態

### 1.1 參數解析 - clap

**clap** 是 Rust 生態中最流行的命令列參數解析庫,提供強大的功能和優秀的使用體驗。

```toml
[dependencies]
clap = { version = "4.5", features = ["derive", "cargo"] }
```

#### 基本使用 (Derive API)

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "mytool")]
#[command(author = "Your Name <you@example.com>")]
#[command(version = "1.0")]
#[command(about = "A powerful CLI tool", long_about = None)]
struct Cli {
    /// 輸入文件路徑
    #[arg(short, long, value_name = "FILE")]
    input: String,

    /// 輸出文件路徑
    #[arg(short, long, value_name = "FILE")]
    output: Option<String>,

    /// 詳細輸出模式
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbose: u8,

    /// 子命令
    #[command(subcommand)]
    command: Option<Commands>,
}

#[derive(Subcommand)]
enum Commands {
    /// 處理數據
    Process {
        /// 處理模式
        #[arg(short, long, default_value = "fast")]
        mode: String,
        
        /// 線程數
        #[arg(short = 'j', long, default_value_t = 4)]
        threads: usize,
    },
    /// 驗證文件
    Validate {
        /// 嚴格模式
        #[arg(short, long)]
        strict: bool,
    },
}

fn main() {
    let cli = Cli::parse();

    // 設置日誌級別
    match cli.verbose {
        0 => println!("Normal mode"),
        1 => println!("Verbose mode"),
        2 => println!("Very verbose mode"),
        _ => println!("Debug mode"),
    }

    // 處理子命令
    match &cli.command {
        Some(Commands::Process { mode, threads }) => {
            println!("Processing with mode: {}, threads: {}", mode, threads);
        }
        Some(Commands::Validate { strict }) => {
            println!("Validating (strict: {})", strict);
        }
        None => {
            println!("No command specified");
        }
    }
}
```

#### 高級特性

```rust
use clap::{Parser, ValueEnum};
use std::path::PathBuf;

#[derive(Copy, Clone, PartialEq, Eq, PartialOrd, Ord, ValueEnum)]
enum Format {
    /// JSON 格式
    Json,
    /// YAML 格式
    Yaml,
    /// TOML 格式
    Toml,
}

#[derive(Parser)]
#[command(name = "converter")]
struct Args {
    /// 輸入文件
    #[arg(value_name = "INPUT")]
    input: PathBuf,

    /// 輸出格式
    #[arg(short, long, value_enum, default_value_t = Format::Json)]
    format: Format,

    /// 配置文件
    #[arg(short, long, env = "CONFIG_PATH")]
    config: Option<PathBuf>,

    /// 覆蓋現有文件
    #[arg(long)]
    force: bool,

    /// 排除的模式 (可多次指定)
    #[arg(short = 'e', long = "exclude")]
    excludes: Vec<String>,
}
```

### 1.2 錯誤處理 - anyhow & thiserror

結合 **anyhow** (應用層) 和 **thiserror** (庫層) 提供最佳錯誤處理體驗。

```toml
[dependencies]
anyhow = "1.0"
thiserror = "1.0"
```

```rust
use anyhow::{Context, Result};
use thiserror::Error;
use std::path::PathBuf;

#[derive(Error, Debug)]
enum CliError {
    #[error("文件不存在: {0}")]
    FileNotFound(PathBuf),

    #[error("無效的格式: {0}")]
    InvalidFormat(String),

    #[error("權限錯誤: {path}")]
    PermissionDenied { path: PathBuf },

    #[error(transparent)]
    IoError(#[from] std::io::Error),
}

fn process_file(path: &PathBuf) -> Result<String> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("無法讀取文件: {:?}", path))?;
    
    if content.is_empty() {
        anyhow::bail!("文件為空: {:?}", path);
    }

    Ok(content)
}

fn main() -> Result<()> {
    let path = PathBuf::from("data.txt");
    
    let content = process_file(&path)
        .context("處理文件時發生錯誤")?;
    
    println!("成功讀取 {} 字節", content.len());
    Ok(())
}
```

### 1.3 終端輸出美化

#### colored - 彩色輸出

```toml
[dependencies]
colored = "2.1"
```

```rust
use colored::*;

fn main() {
    println!("{}", "Success!".green());
    println!("{}", "Warning!".yellow());
    println!("{}", "Error!".red().bold());
    
    println!("{} {} {}", 
        "Info:".blue(),
        "Processing".normal(),
        "file.txt".bright_white()
    );

    // 條件着色
    let status = true;
    println!("{}", 
        if status { "OK".green() } else { "FAILED".red() }
    );
}
```

#### indicatif - 進度條

```toml
[dependencies]
indicatif = "0.17"
```

```rust
use indicatif::{ProgressBar, ProgressStyle};
use std::time::Duration;

fn main() {
    // 簡單進度條
    let pb = ProgressBar::new(100);
    for i in 0..100 {
        pb.inc(1);
        std::thread::sleep(Duration::from_millis(20));
    }
    pb.finish_with_message("完成");

    // 自定義樣式
    let pb = ProgressBar::new(1000);
    pb.set_style(
        ProgressStyle::default_bar()
            .template("[{elapsed_precise}] {bar:40.cyan/blue} {pos:>7}/{len:7} {msg}")
            .unwrap()
            .progress_chars("##-")
    );

    for i in 0..1000 {
        pb.set_message(format!("處理第 {} 個項目", i));
        pb.inc(1);
        std::thread::sleep(Duration::from_millis(5));
    }
    pb.finish_with_message("全部完成");
}
```

#### 多進度條

```rust
use indicatif::{MultiProgress, ProgressBar, ProgressStyle};
use std::thread;
use std::time::Duration;

fn main() {
    let m = MultiProgress::new();
    let style = ProgressStyle::default_bar()
        .template("[{elapsed_precise}] {bar:40.cyan/blue} {pos:>7}/{len:7} {msg}")
        .unwrap();

    let pb1 = m.add(ProgressBar::new(100));
    pb1.set_style(style.clone());
    pb1.set_message("任務 1");

    let pb2 = m.add(ProgressBar::new(200));
    pb2.set_style(style.clone());
    pb2.set_message("任務 2");

    let pb3 = m.add(ProgressBar::new(150));
    pb3.set_style(style);
    pb3.set_message("任務 3");

    let handles: Vec<_> = vec![
        thread::spawn(move || {
            for _ in 0..100 {
                pb1.inc(1);
                thread::sleep(Duration::from_millis(15));
            }
            pb1.finish_with_message("任務 1 完成");
        }),
        thread::spawn(move || {
            for _ in 0..200 {
                pb2.inc(1);
                thread::sleep(Duration::from_millis(8));
            }
            pb2.finish_with_message("任務 2 完成");
        }),
        thread::spawn(move || {
            for _ in 0..150 {
                pb3.inc(1);
                thread::sleep(Duration::from_millis(10));
            }
            pb3.finish_with_message("任務 3 完成");
        }),
    ];

    for h in handles {
        h.join().unwrap();
    }
}
```

#### dialoguer - 交互式提示

```toml
[dependencies]
dialoguer = "0.11"
```

```rust
use dialoguer::{Input, Confirm, Select, MultiSelect, Password};

fn main() {
    // 文本輸入
    let name: String = Input::new()
        .with_prompt("您的名字")
        .with_initial_text("User")
        .interact_text()
        .unwrap();

    // 確認
    let confirmed = Confirm::new()
        .with_prompt(format!("確認名字是 {}?", name))
        .default(true)
        .interact()
        .unwrap();

    // 單選
    let selection = Select::new()
        .with_prompt("選擇一個選項")
        .items(&["選項 1", "選項 2", "選項 3"])
        .default(0)
        .interact()
        .unwrap();

    // 多選
    let selections = MultiSelect::new()
        .with_prompt("選擇多個選項")
        .items(&["A", "B", "C", "D"])
        .interact()
        .unwrap();

    // 密碼輸入
    let password = Password::new()
        .with_prompt("輸入密碼")
        .with_confirmation("確認密碼", "密碼不匹配")
        .interact()
        .unwrap();

    println!("名字: {}, 選擇: {}, 多選: {:?}", name, selection, selections);
}
```

---

## 2. 配置管理

### 2.1 配置文件解析 - config

```toml
[dependencies]
config = "0.14"
serde = { version = "1.0", features = ["derive"] }
```

```rust
use config::{Config, ConfigError, File, Environment};
use serde::Deserialize;
use std::path::PathBuf;

#[derive(Debug, Deserialize)]
struct Settings {
    database: DatabaseConfig,
    server: ServerConfig,
    logging: LoggingConfig,
}

#[derive(Debug, Deserialize)]
struct DatabaseConfig {
    url: String,
    pool_size: u32,
}

#[derive(Debug, Deserialize)]
struct ServerConfig {
    host: String,
    port: u16,
}

#[derive(Debug, Deserialize)]
struct LoggingConfig {
    level: String,
    file: Option<PathBuf>,
}

impl Settings {
    pub fn new() -> Result<Self, ConfigError> {
        let s = Config::builder()
            // 默認配置
            .set_default("server.host", "127.0.0.1")?
            .set_default("server.port", 8080)?
            .set_default("database.pool_size", 10)?
            .set_default("logging.level", "info")?
            // 從配置文件加載 (可選)
            .add_source(File::with_name("config/default").required(false))
            // 根據運行模式加載 (可選)
            .add_source(
                File::with_name(&format!(
                    "config/{}",
                    std::env::var("RUN_MODE").unwrap_or_else(|_| "development".into())
                ))
                .required(false)
            )
            // 從環境變量覆蓋 (MY_APP_SERVER_PORT=9000)
            .add_source(Environment::with_prefix("MY_APP").separator("_"))
            .build()?;

        s.try_deserialize()
    }
}

fn main() -> Result<(), ConfigError> {
    let settings = Settings::new()?;
    println!("{:#?}", settings);
    Ok(())
}
```

### 2.2 環境變量 - dotenv

```toml
[dependencies]
dotenvy = "0.15"
```

```rust
use dotenvy::dotenv;
use std::env;

fn main() {
    // 加載 .env 文件
    dotenv().ok();

    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL 必須設置");
    
    let api_key = env::var("API_KEY")
        .unwrap_or_else(|_| "default_key".to_string());

    println!("Database: {}", database_url);
    println!("API Key: {}", api_key);
}
```

`.env` 文件示例:
```env
DATABASE_URL=postgresql://user:pass@localhost/db
API_KEY=your_secret_key
LOG_LEVEL=debug
```

---

## 3. 日誌記錄

### 3.1 結構化日誌 - tracing

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

```rust
use tracing::{info, warn, error, debug, trace, instrument};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[instrument]
fn process_data(input: &str) -> Result<(), Box<dyn std::error::Error>> {
    debug!("開始處理數據");
    
    info!(input_len = input.len(), "處理輸入數據");
    
    if input.is_empty() {
        warn!("輸入數據為空");
        return Ok(());
    }

    // 模擬處理
    std::thread::sleep(std::time::Duration::from_millis(100));
    
    info!("數據處理完成");
    Ok(())
}

fn main() {
    // 初始化日誌 (支持 RUST_LOG 環境變量)
    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::new(
            std::env::var("RUST_LOG").unwrap_or_else(|_| "info".into())
        ))
        .with(tracing_subscriber::fmt::layer())
        .init();

    info!("應用程序啟動");
    
    process_data("test data").unwrap();
    
    error!(code = 404, "發生錯誤");
}
```

### 3.2 日誌輸出到文件

```rust
use tracing_subscriber::fmt;
use tracing_appender::{non_blocking, rolling};

fn main() {
    // 按日期滾動日誌
    let file_appender = rolling::daily("./logs", "app.log");
    let (non_blocking, _guard) = non_blocking(file_appender);

    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::from_default_env())
        .with(fmt::layer().with_writer(non_blocking))
        .init();

    tracing::info!("應用程序啟動");
}
```

---

## 4. 實戰案例: 文件處理工具

### 4.1 完整 CLI 工具實現

構建一個功能完整的文件批量處理工具 `fileproc`:

```toml
[package]
name = "fileproc"
version = "0.1.0"
edition = "2021"

[dependencies]
clap = { version = "4.5", features = ["derive"] }
anyhow = "1.0"
thiserror = "1.0"
colored = "2.1"
indicatif = "0.17"
walkdir = "2.4"
regex = "1.10"
rayon = "1.8"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

**main.rs**:
```rust
use clap::{Parser, Subcommand};
use anyhow::Result;
use colored::*;
use std::path::PathBuf;

mod commands;
mod utils;
mod errors;

use commands::{find, replace, stats};

#[derive(Parser)]
#[command(name = "fileproc")]
#[command(author = "Your Name")]
#[command(version = "1.0.0")]
#[command(about = "文件批量處理工具", long_about = None)]
struct Cli {
    /// 詳細輸出
    #[arg(short, long, global = true)]
    verbose: bool,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// 查找文件
    Find {
        /// 搜索路徑
        #[arg(value_name = "PATH", default_value = ".")]
        path: PathBuf,

        /// 文件名模式 (支持通配符)
        #[arg(short, long)]
        pattern: Option<String>,

        /// 正則表達式
        #[arg(short, long)]
        regex: Option<String>,

        /// 文件大小過濾 (如: +10M, -1G)
        #[arg(short, long)]
        size: Option<String>,

        /// 遞歸搜索
        #[arg(short = 'R', long)]
        recursive: bool,
    },

    /// 批量替換
    Replace {
        /// 搜索路徑
        #[arg(value_name = "PATH")]
        path: PathBuf,

        /// 查找內容 (正則)
        #[arg(short, long)]
        find: String,

        /// 替換內容
        #[arg(short = 'r', long)]
        replace: String,

        /// 文件擴展名過濾 (如: rs,toml)
        #[arg(short, long)]
        extensions: Option<String>,

        /// 試運行 (不實際修改)
        #[arg(long)]
        dry_run: bool,
    },

    /// 統計信息
    Stats {
        /// 目標路徑
        #[arg(value_name = "PATH", default_value = ".")]
        path: PathBuf,

        /// 輸出格式
        #[arg(short, long, value_enum, default_value = "text")]
        format: OutputFormat,
    },
}

#[derive(Copy, Clone, PartialEq, Eq, clap::ValueEnum)]
enum OutputFormat {
    Text,
    Json,
}

fn main() -> Result<()> {
    let cli = Cli::parse();

    // 初始化日誌
    let log_level = if cli.verbose { "debug" } else { "info" };
    tracing_subscriber::fmt()
        .with_env_filter(log_level)
        .with_target(false)
        .init();

    // 執行命令
    match cli.command {
        Commands::Find { path, pattern, regex, size, recursive } => {
            find::execute(path, pattern, regex, size, recursive)?;
        }
        Commands::Replace { path, find, replace, extensions, dry_run } => {
            replace::execute(path, find, replace, extensions, dry_run)?;
        }
        Commands::Stats { path, format } => {
            stats::execute(path, format)?;
        }
    }

    Ok(())
}
```

**commands/find.rs**:
```rust
use anyhow::Result;
use colored::*;
use indicatif::{ProgressBar, ProgressStyle};
use regex::Regex;
use std::path::PathBuf;
use walkdir::WalkDir;

pub fn execute(
    path: PathBuf,
    pattern: Option<String>,
    regex: Option<String>,
    _size: Option<String>,
    recursive: bool,
) -> Result<()> {
    println!("{} {}", "搜索路徑:".blue().bold(), path.display());

    let regex = regex.map(|r| Regex::new(&r)).transpose()?;
    
    let walker = if recursive {
        WalkDir::new(&path)
    } else {
        WalkDir::new(&path).max_depth(1)
    };

    let pb = ProgressBar::new_spinner();
    pb.set_style(
        ProgressStyle::default_spinner()
            .template("{spinner:.green} {msg}")
            .unwrap()
    );
    pb.set_message("掃描文件...");

    let mut count = 0;

    for entry in walker.into_iter().filter_map(|e| e.ok()) {
        if !entry.file_type().is_file() {
            continue;
        }

        let file_name = entry.file_name().to_string_lossy();

        // 模式匹配
        if let Some(ref pat) = pattern {
            if !file_name.contains(pat) {
                continue;
            }
        }

        // 正則匹配
        if let Some(ref re) = regex {
            if !re.is_match(&file_name) {
                continue;
            }
        }

        count += 1;
        println!("  {}", entry.path().display());
        pb.inc(1);
    }

    pb.finish_and_clear();
    
    println!(
        "\n{} {}",
        "找到文件:".green().bold(),
        count.to_string().yellow()
    );

    Ok(())
}
```

**commands/replace.rs**:
```rust
use anyhow::{Context, Result};
use colored::*;
use indicatif::{ProgressBar, ProgressStyle};
use rayon::prelude::*;
use regex::Regex;
use std::fs;
use std::path::PathBuf;
use walkdir::WalkDir;

pub fn execute(
    path: PathBuf,
    find: String,
    replace: String,
    extensions: Option<String>,
    dry_run: bool,
) -> Result<()> {
    let regex = Regex::new(&find)?;
    let allowed_extensions: Option<Vec<String>> = extensions.map(|e| {
        e.split(',').map(|s| s.trim().to_string()).collect()
    });

    // 收集文件
    let files: Vec<PathBuf> = WalkDir::new(&path)
        .into_iter()
        .filter_map(|e| e.ok())
        .filter(|e| e.file_type().is_file())
        .filter(|e| {
            if let Some(ref exts) = allowed_extensions {
                if let Some(ext) = e.path().extension() {
                    return exts.contains(&ext.to_string_lossy().to_string());
                }
                return false;
            }
            true
        })
        .map(|e| e.path().to_path_buf())
        .collect();

    println!("{} {} 個文件", "準備處理".blue().bold(), files.len());

    if dry_run {
        println!("{}", "試運行模式 (不會實際修改文件)".yellow());
    }

    let pb = ProgressBar::new(files.len() as u64);
    pb.set_style(
        ProgressStyle::default_bar()
            .template("[{elapsed_precise}] {bar:40.cyan/blue} {pos}/{len} {msg}")
            .unwrap()
    );

    let results: Vec<_> = files
        .par_iter()
        .map(|file| {
            pb.inc(1);
            process_file(file, &regex, &replace, dry_run)
        })
        .collect();

    pb.finish_with_message("完成");

    let mut total_replacements = 0;
    for result in results {
        match result {
            Ok(count) => total_replacements += count,
            Err(e) => eprintln!("{}: {}", "錯誤".red(), e),
        }
    }

    println!(
        "\n{} {} 次替換",
        "完成".green().bold(),
        total_replacements
    );

    Ok(())
}

fn process_file(
    path: &PathBuf,
    regex: &Regex,
    replacement: &str,
    dry_run: bool,
) -> Result<usize> {
    let content = fs::read_to_string(path)
        .with_context(|| format!("讀取文件失敗: {:?}", path))?;

    let new_content = regex.replace_all(&content, replacement);

    if content == new_content {
        return Ok(0);
    }

    let count = regex.find_iter(&content).count();

    if !dry_run {
        fs::write(path, new_content.as_bytes())
            .with_context(|| format!("寫入文件失敗: {:?}", path))?;
    }

    Ok(count)
}
```

**commands/stats.rs**:
```rust
use anyhow::Result;
use colored::*;
use serde::Serialize;
use std::collections::HashMap;
use std::path::PathBuf;
use walkdir::WalkDir;

use crate::OutputFormat;

#[derive(Serialize)]
struct Stats {
    total_files: usize,
    total_size: u64,
    by_extension: HashMap<String, ExtensionStats>,
}

#[derive(Serialize)]
struct ExtensionStats {
    count: usize,
    total_size: u64,
}

pub fn execute(path: PathBuf, format: OutputFormat) -> Result<()> {
    let mut stats = Stats {
        total_files: 0,
        total_size: 0,
        by_extension: HashMap::new(),
    };

    for entry in WalkDir::new(&path).into_iter().filter_map(|e| e.ok()) {
        if !entry.file_type().is_file() {
            continue;
        }

        let metadata = entry.metadata()?;
        let size = metadata.len();

        stats.total_files += 1;
        stats.total_size += size;

        let ext = entry
            .path()
            .extension()
            .and_then(|s| s.to_str())
            .unwrap_or("無擴展名")
            .to_string();

        stats
            .by_extension
            .entry(ext)
            .and_modify(|e| {
                e.count += 1;
                e.total_size += size;
            })
            .or_insert(ExtensionStats {
                count: 1,
                total_size: size,
            });
    }

    match format {
        OutputFormat::Text => print_text(&stats),
        OutputFormat::Json => print_json(&stats)?,
    }

    Ok(())
}

fn print_text(stats: &Stats) {
    println!("\n{}", "文件統計".blue().bold());
    println!("  總文件數: {}", stats.total_files.to_string().yellow());
    println!("  總大小: {}", format_size(stats.total_size).yellow());

    println!("\n{}", "按擴展名統計:".blue().bold());
    
    let mut ext_vec: Vec<_> = stats.by_extension.iter().collect();
    ext_vec.sort_by(|a, b| b.1.count.cmp(&a.1.count));

    for (ext, ext_stats) in ext_vec {
        println!(
            "  .{:<10} {:>6} 個文件  {:>10}",
            ext,
            ext_stats.count,
            format_size(ext_stats.total_size)
        );
    }
}

fn print_json(stats: &Stats) -> Result<()> {
    let json = serde_json::to_string_pretty(stats)?;
    println!("{}", json);
    Ok(())
}

fn format_size(bytes: u64) -> String {
    const KB: u64 = 1024;
    const MB: u64 = KB * 1024;
    const GB: u64 = MB * 1024;

    if bytes >= GB {
        format!("{:.2} GB", bytes as f64 / GB as f64)
    } else if bytes >= MB {
        format!("{:.2} MB", bytes as f64 / MB as f64)
    } else if bytes >= KB {
        format!("{:.2} KB", bytes as f64 / KB as f64)
    } else {
        format!("{} B", bytes)
    }
}
```

### 4.2 使用示例

```bash
# 查找文件
fileproc find . -p "*.rs" -R

# 批量替換
fileproc replace ./src -f "old_name" -r "new_name" -e "rs,toml"

# 試運行模式
fileproc replace ./src -f "pattern" -r "replacement" --dry-run

# 統計信息
fileproc stats .
fileproc stats . -f json

# 詳細模式
fileproc -v find . -R
```

---

## 5. 跨平台考慮

### 5.1 路徑處理

```rust
use std::path::{Path, PathBuf};
use std::env;

fn get_config_dir() -> PathBuf {
    if cfg!(target_os = "windows") {
        PathBuf::from(env::var("APPDATA").unwrap())
            .join("myapp")
    } else {
        PathBuf::from(env::var("HOME").unwrap())
            .join(".config")
            .join("myapp")
    }
}

fn normalize_path(path: &Path) -> PathBuf {
    // 處理 ~ 展開
    if path.starts_with("~") {
        if let Ok(home) = env::var("HOME") {
            return PathBuf::from(home).join(path.strip_prefix("~").unwrap());
        }
    }
    path.to_path_buf()
}
```

### 5.2 終端顏色支持檢測

```rust
use colored::*;

fn main() {
    // 自動檢測終端是否支持顏色
    if atty::is(atty::Stream::Stdout) {
        println!("{}", "彩色輸出".green());
    } else {
        println!("純文本輸出");
    }

    // 強制啟用/禁用顏色
    colored::control::set_override(true);  // 強制啟用
    colored::control::set_override(false); // 強制禁用
}
```

---

## 6. 性能優化

### 6.1 並行處理

使用 **rayon** 加速文件批量處理:

```rust
use rayon::prelude::*;
use std::fs;
use std::path::PathBuf;

fn process_files(files: Vec<PathBuf>) -> Vec<Result<(), std::io::Error>> {
    files
        .par_iter()
        .map(|file| {
            let content = fs::read_to_string(file)?;
            // 處理內容
            let processed = content.to_uppercase();
            fs::write(file, processed)?;
            Ok(())
        })
        .collect()
}
```

### 6.2 減少記憶體分配

```rust
use std::io::{BufRead, BufReader, Write};
use std::fs::File;

fn process_large_file(input: &str, output: &str) -> std::io::Result<()> {
    let input_file = File::open(input)?;
    let reader = BufReader::new(input_file);
    
    let output_file = File::create(output)?;
    let mut writer = std::io::BufWriter::new(output_file);

    // 逐行處理,避免一次加載整個文件
    for line in reader.lines() {
        let line = line?;
        let processed = line.to_uppercase();
        writeln!(writer, "{}", processed)?;
    }

    writer.flush()?;
    Ok(())
}
```

---

## 7. 分發與打包

### 7.1 靜態編譯

```toml
# .cargo/config.toml
[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"

[build]
target = "x86_64-unknown-linux-musl"
```

```bash
# 安裝 musl 工具鏈
rustup target add x86_64-unknown-linux-musl

# 靜態編譯
cargo build --release --target x86_64-unknown-linux-musl
```

### 7.2 減小二進制大小

```toml
# Cargo.toml
[profile.release]
opt-level = "z"     # 優化大小
lto = true          # Link-Time Optimization
codegen-units = 1   # 單個代碼生成單元
strip = true        # 移除符號信息 (Rust 1.59+)
panic = "abort"     # Panic 時直接終止
```

進一步壓縮:
```bash
# 使用 UPX 壓縮
upx --best --lzma target/release/mytool
```

### 7.3 跨平台構建

使用 **cross**:
```bash
# 安裝 cross
cargo install cross

# 交叉編譯
cross build --target x86_64-pc-windows-gnu --release
cross build --target aarch64-unknown-linux-musl --release
cross build --target x86_64-apple-darwin --release
```

---

## 8. 最佳實踐

### 8.1 錯誤處理原則

1. **庫代碼**: 使用 `thiserror` 定義具體錯誤類型
2. **應用代碼**: 使用 `anyhow` 簡化錯誤處理
3. **用戶友好**: 提供清晰的錯誤消息
4. **上下文**: 使用 `.context()` 添加錯誤上下文

### 8.2 CLI 設計原則

1. **遵循 UNIX 哲學**: 做好一件事
2. **合理默認值**: 提供智能的默認配置
3. **進度反饋**: 長時間操作提供進度提示
4. **優雅退出**: 處理 Ctrl+C 信號
5. **文檔完善**: `--help` 輸出清晰易懂

### 8.3 性能考慮

1. **並行處理**: 使用 rayon 處理大量文件
2. **緩衝 I/O**: 使用 `BufReader` / `BufWriter`
3. **惰性求值**: 使用迭代器而非提前分配
4. **避免不必要的克隆**: 使用引用

### 8.4 測試策略

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use tempfile::tempdir;

    #[test]
    fn test_file_processing() {
        let dir = tempdir().unwrap();
        let file_path = dir.path().join("test.txt");
        
        std::fs::write(&file_path, "test content").unwrap();
        
        // 測試處理邏輯
        let result = process_file(&file_path).unwrap();
        assert_eq!(result, "expected");
        
        dir.close().unwrap();
    }
}
```

---

## 9. 常用 CLI 工具庫總結

| 功能 | 推薦庫 | 說明 |
|------|--------|------|
| 參數解析 | `clap` | 功能最全,生態最好 |
| 錯誤處理 | `anyhow` + `thiserror` | 應用層 + 庫層組合 |
| 彩色輸出 | `colored` | 簡單易用 |
| 進度條 | `indicatif` | 支持多進度條 |
| 交互提示 | `dialoguer` | 豐富的交互組件 |
| 配置管理 | `config` | 支持多種配置源 |
| 日誌記錄 | `tracing` | 結構化追蹤 |
| 並行處理 | `rayon` | 數據並行 |
| 文件遍歷 | `walkdir` | 跨平台目錄遍歷 |
| 正則表達式 | `regex` | 高性能正則引擎 |
| 序列化 | `serde` + `serde_json` | 數據交換 |

---

## 參考資料

1. [Command Line Applications in Rust](https://rust-cli.github.io/book/)
2. [clap Documentation](https://docs.rs/clap/)
3. [Rust CLI Working Group](https://github.com/rust-cli)
4. [anyhow Documentation](https://docs.rs/anyhow/)
5. [indicatif Examples](https://github.com/console-rs/indicatif)
6. [Command Line Interface Guidelines](https://clig.dev/)
