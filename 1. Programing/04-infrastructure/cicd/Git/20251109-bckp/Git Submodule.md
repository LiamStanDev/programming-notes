### 前情提要
因為項目越來越大，整個項目不同模塊由不同人維護，我們可以使用 Submodule 來將整個大項目猜分成一大堆子項目，可以單獨進行管理。

### 本質
##### 名詞簡介：
* SuperRepo: 是整個大項目的 repo
* SubRepo: 是子項目的 repo

Git submodule 簡單來說就是幫子項目建立新的 repo，在 SuperRepo 中紀錄子項目的 Hash 值，這樣 SuperRepo 可以鎖定 SubRepo 的版本，並且兩者的 commit 可以獨立出來。

### 操作
#### 項目拆分
1. 為每一個子項目建立新的 git repo，也就是在 github 中建立新的項目，該項目就是你的子項目。
2. 在 SuperRepo 中 `git clone <sub_repo_url>`，到指定目錄中
3. 在 SubperRepo 中執行 `git submodule add <sub_repo_url> <folder>`
	* 此時會在 SuperRepo 根目錄中產生 .gitmodules 文件，他跟 gitignore 一樣會被 git 追蹤。
4. 在 SuperRepo 中進行 `git commit` 會將 SuperRepo 鎖定當前 SubRepo 的版本
5. `git push`

#### 項目 Clone
```shell
git clone <super_repo> --recursive # 整體 clone

# 若已經拉下來 SuperRepo 後在 clone SubRepo
git submodule init 
git submodule update
```
