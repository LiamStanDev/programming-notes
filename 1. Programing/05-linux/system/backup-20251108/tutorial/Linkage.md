### 分類
#### Hard Link
硬連接是對該文件的直接連接，它指向的是與原文件指向相同的**物理地址**。
* 原文件被刪除，硬連接還是能用
* 不能用於目錄：因為他是指向真實的物理地址，而目錄只是抽象。
* 不會佔用空間。
```shell
ln original.txt hardlink.txt 
```
#### Soft Link
軟連接為原文件的的路徑信息。
* 原文件被刪除，軟連接會懸空
* 可以用於目錄。
* 會消耗一個指向的空間。
```shell
ln -s original.txt softlink.txt
```
