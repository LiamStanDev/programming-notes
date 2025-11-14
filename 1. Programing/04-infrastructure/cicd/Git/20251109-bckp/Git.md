# 如何寫 Commit
---
```text
Header: <type>(<scope>): <subject>
 - type: 代表 commit 的類別：feat, fix, docs, style, refactor, test, chore。
 - scope (optional): 代表 commit 影響的範圍，例如資料庫、控制層、模板層等等，視專案不同而不同。
 - subject: 代表此 commit 的簡短描述，不要超過 50 個字元，結尾不要加句號。

Body: 72-character wrapped. This should answer:
 - Body 部份是對本次 Commit 的詳細描述，可以分成多行，每一行不要超過 72 個字元。
 - 說明程式碼變動的項目與原因，還有與先前行為的對比。

Footer: 
 - 填寫任務編號（如果有的話）.
 - BREAKING CHANGE（可忽略），記錄不兼容的變動，
   以 BREAKING CHANGE: 開頭，後面是對變動的描述、以及變動原因和遷移方法。
```

#### type 種類
- feat：新增或修改功能（feature）
- fix：修補 bug（bug fix）
- docs：文件（documentation）
- style：格式
    - 不影響程式碼運行的變動，例如：white-space, formatting, missing semi colons
- refactor：重構
    - 不是新增功能，也非修補 bug 的程式碼變動
- perf：改善效能（improves performance）
- test：增加測試（when adding missing tests）
- chore：maintain
    - 不影響程式碼運行，建構程序或輔助工具的變動，例如修改 config、Grunt Task 任務管理工具
- revert：撤銷回覆先前的 commit
    - 例如：`revert：type(scope):subject`

# Commands
---
### 基礎
##### 配置
```shell
git config --global user.name "Liam"
git config --global user.email "geffc1454@gmail"
git config --global init.defaultBranch main # 將默認 master 改為 main
git config --global pull.rebase true
```

##### 克隆項目
```shell
git clone [url] --recursive
```
> --recursive 可以將子模塊一併拉下來

##### 添加到 staging
```shell
git add [file]
```
> file 改為 . 表示全部

##### commit
```shell
git commit -m "[commit message]"
```

##### 狀態
```shell 
git status
```


### Branching and Merging
```shell
git branch # list branches
git branch [branch-name] # new branch
git branch -m [new-branch-name] # rename  -m:stand for move
git chekout [branch-name] # change
git branch -d [branch-name] # delete 

git merge [branch]
```


### Sharing and Updatin
```shell
# 遠端倉庫
git remote add [alias] [url]
git remove -v # list all remote repositories
# 推送與拉取
git push [alias] [branch]
git pull
```

```shell
git push -u origin main # -u 表示將遠程倉庫指分支定到本地分支
# 之後就可以直接
git push
git pull
```

### Inspection and Comparison
```shell
git log --oneline --graph # 列出所有的 commit ，後面的選項只是比較好看而已
git show [commit] | bat
git diff [commit1] [commit2]
git diff [branch1] [branch2]
```
> options 這樣配置顯示比較好看


### Undoing Changes
```shell
git revert [commit] # 撤銷提交，但是會新增一個提交，這個提交內容就是撤銷指定的提交，也就是反操作
git reset --soft [file]  # 與 git add 相反的操作
git reset --soft [commit] # 撤銷到指定的 commit
```
> reset --soft: 回去後還能 commit 回來
> 	重設 HEAD 到指定的提交，但保留工作目錄和暫存區的狀態。
> reset --hard: 回到那時候
> 	重設 HEAD 到指定的提交，並更新暫存區和工作目錄以匹配該提交。
> reset --mixed: 回去後需要 add 再 commit 才能回來
> 	重設 HEAD 到指定的提交，並更新暫存區以匹配該提交，但不更改工作目錄

### Advanced Commands
```shell
git stash # temporarily store all modified tracked files
git rebase
```
