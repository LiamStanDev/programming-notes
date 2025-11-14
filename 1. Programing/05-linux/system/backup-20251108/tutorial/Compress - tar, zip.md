### tar
tar 本身是一個打包工具。
#### 解壓縮
```shell
tar -xzvf xxx.tar.gz -C ~/Document # 解壓縮
```
- -x : 表示拆包
- -z : 使用gzip壓縮(壓縮能力差)
- -j : 使用bzip壓縮(壓縮能力中)
- -J : 使用xz壓縮(壓縮能力高)
- -v : verbose
- -f : 後面跟者目標
- -C : 輸出目錄位置
#### 壓縮
```shell
tar -czvf  xxx.tar.gz xxx
```
* -c : create表示打包

### zip, unzip
#### 壓縮
```shell
zip -r -v xxx.zip ~/Document/xxx
```
* -r : recursive,表示打包壓縮
#### 解壓縮
```shell
unzip xxx.zip
```
