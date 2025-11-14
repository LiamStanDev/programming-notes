# 07-GitHub Actions

## 7.1 GitHub Actions 概念與架構

GitHub Actions 是 GitHub 官方提供的 CI/CD（持續整合與持續部署）平台，讓開發者能直接在 GitHub 上自動化各種軟體開發流程。其核心概念為「事件驅動」，可根據如 push、pull request、release 等事件自動觸發工作流程（Workflow）。架構上，GitHub Actions 由 Workflow、Job、Step、Runner 與 Action 組成，支援跨平台（Linux、Windows、macOS）執行，並可整合第三方服務。

## 7.2 Workflow 與 Job 基本結構

- **Workflow**：定義於 `.github/workflows/` 目錄下的 YAML 檔案，是自動化流程的最外層單位。
- **Job**：Workflow 內可包含多個 Job，每個 Job 通常在獨立的 Runner 上執行，彼此可並行或串行。
- **Step**：Job 由多個 Step 組成，每個 Step 執行一個命令或 Action。
- **Runner**：執行 Job 的主機，可使用 GitHub 提供的雲端 Runner 或自架 Runner。
- **Action**：可重複使用的自動化腳本，官方與社群皆可開發與分享。

## 7.3 YAML 配置語法與範例

### 基本語法

```yaml
name: CI Example
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: 檢出程式碼
        uses: actions/checkout@v4
      - name: 安裝 Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      - name: 安裝依賴
        run: npm install
      - name: 執行測試
        run: npm test
```

### 實際輸出範例

```
Run actions/checkout@v4
Checked out refs/heads/main to path
Run actions/setup-node@v4
Using Node.js version 20.x
Run npm install
added 200 packages in 3s
Run npm test
PASS  ./sum.test.js
```

## 7.4 常見自動化流程（CI、CD、Lint、Test）

- **CI（持續整合）**：自動編譯、測試、靜態檢查
- **CD（持續部署）**：自動部署至測試或生產環境
- **Lint**：自動執行程式碼風格檢查
- **Test**：自動化單元測試與整合測試

#### 範例：CI + Lint + Test

```yaml
name: Node.js CI
on: [push]
jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run lint
      - run: npm test
```

#### 範例：CD 自動部署

```yaml
name: Deploy to Production
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: 部署腳本
        run: ./deploy.sh
```

## 7.5 Marketplace 與自訂 Action

- **Marketplace**：GitHub Marketplace 提供數千個現成 Action，可直接引用（如 `actions/checkout`、`actions/setup-node`）。
- **自訂 Action**：可用 JavaScript、Docker 或 Composite（YAML）撰寫自訂 Action，並於專案或 Marketplace 發布。

#### 自訂 JavaScript Action 範例

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ./my-actions/hello-world
    with:
      who-to-greet: '工程師'
```

## 7.6 實戰案例：自動化部署與測試

### 案例：自動部署至 GitHub Pages

```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [main]
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: 安裝依賴
        run: npm ci
      - name: 建置靜態網站
        run: npm run build
      - name: 部署至 GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./dist
```

### 案例：自動化測試與 Slack 通知

```yaml
name: Test and Notify
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
      - name: 通知 Slack
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          fields: repo,message,commit,author,action,eventName,ref,workflow,job,took
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## 7.7 進階技巧與最佳實踐

### 進階技巧

- **矩陣（Matrix）策略**：同時測試多種環境
  ```yaml
  strategy:
    matrix:
      node: [16, 18, 20]
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node }}
  ```
- **條件執行**：根據分支、檔案變更、Job 狀態決定是否執行
  ```yaml
  if: github.ref == 'refs/heads/main'
  ```
- **快取依賴**：加速安裝流程
  ```yaml
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node-
  ```
- **自架 Runner**：提升效能或存取內部資源

### 最佳實踐

- 將敏感資訊（如 API 金鑰）存放於 GitHub Secrets
- 每個 Job 保持單一職責，易於維護與除錯
- 善用 Marketplace Action，減少重複造輪子
- 定期檢查 Workflow 執行時間與資源消耗
- 使用 `pull_request` 事件保證合併前品質

### 常見錯誤排查

- **YAML 語法錯誤**：建議使用 YAML Linter 工具
- **權限不足**：檢查 Token 權限與 Secrets 設定
- **Action 版本錯誤**：確認引用的 Action 版本正確
- **Runner 資源不足**：可考慮自架 Runner 或優化流程
- **Job 依賴失敗**：檢查 Job 之間的依賴與條件設定

---

GitHub Actions 是現代 DevOps 流程不可或缺的工具，熟練掌握其配置與進階技巧，能大幅提升開發效率與專案品質。