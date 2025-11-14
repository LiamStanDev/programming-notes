# 1.1 Git 版本控制簡介

Git 是一套分散式版本控制系統，廣泛用於軟體開發協作與原始碼管理。其核心概念在於「快照」(snapshot) 與「分支」(branch)，每次提交 (commit) 都會記錄當下專案的完整狀態，並可隨時回溯、切換或合併不同開發歷程。Git 以高效、可靠、支援離線操作著稱，適合個人與團隊協作。

---

# 1.2 Git 物件模型（Blob、Tree、Commit、Tag）

Git 內部以物件（Object）為基礎，主要分為四種型態：

#### 1. Blob（Binary Large Object）
- **內容**：只儲存檔案的內容，不包含檔名、目錄結構或權限。
- **用途**：每個檔案內容唯一對應一個 blob，內容相同的檔案只儲存一次。
- **查詢指令**：
  - 取得某 commit 下檔案的 blob SHA：
    ```sh
    git ls-tree <commit>    # 例如 git ls-tree HEAD
    ```
  - 查看 blob 內容：
    ```sh
    git show <blob_sha>
    ```


#### 2. Tree
- **內容**：類似目錄，記錄檔案名稱、權限、以及對應的 blob 或子 tree（目錄）。
- **用途**：描述一個目錄（含子目錄與檔案）的結構。
- **查詢指令**：
  - 查看某 tree 物件內容（目錄結構）：
    ```sh
    git ls-tree <tree_sha>
    ```
  - 取得 commit 對應的 tree：
    ```sh
    git cat-file -p <commit_sha>
    ```

#### 3. Commit
- **內容**：一次提交的快照，包含指向 tree 的指標、作者、提交訊息、父 commit 等。
- **用途**：串連專案歷史，記錄每次變更。
- **查詢指令**：
  - 查看 commit 詳細內容：
    ```sh
    git cat-file -p <commit_sha>
    ```
  - 取得 HEAD commit SHA：
    ```sh
    git rev-parse HEAD
    ```

> 每次 commit 都會建立一個新的 tree 結構，指向所有檔案的 blob。  
> 只要檔案內容有變動，就會產生新的 blob，保存整個檔案內容（**不是只存修改的部分**）。

#### 4. Tag
- **內容**：為 commit 加上標籤，可包含簽名與說明。
- **用途**：標記重要版本（如 v1.0.0）。
- **查詢指令**：
  - 列出所有 tag：
    ```sh
    git tag
    ```
  - 查看 tag 詳細內容（含指向的 commit）：
    ```sh
    git cat-file -p <tag_name>
    ```

# 1.3 索引（Index/Stage）與工作區（Working Directory）

- **工作區 (Working Directory)**：實際檔案所在目錄，開發者直接編輯的地方。
- **索引 (Index/Stage)**：暫存區，記錄哪些檔案將納入下次 commit。
  變更需先 `git add` 進入索引，才能 `git commit` 寫入歷史。

流程圖：
```
工作區 → (git add) → 索引 → (git commit) → 本地儲存庫
```

---

# 1.4 Commit 與歷史追蹤原理

每次 commit 會產生一個新的 commit 物件，指向當下 tree 物件與父 commit。Git 以此形成有向無環圖（DAG），可完整追蹤每次變更來源與分支合併歷程。

範例：
```bash
$ git log --oneline --graph
* 1a2b3c4 (HEAD -> main) 新增 README
* 5d6e7f8 修正 bug
* 9a0b1c2 初始提交
```
輸出顯示 commit 之間的關聯與分支結構。

---

# 1.5 Branch 與 Tag 的運作機制

- **Branch**：分支本質上是指向 commit 的可變指標（ref），可用於平行開發、功能隔離。
- **Tag**：標籤是指向特定 commit 的不可變指標，常用於標記版本釋出點。

建立與切換範例：
```bash
$ git branch feature/login
$ git checkout feature/login
$ git tag v1.0.0
$ git checkout main
```

---

# 1.6 Git 內部資料結構與儲存原理

Git 以 `.git` 目錄儲存所有版本資料，主要結構如下：

- `objects/`：存放所有物件（blob、tree、commit、tag），以 SHA-1 命名。
- `refs/`：分支與標籤的指標。
- `HEAD`：目前檢出（checkout）的分支指標。
- `index`：暫存區狀態檔案。

查詢物件範例：
```bash
$ git cat-file -p HEAD
tree 1a2b3c...
author Alice <alice@example.com>
...
```

---

# 1.7 常用命令與實戰案例

## 初始化與基本操作

```bash
$ git init
$ git add .
$ git commit -m "初始化專案"
```
輸出：
```
Initialized empty Git repository in /path/to/repo/.git/
[main (root-commit) 9a0b1c2] 初始化專案
 3 files changed, 120 insertions(+)
```

## 分支管理

```bash
$ git branch dev
$ git checkout dev
$ git merge main
```
輸出：
```
Switched to branch 'dev'
Updating 1a2b3c4..5d6e7f8
Fast-forward
```

## 追蹤檔案狀態

```bash
$ git status
```
輸出：
```
On branch main
Changes not staged for commit:
  modified:   app.js
Untracked files:
  README.md
```

## 檢視歷史與差異

```bash
$ git log --oneline
$ git diff HEAD~1 HEAD
```

---

# 1.8 進階技巧與最佳實踐

## 進階技巧

- **Rebase 整理歷史**
  用 `git rebase` 讓 commit 歷史更線性，便於審查與回溯。
  ```bash
  $ git rebase -i HEAD~3
  ```
- **Stash 暫存變更**
  快速切換分支時可用 `git stash` 暫存未提交變更。
  ```bash
  $ git stash
  $ git stash pop
  ```
- **Cherry-pick 精選提交**
  將特定 commit 應用到當前分支。
  ```bash
  $ git cherry-pick <commit-hash>
  ```

## 最佳實踐

- 經常 commit，訊息具體明確。
- 分支命名規範（如 feature/、bugfix/）。
- 合併前先同步主分支，減少衝突。
- 定期檢查 `git status`，避免遺漏變更。

## 常見錯誤排查

- **合併衝突**：發生時編輯衝突檔案，解決後 `git add` 並 `git commit`。
- **誤刪分支/commit**：可用 `git reflog` 查詢歷史指標，找回遺失 commit。
  ```bash
  $ git reflog
  $ git checkout <commit-hash>
  ```

---

本文件適合工程師面試、實務開發與進階學習，掌握 Git 內部原理與高效操作技巧。