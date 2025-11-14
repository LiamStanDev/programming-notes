Rust 提供另外一種錯誤處裡的思路，分為 `panic` 與 `Result`。

# panic
---
當存在不該發生的錯誤時，他並不是可復原的需要立刻停止程序以免後續繼續發生錯誤，此時就需要使用 panic，有兩種形式:
1. 棧展開 (默認)
2. 中止 

正常情況我們並沒有義務去處理 panic，因為 panic 一般來說都是邏輯錯誤，並不是可恢復的錯誤。

### 棧展開
棧展開時任何函數內的臨時值、局部變量、參數等會按照創建的相反順序被 drop，再往掉用者以此類推，直到退出 panic 的線程，若為主線程 panic 會導致進程退出。
> 子線程 panic 若沒導致主線程 panic 並不會導致進程退出。

### 中止
中止並不是默認行為，中止會退出整個進程，有兩個方式會導致中止:
1. 編譯參數使用 `-C panic=abort` 
2. 在棧展開時 drop 過程又發生 panic

# Result
---
```rust
fn get_weather(location: LatLng) -> Result<WeatherReport, io::Error>
```
Result 類型表明可能會失敗的操作，可能返回成功的結果 `Ok(weather)`，其中 weather 是成功的結果，或者失敗返回 `Err(error_value)`，其中 error_value 是 io::Error 的錯誤信息。

這樣設計的好處是，我們想要得到期望的值，就必須進行錯誤處裡。若 Result 返回值沒被使用編譯器也會警告。
### Result 別名
若我們看到 `Result<T>`，就是 `Result<T, Err>` 的別名，常用於模塊中以簡化重複書寫相同的錯誤類型，以下為標準庫 `std::io` 中的 Result 別名，
```rust
pub type Result<T> = std::result::Result<T, std::io::Error> // std::result 一定要寫，不然編譯器會搞混

fn remove_file(path: &Path) -> Result<()> // 使用 Result 別名作為返回值
```


### 錯誤補捉
1. match 表達式
2. `result.is_ok()`, `result.is_err()`: 返回布林值
3. `result.ok()`: 會以 Option 返回結果，成功返回 `Some(_)` 失敗返回 None
3. `result.err()`: 會以 Option 返回結果，失敗返回 `Some(_)` 成功返回 None
4. `result.unwrap_or(fallback)`: 成功返回值，失敗返回 fallback 的值
5. `result.unwrap_or_else(fallback_fn)`: 傳遞一個 closure，並在錯誤時執行，通常用於 fallback 的值無法直接得出。
6. `result.unwrap()`: 成功的化返回值，失敗 panic
7. `result.expect(message)`: 同 `unwrap()` 但可以定義訊息
8. `result.as_ref()`: 將 `Result<T, E>` 轉為 `Result<&T, &E>`，可以不用消耗 Result，而是採取共享借用。
9. `result.as_mut()`: 同上，但是是可變借用

### 錯誤打印
標準庫定義的錯誤 
1. `std::io::Error`
2. `std::fmt::Error`
3. `std::str::Utf8Error`
等等，都實現了 `std::error::Error` trait，故他們共享一些特性:

1. `println!()`: 可以使用 `{}` 與 `{:?}` 打印錯誤信息。
2. `err.to_string()`: 返回 `String` 的錯誤信息
3. `err.source()`: 返回一個 Option，表示導致該錯誤的源頭是哪個錯誤，可能沒有源頭，如標準庫的錯誤掉用這個方法只會返回 `None`，因為標準庫已經是底層錯誤。


### 錯誤傳播
Rust 提供 `?` 運算符，在 Result 後使用，可以進行傳播錯誤。相比使用 match 更加簡潔。
```rust
let weather = get_weather(hometown)?

// 等同於
let weather = match get_weather {
	Ok(success_val) => success_val,
	Err(err) => return Err(err), // 注意是直接 return
}
```
> `?` 也可以用於 `Option`，若為 `None` 也會直接 `return None`

#### 多種錯誤
```rust
use std::io::{self, BufRead};

fn main() {
    println!("Hello, world!");
}

fn read_number(file: &mut dyn BufRead)  -> Result<Vec<i64>, io::Error> {
    let mut numbers = vec![];

    for line_result in file.lines() {
        let line = line_result?; // 返回 std::io::Error
        numbers.push(line.parse()?); // 返回 std::num::ParseIntError
    }
    Ok(numbers)
}
```

故以上返回值需要修改成更加廣泛的類型，方式有以下幾種:
1. 自訂義錯誤類型，並提供轉換成自訂義錯誤類型的轉換方法
2. 使用 trait object，可以適用於實現 `std::io::Error` 的錯誤，如下:
```rust
type GenericError = Box<dyn std::error::Error + Send + Sync + 'static>
```
* 使用類型別名簡化書寫
* `dyn std::error::Error`: 表示任何錯誤。
* `Send + Sync + 'static`: 表示可以在線程中安全傳遞。
可以改寫如下:
```rust
type GenericResult<T> = std::result::Result<T, GenericError>;

fn read_number(file: &mut dyn BufRead)  -> GenericResult<Vec<i64>> {
    let mut numbers = vec![];

    for line_result in file.lines() {
        let line = line_result?;
        numbers.push(line.parse()?);
    }
    Ok(numbers)
}
```

###### 小註解
以上其實 ? 本質上是調用 From trait 的 from() 關聯函數，`GenericError::from(io_error)`。

### 忽略錯誤
當我們返回 Result 不進行銷耗時，編譯器就會釋出警告，但如果真的沒必要處裡，可以使用以下方式進行忽略:
```rust
let _ = writeln!(stderr(), "error: {}", err); // writeln! 會返回 Result，但我們並不想處裡
```

### 自訂義錯誤
若我們自己的模塊想要建立屬於該模塊的錯誤，可以使用自訂義錯誤。但我們又希望向標準庫錯誤一樣可以被打印，就需要在錯誤類型上實現 `fmt::Display` 如下
```rust
#[derive(Debug, Clone)]
pub struct JsonError {
    pub message: String,
    pub line: usize,
    pub column: usize,
}

impl fmt::Display for JsonError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} ({}:{})", self.message, self.line, self.column)
    }
}
```

但是我們可以使用好用的第三方庫，`thiserror` 會更簡潔方便。
```rust
use thiserror::Error;

#[derive(Error, Debug)]
#[error("{message} ({line}:{column})")]
pub struct JsonError {
    pub message: String,
    pub line: usize,
    pub column: usize,
}
```