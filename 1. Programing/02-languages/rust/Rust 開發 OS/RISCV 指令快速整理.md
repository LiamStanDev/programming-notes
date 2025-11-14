### 寄存器別名


| 寄存器名稱     | 別名                  | 描述          |
| --------- | ------------------- | ----------- |
| x0        | zero                | 常值 0        |
| x1        | ra (Return Address) | 返回地址        |
| x2        | sp (Stack Pointer)  | 棧指針         |
| x3        | gp (Global Pointer) | 全局指針        |
| x4        | tp (Thread Pointer) | 線程指針        |
| x5 - x7   | t0 - t2             | 臨時寄存器       |
| x8        | s0/fp               | 保存寄存器/幀指針   |
| x9        | s1                  | 保存寄存器       |
| x10       | a0                  | 函數參數/返回值寄存器 |
| x10-x17   | a1-a7               | 函數參數寄存器     |
| x18-x27   | s2 - s11            | 保存寄存器       |
| x28 - x31 | t3 - t6             | 臨時寄存器       |
> 函數中使用 saved register 時候需要保存舊值，才能使用在函數退出前要復原。
> 函數使用 temp register 時不用保存可以直接用，但就是 Caller 要在調用前就保存（若後續要用）。
### 操作
#### 加減法
```shell
# 寄存器間
ADD x5, x1, x2 # 將 x5 = x1 + x2
SUB x5, x1, x2 # x5 = x1 - x2

# 立即數
ADDI x5, x1, 10 # x5 = x1 + 10
```
#### 存儲
```shell
# 32 位元使用 LW
LD x5, 0(x1) # 把 x1 地址的值放到 x5 
SD x5, 0(x1) # 把 x1 地址的值放到 x5
```

#### 邏輯
```shell
AND x5, x1, x2 # x5 = x1 & x2
OR x5, x1, x2 # x5 = x1 | x2
XOR x5, x1, x2 # x5 = x1 ^ x2
```
#### 位移
```python
# Shift Left Logic
SLL x5, x1, x2 # x5 = x1 << x2
# Shift Right Logic
SRL x5, x1, x2 # x5 = x1 >> x2
```

#### 分支跳轉
```python
BEQ x1, x2, 8 # if (x1 == x2) PC = PC + 8
BNE x1, x2, 8 # if (x1 != x2) PC = PC + 8
BLT x1, x2, 8 # if (x1 < x2) PC = PC + 8
BGE x1, x2, 8 # if (x1 > x2) PC = PC + 8
```

#### 無條件跳轉
```python
JAL x1, 16 # x1 = PC + 8; PC = PC + 16
JALR x1, x2, 8 # x1 = PC + 8; PC = x2 + 8
```
> rd 保存跳轉的下一個指令地址

### 偽指令
#### 加載立即數字
```shell
LI x5, 1000 # x5 = 1000

# 等同於
ADDI x5, x0, 1000
```

#### 設置寄存為零
```shell
CLR x5

# 等同於
ADDI x5, x0, 0
```

#### 複製
```shell
MV x5, x1 # x5 = x1

# 等同於
ADDI x5, x1, 0
```

#### 跳轉
* 跳轉一個立即數
```shell
J 8 # PC = PC + 8

# 等同於
JAL x0, 8
```
* 跳轉到寄存器
```shell
JR x1 # PC = x1

# 等同於
JALR x0, 0(x1)
```
* 返回
```shell
RET

# 等同於
JALR x0, 0(ra)
```

#### 內存與 IO 屏障
```shell
FENCE.I # 用於指令可以看到任何已經修改的內容，清空 I-Cache，常用於改動 .data 段數據

FENCE [pred] [succ] # 用於保證 pred 一定會執行完之後才會執行 succ，保證不會重排
SFENCE.VMA # 清空 TLB 緩存，用於更新 satp CSR 的時候
```