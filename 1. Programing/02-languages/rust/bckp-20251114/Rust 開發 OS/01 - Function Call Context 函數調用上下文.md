### RISC-V 匯編指令部分含義
---
通常爲以下三個內容組成：
* `rs`: source regilster，源寄存器
* `rd`: destination register，目標寄存器用來存放輸出
* `imm`: immediate，立即數，表示一個常數
rs 與 rd 可以隨意的從 `x0` ~ `x31` 中選取。
##### 案例
```shell
# 跳轉指令定義
jalr rd, (imm)rs

# 使用
jalr x3, 2(x1) # 跳轉到 x1 + 2 並紀錄在 x3 中
```

### 函數本質
---
#### CPU 如何運行指令？
CPU 會依照物理地址的順序依序執行匯編指令。

#### 如何處理控制流？
當 CPU 執行到的匯編指令爲跳轉指令 (`j`, `jr` 等等)， 就可以形成多種控制流 Control Flow 如 `if/switch`, `while/for` 等，**也就是將 `pc` 寄存器設置到指定地址就好** (跳轉指令背後會完成設置 `pc`)。
> 該位置在編譯時期就已經固定。

##### 函數調用控制流
函數調用 (Function Call) 爲更複雜的控制流，與一般跳轉不同：

函數完成之後**需要返回原**執行位置的下一條指令：簡單控制流沒有返回機制
* 在不同地方調用相同函數，返回時位置不同 => 需要紀錄函數該返回的位置 (運行時/被調用時才能知道)
* 在 RISC-V 中會使用 `x1` 也就是 `ra` 寄存器來存放返回地址，會使用 `ret` 偽指令返回
	* `ret` 偽指令等於 `jalr x0, 0(x1)`，表示跳轉至 x1 + 0 位置，並保存在 x0 (恆爲 0) 寄存器。

##### 函數調用上下文
為了實現正確返回必須保證 `ra` 寄存器從執行函數開始至結束都不能改變，試想以下情況：
1. 沒有子函數：那只函數中要不要使用 `ra` 寄存器就行
2. 有子函數：因為編譯器是獨立編譯每一個函數，故 `ra` 寄存器在子函數調用時肯定會被修改成子函數返回地址。

**為了解決這個問題，我們需要將其進行保存，但不僅限於 `ra`，而是所有通用寄存器，而我們只有一套寄存器，所以必須保存在內存的某一塊空間 (Stack 棧)，這些因為函數調用導致的控制流轉移前後需要保持不變的寄存器稱為函數調用上下文 (Function Call Context)**
> 僅管函數中沒有使用任何空間紀錄局部變量，也會佔據棧空間用於存放函數調用上下文。

* 恢復與保存函數調用上下文，只要不要自己寫匯編的話，**編譯器會幫我們完成**。

### 寄存器的含義
---
RISC-V 中 C 語言的調用規範，

| registers                | 保存者    | 功能                                                                                             |
| ------------------------ | ------ | ---------------------------------------------------------------------------------------------- |
| a0~a7 <br>(x10~x17)      | Caller | 參數傳遞                                                                                           |
| t0~t6<br>(x5~7, x28~31)  | Caller | 臨時寄存器，可給 Callee 隨意使用                                                                           |
| s0~s11<br>(x8~9, x18~27) | Callee | 臨時寄存器，Callee 自己保存後才能使用                                                                         |
| zero (x0)                | None   | 恆為零，對他操作不會有任何結果                                                                                |
| ra (x1)                  | Callee | 保存函數返回地址，Callee 也可能再調用子函數，所以由自己保存，因為 Caller 並不知道你有沒有嵌套調用。                                      |
| sp (x2)                  | Callee | Stack Pointer，指向下一個函數調用棧的起始位置，也就是現在函數所占用的棧頂。                                                   |
| fp (s0)                  | Callee | Frame Pointer，表示當前棧底位置，也就是當前函數棧的起始位置。                                                          |
| gp (x3), tp (x4)         | None   | 函數運行期間都不會變化，不是函數調用上下文。<br>gp: global pointer，紀錄全局變量起始地址。<br>tp: thread pointer，紀錄線程局部局部數據起始地址。 |
> 1. 以 p 結尾的都是 pointer 用來記錄地址。
> 2. 以該函數的角度，我們只需要保存 Callee-saved 寄存器，剩下都要透過局部變量保存。

##### sp 有什麼用？
寄存器保存的內存位置爲 Stack，**函數調用當下 `sp` 寄存器表示棧起始位置**。函數中會有兩部分使用：
* 開場時會將 `sp` 減去對應的 bytes 數，故 `[新sp, 舊sp)` 形成該函數的棧空間 (Stack Frame)。
* 結尾時會將 `sp` 加上對應的 bytes 數，來使棧空間歸零回收。

###### 內核棧 (Kernel Stack) vs. 函數棧空間 (Stack Frame)
所有函數執行都會使用同一個棧，在內核中稱為 Kernel Stack，函數調用時會佔據內核棧的一部分空間，結束時會釋放函數棧的空間還給 Kernel Stack。

> `fp` 又是什麼？
> 可以與 `sp` 共同表示函數棧空間，但是 `fp` 信息除了在 Debug 下沒有什麼用，所以我們並不太關心他。但若要 Debug 的話需要在編譯選項中添加  `-Cforce-frame-pointers=yes` 保證編譯器將 `fp` 保存在函數調用上下文中。

![[Pasted image 20240526180801.png]]


### 實務：為 Rust 函數建立棧空間
```shell
    .section .text.entry
    .global  _start
_start:
    la sp, boot_stack_top # 將當前 sp 設定爲 boot stack 起始，使得 rust 函數從這邊開始分配 stack frame
    call rust_main # 呼叫 rust 函數

# 為了 rust 函數建立 boot stack 空間
	.global  _boot_stack_lower_bound
	.section .bss.stack
boot_stack_lower_bound:
    .space 4096 * 16 # 4 KB * 16

    .global boot_stack_top
boot_stack_top:
```
