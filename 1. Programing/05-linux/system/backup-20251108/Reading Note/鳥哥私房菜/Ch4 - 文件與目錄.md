# 權限管理
---
##### chgrp
改變文件所屬群組。
```shell
chgrp wheel .viminfo # 將 .viminfo 文件改為 wheel 群組
chgrp wheel -R Documents # 將 Document 所有文件(含)，都改為 wheel 群組
```

##### chown
改變文件擁有者。
```shell
chown root .viminfo
chown -R root Documents
chown root:wheel .viminfo # 同時修改 owner 與群組
```

##### chmod
改變文件的權限。
```shell
chmod a+x .bashrc # 全部添加 x
chmod u=rwx,go=rx .bashrc # owner: rwx, gourp,other: rx
chmod u+x .bashrc # owner 添加 x
```

##### umask
查看目錄下的默認權限(創建文件的初始權限)
```shell
umask -S
```
# 文件與目錄管理
---
### 增刪改查
#### 基本
##### ls
```shell
ls -la # all
ls -lR # recursive
ls -l --time=atime
```
##### cp
```shell
cp ~/.bashrc /tmp/bashrc # 複製並更名
cp -i ~/.bashrc /tmp/bashrc # 交互式，詢問是否覆蓋, -f 表示直接覆蓋
cp -a /var/log/wtmp wtemp_2 # 所有資訊都複製，包含權限
cp -r ~/Documents /tmp/Documents # 遞迴
cp -s ~/.bashrc /tmp/bashrc # 建立 soft link
cp -l ~/.bashrc /tmp/bashrc # 建立 hard link
```
> 不使用 `-a` 會用默認該目錄的默認權限
##### rm
```shell
rm -i /tmp/bashrc # 交互式，會詢問是否刪除，-f 表示直接刪除
rm -r /tmp/Documents # 遞迴
```
##### mv 
```shell
mv -i ~/.bashrc bashrc # 交互式
```

##### touch
* 文件紀錄的變動時間分為: 
	* mtime: 最後修改內容的時間
	* ctime: 最後修改權限與屬性的時間
	* atime: 最後讀取時間
```shell
touch bashrc # 建立新文件
touch bashrc -t 201406150202 bashrc # 會將 bashrc 的 atime mtime 修改為 201406150202
```
#### 取得檔名或目錄名稱
```shell
basename /etc/sysconfig/network # 檔案名稱
dirname /etc/sysconfig/network # 目錄名稱
```

### 文件查閱
##### file 
查看文件類型
```shell
file /bin/bash
```

##### cat
* 全部打印在終端上
```shell
cat ~/.bashrc
```

##### less
* 分頁查看
```shell
less ~/.bashrc
```

##### od
* 查看二進制文件
```shell
# -t 表示 type，莫認為二進制
od -t x /usr/bin/passwd # 16 進制
od -t c /usr/bin/passwd # 文本
```

# 文件搜尋
---
### 查找文件
##### find
```shell
find ~/Document -name "*.cfg*"
```
* -name: 後面可以匹配文件名稱
	* * : 表示前面多個文字
##### fd
為 rust 重寫版的 find，速度更快更易用。在 ubuntu 下為 `fdfind`
```shell
fd '^x.*rc$' # regex
fd passwd /etc # 指定路徑
fd -e zip # 找到指定後墜
```

### 查找文本
##### 正則表達式
| 特别字符 | 描述                                                                                        |     |
| ---- | ----------------------------------------------------------------------------------------- | --- |
| $    | 匹配輸入字符串的結尾位置。                                                                             |     |
| ^    | 1. 匹配输入字符串的开始位置 2. 在方括号表达式中使用，此时它表示不接受该字符集合。                                              |     |
| {}   | 前面表達式重複幾次。                                                                                |     |
| \[\] | 表达式的开始。                                                                                   |     |
| (,)  | 標記一個子表達式的開始和結束位置。子表達式可以獲取供以後使用。                                                           |     |
| \|   | 指明两项之间的一个选择。                                                                              |     |
| *    | 匹配前面的子表達式零次或多次。                                                                           |     |
| +    | 匹配前面的子表達式一次或多次。                                                                           |     |
| .    | 匹配除换行符 \n 之外的任何单字符。                                                                       |     |
| ?    | 匹配前面的子表达式零次或一次，或指明一个非贪婪限定符。                                                               |     |
| \    | 将下一个字符标记为或特殊字符、或原义字符、或向后引用、或八进制转义符。例如， n 匹配字符 n。\n 匹配换行符。序列 \\ 匹配 '\' 字符，而 \( 则匹配 '(' 字符。 |     |
```text
() : 用於捕獲子表達式，之後會使用
{} : 表示前前表達式的次數
[] : 用於字符的集合
	e.g. [abc] : a or b or c
		 [0-9] : 0-9
		 [^abc] : 除了abc
```
#### 案例
匹配.txt結尾的文件
```text
^.*\.txt$
```

##### grep
```shell
grep -n -w -r "^.*\.txt$" ~/Document
```
* -n : 顯示行號
* -w : 整詞查找
* -r : recursive, 用於查找整個目錄
🔥 可以使用更快的 `rg` 工具

##### rg
ripgrep 使用 rust 重寫的 grep，效率快非常多。
```bash
rg -n -w "^.*\.txt$" ~/Document
```
* -n: 
* -w:
> rg 默認遞歸查找


### 查找二進制文件
##### which
查找 `$PATH` 環境變量中的可執行程序的地址。
```shell
which bash
```

##### whereis
與which相同是用來查找二進制文件的，但是還會一起查找man說明文件路徑，與源文件路徑
```shell
whereis shell
```