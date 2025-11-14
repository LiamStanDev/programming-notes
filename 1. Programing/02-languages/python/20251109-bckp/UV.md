## 專案管理
---
#### 建立項目

##### 方式一：舊項目
```shell
# 建立專案資料夾
mkdir myproject && cd myproject

# 建立虛擬環境
uv init
```

##### 方式二：新項目
```shell
uv init myproject
```

> uv.lock 檔案會在你每次運行 `uv sync`, `uv run` 產生/更新

#### 依賴管理
```shell
# 添加依賴
uv add <pkgs>

# 從 requirements.txt 遷移
uv add -r requirements.txt

# 移除依賴
uv remove requests

# 更新依賴: 更新到最新兼容版本
uv lock --upgrade-package <pkgs>
```

#### 執行
```shell
# 運行
uv run example.py
```

> 1. 使用 `uv run` 可以不在虛擬環境中就能運行
> 2. `uv run` 會依照 `pyproject` 描述更新 `uv.lock` 後運行


#### 構建
產生 `source distributions` 與 `binary distributions (wheel)`
```shell
uv build
ls /dist/
```

結果如下
```shell
$ ls dist/
hello-world-0.1.0-tar.gz # source
hello-world-0.1.0-py3-none-any.whl # wheel
```


## 命令分類
---
#### python 管理
```shell
# 列出可下載的 python 版本
uv python list
# 安裝指定的 python 版本
uv python install <python-version>
# 卸載 python 版本
uv python uninstall <python-verison>
# 設定當前項目使用的 python 版本
uv python pin <python-version>
```

#### 項目管理
```shell
# 初始戶項目
uv init --python=3.13
# 添加項目依賴
uv add <pkgs>
# 移除依賴 
uv remove <pkgs>
# 同步項目依賴
uv sync
# 產出 lock 檔案
uv lock
# 在 project env 下執行
uv run <python-file>
# 查看依賴樹
uv tree
# 打包
uv build
# 釋出
uv publish
```

#### Tools
```shell
# 運行工具 e.g. ruff, black, ...
uvx <tool> # or uv tool run <tool>
# 安裝工具
uv tool install <tool>
# 卸載工具
uv tool uninstall <tool>
# 查看安裝的工具
uv tool list
# update shell env to executables
uv tool update-shell
```

#### Legacy workflows (PIP interface)
```shell
# 建立虛擬環境
uv venv --python=3.13
# 管理包
uv pip install <pkgs>
uv pip uninstall <pkgs>
uv pip show <pkgs>
uv pip freeze
```
