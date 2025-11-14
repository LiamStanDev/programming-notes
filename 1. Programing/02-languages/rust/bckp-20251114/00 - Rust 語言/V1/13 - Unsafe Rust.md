# 神器
---
我們可以在 unsafe block 中進行以下五種操作，讓我們可以逃出 rustc 的控制，讓我們與其他語言交互、提高性能或者編譯通過正確但卻無法在 safe rust 下通過的程式。

### 一、解引用裸指針
```rust
// 數值
let mut num = 5;
let r1 = &num as *const i32; // 使用 as 將引用轉為不可變指針
let r2 = &num as *mut i32; // 使用 as 將引用轉為可變指針

// 數組
let mut v = vec![1, 2, 3];
let r3 = v.as_ptr();
let r4 = v.as_mut_ptr();

// 動態數組
let mut v = vec![1, 2, 3];
let r3 = v.as_ptr();
let r4 = v.as_mut_ptr();

// Box
let mut b = Box::new(10);
let r5 = b.as_ref() as *const i32; // 先取得 & 在轉為不可變指針
let r6 = b.as_mut() as *mut i32; // 先取得 &mut 在轉為可變指針

unsafe {
	println!("{:?}", *r3.add(2).sub(1)); // 解引用裸指針
}
```

> 註: 
> 1. 創建指針是安全操作
> 2. 指針操作並沒有重載 + - ，只能使用`add` 與 `sub` 

### 二、調用 unsafe 方法
這就不用講解了。

### 三、FFI （Foreign Function Interface）
若調用庫是使用其他語言編寫時，有以下三種方式可以解決:
1. 重寫為 rust 版本
2. 使用 FFI
3. 用 HTTP / gRPC 等網路方式調用

#### 調用 C 函數
```rust
extern "C" {
    fn abs(x: i32) -> i32;
}

fn main() {
    unsafe { // 需要再 unsafe 裡面使用
        println!("{}", abs(-3));
    }
}
```
> `"C"` 表示調用 C 語言標準的(最常見) ABI 二進制接口，ABI 定義匯編層面該如何調用函數。


### 四、修改靜態可變變量
```rust
static mut REQUEST_RECV: usize = 0;

fn main() {
   unsafe {
        REQUEST_RECV += 1;
        assert_eq!(REQUEST_RECV, 1);
   }
}
```
> 因為這一定是無法確保多線程訪問的競爭條件的。