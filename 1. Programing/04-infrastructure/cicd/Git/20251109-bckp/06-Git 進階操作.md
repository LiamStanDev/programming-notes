# 06-Git 進階操作

本章節將深入探討 Git 的進階操作，涵蓋救援、錯誤查找、歷史重寫、自動化與高階命令組合，並提供實務案例、最佳實踐與常見錯誤排查，適合資深工程師與面試準備。

---

## 6.1 reflog 操作與救援

### 主題簡介
`git reflog` 記錄所有 HEAD 及分支引用的移動歷史，能追蹤被刪除或重設的 commit，為資料救援與誤操作回復的關鍵工具。

### 原理說明
Git 會在 `.git/logs/refs/` 下記錄每次 HEAD、分支、rebase、reset 等操作的變動。即使 commit 被 `reset` 或 `rebase` 移除，reflog 仍可追溯其 SHA-1。

### 常用命令與語法範例

```shell
$ git reflog
```
**輸出範例：**
```
a1b2c3d HEAD@{0}: reset: moving to HEAD~1
e4f5g6h HEAD@{1}: commit: 修正 bug
...
```

**回復誤刪 commit：**
```shell
$ git reset --hard HEAD@{1}
```

**查找特定操作前的狀態：**
```shell
$ git reflog show feature-branch
```

### 實際開發場景與案例
- 誤用 `git reset --hard` 導致 commit 消失，可用 reflog 找回。
- 誤刪分支後，利用 reflog 查找分支最後一個 commit 並恢復。

### 最佳實踐與常見錯誤排查
- **最佳實踐：** 定期檢查 reflog，重要操作前建立 tag。
- **常見錯誤：** reflog 有保存期限（預設 90 天），過期後無法救援。

---

## 6.2 bisect 二分查找錯誤

### 主題簡介
`git bisect` 透過二分搜尋法自動定位引入 bug 的 commit，適用於大型專案或歷史悠久的代碼庫。

### 原理說明
指定一個「正常」與「有錯」的 commit，Git 會自動 checkout 中間點，工程師標記好壞，直到找出問題 commit。

### 常用命令與語法範例

```shell
$ git bisect start
$ git bisect bad                # 標記目前 commit 有 bug
$ git bisect good v1.0.0        # 標記 v1.0.0 為正常
```
**每次自動切換後，執行測試並標記：**
```shell
$ git bisect good
$ git bisect bad
```
**結束並回到原分支：**
```shell
$ git bisect reset
```

### 實際開發場景與案例
- 追查某功能何時壞掉，快速定位引入 bug 的 commit。
- 可結合自動化測試腳本，完全自動化查找。

### 最佳實踐與常見錯誤排查
- **最佳實踐：** 配合自動化測試腳本提升效率。
- **常見錯誤：** 標記 good/bad 時誤判，導致結果不準確。

---

## 6.3 filter-branch 與 git filter-repo

### 主題簡介
用於批次重寫 Git 歷史（如移除敏感資訊、調整作者、目錄重構），`git filter-branch` 與 `git filter-repo` 是兩大主力工具。

### 原理說明
- `git filter-branch` 透過 shell script 遍歷 commit 並重寫歷史，速度較慢。
- `git filter-repo`（推薦）以 Python 實作，效能高、語法簡潔，支援更複雜的過濾需求。

### 常用命令與語法範例

**移除檔案（filter-branch）：**
```shell
$ git filter-branch --tree-filter 'rm -f secret.txt' -- --all
```

**更換作者（filter-repo）：**
```shell
$ git filter-repo --mailmap my-mailmap.txt
```

**只保留某目錄（filter-repo）：**
```shell
$ git filter-repo --path src/ --force
```

### 實際開發場景與案例
- 誤將密碼寫入 commit，需移除歷史紀錄。
- 專案目錄重構，僅保留特定子目錄歷史。
- 公司合併，統一作者資訊。

### 最佳實踐與常見錯誤排查
- **最佳實踐：** 操作前備份 repo，操作後強制推送（force push）。
- **常見錯誤：** 忘記通知協作者，導致 push/pull 衝突。

---

## 6.4 hooks 自動化流程

### 主題簡介
Git hooks 允許在特定事件（如 commit、push）自動執行腳本，實現自動化檢查、格式化、部署等流程。

### 原理說明
Git 於 `.git/hooks/` 目錄下提供多種 hook 範本（如 `pre-commit`、`pre-push`），可自訂 shell、Python、Node.js 腳本。

### 常用命令與語法範例

**建立 pre-commit hook：**
```shell
$ vi .git/hooks/pre-commit
```
內容範例：
```bash
#!/bin/sh
npm run lint
if [ $? -ne 0 ]; then
  echo "Lint failed!"
  exit 1
fi
```
**賦予執行權限：**
```shell
$ chmod +x .git/hooks/pre-commit
```

### 實際開發場景與案例
- 提交前自動格式化代碼、檢查測試。
- push 前自動執行 CI/CD 部署腳本。

### 最佳實踐與常見錯誤排查
- **最佳實踐：** 搭配 Husky 等工具集中管理 hooks。
- **常見錯誤：** 忘記賦予執行權限，導致 hook 無效。

---

## 6.5 進階命令組合實戰

### 主題簡介
結合多個 Git 命令與 shell 工具，實現高效批次操作與自動化，提升日常開發效率。

### 原理說明
利用管線（|）、xargs、find 等 shell 工具，批次處理分支、標籤、提交等。

### 常用命令與語法範例

**批次刪除本地已合併分支：**
```shell
$ git branch --merged | grep -v '\*' | xargs git branch -d
```

**查找含特定關鍵字的 commit：**
```shell
$ git log --all --grep="fix bug"
```

**統計每人提交數量：**
```shell
$ git shortlog -s -n --all
```

### 實際開發場景與案例
- 清理長期未用分支。
- 追蹤某功能相關所有 commit。
- 產生貢獻者排行榜。

### 最佳實踐與常見錯誤排查
- **最佳實踐：** 命令前先 dry-run，避免誤刪。
- **常見錯誤：** xargs 未處理空白分支名，導致刪除失敗。

---

## 6.7 git worktree 多工作目錄管理

### 主題簡介
`git worktree` 允許一個 Git 倉庫同時檢出多個分支於不同目錄，適合多任務開發、熱修復、跨版本維護等場景。

### 原理說明
Git 預設一個倉庫僅能在單一目錄檢出一個分支。`git worktree` 透過建立額外工作目錄，讓同一個 repo 可同時操作多個分支，彼此獨立但共用物件資料庫（.git/objects）。

### 常用命令與語法範例

```shell
# 新增一個工作目錄並檢出分支
$ git worktree add ../feature-x feature-x

# 列出所有 worktree
$ git worktree list

# 移除工作目錄
$ git worktree remove ../feature-x
```

### 實戰案例

假設你在主目錄開發主線，需同時修復舊版分支，可：

```shell
$ git worktree add ../hotfix hotfix-branch
$ cd ../hotfix
# 修復後提交
$ git commit -am "fix: hotfix"
```

### 結構圖示

```text
主倉庫/
├── .git/
├── src/
├── ...
├── worktree-feature-x/   ← git worktree add 建立
│   └── (feature-x 分支)
└── worktree-hotfix/      ← git worktree add 建立
    └── (hotfix-branch 分支)
```

---

## 6.8 git rebase 進階操作

### 主題簡介
`git rebase` 用於重寫分支歷史、整理 commit、合併分支。常見進階用法包含互動式 rebase、--onto、squash 等。

### 原理說明
rebase 會將當前分支的一系列 commit 取下，重新套用到指定基底（base）之後，讓歷史更線性、易讀。

### 常用命令與語法範例

#### 1. 互動式 rebase

```shell
$ git rebase -i HEAD~5
```
可編輯、合併、調整順序、squash commit。

#### 2. --onto 用法

```shell
$ git rebase --onto new-base old-base feature
```
將 feature 分支自 old-base 之後的 commit，移動到 new-base 之後。

#### 3. squash 合併多個 commit

```shell
$ git rebase -i HEAD~3
# 將多個 pick 改為 squash
```

### 圖示：rebase 前後分支結構

```text
# rebase 前
master:   A---B---C
                \
feature:         D---E

# git rebase master
master:   A---B---C
                        \
feature:                 D'--E'
```

#### --onto 圖示

```text
# rebase --onto new-base old-base feature
main:     A---B---C (old-base)
                   \
feature:            D---E
new-base:      X---Y

# 執行後
main:     A---B---C
new-base:      X---Y---D'---E'
```

### 實戰案例

- 合併多個 commit，優化 PR 歷史。
- 將 feature 分支從舊基底移動到新基底，避免 merge commit。

---

## 6.9 git squash 操作

### 主題簡介
`squash` 將多個 commit 合併為一，常用於整理開發歷史、提交 PR 前壓縮 commit。

### 原理說明
透過 rebase -i 或 merge --squash，將多個 commit 內容合併，僅保留一筆 commit 記錄，讓歷史更乾淨。

### 常用命令與語法範例

#### 1. 互動式 rebase squash

```shell
$ git rebase -i HEAD~3
# 將第二、三個 pick 改為 squash
```

#### 2. merge --squash

```shell
$ git checkout main
$ git merge --squash feature
$ git commit -m "squash: 合併 feature 所有更動"
```

### 圖示：多 commit 合併為一

```text
# squash 前
A---B---C (feature)

# squash 後
A---D (squash commit)
```

### 實戰案例

- PR 前將多個修正 commit 壓縮為一，提升審查效率。
- 合併 feature 分支時，僅保留一筆合併紀錄。

---

## 6.6 進階技巧與最佳實踐

### 主題簡介
彙整資深工程師常用的 Git 進階技巧、最佳實踐與常見錯誤排查，提升團隊協作與代碼品質。

### 原理說明
- 善用 alias、rebase、cherry-pick、stash 等進階功能。
- 建立嚴謹的分支策略與 commit message 規範。

### 常用命令與語法範例

**自訂 alias：**
```shell
$ git config --global alias.lg "log --oneline --graph --all"
$ git lg
```

**互動式 rebase：**
```shell
$ git rebase -i HEAD~5
```

**挑選特定 commit：**
```shell
$ git cherry-pick <commit-sha>
```

**暫存工作區：**
```shell
$ git stash
$ git stash pop
```

### 實際開發場景與案例
- 互動式 rebase 合併多個 commit，優化歷史。
- cherry-pick 熱修復至多個分支。
- stash 暫存未完成工作，切換緊急任務。

### 最佳實踐與常見錯誤排查
- **最佳實踐：** commit message 清楚、原子性高，rebase 前先 pull 最新代碼。
- **常見錯誤：** rebase 衝突未解決、stash 遺漏導致資料丟失。

---
