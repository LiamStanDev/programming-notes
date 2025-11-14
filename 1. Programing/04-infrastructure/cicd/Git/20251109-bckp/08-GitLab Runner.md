# 08-GitLab Runner

## 8.1 GitLab Runner 概念與架構

GitLab Runner 是 GitLab CI/CD 的執行代理程式，負責接收 GitLab 伺服器下發的 pipeline 任務，並在指定環境中執行自動化流程（如建置、測試、部署等）。Runner 支援多種執行模式（Shell、Docker、Kubernetes 等），可根據專案需求彈性配置。核心概念包含：

- **Runner**：負責執行 CI/CD 任務的代理程式。
- **Executor**：Runner 執行任務的方式（如 Shell、Docker）。
- **Pipeline**：由多個 Job 組成的自動化流程。
- **Job**：單一自動化任務（如 build、test）。

架構上，GitLab Runner 可分為 Shared Runner（全組織共用）與 Specific Runner（專案專用），並可橫跨多台主機部署，提升彈性與擴展性。

---

## 8.2 Pipeline 配置語法與範例

GitLab CI/CD pipeline 以 `.gitlab-ci.yml` 配置，描述各階段（stages）、任務（jobs）及執行條件。基本語法如下：

```yaml
stages:
  - build
  - test
  - deploy

build-job:
  stage: build
  script:
    - echo "Building..."
    - make build

test-job:
  stage: test
  script:
    - echo "Testing..."
    - make test

deploy-job:
  stage: deploy
  script:
    - echo "Deploying..."
    - ./deploy.sh
  only:
    - main
```

**實際輸出範例：**

```
$ echo "Building..."
Building...
$ make build
[build output]
$ echo "Testing..."
Testing...
$ make test
[testing output]
$ echo "Deploying..."
Deploying...
$ ./deploy.sh
[deploy output]
```

---

## 8.3 Runner 安裝與註冊流程

### 安裝步驟（以 Linux 為例）：

1. 下載並安裝 Runner：
   ```bash
   sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64
   sudo chmod +x /usr/local/bin/gitlab-runner
   sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
   sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
   sudo gitlab-runner start
   ```

2. 註冊 Runner：
   ```bash
   sudo gitlab-runner register
   ```
   依照提示輸入 GitLab URL、註冊 token、描述、標籤、執行模式（executor）。

### 註冊流程說明

- **GitLab URL**：GitLab 伺服器網址。
- **Token**：於專案或群組設定頁取得。
- **Executor**：選擇 Shell、Docker、Kubernetes 等。

---

## 8.4 執行模式（Shell、Docker、Kubernetes 等）

GitLab Runner 支援多種執行模式：

- **Shell Executor**：直接在主機 shell 執行，適合簡單任務或自架主機。
- **Docker Executor**：每個 Job 於獨立容器執行，隔離性高，易於管理依賴。
- **Docker Machine Executor**：自動建立/銷毀雲端 VM，適合動態擴展。
- **Kubernetes Executor**：於 K8s 叢集動態建立 Pod 執行 Job，適合大規模自動化。
- **Custom Executor**：自訂執行邏輯，彈性最高。

**範例：Docker Executor 設定片段**
```yaml
image: node:18

stages:
  - test

test-job:
  stage: test
  script:
    - npm install
    - npm test
```
此時 Runner 會自動拉取 `node:18` 容器執行 Job。

---

## 8.5 常見自動化流程（CI、CD、Lint、Test）

### CI（持續整合）範例

```yaml
stages:
  - lint
  - test

lint-job:
  stage: lint
  script:
    - npm run lint

test-job:
  stage: test
  script:
    - npm test
```

### CD（持續部署）範例

```yaml
deploy-prod:
  stage: deploy
  script:
    - ./deploy-prod.sh
  only:
    - tags
```

### Lint、Test 輸出範例

```
$ npm run lint
No lint errors found!
$ npm test
PASS  tests/app.test.js
```

---

## 8.6 實戰案例：多環境部署與測試

### 多環境部署範例

```yaml
stages:
  - build
  - test
  - deploy

deploy-dev:
  stage: deploy
  script:
    - ./deploy.sh dev
  environment:
    name: development
  only:
    - develop

deploy-prod:
  stage: deploy
  script:
    - ./deploy.sh prod
  environment:
    name: production
  only:
    - main
```

### 多環境測試範例

```yaml
test-dev:
  stage: test
  script:
    - npm run test:dev
  only:
    - develop

test-prod:
  stage: test
  script:
    - npm run test:prod
  only:
    - main
```

**實際開發場景：**
- 開發分支自動部署至測試環境，主分支合併後自動部署至正式環境。
- 不同分支可執行不同測試腳本，確保各環境穩定。

---

## 8.7 進階技巧與最佳實踐

### 進階技巧

- **Cache 機制**：加速依賴安裝
  ```yaml
  cache:
    paths:
      - node_modules/
  ```
- **Artifacts**：保存建置產物供後續 Job 使用
  ```yaml
  artifacts:
    paths:
      - dist/
  ```
- **Matrix Build**：多版本/多平台測試
  ```yaml
  test:
    script: npm test
    parallel:
      matrix:
        - NODE_VERSION: [16, 18, 20]
  ```
- **自訂 Runner 標籤**：針對不同硬體/環境分流 Job
  ```yaml
  tags:
    - gpu
  ```

### 最佳實踐

- 明確區分 stages，避免 Job 過於複雜。
- 善用 artifacts、cache 提升效率。
- 使用 secret variables 管理敏感資訊。
- 定期更新 Runner 版本，確保安全與新功能。
- 監控 Runner 狀態，避免資源瓶頸。

### 常見錯誤排查

- **Job 卡住不執行**：檢查 Runner 是否離線、標籤是否對應。
- **權限問題**：確認 Runner 執行帳號權限、CI 變數設置。
- **Docker 相關錯誤**：檢查 Docker Daemon 狀態、映像檔可用性。
- **YAML 語法錯誤**：可用 `CI Lint` 工具驗證 `.gitlab-ci.yml`。

---

本章節涵蓋 GitLab Runner 的核心原理、安裝註冊、常見配置與進階技巧，適合工程師實務操作與面試準備。