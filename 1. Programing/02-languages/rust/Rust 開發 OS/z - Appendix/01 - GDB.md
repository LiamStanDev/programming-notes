# Debug 指令
---
* b: 設定斷點
* r: 運行
* l: 顯示程式碼
* p: 打印變量
* s: step into
* n: step over
* c: continue
* display: 觀察變數
* watch: 會把改變前與改變後都打印
* quit: 離開

##### 用於機器碼
* si: step instruction，執行一條機器指令
* x: 檢查記憶體內容
```shell
x/10i $pc # /10i 為格式化輸出，10 個 instruction
```