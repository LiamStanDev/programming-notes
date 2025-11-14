# 09-CICD 全面指南

本章將深入探討現代軟體開發中不可或缺的 CI/CD（持續整合與持續部署）技術，涵蓋從版本發布策略、自動化測試、部署策略、安全品質、Pipeline 優化到端到端實戰案例與進階技巧，協助工程師建立高效、穩健的自動化交付流程。

---

## 9.1 版本發布策略與自動化

### 主題簡介
版本發布策略決定了軟體如何從開發階段順利推進到生產環境。自動化發布能減少人為錯誤、提升效率，並確保每次發布皆可追溯。

### 詳細原理說明
常見策略包括：
- **語意化版本控制（SemVer）**：`主.次.修`，如 `1.2.3`。
- **標籤（Tagging）自動化**：CI 工具自動產生 Git 標籤。
- **Changelog 生成**：自動彙整變更紀錄。

### 常用命令與配置範例

**Git 標籤自動化：**
```bash
git tag v1.2.3
git push origin v1.2.3
```
**GitHub Actions 自動標籤：**
```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]
jobs:
  tag:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Create Tag
        run: |
          git tag v${{ github.run_number }}
          git push origin v${{ github.run_number }}
```
**實際輸出：**
```
To github.com:repo/project.git
 * [new tag]         v1.2.3 -> v1.2.3
```

### 實際開發場景與案例
- 發布新功能時自動產生版本號與 changelog。
- 修復緊急 bug 時自動標記 hotfix 版本。

### 最佳實踐與常見錯誤排查
- **最佳實踐**：嚴格遵循語意化版本規則，避免手動標籤。
- **常見錯誤**：標籤重複、未同步遠端，解法：`git tag -d v1.2.3 && git push origin :refs/tags/v1.2.3`

### 進階技巧
- 結合 Conventional Commits，自動產生 changelog 與版本號。
- 使用 release bot 實現全自動發布。

---

## 9.2 自動化測試設計與實踐

### 主題簡介
自動化測試確保每次提交都能即時驗證程式正確性，減少回歸錯誤。

### 詳細原理說明
- **單元測試（Unit Test）**：驗證最小功能單元。
- **整合測試（Integration Test）**：驗證模組間協作。
- **端到端測試（E2E）**：模擬用戶操作全流程。

### 常用命令與配置範例

**Jest 單元測試：**
```bash
npx jest
```
**GitLab CI 測試配置：**
```yaml
# .gitlab-ci.yml
stages:
  - test
test:
  stage: test
  script:
    - npm ci
    - npm test
```
**實際輸出：**
```
PASS  src/app.test.js
✓ add() returns correct sum (5 ms)
Test Suites: 1 passed, 1 total
```

### 實際開發場景與案例
- PR 合併前自動執行所有測試。
- 部署前阻擋測試失敗的版本。

### 最佳實踐與常見錯誤排查
- **最佳實踐**：測試覆蓋率納入 CI Gate，確保關鍵路徑皆被測試。
- **常見錯誤**：測試環境依賴未正確 mock，導致測試不穩定。

### 進階技巧
- 並行執行多組測試以加速 Pipeline。
- 使用 Test Report 上傳與自動通知失敗原因。

---

## 9.3 部署策略（藍綠部署、滾動更新、Canary）

### 主題簡介
部署策略決定新版本如何平滑上線，降低服務中斷與風險。

### 詳細原理說明
- **藍綠部署（Blue-Green）**：同時維護兩套環境，切換流量。
- **滾動更新（Rolling Update）**：逐步替換實例，無縫升級。
- **Canary 部署**：先讓部分用戶使用新版本，觀察無誤再全面推廣。

### 常用命令與配置範例

**Kubernetes 滾動更新：**
```bash
kubectl rollout status deployment/my-app
```
**K8s Canary 配置：**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
```
**實際輸出：**
```
deployment "my-app" successfully rolled out
```

### 實際開發場景與案例
- 金融服務採用藍綠部署，確保零停機。
- 新功能先以 5% 用戶 Canary 部署，觀察指標。

### 最佳實踐與常見錯誤排查
- **最佳實踐**：自動化健康檢查與回滾機制。
- **常見錯誤**：未設置 readiness probe，導致流量導向未就緒實例。

### 進階技巧
- 結合 Service Mesh（如 Istio）實現流量細緻分流。
- 自動化異常監控與自動回滾。

---

## 9.4 安全掃描與品質保證

### 主題簡介
安全與品質是 CI/CD 不可或缺的一環，能及早發現漏洞與潛在問題。

### 詳細原理說明
- **靜態程式碼分析（SAST）**：掃描原始碼漏洞。
- **依賴漏洞掃描（Dependency Scan）**：檢查第三方套件風險。
- **動態分析（DAST）**：模擬攻擊測試應用程式。

### 常用命令與配置範例

**GitLab SAST 配置：**
```yaml
include:
  - template: Security/SAST.gitlab-ci.yml
```
**OWASP ZAP 動態掃描：**
```bash
zap-cli quick-scan --self-contained http://localhost:8080
```
**實際輸出：**
```
PASS: No vulnerabilities found
FAIL: SQL Injection detected in /api/user
```

### 實際開發場景與案例
- PR 合併前自動執行 SAST，阻擋有高風險漏洞的程式碼。
- 定期掃描依賴，主動升級有漏洞的套件。

### 最佳實踐與常見錯誤排查
- **最佳實踐**：將安全掃描納入每次 CI 流程。
- **常見錯誤**：掃描誤報，需定期調整規則與白名單。

### 進階技巧
- 整合多種掃描工具，提升偵測覆蓋率。
- 自動產生安全報告並通知相關人員。

---

## 9.5 高效能 Pipeline 設計

### 主題簡介
高效能 Pipeline 能大幅縮短交付週期，提升團隊生產力。

### 詳細原理說明
- **階段並行化**：同時執行多個 Job。
- **快取（Cache）與重用**：減少重複下載與建構。
- **條件執行**：根據分支、標籤或檔案變更決定執行內容。

### 常用命令與配置範例

**GitHub Actions 並行 Job：**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [...]
  test:
    runs-on: ubuntu-latest
    steps: [...]
```
**GitLab CI 快取設定：**
```yaml
cache:
  paths:
    - node_modules/
```
**實際輸出：**
```
Restoring cache
Job succeeded
```

### 實際開發場景與案例
- 前端、後端分開 build 與測試，並行加速。
- 只針對有變更的模組執行對應 Job。

### 最佳實踐與常見錯誤排查
- **最佳實踐**：合理拆分 Job，避免單一 Job 過大。
- **常見錯誤**：快取未命中，導致重複下載。

### 進階技巧
- 動態生成 Pipeline，根據 PR 內容自動調整流程。
- 使用 Matrix 策略測試多版本組合。

---

## 9.6 實戰案例：端到端自動化流程

### 主題簡介
端到端自動化流程涵蓋從程式提交、測試、建構、部署到監控的全流程。

### 詳細原理說明
- **觸發條件**：如 PR、Push、定時任務。
- **多階段串接**：測試、建構、部署、驗證。
- **自動回報**：失敗即時通知、產生報告。

### 常用命令與配置範例

**GitHub Actions 端到端流程：**
```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
      - run: npm run build
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to production..."
```
**實際輸出：**
```
Build succeeded
Tests passed
Deploying to production...
```

### 實際開發場景與案例
- 每次合併主分支即自動部署至測試環境。
- 失敗時自動回滾並通知 Slack 頻道。

### 最佳實踐與常見錯誤排查
- **最佳實踐**：流程全自動化，減少人工干預。
- **常見錯誤**：部署權限不足，需預先配置憑證。

### 進階技巧
- 整合 ChatOps，讓團隊可用聊天指令觸發部署。
- 自動化驗證部署後服務健康狀態。

---

## 9.7 進階技巧與最佳實踐

### 主題簡介
進階技巧與最佳實踐能讓 CI/CD 流程更穩健、可維護且具備彈性。

### 詳細原理說明
- **動態參數化**：根據環境自動切換配置。
- **秘密管理**：安全儲存與調用敏感資訊。
- **觀測性（Observability）**：全流程監控與追蹤。

### 常用命令與配置範例

**GitHub Actions Secrets：**
```yaml
- name: Use secret
  run: echo ${{ secrets.MY_SECRET }}
```
**Kubernetes Secret 管理：**
```bash
kubectl create secret generic db-pass --from-literal=password=123456
```
**實際輸出：**
```
secret/db-pass created
```

### 實際開發場景與案例
- 依據不同環境自動切換 API endpoint。
- 部署時自動注入憑證與密碼。

### 最佳實踐與常見錯誤排查
- **最佳實踐**：所有敏感資訊皆透過 Secret 管理，嚴禁硬編碼。
- **常見錯誤**：憑證過期未自動更新，導致部署失敗。

### 進階技巧
- 結合 IaC（Infrastructure as Code）自動化管理基礎設施。
- Pipeline 內建自我修復與異常通知機制。

---
