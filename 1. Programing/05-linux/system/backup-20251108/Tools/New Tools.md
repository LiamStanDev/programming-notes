### dust
* 作為 du 的替代品
* 功能：只列出大文件，並用圖形化方式列出。

### duf
* 為 df 的替代品
* 用於查看文件系統使用的硬盤空間情形

### hyperfine
* 用於進行壓力測試的終端工具
1. 測試單一軟體執行
```shell
hyperfine 'sleep 0.3'
```
2. 比較兩個軟體
```shell
hyperfine --warmup 3 'hexdump package.json' 'xxd package.json'
```
> --warmup 表示暖機 3 次



### tokie
分析目錄下 code 的語言與行數

### zoxide
一個有記憶性的旨在取代 cd 的工具 

### jq
#### 美化輸出
```shell
cat file.json | jq
```

#### 提取特定字段
```shell
jq '.name' appsettings.Development.json

jq '.ConnectionStrings.MongoDBConnection' appsettings.Development.json   
```
```text
output:
{
  "MongoDBConnection": "mongodb://root:mongopw@localhost"
}

"mongodb://root:mongopw@localhost"
```
#### 多字段提取
```shell
echo '{"name": "John", "age": 31}' | jq '{Name: .name, Age: .age}'
```