### 常用命令
注意: 編譯時不要使用 -O 參數，且要添加 -g 參數
```shell
gdb target_file
```

| 命令       | 簡寫  | 說明                            |
| -------- | --- | ----------------------------- |
| set args |     | e.g. set args A B C           |
| break    | b   | 設定斷點                          |
| run      | r   | 運行程序直到運行到斷點                   |
| next     | n   | step over                     |
| step     | s   | step into                     |
| print    | p   | 顯示變量或者表達式的值                   |
| continue | c   | 繼續運行，直到下一個斷點                  |
| set var  |     | 設定變量的值<br>e.g. set var i = 10 |
| quit     | q   | 退出                            |

### 調適 core 文件
當程序運行中發生內存操作問題，會被內核強行終止，顯示 `Segment fault`，內存狀態會被保存在 core 文件中，使我們可以進一步分析。
> Linux 默認不會生成 core 文件，需要修改系統參數
> 1. `ulimit -a` 查看該用戶資源限制參數。其中前面的表示該參數的選項，core 文件為 `-c`
> 2. `ulimit -c unlimited` 將 core file size 改為 unlimited
> 3. 在運行一次程序即會生成 core 文件

```shell
gdb target_file core.8872
```
* 會顯示在哪裡掛掉。
* 輸入 `bt` 顯示調用棧。

### 調適運行程序
```shell
# 取得進程編號
ps -ef | grep target_name
# 執行 gdb
gdb target_fil -p 8029
```

