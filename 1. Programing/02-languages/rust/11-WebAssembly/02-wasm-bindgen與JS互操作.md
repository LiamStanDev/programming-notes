# wasm-bindgen 與 JS 互操作

## 概述

**wasm-bindgen** 是 Rust 與 JavaScript 之間的橋樑，提供高階綁定，讓兩種語言能無縫交互。它自動處理類型轉換、記憶體管理和 API 綁定。

### 核心功能

- **Rust → JS**: 導出 Rust 函數、結構體給 JavaScript
- **JS → Rust**: 導入 JS 函數、Web APIs 到 Rust
- **類型轉換**: 自動處理基本類型和複雜對象
- **記憶體管理**: 安全的跨邊界記憶體操作

---

## 基本類型轉換

### Rust → JavaScript

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn primitives() -> JsValue {
    // 數字
    42_i32.into()  // JavaScript Number
}

#[wasm_bindgen]
pub fn return_string() -> String {
    "Hello from Rust".to_string()  // JavaScript String
}

#[wasm_bindgen]
pub fn return_bool() -> bool {
    true  // JavaScript Boolean
}

#[wasm_bindgen]
pub fn return_option(flag: bool) -> Option<String> {
    if flag {
        Some("Value".to_string())  // 轉為字串
    } else {
        None  // 轉為 undefined
    }
}

#[wasm_bindgen]
pub fn return_result() -> Result<i32, JsValue> {
    // Ok → 直接返回值
    // Err → JavaScript 拋出異常
    Ok(42)
}
```

### JavaScript → Rust

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn accept_number(n: i32) {
    web_sys::console::log_1(&format!("Got: {}", n).into());
}

#[wasm_bindgen]
pub fn accept_string(s: &str) {
    // &str 比 String 更高效，避免複製
    web_sys::console::log_1(&s.into());
}

#[wasm_bindgen]
pub fn accept_option(opt: Option<String>) {
    match opt {
        Some(s) => console::log_1(&format!("Got: {}", s).into()),
        None => console::log_1(&"Got nothing".into()),
    }
}

#[wasm_bindgen]
pub fn accept_array(arr: &[u8]) {
    // JavaScript Uint8Array → Rust &[u8]
    console::log_1(&format!("Length: {}", arr.len()).into());
}
```

---

## 類型對照表

| Rust 類型 | JavaScript 類型 | 備註 |
|-----------|----------------|------|
| `i8..i64`, `u8..u64` | `Number` | i64/u64 需 BigInt feature |
| `f32`, `f64` | `Number` | |
| `bool` | `Boolean` | |
| `char` | `String` | 單字符字串 |
| `String` | `String` | 需複製 |
| `&str` | `String` | 借用，更高效 |
| `Vec<T>` | `Array` | 需複製 |
| `&[T]` | TypedArray | 零拷貝（視類型） |
| `Option<T>` | `T | undefined` | |
| `Result<T, E>` | `T` 或拋出異常 | |
| `JsValue` | `any` | 通用類型 |

---

## 導出結構體與方法

### 基本結構體

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct Person {
    name: String,
    age: u32,
}

#[wasm_bindgen]
impl Person {
    /// 構造函數（在 JS 中使用 new）
    #[wasm_bindgen(constructor)]
    pub fn new(name: String, age: u32) -> Person {
        Person { name, age }
    }

    /// Getter（自動成為 JS 屬性）
    #[wasm_bindgen(getter)]
    pub fn name(&self) -> String {
        self.name.clone()
    }

    /// Setter
    #[wasm_bindgen(setter)]
    pub fn set_name(&mut self, name: String) {
        self.name = name;
    }

    /// 普通方法
    pub fn greet(&self) -> String {
        format!("Hi, I'm {}, {} years old", self.name, self.age)
    }

    /// 靜態方法
    pub fn create_adult(name: String) -> Person {
        Person { name, age: 18 }
    }
}
```

### JavaScript 使用

```javascript
import { Person } from './pkg/my_project.js';

const person = new Person("Alice", 30);
console.log(person.name);         // "Alice" (getter)
person.name = "Bob";               // setter
console.log(person.greet());      // "Hi, I'm Bob, 30 years old"

const adult = Person.create_adult("Charlie");  // 靜態方法
```

### 複雜結構體

```rust
use wasm_bindgen::prelude::*;
use serde::{Serialize, Deserialize};

#[wasm_bindgen]
#[derive(Serialize, Deserialize)]
pub struct Config {
    #[wasm_bindgen(skip)]  // 不導出此字段
    internal_data: Vec<u8>,
    
    pub timeout: u32,
    pub enabled: bool,
}

#[wasm_bindgen]
impl Config {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Config {
        Config {
            internal_data: vec![],
            timeout: 5000,
            enabled: true,
        }
    }

    /// 使用 JsValue 導出複雜對象
    pub fn to_js(&self) -> Result<JsValue, JsValue> {
        serde_wasm_bindgen::to_value(self)
            .map_err(|e| JsValue::from_str(&e.to_string()))
    }

    /// 從 JS 對象導入
    pub fn from_js(val: JsValue) -> Result<Config, JsValue> {
        serde_wasm_bindgen::from_value(val)
            .map_err(|e| JsValue::from_str(&e.to_string()))
    }
}
```

---

## 導入 JavaScript 函數

### 導入全局函數

```rust
use wasm_bindgen::prelude::*;

// 聲明外部 JavaScript 函數
#[wasm_bindgen]
extern "C" {
    // alert 函數
    pub fn alert(s: &str);
    
    // 帶命名空間
    #[wasm_bindgen(js_namespace = console)]
    pub fn log(s: &str);
    
    #[wasm_bindgen(js_namespace = ["console", "error"])]
    pub fn error(s: &str);
    
    // 可變參數
    #[wasm_bindgen(js_namespace = console, variadic)]
    pub fn log_many(values: &[JsValue]);
}

// 使用
#[wasm_bindgen]
pub fn test_imports() {
    alert("Hello!");
    log("This is a log");
    error("This is an error");
    
    log_many(&[
        JsValue::from_str("Multiple"),
        JsValue::from_f64(42.0),
        JsValue::from_bool(true),
    ]);
}
```

### 導入 JavaScript 類

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    // Date 類
    pub type Date;
    
    #[wasm_bindgen(constructor)]
    pub fn new() -> Date;
    
    #[wasm_bindgen(method)]
    pub fn getTime(this: &Date) -> f64;
    
    #[wasm_bindgen(static_method_of = Date)]
    pub fn now() -> f64;
}

// 使用
#[wasm_bindgen]
pub fn get_timestamp() -> f64 {
    let date = Date::new();
    date.getTime()
}
```

---

## Web APIs 綁定 (web-sys)

**web-sys** 提供完整的 Web API 綁定。

### Cargo.toml 配置

```toml
[dependencies]
wasm-bindgen = "0.2"
web-sys = { version = "0.3", features = [
    "console",
    "Document",
    "Element",
    "HtmlElement",
    "Node",
    "Window",
] }
```

### DOM 操作

```rust
use wasm_bindgen::prelude::*;
use web_sys::{Document, Element, HtmlElement, Window};

#[wasm_bindgen]
pub fn manipulate_dom() -> Result<(), JsValue> {
    // 獲取 window 對象
    let window = web_sys::window().expect("no global window");
    let document = window.document().expect("no document");
    
    // 創建元素
    let div = document.create_element("div")?;
    div.set_id("my-div");
    div.set_class_name("container");
    div.set_inner_html("<h1>Hello from Rust!</h1>");
    
    // 添加到 body
    let body = document.body().expect("no body");
    body.append_child(&div)?;
    
    Ok(())
}

#[wasm_bindgen]
pub fn query_selector() -> Result<(), JsValue> {
    let document = web_sys::window()
        .unwrap()
        .document()
        .unwrap();
    
    // querySelector
    if let Some(element) = document.query_selector("#my-div")? {
        element.set_text_content(Some("Updated!"));
    }
    
    Ok(())
}
```

### 事件處理

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{EventTarget, MouseEvent};

#[wasm_bindgen]
pub fn add_click_handler() -> Result<(), JsValue> {
    let document = web_sys::window().unwrap().document().unwrap();
    let button = document.get_element_by_id("my-button").unwrap();
    
    // 創建閉包
    let closure = Closure::<dyn FnMut(_)>::new(move |event: MouseEvent| {
        web_sys::console::log_1(&"Button clicked!".into());
        web_sys::console::log_1(&format!("X: {}, Y: {}", 
            event.client_x(), event.client_y()).into());
    });
    
    // 添加事件監聽器
    button.add_event_listener_with_callback(
        "click",
        closure.as_ref().unchecked_ref()
    )?;
    
    // 防止閉包被釋放
    closure.forget();
    
    Ok(())
}
```

### Fetch API

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Request, RequestInit, Response};

#[wasm_bindgen]
pub async fn fetch_data(url: String) -> Result<JsValue, JsValue> {
    let mut opts = RequestInit::new();
    opts.method("GET");
    
    let request = Request::new_with_str_and_init(&url, &opts)?;
    
    let window = web_sys::window().unwrap();
    let resp_value = JsFuture::from(window.fetch_with_request(&request)).await?;
    let resp: Response = resp_value.dyn_into()?;
    
    // 獲取 JSON
    let json = JsFuture::from(resp.json()?).await?;
    
    Ok(json)
}
```

---

## 閉包與回調

### 一次性回調

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn set_timeout(millis: i32) {
    let closure = Closure::once(move || {
        web_sys::console::log_1(&"Timeout!".into());
    });
    
    web_sys::window()
        .unwrap()
        .set_timeout_with_callback_and_timeout_and_arguments_0(
            closure.as_ref().unchecked_ref(),
            millis,
        )
        .unwrap();
    
    // 閉包會自動釋放
    closure.forget();
}
```

### 可重複使用的閉包

```rust
use wasm_bindgen::prelude::*;
use std::cell::RefCell;
use std::rc::Rc;

#[wasm_bindgen]
pub fn set_interval(millis: i32) -> Result<i32, JsValue> {
    let counter = Rc::new(RefCell::new(0));
    let counter_clone = counter.clone();
    
    let closure = Closure::wrap(Box::new(move || {
        *counter_clone.borrow_mut() += 1;
        web_sys::console::log_1(
            &format!("Tick: {}", counter_clone.borrow()).into()
        );
    }) as Box<dyn FnMut()>);
    
    let interval_id = web_sys::window()
        .unwrap()
        .set_interval_with_callback_and_timeout_and_arguments_0(
            closure.as_ref().unchecked_ref(),
            millis,
        )?;
    
    closure.forget();
    
    Ok(interval_id)
}
```

---

## JavaScript → Rust 高階類型

### 導入 Promise

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;

#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_name = fetchData)]
    fn fetch_data_js() -> js_sys::Promise;
}

#[wasm_bindgen]
pub async fn use_promise() -> Result<JsValue, JsValue> {
    let promise = fetch_data_js();
    let result = JsFuture::from(promise).await?;
    Ok(result)
}
```

### 導出 Promise

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::future_to_promise;

#[wasm_bindgen]
pub fn async_operation() -> js_sys::Promise {
    future_to_promise(async {
        // 異步操作
        let result = expensive_computation().await;
        Ok(JsValue::from(result))
    })
}

async fn expensive_computation() -> i32 {
    // 模擬異步計算
    42
}
```

---

## js-sys: JavaScript 標準庫綁定

```toml
[dependencies]
js-sys = "0.3"
```

### 使用 JavaScript 內建對象

```rust
use wasm_bindgen::prelude::*;
use js_sys::{Array, Object, Reflect, Map, Set};

#[wasm_bindgen]
pub fn use_js_types() -> Result<JsValue, JsValue> {
    // Array
    let arr = Array::new();
    arr.push(&JsValue::from(1));
    arr.push(&JsValue::from(2));
    arr.push(&JsValue::from(3));
    
    // Object
    let obj = Object::new();
    Reflect::set(&obj, &"name".into(), &"Rust".into())?;
    Reflect::set(&obj, &"version".into(), &1.75.into())?;
    
    // Map
    let map = Map::new();
    map.set(&"key1".into(), &"value1".into());
    
    // Set
    let set = Set::new(&JsValue::undefined());
    set.add(&"item1".into());
    
    Ok(obj.into())
}

#[wasm_bindgen]
pub fn array_operations(js_array: &Array) -> Vec<String> {
    let mut result = Vec::new();
    
    for i in 0..js_array.length() {
        if let Some(val) = js_array.get(i).as_string() {
            result.push(val);
        }
    }
    
    result
}
```

---

## 記憶體管理

### 手動釋放

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct LargeData {
    data: Vec<u8>,
}

#[wasm_bindgen]
impl LargeData {
    #[wasm_bindgen(constructor)]
    pub fn new(size: usize) -> LargeData {
        LargeData {
            data: vec![0; size],
        }
    }
}

// JavaScript 中
// const data = new LargeData(1000000);
// // 使用後手動釋放
// data.free();
```

### 使用 Weak 引用避免循環

```rust
use wasm_bindgen::prelude::*;
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[wasm_bindgen]
pub struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}
```

---

## 性能優化技巧

### 1. 避免頻繁字串複製

```rust
// ❌ 每次調用都複製
#[wasm_bindgen]
pub fn bad(s: String) -> String {
    s.to_uppercase()
}

// ✅ 使用借用
#[wasm_bindgen]
pub fn good(s: &str) -> String {
    s.to_uppercase()
}
```

### 2. 使用 TypedArray 傳遞大量數據

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn process_data(data: &[u8]) -> Vec<u8> {
    // 零拷貝讀取
    data.iter().map(|x| x.wrapping_add(1)).collect()
}

// JavaScript
// const data = new Uint8Array([1, 2, 3]);
// const result = process_data(data);
```

### 3. 批量操作

```rust
// ❌ 多次調用
for i in 0..1000 {
    wasm_function(i);
}

// ✅ 一次傳遞整個陣列
wasm_batch_function(array);
```

---

## 錯誤處理最佳實踐

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn may_fail(input: &str) -> Result<String, JsValue> {
    // 使用 ? 運算子
    let parsed: i32 = input.parse()
        .map_err(|e| JsValue::from_str(&format!("Parse error: {}", e)))?;
    
    if parsed < 0 {
        return Err(JsValue::from_str("Negative numbers not allowed"));
    }
    
    Ok(format!("Success: {}", parsed))
}

// JavaScript 中
// try {
//     const result = may_fail("123");
// } catch (e) {
//     console.error(e);
// }
```

---

## 調試技巧

### 1. 控制台日誌

```rust
use web_sys::console;

console::log_1(&"Simple message".into());
console::log_2(&"Key:".into(), &"Value".into());
console::error_1(&"Error message".into());
console::warn_1(&"Warning".into());

// 使用宏簡化
macro_rules! console_log {
    ($($t:tt)*) => {
        web_sys::console::log_1(&format!($($t)*).into())
    }
}

console_log!("Number: {}", 42);
```

### 2. 檢查類型

```rust
use wasm_bindgen::JsCast;

fn check_type(val: JsValue) {
    if val.is_string() {
        console_log!("It's a string");
    } else if val.is_function() {
        console_log!("It's a function");
    } else if let Some(arr) = val.dyn_ref::<js_sys::Array>() {
        console_log!("It's an array of length {}", arr.length());
    }
}
```

---

## 參考資料

1. [wasm-bindgen Book](https://rustwasm.github.io/wasm-bindgen/)
2. [web-sys Documentation](https://rustwasm.github.io/wasm-bindgen/web-sys/index.html)
3. [js-sys Documentation](https://docs.rs/js-sys/)
4. [wasm-bindgen API Reference](https://docs.rs/wasm-bindgen/)
5. [MDN Web Docs](https://developer.mozilla.org/)
