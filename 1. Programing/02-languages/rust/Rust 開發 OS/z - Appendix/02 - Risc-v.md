## 0.1 計算機組成原理
---
### Von Neumann architecture
![[Screenshot 2023-06-18 045254.png]]
- CPU
    - Contro Unit（控制單元）:　從內存取出![[Screenshot 2023-06-18 051040.png]]指令
        - Instruction Register: 存放執行的指令
        - Program Counter(PC): 紀錄執行到第幾個
    - Arithmetic Logical Unit（ALU）: 處理算術運算與邏輯運算
    - register: 存放數據
- IO Bridge: 連接Main Memory與Bus
    - 匯流排(Bus, 大陸:總線)連接所有外設

<aside> 💡 一個程序的執行: 加載到內存: Disk → Disk Controller → Bus → I/O Bridge → Main Memory 運行: CU將一條指令取出來(透過IO Bridge)，然後依照指令進行Fetch(從內存取數據到register) → Deconde（翻譯） → execute，一直重複。

</aside>

## 0.2 編譯與連接
---
### GCC命令格式

|常用選項|含意|
|---|---|
|-E|預處理 ⇒ .i|
|-c|編譯但不連接 ⇒ .s|
|-S|生成匯編代碼 ⇒ .o|
|-o file|指定輸出文件|
|-g|加入調適信息|
|-v|顯示詳細命令執行過程|

![[Screenshot 2023-06-18 051608.png]]
- 編譯(cc1): 完成預處裡+編譯 ⇒ 生成匯編指令存放在.s文件
- 匯編(as): 匯編器將匯編指令轉成機器指令
- 鏈接(ld): 鏈接器將匯編生成的目標文件形成最終的可執行文件
>  靜態共享庫擴展名為.so (share object)


### ELF(Executable Linkable Format)
> 是一種unix-like系統上的二進制文件格式的標準

|類型|說明|實例|
|---|---|---|
|Relocatable|包含代碼與數據，可以被鏈接成可執行文件或者共享目標文件。|.o|
|Executable|可以直接運行的程序|.out|
|Shared Object|包含代碼與數據||

1. 作為鍊接器的輸入，在鏈接階（靜態連接）段與Relocatable files或者shared objects鏈結成新的object file
2. 作為動態鏈接器的輸入，在運行階段與Executable File結合作為進程的一部分 | .so | | Core Dump File | 進程意外終止時，用來保存該進程狀態的文件 | core |
- 具體的ELF格式
    ![[Screenshot 2023-06-18 051608 1.png]]
    
    - 因為內存一次讀取為4K，為了節省內存，利用Segment的方式讀取會比Section讀取更有效率

### ELF文件處理工具(Binutils)
- ar: 打包用
- as: 匯編器
- ld: 鏈結器
- objcopy: 文件格式轉換
- objdump: 顯示ELF文件信息
- readelf: 顯示更多EFL文件信息
    
    ```bash
    readelf -h hello.o # -h: header
    readelf -SW hello.o # -S: section, -W: display wide
    objdump -S hello.o # 反匯編, Note: gcc -c -g hello.c, 需要帶調適信息
    ```
    
## 0.3 嵌入式開發
---
指在特定硬體環境下針對某款硬體進行開發, 是一種系統級別的軟體開發技術。

### 交叉編譯
> 就是在開發程式的電腦中，編譯目標機器運行的軟體


<aside> 💡 我們要使用的交叉編譯工具是基於riscv64下的，命名格式為 `riscv64-unknown-elf-gcc riscv64-unknown-objdump`

</aside>

### GDB
- 本地調適
- 遠程調適: 利用網路傳輸來調適遠程電腦的程序

### Qemu

> 是一個計算機系統模擬軟體，在Gnu/Linux平台廣泛使用，支持: IA-32(x86), amd 64, RSIC-V 32/64等。

有兩種主要的模式:

- User mode: 模擬了Microarchitecture(硬體設備) + Operating system
- System mode: 只模擬了Microarchitecture

```bash
# 編譯成riscv64的elf文件
riscv64-unknown-elf-gcc hello.c
# 查看elf資訊
readelf a.out # or file a.out
# 利用qemu模擬riscv64環境(User mode),來執行a.out
qemu-riscv64 ./a.out
```


## 0.4 Assembly Language (RISC-V)
---
### ISA(Instruction Set Architecture)
> 是底層硬體電路面上上層軟件程序提供的一層接口規範

![[Screenshot 2023-06-18 175711.png]]
- 分為CISC(複雜指令集) vs. RISC(精簡指令集)

### RISC 模塊化
- RISC ISA = 一個基本整數指令集 + 多個可選的擴展指令集

|基本指令集|描述|
|---|---|
|RV32I|32位整數指令集|
|RV32E|RV32I的子集合，用於小型嵌入式|
|RV64I|64位整數指令集，兼容RV32I|
|RV128I|128位整數指令集，兼容RV64I|

|擴展指令集|描述|
|---|---|
|M|整數乘法|
|A|存储原子指令集，原子操作|
|F|單精度浮點|
|D|雙精度浮點，兼容F|
|C|壓縮指令|
|…||

![20230619_19h21m35s_grim.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/12c14232-79fd-4a58-b998-f5c7de5746e2/20230619_19h21m35s_grim.png)

<aside> 💡 e.g. RV32I、RV32IMAC、RC64G(G表示支持IMAFD)

</aside>

### 指令基礎
<aside> 💡 RISC-V: 利用模塊化來區分支持的指令，但是一定要支持I (Integer)

</aside>

一個完整的RISC-V匯編程序有多條statement，一個statement由三個部分組成

```nasm
[lable:] [operation] [commnet]
```

- label: 標記
- operation: 分為以下幾種
    1. instruction: 直接對應二進制的字串
    2. pseudo-instruction: 可以想像成內聯函數或者指令的別名
    3. directive: 以.開頭，用來告知匯編器如何控制代碼生成(與risc-v無關)，很像預處理
    4. macro: 採用.marcor/.endm自定義的宏
- comment: 常用#

<aside> 💡 RISC-V是以小端序排列

</aside>

### 寄存器
- 有32個通用寄存器，有一個零寄存器可讀不可寫
- risc-v中pc是無法訪問的
    ![[Screenshot 2023-06-18 154805.png]]

### Host Byte Order (HBO)
> 一個**多字節整數**在計算機內存中存處的字節順序，不同類型的CPU的HBO不同，分為大端序(Big-Endian)與小端序(Little-Endian)。


### RISC-V 指令格式

![[Screenshot 2023-06-18 154517 2.png]]
- r開頭的都是register，用5bit = 32來存放32種不同的寄存器

![[Screenshot 2023-06-18 175135.png]]
### 進制相關
- 16進制(0~F)如何轉2進制
    - 一個16進制對應4個2進制位置
    ```
    
    16進制: 
    0     1    2    .....  A     B    .... F
    2進制: 
    0000  0001 0010 .....  1010  1011 .... 1111
    
    e.g. 0xfb = 1111 1011
    ```
    
    - 兩位16進制為一個byte空間也就是8個二進制位
- 有符號數在計算機的表示: 二進制補碼(two’s complement)
    
    - 口訣: 絕對值取反加一符號擴展(依據數據占用大小)
    - e.g. **-4** → 4 → 100 → 011(取反後) → 100(加一後) → **1111 1100**(符號擴展成8 bit)
- 符號擴展 vs. 零擴展
    
    - 符號擴展: 表示若沒填滿左邊使用符號位填滿
    - 零擴展(無符號擴展): 表示若沒填滿左邊使用0填滿

### 算術運算指令

##### ADD
```c
add x5, x6, x7 # x5 = x6 + x7
```
    
```c
# .開頭是給匯編器看的
	.text   		 # Define beginning of text section # elf格式
	.global    _start   	 # Define entry _start

_start:
	li x6, 1   	 # x6 = 1
	li x7, 2   	 # x7 = 2
	add x5, x6, x7   	 # x5 = x6 + x7

stop:
	j stop   		 # Infinite loop to stop execution

	  .end         # End of file # Add
```
    
##### SUB
```c
.text   		 # Define beginning of text section
	.global    _start   	 # Define entry _start

_start:
	li x6, -1   	 # x6 = -1
	li x7, -2   	 # x7 = -2
	sub x5, x6, x7   	 # x5 = x6 - x7

stop:
	j stop   		 # Infinite loop to stop execution

	.end   		 # End of file    .text   		 # Define beginning of text section
```
    
##### ADDI (ADD Immediate):
![[Screenshot 2023-06-18 222301.png]]
    
- immediate數只能**使用12bit空間[-2048, 2047]**

<aside> 💡 immediate是有符號數表示第一位為符號位

</aside>

```c
addi x5, x6, -5 # x5 = x6 + (-5)
# -5 is immediate which is less memory
```
    
- 基於算術運算的偽指令

| 偽指令 | 語法         | 等價             | 描述                         |
| --- | ---------- | -------------- | -------------------------- |
| neg | neg rd, rs | sub rd, x0, rs | 對rs中的值取反放入rd中              |
| mv  | mv rd, rs  | addi rd, rs, 0 | 將rs中的值拷貝到rd中               |
| nop | nop        | addi x0, x0, 0 | 甚麼都不做，因為x0不能寫，cpu解讀時就啥事都不做 |
    
##### LUI (Load Upper Immediate)
![[Screenshot 2023-06-18 222232.png]]
    
- **用來構造大數，也就是將一個大數放到寄存器，但不改變或使用其他寄存器**
	
- 如何解決ADDI立即數太小，想要處理32bit的大數該怎麼辦?
	- 引入一個新的命令(lui)，先設置高20位，存放在rs1
	- 然後使用addi加rs1與剩下的12位即可
- 會構造一個32bits的immediate
	
```c
# 想來為寄存器加載一個大數 0x12345678
lui x1, 0x12345 #  x1 = 0x12345000
addi x1, x1, 0x678 # x1 = 0x12345678

# 想來為寄存器加載一個大數 0x12345FFF
# 想法一(不正確)
lui x1, 0x12345 # x1 = 0x12346000
addi x1, x1, 0xFFF # 但是不幸的是這個12bit為1111 1111會被當成-1

# 想法二 999 可以用1000 -1 來表示
lui x1, 0x123456 # 先加一
addi x1, x1, -1 # 減去一 
```
        
##### **LI (Load Immediate)**
- 因為用LUI + ADDI構造Immediate太複雜，提供呢LI偽指令(匯編器會幫你處裡)

##### AUIPC
![[Screenshot 2023-06-18 222200.png]]
- 因為動態鏈接庫，在鏈接的時候都是採取相對地址，一般的處理方式是採取相對於pc來構造
- auipc也會構造一個32bits的立即數，這個立即數的高20位會放到imm中，低12位清零，但與lui不同的是auipc會將這個立即數和pc相加，結果存放在rd中

```c
auipc x5, 0x12345
```
    
##### **LA(Load Address)**
- 是一個偽指令，編譯器會根據實際情況利用auipc和其他指令自動生成正確的指令
- 用於加載一個函數或者一個變量的地址

```c
	.text   		 # Define beginning of text section
	.global    _start   	 # Define entry _start

_start:
	la x5, _start   	 # x5 = _start
	jr x5

stop:
	j stop   		 # Infinite loop to stop execution

exit:

	.end   		 # End of file
```
    

### 邏輯運算指令(Logical Instructions)
![[Screenshot 2023-06-18 223451.png]]

- 沒有非操作但是提供一個NOT偽指令
![[Screenshot 2023-06-18 223739.png]]
    
<aside> 💡 與-1取xor就是取非

</aside>
    
### 移位運算指令(Shifting Instructions)

##### 邏輯移偽置
![[Screenshot 2023-06-18 224028.png]]
<aside> 💡 不管左移又移都是補0，可用於

1. 乘法(只能正數)
2. 將兩個8位的數拼成16位的數 e.g. 1001 0011與 1111 0110拼成16為 1001 0011 0000 0000 + 0000 0000 1111 0110

</aside>

##### 算數移位
![[Screenshot 2023-06-18 224615.png]]
<aside> 💡 只有右移，按照符號位補足，可用於除法(正數、負數)

</aside>
    
### 內存讀寫(Load and Store Instruction)
##### 內存讀
![[Screenshot 2023-06-18 225325.png]]
- IMM給出的偏移量範圍為[-2048, 2047]，immediate是12bit的
    
<aside> 💡 注意符號擴展與零擴展 e.g LB 讀入時為1100 1000 → 1111 1111 1111 1111 1111 1111 1100 1000 (符號擴展) LB 讀入時為0100 1000 → 0000 0000 0000 0000 0000 0111 1100 1000 (符號擴展)

</aside>
- `ld`: load double word (64 bits)，在64位下更常用

##### 內存寫
![[Screenshot 2023-06-18 225743 1.png]]
- `sd`: store double word (64 bits)，在64位下更常用

### 條件分支指令(Conditional Branch Instructions)
![[Screenshot 2023-06-18 230704.png]]
![[Screenshot 2023-06-18 230716.png]]

- 具體寫代碼時不直接會寫imm，而是用lable代替，交給匯編器進行處理
    
- 偽指令
![[Screenshot 2023-06-18 231551.png]]
- 實現while loop
    
```c
# int i = 0
# while(i < 5) i++;

		.text
		.global _start
_start:
		li x5, 0
		li x6, 5
loop:
		addi x5, x5, 1
		bne x5, x6, loop
```
    

### 無條件跳轉指令(Unconditional Jump Instructions)
##### JAL (Jump And Link)
![[20230619_19h21m35s_grim.png]]
<aside>💡 以`pc`值為基準進行跳轉</aside>
- 用於調用子過程（subroutine / funtion）：使用lable
- rd 會存放當前調用的下一個地址（也就是執行完子過程後的下一個位置，也就是下一行）
    
##### JALR (Jump And Link Register)
![[20230619_19h25m12s_grim.png]]
<aside>💡 以`rs1`為基準進行跳轉</aside>
- 用於調用子過程（subroutine / funtion）：使用register
    
```c
	.text
    .global _start

_start:
    li x6, 1      # x6 = 1
    li x7, 1      # x7 = 1
    jal x5, sum   # x5 = call sum, return address is saved in x5

stop:
    j stop

sum:
    add x6, x6, x7   # x6 = x6 + x7
    jalr x0, 0(x5)   # return to x5
```

<aside> 💡 以上指令只能做有限跳轉，因為imm限制只能跳轉1MB，可以藉由auipc + jalr達成長跳轉，但後面還有call偽指令可以作到長跳轉

</aside>

- 偽指令
![[20230619_20h10m41s_grim.png]]
- 用於跳轉但是不返回

### 函數調用的過程與匯編函數約定
- A → B → C
![[20230619_20h49m29s_grim.png]]
- 若C再調用函數D則會發生stack overflow

#### Calling Conventions: 
匯編時調用參數，返回地址，返回參數都需要放在寄存器，要做約定，不然亂使用寄存器導致程式可讀性會很差

| 寄存器名稱              | ABI名稱（編程用名）                   | 用途約定                                                  | 誰負責維護（保持寄存器狀態）該寄存器   |
| ------------------ | ----------------------------- | ----------------------------------------------------- | -------------------- |
| x0                 | zero                          | 讀取總為0，寫入不起作用                                          | N/A                  |
| x1                 | ra                            | return address                                        | Caller               |
| x2                 | sp                            | stack pointer(棧針)                                     | Callee(入棧出棧是由子函數完成的) |
| x5~x7,<br>x28~x31  | t0~t2, t3~t6                  | temporaries（對於Caller是臨時的）可能會被Callee改掉，但Caller要負責保持原值  | Caller               |
| x8, x9,<br>x18~x27 | s0(Frame Pointer), s1, s2~s11 | saved （對於Caller是不變的）Callee需要保證這些寄存器的值在函數返回時保持與調用前的值相同 | Callee               |
| x10, x11           | a0, a1                        | argument ＋ 返回值                                        | Caller               |
| x12, x17           | a2~a7                         | argument                                              | Caller               |

>💡 
>函數傳參採用（a0~a7），首先使用a0 
>返回值採用（a0, a1），首先使用a0 
>函數的local variable採用（s0~s11）
    
<aside> 💡 在實際寫程式時最好是寫ABI名</aside>

##### 常用的偽指令
- **`call`**: 長調轉調用函數，常用於調用函數，等價於
```nasm
auipc x1, offset[31:12] + offset[11]
jalr x1, offset[11:0](x1)
```
	
- **`ret`**: 從Callee返回，等價於
```nasm
jalr x0, 0(x1)
```
	
- **`tail`**: 長跳轉尾調用（尾調用e.g. `return function()`），等價於
```nasm
auipc x6, offset[32:12] + offset[11]
jalr x0, offset[11:0](x6)
```
        
![[20230619_21h29m43s_grim.png]]

### 匯編對應函數調用
1. 函數的開始(Prologue) Note: 在函數中
    1. 依據本函數使用saved寄存器，以及local變量的多少，來減少sp的值(也就是壓棧，棧針下移)，用來開闢棧空間。
    2. 將saved寄存器保存到棧中（Caller要維護，所以放入內存保持值）
    3. 若函數還調用其他函數，則將ra寄存器的值保存在棧中
2. 執行函數體
3. 函數退出（Epilogue） Note: 在函數中
    1. 從棧中恢復saved寄存器
    2. 如果需要的話從棧中恢復ra寄存器
    3. 增加sp的值（出棧），恢復到進入本函數前的狀態
    4. 調用ret返回

#### 例子一
- C
```c
void _start() {
		square(3);
}

int square(int num) {
		return num * num;
}
```
    
- risc-v
```c
	.text
	.global _start
_start:
	la sp, stack_end   # load address to stack pointer with low address
	li a0, 3           # load immediate argument0
	call square

stop: 
	j stop

square:
	# prologue
	addi sp, sp, -8    # push to stack (4 bytes)
	sw s0, 0(sp)       # save s0 (4 bytes)
	sw s1, 4(sp)       # save s1 (4 bytes)

	mv s0, a0          # move a0 to s0
	mul s1, s0, s0     # multiply s0 and s0 to  s1

	mv a0, s1          # move s1 to a0 (use a0 to pass return value)

	# epilogue
	lw s0, 0(sp)       # load s0 from memory
	lw s1, 4(sp)       # load s1 from memory
	addi sp, 8         # pop
	ret                # return

# create a stack with 4 * 12 byte
# 只能放最後面
stack_start:
	.rept 12  # repeate the code between rept and .endr
	.word 0   # set a word(here is 32bit) to 0000 0000 0000 0000 0000 0000 0000 0000
	.endr

stack_end:
	
	.end        # End of file
```
    
#### 例子二
- C
```c
void _start()
{
	// calling nested routine
	aa_bb(3, 4);
}

int aa_bb(int a, int b)
{
	return square(a) + square(b);
}

int square(int num)
{
	return num * num;
}
```
        
- risc-v
```nasm
.file
	.global _start

_start:
	la sp, stack_end
	li a0, 3
	li a1, 4
	
	call aa_bb

stop: 
	j stop

aa_bb:
	# prologue
	addi sp, sp, -16
	sw s0, 0(sp)    # for local variable
	sw s1, 4(sp)    # for local variable
	sw s2, 8(sp)    # for the sum variable 
	sw ra, 12(sp)   # because it will call other function

	mv s0, a0       # pass argument
	mv s1, a1       # pass argument

	# initialize the sum
	li s2, 0   

	mv a0, s0       # pass local varialbe to square function argument
	call square      # call square
	add s2, s2, a0

	mv a0, s1      # pass local variable to square function arguments
	call square
	add s2, s2, a0

	mv a0, s2     # return value
	
	# epilogue
	sw s0, 0(sp)    
	sw s1, 4(sp)    
	sw s2, 8(sp)    
	sw ra, 12(sp)
	addi sp, sp, 16
	ret

square:
	# prologue
	addi sp, sp, -8
	sw s0, 0(sp)
	sw s1, 4(sp)

	mv s0, a0
	mul s1, s0, s0

	mv a0, s1

	# epilogue
	lw s0, 0(sp)
	lw s1, 4(sp)
	addi sp, sp, 8
	ret

# create 4 * 12 bytes stack
stack_start:
	.rept 12 
	.word 0
	.endr
stack_end:

	.end
```
        

### C 與 risc-v混合編程

#### C嵌入asm
- 寫法一
```c
int foo(int a, int b) {
		int c;
		asm volatile (
				"add %[sum], %[add1], %[add2]"
				:[sum]"r="(c)
				:[add1]"r"(a). [add2]"r"(b)
		);
		return c;
}
```
- volatiile：可選項，用來表示編譯器不要優化
- “r=” 表示用register綁定**輸出**操作數
- “r”表示用register綁定**輸入**操作數

<aside> 💡 sum, add1, add2具體是哪個register交給編譯器決定。

</aside>
        
- 寫法二
```c
int foo(int a, int b) {
		int c;
		asm volatile (
				"add %0,%1, %2"
				:"=r"(c)
				:"r"(a), "r"(b)
		);
		return c;
}
```
        
- 下面的輸入操作數與輸出操作數是按照出現順序對應0, 1, 2
	
	<aside> 💡 c先出現對應0, a對應1, b對應2
	
	</aside>
            
##### asm嵌入c
- 外部的函數名要聲明在`.globl`中
- 因為ｃ也會編譯成匯編所以沒那麼複雜

## 0.5 Assembly Language (RISC-V) - CSRs

---
> CSRs (Control and Status Registers)

### 常用的CSR指令

##### `CSRRW` (Atomic Read/write CSR)
```nasm
csrrw t1, mscratch, t2 # t1 = mscratch; mscratch = t2
```
- Note: 為零擴展
- 偽指令

##### `csrw` (寫入操作) ：等價於csrrw x0, csr, rs
```nasm
csrw csr, rs
```

##### `csrr` (讀取操作) ：
```nasm
csrr rd, csr
```