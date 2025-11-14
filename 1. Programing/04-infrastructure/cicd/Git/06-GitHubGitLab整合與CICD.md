# GitHub/GitLab 整合與 CI/CD

## 1. 平台功能比較與選擇

### 1.1 GitHub vs GitLab 功能對比

```mermaid
graph TB
    subgraph "GitHub"
        A1[GitHub Actions] --> A2[Marketplace]
        A3[Issues & Projects] --> A4[GitHub Pages]
        A5[Security Features] --> A6[Codespaces]
        A7[Discussions] --> A8[Packages]
    end
    
    subgraph "GitLab"  
        B1[GitLab CI/CD] --> B2[Built-in Registry]
        B3[Issue Boards] --> B4[GitLab Pages]
        B5[Security Scanning] --> B6[Web IDE]
        B7[Wiki] --> B8[Container Registry]
    end
    
    subgraph "共同特性"
        C1[Git Repository]
        C2[Pull/Merge Requests]
        C3[Branch Protection]
        C4[Webhooks]
    end
```

| 功能 | GitHub | GitLab | 說明 |
|------|--------|--------|------|
| CI/CD | GitHub Actions | GitLab CI/CD | GitLab 內建，GitHub 需要額外配置 |
| 容器註冊表 | GitHub Packages | 內建 Container Registry | GitLab 整合度更高 |
| 安全掃描 | Advanced Security | 內建 Security Scanning | 都支援 SAST/DAST/依賴掃描 |
| 項目管理 | Projects Beta | Issue Boards | GitLab 功能更完整 |
| 託管方式 | 雲端為主 | 雲端 + 自託管 | GitLab 自託管選項更豐富 |

### 1.2 選擇決策因素

```bash
# 評估腳本：選擇合適的平台
cat > scripts/platform-evaluation.sh << 'EOF'
#!/bin/bash

echo "🔍 Git Platform Evaluation Tool"
echo "================================"

score_github=0
score_gitlab=0

# 團隊規模
read -p "Team size (1-Small <10, 2-Medium 10-50, 3-Large >50): " team_size
case $team_size in
    1) score_github=$((score_github + 2)); score_gitlab=$((score_gitlab + 1)) ;;
    2) score_github=$((score_github + 1)); score_gitlab=$((score_gitlab + 2)) ;;
    3) score_github=$((score_github + 1)); score_gitlab=$((score_gitlab + 3)) ;;
esac

# 預算考量
read -p "Budget priority (1-Low cost, 2-Medium, 3-Feature rich): " budget
case $budget in
    1) score_gitlab=$((score_gitlab + 3)); score_github=$((score_github + 1)) ;;
    2) score_github=$((score_github + 2)); score_gitlab=$((score_gitlab + 2)) ;;
    3) score_github=$((score_github + 3)); score_gitlab=$((score_gitlab + 1)) ;;
esac

# 安全要求
read -p "Security requirements (1-Basic, 2-Advanced, 3-Enterprise): " security
case $security in
    1) score_github=$((score_github + 2)); score_gitlab=$((score_gitlab + 2)) ;;
    2) score_github=$((score_github + 2)); score_gitlab=$((score_gitlab + 3)) ;;
    3) score_github=$((score_github + 1)); score_gitlab=$((score_gitlab + 3)) ;;
esac

# CI/CD 需求
read -p "CI/CD complexity (1-Simple, 2-Moderate, 3-Complex): " cicd
case $cicd in
    1) score_github=$((score_github + 2)); score_gitlab=$((score_gitlab + 2)) ;;
    2) score_github=$((score_github + 2)); score_gitlab=$((score_gitlab + 3)) ;;
    3) score_github=$((score_github + 1)); score_gitlab=$((score_gitlab + 3)) ;;
esac

echo ""
echo "📊 Evaluation Results:"
echo "GitHub Score: $score_github"
echo "GitLab Score: $score_gitlab"

if [ $score_github -gt $score_gitlab ]; then
    echo "🏆 Recommendation: GitHub"
    echo "Reasons: Better for smaller teams, strong ecosystem, excellent for open source"
elif [ $score_gitlab -gt $score_github ]; then
    echo "🏆 Recommendation: GitLab"  
    echo "Reasons: Better DevOps integration, self-hosting options, comprehensive feature set"
else
    echo "🤝 Both platforms are equally suitable for your needs"
fi
EOF

chmod +x scripts/platform-evaluation.sh
```

## 2. GitHub Actions 深度應用

### 2.1 GitHub Actions 架構

```mermaid
graph TB
    A[Workflow 觸發] --> B[Runner 分配]
    B --> C[Job 1]
    B --> D[Job 2]
    B --> E[Job 3]
    
    C --> C1[Step 1: Checkout]
    C --> C2[Step 2: Setup]
    C --> C3[Step 3: Build]
    C --> C4[Step 4: Test]
    
    D --> D1[Step 1: Security Scan]
    D --> D2[Step 2: Code Quality]
    
    E --> E1[Step 1: Deploy Staging]
    E --> E2[Step 2: Integration Test]
    E --> E3[Step 3: Deploy Production]
```

### 2.2 完整的 CI/CD Pipeline

#### Node.js 應用的企業級 CI/CD

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  release:
    types: [ published ]

env:
  NODE_VERSION: '18.x'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # 代碼質量檢查
  code-quality:
    runs-on: ubuntu-latest
    outputs:
      cache-key: ${{ steps.cache-key.outputs.key }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        fetch-depth: 0  # 完整歷史，用於 SonarQube

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ env.NODE_VERSION }}
        cache: 'npm'

    - name: Generate cache key
      id: cache-key
      run: echo "key=node-modules-${{ hashFiles('**/package-lock.json') }}" >> $GITHUB_OUTPUT

    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: node_modules
        key: ${{ steps.cache-key.outputs.key }}
        restore-keys: |
          node-modules-

    - name: Install dependencies
      run: npm ci

    - name: Run ESLint
      run: npm run lint -- --format=json --output-file=eslint-report.json
      continue-on-error: true

    - name: Run Prettier check
      run: npm run format:check

    - name: TypeScript type check
      run: npm run type-check

    - name: Upload ESLint report
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: eslint-report
        path: eslint-report.json

  # 單元測試和覆蓋率
  test:
    runs-on: ubuntu-latest
    needs: code-quality
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'

    - name: Restore dependencies cache
      uses: actions/cache@v3
      with:
        path: node_modules
        key: ${{ needs.code-quality.outputs.cache-key }}

    - name: Install dependencies
      run: npm ci

    - name: Run unit tests
      run: npm test -- --coverage --watchAll=false

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/lcov.info
        flags: unittests
        name: codecov-umbrella

  # 安全掃描
  security:
    runs-on: ubuntu-latest
    needs: code-quality
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'

    - name: Run npm audit
      run: npm audit --audit-level high

    - name: Run Snyk security scan
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high

  # 建構和容器化
  build:
    runs-on: ubuntu-latest
    needs: [code-quality, test, security]
    if: github.event_name != 'pull_request'
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ env.NODE_VERSION }}
        cache: 'npm'

    - name: Restore dependencies cache
      uses: actions/cache@v3
      with:
        path: node_modules
        key: ${{ needs.code-quality.outputs.cache-key }}

    - name: Install dependencies
      run: npm ci

    - name: Build application
      run: npm run build

    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=sha,prefix=sha-

    - name: Build and push Docker image
      id: build
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  # 部署到測試環境
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.example.com
    steps:
    - name: Deploy to staging
      uses: azure/webapps-deploy@v2
      with:
        app-name: 'myapp-staging'
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE_STAGING }}
        images: ${{ needs.build.outputs.image-tag }}

    - name: Run smoke tests
      run: |
        curl -f https://staging.example.com/health || exit 1
        curl -f https://staging.example.com/api/status || exit 1

  # 整合測試
  integration-test:
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/develop'
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ env.NODE_VERSION }}
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Run integration tests
      run: npm run test:integration
      env:
        DATABASE_URL: postgres://postgres:postgres@localhost:5432/testdb
        API_BASE_URL: https://staging.example.com

  # 部署到生產環境
  deploy-production:
    runs-on: ubuntu-latest
    needs: [build, integration-test]
    if: github.event_name == 'release'
    environment:
      name: production
      url: https://example.com
    steps:
    - name: Deploy to production
      uses: azure/webapps-deploy@v2
      with:
        app-name: 'myapp-production'
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE_PRODUCTION }}
        images: ${{ needs.build.outputs.image-tag }}

    - name: Post-deployment verification
      run: |
        # 等待部署完成
        sleep 30
        
        # 健康檢查
        curl -f https://example.com/health || exit 1
        
        # 功能驗證
        curl -f https://example.com/api/status || exit 1

    - name: Notify deployment success
      uses: 8398a7/action-slack@v3
      with:
        status: success
        text: "🚀 Production deployment successful!"
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

  # 效能測試
  performance-test:
    runs-on: ubuntu-latest
    needs: deploy-production
    if: github.event_name == 'release'
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Run load tests
      uses: grafana/k6-action@v0.2.0
      with:
        filename: tests/performance/load-test.js
      env:
        TARGET_URL: https://example.com

    - name: Upload performance results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: performance-results
        path: results/
```

### 2.3 自定義 GitHub Actions

#### 創建可重用的 Action

```yaml
# .github/actions/setup-app/action.yml
name: 'Setup Application'
description: 'Setup Node.js application with caching'
inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '18.x'
  cache-dependency-path:
    description: 'Path to dependency files'
    required: false
    default: '**/package-lock.json'
  install-dependencies:
    description: 'Whether to install dependencies'
    required: false
    default: 'true'
outputs:
  cache-hit:
    description: 'Whether dependencies were restored from cache'
    value: ${{ steps.cache.outputs.cache-hit }}
runs:
  using: 'composite'
  steps:
  - name: Setup Node.js
    uses: actions/setup-node@v4
    with:
      node-version: ${{ inputs.node-version }}
      cache: 'npm'
      cache-dependency-path: ${{ inputs.cache-dependency-path }}

  - name: Cache node modules
    id: cache
    uses: actions/cache@v3
    with:
      path: node_modules
      key: ${{ runner.os }}-node-${{ hashFiles(inputs.cache-dependency-path) }}
      restore-keys: |
        ${{ runner.os }}-node-

  - name: Install dependencies
    if: inputs.install-dependencies == 'true' && steps.cache.outputs.cache-hit != 'true'
    shell: bash
    run: npm ci

  - name: Verify installation
    shell: bash
    run: npm list --depth=0
```

#### 使用自定義 Action

```yaml
# .github/workflows/custom-action-example.yml
name: Using Custom Actions

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup application
      uses: ./.github/actions/setup-app
      with:
        node-version: '18.x'
        install-dependencies: 'true'

    - name: Run tests
      run: npm test
```

## 3. GitLab CI/CD 深度應用

### 3.1 GitLab CI/CD 架構

```mermaid
graph LR
    A[.gitlab-ci.yml] --> B[Pipeline]
    B --> C[Stage 1: Build]
    B --> D[Stage 2: Test]
    B --> E[Stage 3: Deploy]
    
    C --> C1[Job 1.1]
    C --> C2[Job 1.2]
    
    D --> D1[Job 2.1]
    D --> D2[Job 2.2]
    D --> D3[Job 2.3]
    
    E --> E1[Job 3.1]
```

### 3.2 企業級 GitLab CI/CD Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - security
  - package
  - deploy-staging
  - integration-test
  - deploy-production

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  NODE_VERSION: "18.16.0"
  APP_NAME: "ecommerce-api"
  STAGING_URL: "https://staging-api.company.com"
  PRODUCTION_URL: "https://api.company.com"

# 全局緩存配置
.cache_template: &cache_definition
  cache:
    key: "$CI_COMMIT_REF_SLUG-$CI_PROJECT_DIR"
    paths:
      - node_modules/
      - .npm/
    policy: pull

# 基礎配置模板
.base_job:
  image: node:${NODE_VERSION}-alpine
  before_script:
    - apk add --no-cache git curl
    - npm config set cache .npm
    - npm ci --cache .npm --prefer-offline

# 建構階段
build:
  stage: build
  extends: .base_job
  cache:
    <<: *cache_definition
    policy: pull-push
  script:
    - echo "Building application..."
    - npm run build
    - echo "Build completed successfully"
  artifacts:
    paths:
      - dist/
      - node_modules/
    expire_in: 1 hour
  only:
    - branches
    - tags

# 代碼質量檢查
lint:
  stage: test
  extends: .base_job
  <<: *cache_definition
  needs: ["build"]
  script:
    - echo "Running code quality checks..."
    - npm run lint -- --format=junit --output-file=lint-report.xml
    - npm run format:check
    - npm run type-check
  artifacts:
    reports:
      junit: lint-report.xml
    when: always
    expire_in: 1 week

# 單元測試
unit-test:
  stage: test
  extends: .base_job
  <<: *cache_definition
  needs: ["build"]
  services:
    - name: redis:6-alpine
      alias: redis
    - name: postgres:13-alpine
      alias: postgres
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
    REDIS_URL: redis://redis:6379
    DATABASE_URL: postgres://testuser:testpass@postgres:5432/testdb
  script:
    - echo "Running unit tests..."
    - npm run test:unit -- --coverage --reporter=junit --outputFile=test-results.xml
  artifacts:
    reports:
      junit: test-results.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    paths:
      - coverage/
    when: always
    expire_in: 1 week
  coverage: '/Lines\s*:\s*(\d+\.\d+)%/'

# 整合測試
integration-test:
  stage: test
  extends: .base_job
  <<: *cache_definition
  needs: ["build"]
  services:
    - name: postgres:13-alpine
      alias: postgres-integration
    - name: redis:6-alpine
      alias: redis-integration
  variables:
    POSTGRES_DB: integrationdb
    POSTGRES_USER: integrationuser  
    POSTGRES_PASSWORD: integrationpass
    DATABASE_URL: postgres://integrationuser:integrationpass@postgres-integration:5432/integrationdb
    REDIS_URL: redis://redis-integration:6379
  script:
    - echo "Running integration tests..."
    - npm run test:integration
  artifacts:
    reports:
      junit: integration-test-results.xml
    when: always
    expire_in: 1 week

# 安全掃描
security-scan:
  stage: security
  image: owasp/zap2docker-stable
  needs: []
  script:
    - echo "Running security scans..."
    # 依賴漏洞掃描
    - npm audit --audit-level=moderate --json > npm-audit.json || true
    # SAST 掃描
    - /zap/zap-baseline.py -t $STAGING_URL -J zap-report.json || true
  artifacts:
    reports:
      sast: gl-sast-report.json
    paths:
      - npm-audit.json
      - zap-report.json
    when: always
    expire_in: 1 week
  allow_failure: true

# 許可證檢查
license-check:
  stage: security
  extends: .base_job
  <<: *cache_definition
  needs: ["build"]
  script:
    - echo "Checking licenses..."
    - npx license-checker --json --out license-report.json
    - npx license-checker --failOn 'GPL;AGPL'
  artifacts:
    paths:
      - license-report.json
    expire_in: 1 week
  allow_failure: true

# Docker 映像建構
build-image:
  stage: package
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  needs: ["build", "lint", "unit-test"]
  variables:
    IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    LATEST_TAG: $CI_REGISTRY_IMAGE:latest
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - echo "Building Docker image..."
    - |
      cat > Dockerfile << 'EOF'
      FROM node:18.16.0-alpine

      WORKDIR /app

      # 複製依賴文件
      COPY package*.json ./
      RUN npm ci --only=production && npm cache clean --force

      # 複製應用程式碼
      COPY dist/ ./dist/
      COPY public/ ./public/

      # 創建非 root 用戶
      RUN addgroup -g 1001 -S nodejs
      RUN adduser -S nextjs -u 1001
      USER nextjs

      EXPOSE 3000

      CMD ["node", "dist/server.js"]
      EOF
    
    - docker build -t $IMAGE_TAG -t $LATEST_TAG .
    - docker push $IMAGE_TAG
    - docker push $LATEST_TAG
    - echo "Docker image built and pushed successfully"
  artifacts:
    reports:
      dotenv: docker.env
  only:
    - main
    - develop
    - tags

# 部署到 Staging 環境
deploy-staging:
  stage: deploy-staging
  image: alpine/helm:3.12.0
  needs: ["build-image"]
  environment:
    name: staging
    url: $STAGING_URL
    deployment_tier: staging
  variables:
    KUBECONFIG: /tmp/kubeconfig
    NAMESPACE: ecommerce-staging
  before_script:
    - echo "$KUBE_CONFIG_STAGING" | base64 -d > $KUBECONFIG
    - helm repo add bitnami https://charts.bitnami.com/bitnami
    - helm repo update
  script:
    - echo "Deploying to staging environment..."
    - |
      helm upgrade --install $APP_NAME-staging bitnami/node \
        --namespace $NAMESPACE \
        --create-namespace \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --set image.tag=$CI_COMMIT_SHA \
        --set service.type=ClusterIP \
        --set ingress.enabled=true \
        --set ingress.hostname=staging-api.company.com \
        --set env.NODE_ENV=staging \
        --set env.DATABASE_URL="$STAGING_DATABASE_URL" \
        --set env.REDIS_URL="$STAGING_REDIS_URL" \
        --wait --timeout=10m
    - echo "Staging deployment completed"
  after_script:
    - kubectl get pods -n $NAMESPACE
    - kubectl get ingress -n $NAMESPACE
  only:
    - develop

# Staging 環境驗證
verify-staging:
  stage: deploy-staging
  image: curlimages/curl:8.1.0
  needs: ["deploy-staging"]
  script:
    - echo "Verifying staging deployment..."
    - sleep 30  # 等待服務啟動
    - curl -f $STAGING_URL/health || exit 1
    - curl -f $STAGING_URL/api/status || exit 1
    - echo "Staging verification completed"
  only:
    - develop

# 端到端測試
e2e-test:
  stage: integration-test
  image: cypress/included:12.17.0
  needs: ["verify-staging"]
  variables:
    CYPRESS_BASE_URL: $STAGING_URL
  script:
    - echo "Running end-to-end tests..."
    - cypress run --record --key $CYPRESS_RECORD_KEY
  artifacts:
    when: always
    paths:
      - cypress/videos/
      - cypress/screenshots/
    expire_in: 1 week
  only:
    - develop
  allow_failure: true

# 效能測試
performance-test:
  stage: integration-test
  image: grafana/k6:0.45.0
  needs: ["verify-staging"]
  script:
    - echo "Running performance tests..."
    - |
      cat > load-test.js << 'EOF'
      import http from 'k6/http';
      import { check, sleep } from 'k6';

      export let options = {
        stages: [
          { duration: '2m', target: 20 },
          { duration: '5m', target: 20 },
          { duration: '2m', target: 0 },
        ],
        thresholds: {
          http_req_duration: ['p(95)<500'],
          http_req_failed: ['rate<0.1'],
        },
      };

      export default function () {
        let response = http.get(`${__ENV.STAGING_URL}/api/products`);
        check(response, {
          'status is 200': (r) => r.status === 200,
          'response time < 500ms': (r) => r.timings.duration < 500,
        });
        sleep(1);
      }
      EOF
    - k6 run --env STAGING_URL=$STAGING_URL load-test.js
  artifacts:
    reports:
      performance: performance-results.json
    when: always
    expire_in: 1 week
  only:
    - develop
  allow_failure: true

# 生產環境部署
deploy-production:
  stage: deploy-production
  image: alpine/helm:3.12.0
  needs: ["build-image", "e2e-test", "performance-test"]
  environment:
    name: production
    url: $PRODUCTION_URL
    deployment_tier: production
  variables:
    KUBECONFIG: /tmp/kubeconfig-prod
    NAMESPACE: ecommerce-production
  before_script:
    - echo "$KUBE_CONFIG_PRODUCTION" | base64 -d > $KUBECONFIG
    - helm repo add bitnami https://charts.bitnami.com/bitnami
    - helm repo update
  script:
    - echo "Deploying to production environment..."
    - |
      # 藍綠部署策略
      CURRENT_RELEASE=$(helm list -n $NAMESPACE -o json | jq -r '.[] | select(.name=="'$APP_NAME'") | .app_version' || echo "none")
      NEW_RELEASE="$APP_NAME-blue"
      
      if [[ "$CURRENT_RELEASE" == *"blue"* ]]; then
        NEW_RELEASE="$APP_NAME-green"
      fi
      
      echo "Deploying new release: $NEW_RELEASE"
      
      helm upgrade --install $NEW_RELEASE bitnami/node \
        --namespace $NAMESPACE \
        --create-namespace \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --set image.tag=$CI_COMMIT_SHA \
        --set service.type=ClusterIP \
        --set ingress.enabled=true \
        --set ingress.hostname=api.company.com \
        --set env.NODE_ENV=production \
        --set env.DATABASE_URL="$PRODUCTION_DATABASE_URL" \
        --set env.REDIS_URL="$PRODUCTION_REDIS_URL" \
        --set replicaCount=3 \
        --set resources.requests.memory="256Mi" \
        --set resources.requests.cpu="200m" \
        --set resources.limits.memory="512Mi" \
        --set resources.limits.cpu="500m" \
        --wait --timeout=15m
      
      # 健康檢查
      sleep 60
      if curl -f $PRODUCTION_URL/health; then
        echo "Health check passed, switching traffic..."
        # 這裡應該有流量切換邏輯
        echo "Production deployment successful"
      else
        echo "Health check failed, rolling back..."
        helm rollback $NEW_RELEASE 0 -n $NAMESPACE
        exit 1
      fi
  when: manual
  only:
    - main
    - tags

# 部署後驗證
verify-production:
  stage: deploy-production
  image: curlimages/curl:8.1.0
  needs: ["deploy-production"]
  script:
    - echo "Verifying production deployment..."
    - curl -f $PRODUCTION_URL/health || exit 1
    - curl -f $PRODUCTION_URL/api/status || exit 1
    - echo "Production verification completed"
  after_script:
    # 發送通知
    - |
      curl -X POST -H 'Content-type: application/json' \
        --data '{"text":"🚀 Production deployment successful for '$APP_NAME' version '$CI_COMMIT_SHA'"}' \
        $SLACK_WEBHOOK_URL
  when: manual
  only:
    - main
    - tags
```

### 3.3 GitLab Runner 自定義配置

```toml
# config.toml - GitLab Runner 配置
concurrent = 10
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "docker-runner-1"
  url = "https://gitlab.company.com/"
  token = "your-registration-token"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
      ServerAddress = "minio.company.com"
      AccessKey = "minio-access-key"
      SecretKey = "minio-secret-key"
      BucketName = "gitlab-runner-cache"
      Insecure = false
  [runners.docker]
    tls_verify = false
    image = "node:18-alpine"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0

[[runners]]
  name = "k8s-runner"
  url = "https://gitlab.company.com/"
  token = "your-k8s-registration-token"
  executor = "kubernetes"
  [runners.kubernetes]
    host = "https://k8s-api.company.com"
    namespace = "gitlab-runner"
    image = "node:18-alpine"
    cpu_limit = "1"
    memory_limit = "2Gi"
    service_cpu_limit = "500m"
    service_memory_limit = "1Gi"
    helper_cpu_limit = "100m"
    helper_memory_limit = "128Mi"
    poll_timeout = 180
    pod_termination_grace_period_seconds = 10
```

## 4. 進階 CI/CD 模式

### 4.1 多環境部署策略

```mermaid
graph TB
    A[Source Code] --> B[Build]
    B --> C[Development]
    C --> D[Testing]
    D --> E[Staging]
    E --> F{Manual Approval}
    F -->|Approved| G[Production]
    F -->|Rejected| H[Rollback]
    
    subgraph "Deployment Strategies"
        I[Blue-Green]
        J[Canary]
        K[Rolling Update]
        L[Feature Flags]
    end
    
    G --> I
    G --> J
    G --> K
    G --> L
```

#### 藍綠部署實現

```bash
# scripts/blue-green-deploy.sh
#!/bin/bash

set -e

NAMESPACE=${1:-default}
APP_NAME=${2:-myapp}
NEW_IMAGE=${3:-myapp:latest}
HEALTH_CHECK_URL=${4:-http://localhost:8080/health}

echo "🚀 Starting Blue-Green deployment for $APP_NAME"

# 檢查當前活動的部署
CURRENT_ACTIVE=$(kubectl get service $APP_NAME -n $NAMESPACE -o jsonpath='{.spec.selector.version}' 2>/dev/null || echo "blue")

if [ "$CURRENT_ACTIVE" = "blue" ]; then
    INACTIVE="green"
    ACTIVE="blue"  
else
    INACTIVE="blue"
    ACTIVE="green"
fi

echo "Current active: $ACTIVE, deploying to: $INACTIVE"

# 部署到非活動環境
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $APP_NAME-$INACTIVE
  namespace: $NAMESPACE
spec:
  replicas: 3
  selector:
    matchLabels:
      app: $APP_NAME
      version: $INACTIVE
  template:
    metadata:
      labels:
        app: $APP_NAME
        version: $INACTIVE
    spec:
      containers:
      - name: $APP_NAME
        image: $NEW_IMAGE
        ports:
        - containerPort: 8080
        env:
        - name: VERSION
          value: $INACTIVE
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: $APP_NAME-$INACTIVE
  namespace: $NAMESPACE
spec:
  selector:
    app: $APP_NAME
    version: $INACTIVE
  ports:
  - port: 80
    targetPort: 8080
EOF

# 等待部署完成
echo "⏳ Waiting for $INACTIVE deployment to be ready..."
kubectl rollout status deployment/$APP_NAME-$INACTIVE -n $NAMESPACE --timeout=300s

# 健康檢查
echo "🏥 Performing health check..."
sleep 30

# 獲取測試 URL
TEST_URL=$(kubectl get service $APP_NAME-$INACTIVE -n $NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || echo "localhost")

# 執行健康檢查
for i in {1..10}; do
    if curl -f "http://$TEST_URL:80/health" >/dev/null 2>&1; then
        echo "✅ Health check passed"
        break
    else
        echo "❌ Health check failed, attempt $i/10"
        if [ $i -eq 10 ]; then
            echo "🚨 Health check failed after 10 attempts, aborting deployment"
            kubectl delete deployment $APP_NAME-$INACTIVE -n $NAMESPACE
            exit 1
        fi
        sleep 10
    fi
done

# 切換流量
echo "🔄 Switching traffic to $INACTIVE environment..."
kubectl patch service $APP_NAME -n $NAMESPACE -p '{"spec":{"selector":{"version":"'$INACTIVE'"}}}'

# 驗證切換
sleep 10
echo "🔍 Verifying traffic switch..."
if curl -f $HEALTH_CHECK_URL >/dev/null 2>&1; then
    echo "✅ Traffic switch successful"
    
    # 清理舊部署
    echo "🧹 Cleaning up old deployment ($ACTIVE)..."
    kubectl delete deployment $APP_NAME-$ACTIVE -n $NAMESPACE --ignore-not-found=true
    kubectl delete service $APP_NAME-$ACTIVE -n $NAMESPACE --ignore-not-found=true
    
    echo "🎉 Blue-Green deployment completed successfully!"
else
    echo "🚨 Traffic switch verification failed, rolling back..."
    kubectl patch service $APP_NAME -n $NAMESPACE -p '{"spec":{"selector":{"version":"'$ACTIVE'"}}}'
    kubectl delete deployment $APP_NAME-$INACTIVE -n $NAMESPACE
    exit 1
fi
```

### 4.2 Canary 部署策略

```yaml
# canary-deployment.yml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: ecommerce-api-canary
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 2m}
      - setWeight: 25
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 75
      - pause: {duration: 10m}
      analysis:
        templates:
        - templateName: success-rate
        startingStep: 2
        args:
        - name: service-name
          value: ecommerce-api
      trafficRouting:
        nginx:
          stableService: ecommerce-api-stable
          canaryService: ecommerce-api-canary
          annotationPrefix: nginx.ingress.kubernetes.io
  selector:
    matchLabels:
      app: ecommerce-api
  template:
    metadata:
      labels:
        app: ecommerce-api
    spec:
      containers:
      - name: ecommerce-api
        image: myregistry/ecommerce-api:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 2m
    count: 5
    successCondition: result[0] >= 0.95
    failureLimit: 2
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          sum(rate(http_requests_total{job="{{args.service-name}}",code!~"5.."}[2m])) /
          sum(rate(http_requests_total{job="{{args.service-name}}"}[2m]))
```

## 5. 監控與可觀測性整合

### 5.1 Pipeline 監控

```yaml
# .github/workflows/monitoring.yml
name: Pipeline Monitoring

on:
  workflow_run:
    workflows: ["CI/CD Pipeline"]
    types: [completed]

jobs:
  collect-metrics:
    runs-on: ubuntu-latest
    steps:
    - name: Collect pipeline metrics
      run: |
        # 發送指標到監控系統
        curl -X POST "https://monitoring.company.com/api/metrics" \
          -H "Authorization: Bearer ${{ secrets.MONITORING_TOKEN }}" \
          -d "{
            \"pipeline_name\": \"${{ github.workflow }}\",
            \"status\": \"${{ github.event.workflow_run.conclusion }}\",
            \"duration\": ${{ github.event.workflow_run.updated_at - github.event.workflow_run.created_at }},
            \"commit_sha\": \"${{ github.sha }}\",
            \"branch\": \"${{ github.ref }}\",
            \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"
          }"

  notify-on-failure:
    runs-on: ubuntu-latest
    if: github.event.workflow_run.conclusion == 'failure'
    steps:
    - name: Send failure notification
      uses: 8398a7/action-slack@v3
      with:
        status: failure
        text: |
          🚨 Pipeline Failed!
          Repository: ${{ github.repository }}
          Branch: ${{ github.ref }}
          Commit: ${{ github.sha }}
          Workflow: ${{ github.event.workflow_run.name }}
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### 5.2 應用程式監控整合

```javascript
// monitoring/pipeline-metrics.js
const prometheus = require('prom-client');

// 建立指標
const pipelineExecutions = new prometheus.Counter({
  name: 'cicd_pipeline_executions_total',
  help: 'Total number of CI/CD pipeline executions',
  labelNames: ['status', 'branch', 'stage']
});

const pipelineDuration = new prometheus.Histogram({
  name: 'cicd_pipeline_duration_seconds',
  help: 'Duration of CI/CD pipeline executions',
  labelNames: ['branch', 'stage'],
  buckets: [30, 60, 120, 300, 600, 1200, 1800, 3600]
});

const deploymentFrequency = new prometheus.Counter({
  name: 'deployment_frequency_total',
  help: 'Total number of deployments',
  labelNames: ['environment', 'service']
});

const leadTimeForChanges = new prometheus.Histogram({
  name: 'lead_time_for_changes_hours',
  help: 'Lead time for changes from commit to deployment',
  labelNames: ['environment', 'service'],
  buckets: [1, 4, 8, 24, 48, 72, 168]
});

// DORA 指標追蹤
class DORAMetrics {
  constructor() {
    this.registry = new prometheus.Registry();
    this.registry.registerMetric(pipelineExecutions);
    this.registry.registerMetric(pipelineDuration);
    this.registry.registerMetric(deploymentFrequency);
    this.registry.registerMetric(leadTimeForChanges);
  }

  recordPipelineExecution(status, branch, stage, duration) {
    pipelineExecutions.inc({ status, branch, stage });
    pipelineDuration.observe({ branch, stage }, duration);
  }

  recordDeployment(environment, service, leadTime) {
    deploymentFrequency.inc({ environment, service });
    leadTimeForChanges.observe({ environment, service }, leadTime);
  }

  getMetrics() {
    return this.registry.metrics();
  }
}

module.exports = DORAMetrics;
```

## 6. 安全與合規

### 6.1 安全掃描整合

```yaml
# .github/workflows/security.yml
name: Security Scanning

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 2 * * *'  # 每日 2AM

jobs:
  secret-scanning:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: Run TruffleHog
      uses: trufflesecurity/trufflehog@main
      with:
        path: ./
        base: main
        head: HEAD

  dependency-scanning:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Run Snyk to check for vulnerabilities
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high --json > snyk-results.json

    - name: Upload Snyk results to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: snyk-results.json

  sast-scanning:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Initialize CodeQL
      uses: github/codeql-action/init@v2
      with:
        languages: javascript

    - name: Autobuild
      uses: github/codeql-action/autobuild@v2

    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v2

  container-scanning:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Build Docker image
      run: docker build -t test-image .

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'test-image'
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'

  compliance-check:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Run compliance checks
      run: |
        # 檢查是否有必需的安全標頭
        if ! grep -r "helmet" package.json; then
          echo "❌ Missing security headers middleware"
          exit 1
        fi
        
        # 檢查是否有 HTTPS 重定向
        if ! grep -r "https" src/; then
          echo "⚠️  Warning: No HTTPS configuration found"
        fi
        
        # 檢查環境變數配置
        if [ ! -f ".env.example" ]; then
          echo "❌ Missing .env.example file"
          exit 1
        fi
        
        echo "✅ Compliance checks passed"
```

## 7. 成本優化與資源管理

### 7.1 CI/CD 成本監控

```yaml
# .github/workflows/cost-optimization.yml
name: Cost Optimization

on:
  schedule:
    - cron: '0 6 * * 1'  # 每週一 6AM
  workflow_dispatch:

jobs:
  analyze-runner-usage:
    runs-on: ubuntu-latest
    steps:
    - name: Analyze GitHub Actions usage
      run: |
        # 使用 GitHub API 分析運行時間
        curl -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
             -H "Accept: application/vnd.github.v3+json" \
             "https://api.github.com/repos/${{ github.repository }}/actions/runs?per_page=100" \
             | jq '.workflow_runs[] | {id: .id, name: .name, status: .status, created_at: .created_at, updated_at: .updated_at}' \
             > workflow_analysis.json

    - name: Generate cost report
      run: |
        echo "# CI/CD Cost Analysis Report" > cost_report.md
        echo "Generated on: $(date)" >> cost_report.md
        echo "" >> cost_report.md
        
        # 分析最耗時的 workflow
        echo "## Top Time-Consuming Workflows" >> cost_report.md
        jq -r '.[] | select(.status == "completed") | "\(.name): \((.updated_at | strptime("%Y-%m-%dT%H:%M:%SZ") | mktime) - (.created_at | strptime("%Y-%m-%dT%H:%M:%SZ") | mktime)) seconds"' workflow_analysis.json \
        | sort -nr -k2 | head -10 >> cost_report.md

    - name: Upload cost report
      uses: actions/upload-artifact@v3
      with:
        name: cost-analysis-report
        path: cost_report.md

  optimize-docker-images:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Analyze Docker image sizes
      run: |
        echo "Analyzing Docker image layers..."
        docker build -t analysis-image .
        
        # 使用 dive 分析映像層
        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
          wagoodman/dive:latest analysis-image --ci

    - name: Suggest optimizations
      run: |
        echo "🔍 Docker Optimization Suggestions:" > docker_optimization.md
        echo "1. Use multi-stage builds to reduce final image size" >> docker_optimization.md
        echo "2. Use .dockerignore to exclude unnecessary files" >> docker_optimization.md
        echo "3. Combine RUN commands to reduce layers" >> docker_optimization.md
        echo "4. Use alpine-based images when possible" >> docker_optimization.md
```

## 8. 最佳實踐總結

### Platform 選擇指南

| 考量因素 | GitHub | GitLab | 建議 |
|---------|---------|---------|------|
| 團隊規模 | 小到中型 | 中到大型 | 根據團隊規模和複雜度選擇 |
| 預算 | 較高 | 較低 | GitLab 自託管可節省成本 |
| DevOps 成熟度 | 中等 | 高 | GitLab 提供更完整的 DevOps 工具鏈 |
| 社群生態 | 豐富 | 適中 | GitHub 開源生態更活躍 |
| 企業功能 | 強 | 很強 | 兩者都有企業級功能 |

### CI/CD 最佳實踐

1. **管道設計原則**
   - 快速失敗：在早期階段捕獲錯誤
   - 並行執行：最大化資源利用率
   - 安全第一：整合安全掃描到每個階段

2. **部署策略**
   - 測試環境：使用與生產環境相同的配置
   - 藍綠部署：無停機部署
   - Canary 發布：逐步推廣新版本

3. **監控與可觀測性**
   - DORA 指標：追蹤部署頻率、前置時間、故障恢復時間
   - 管道監控：監控 CI/CD 管道本身的健康狀況
   - 成本監控：優化資源使用和成本

### 下一步學習

完成本章後，你應該掌握：
- GitHub Actions 和 GitLab CI/CD 的深度應用
- 企業級 CI/CD 管道設計
- 安全掃描和合規檢查整合
- 多環境部署策略實施
- 成本優化和監控實踐

**下一章將學習**：Git 最佳實踐與代碼審查，包括團隊協作規範和質量保證流程。