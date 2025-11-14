## 流編輯器
### sed
是一個流編輯工具，會將文本逐row讀入，並做處理後逐行寫入(不會更改)
動作如下：

|命令|功能|
|---|---|
|a|append(添加在指定行數的下一行)|
|c|change|
|d|delete|
|i|insert|
|p|print|
|r|replace|
* c: 用於整行
* r: 用於字符
#### 輸出
```shell
sed -n "2p" file.txt # 輸出第2行
sed -n "3,10p" file.txt # 輸出3-10行
sed -n "/he/p" file.txt # 打印有he的行
```
* -n : 與p一起使用的
* // 之中是使用正則表達式判斷
#### 刪除
```shell
sed "3d" file.txt
sed "2,4d" file.txt 
sed "/he/d" file.txt
```
* 不用添加 `-n` 選項
#### 插入與追加
```shell
sed "3a nice" file.txt # 在第三行追加nice，故第四行為nice
sed "3i nice" file.txt # 在第三行插入nice，故第二行為nice
```
* 注意要有空格
#### 行替換
```shell
sed "3c happy" file.txt # 將第3行替換成happy
sed "2,5c happy" file.txt # 將第2-5行替換成happy
```
#### 字符替換
```shell
sed "s/ha/hk/g" file.txt # 將整個文本中ha替換成hk
sed "3s/ha/hk/g" file.txt # 將第三行中ha替換成hk
sed -e "s/hi//g;s/foot/dinner/g" # 多個條件
```
* 與neovim中的命令一致。
* -e : 用於多個替換,使用 `;` 分隔

#### 寫入
以上的內容加上`-i` 就會進行寫入
```shell
sed -i -e "s/hi//g;s/foot/dinner/g" # 多個條件
```


### awk
與sed一樣，但是是以column為單位。
使用方式： `awk [-options] '條件1{動作1}條件2{動作2}' file.txt`
* 注意要使用單引號
* 內不字符使用雙引號
* 可以使用//作為正則表達式匹配

#### 案例一: 取出df的 1, 3 columns
```shell
df -h | awk '{print $1 "\t" $3}'
```
* 默認使用空格做為分隔符號
* '': 不會轉譯, "": 會轉譯

#### 案例二: 取出/etc/passwd的 1, 3 columns
```shell
awk -F : '{print $1 "\t" $3}' /etc/passwd
```
* -F : 指定分隔符, 此例子使用 `:` 做為分隔符號

#### 案例三：取出/etc/passwd中uid > 500 的 column 1, 3
```shell
awk -F : '$3>500{print $1 "\t" $3}' /etc/passwd
```

#### 案例四：取出/etc/passwd中 名稱為liam 的 全部內容
```shell
awk -F : '/liam/' /etc/passwd
```
* 沒有{}表示全部打印