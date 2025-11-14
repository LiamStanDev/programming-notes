# Docker 實戰筆記

---

## 1. 基礎概念

Docker 是一套開源的容器化平台，能將應用程式及其依賴環境打包成「容器」（Container），實現跨平台、一致的部署。其核心原理為利用作業系統層級的虛擬化（Linux namespaces、cgroups），讓多個容器共享同一作業系統核心，達到輕量、快速啟動與高效資源利用。

### 傳統虛擬機 vs Docker 容器（ASCII 圖）

```
+---------------------+      +---------------------+
|   傳統虛擬機架構     |      |    Docker 容器架構   |
+---------------------+      +---------------------+
| 應用程式            |      | 應用程式            |
| 作業系統 (多份)     |      | 容器引擎 (單一核心) |
| Hypervisor          |      | 作業系統核心        |
| 實體硬體            |      | 實體硬體            |
+---------------------+      +---------------------+
```

### Docker 核心組件（ASCII 圖）

```
+-------------------+
|   Registry        |
+--------+----------+
         |
         v
+--------+----------+
|      Image        |
+--------+----------+
         |
         v
+--------+----------+
|    Container      |
+--------+----------+
   |           |
   v           v
Volume      Network
```

---

## 2. 安裝

### Windows

1. 下載 [Docker Desktop](https://www.docker.com/products/docker-desktop/) 並安裝。
2. 啟動 Docker Desktop，確認 Docker Engine 啟動。
3. 開啟終端機執行 `docker version` 驗證安裝。

**注意：** 需啟用 WSL 2，Windows 10 需 1903 版以上。

### macOS

1. 下載 [Docker Desktop](https://www.docker.com/products/docker-desktop/) Mac 版本。
2. 安裝並授權權限。
3. 終端機執行 `docker version` 驗證。

**注意：** Apple Silicon 請下載對應版本。

### Linux（以 Ubuntu 為例）

```sh
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl start docker
sudo docker version
```

**建議：** 安裝後將使用者加入 `docker` 群組（免 sudo）：

```sh
sudo usermod -aG docker $USER
```

需重新登入。

### 常見安裝錯誤排查（ASCII 流程圖）

```
[啟動失敗]
     |
     v
[檢查 Docker Daemon 是否啟動]
     |
     v
[檢查權限/群組]
     |
     v
[檢查網路或鏡像源]
```

---

## 3. 映像檔

### 建立映像檔

```sh
docker build -t myimage:latest .
```

### 下載映像檔

```sh
docker pull nginx:latest
```

### 管理映像檔

```sh
docker images
docker rmi myimage:latest
docker tag myimage:latest myrepo/myimage:v1
```

### 分享映像檔

```sh
docker login
docker push myrepo/myimage:v1
```

### Dockerfile 基本語法

| 指令          | 說明         | 範例                                  |
| ----------- | ---------- | ----------------------------------- |
| FROM        | 指定基礎映像檔    | FROM ubuntu:22.04                   |
| RUN         | 執行命令產生新映像層 | RUN apt-get update                  |
| COPY        | 複製檔案       | COPY . /app                         |
| CMD         | 預設執行命令     | CMD ["nginx", "-g", "daemon off;"]  |
| ENTRYPOINT  | 必定執行命令     | ENTRYPOINT ["python3", "app.py"]    |
| EXPOSE      | 宣告對外埠號     | EXPOSE 80                           |
| VOLUME      | 宣告掛載點      | VOLUME ["/data"]                    |
| USER        | 指定執行者      | USER appuser                        |
| ARG/ENV     | 建置/環境變數    | ARG VERSION=1.0 / ENV NODE_ENV=prod |
| HEALTHCHECK | 健康檢查       | HEALTHCHECK CMD curl --fail ...     |

#### 多階段建置（Multi-stage Build）

```dockerfile
FROM node:21-alpine AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

#### CI/CD 自動化建置流程（ASCII 流程圖）

```
[開發者 push 代碼]
        |
        v
[CI/CD Pipeline 啟動]
        |
        v
[自動建置映像檔]
        |
        v
[推送至 Registry]
        |
        v
[自動部署]
```

---

## 4. 容器

### 啟動、停止、刪除、查詢

```sh
docker run -d --name webserver -p 8080:80 nginx:latest
docker stop webserver
docker rm webserver
docker ps -a # -s 可以看佔用空間
docker inspect webserver
```

### 日誌與資源查詢

```sh
docker logs -f webserver
docker stats
```

### CLI 與 GUI 工具

- CLI：`docker` 指令
- GUI：Docker Desktop 支援圖形化管理

### 實例：帶 Volume 與自訂網路的 Nginx 容器

```sh
docker volume create webdata
docker network create webnet
docker run -d --name mynginx --network webnet -v webdata:/usr/share/nginx/html nginx
```

---

## 5. Volume / Network

### Volume/Network 架構

```
+-------------------+
|   Host            |
|  +-------------+  |
|  |  Volume     |<-------------------+
|  +-------------+  |                 |
+-------------------+                 |
         ^                            |
         |                            |
+-------------------+      +-------------------+
|   Container A     |      |   Container B     |
|  /data <----------+      |  /data <----------+
+-------------------+      +-------------------+
         |                            |
         +------------[Network]-------+
```


### Docker 網路

```shell
❯ docker network ls
NETWORK ID     NAME                     DRIVER    SCOPE
fae22dab59a7   bridge                   bridge    local  # 默認 docker0 網橋（虛擬交換機）
560e12c843b0   host                     host      local  # 宿主網路
ad9bd40f6a9d   none                     null      local  # 無網路
6b6e21421011   ng-bridge                bridge    local  # 自定義網橋
```

* 以上三種網路驅動都是 Docker 默認創建的


|     | 默認 Bridge (`docker0`) | 自定義 bridge                       | Host           | None |
| --- | --------------------- | -------------------------------- | -------------- | ---- |
| 優點  | 創建容器自動加入              | 自動 DNS 隔離，可以使用容器名 or Hostname 連通 | 性能高            | 完全隔離 |
| 缺點  | 不會自動進行 DNS            | 因爲地址不同故出網需要進行 NAT，會消耗性能          | 因容器完全暴露故有安全性問題 | 不能聯網 |
| 場景  | 不建議任何場景使用             | 單宿主多容器                           | 單容器多端口         |      |

> 爲什麼創建容器就會增加一個 veth 開頭的網路界面？
> veth 網路界面是 linux 提供的虛擬網路線，連接指定的 bridge 與 Container。如圖所示：
> ![[Pasted image 20250920205158.png|800]]


### Network 操作

```sh
docker network create ng-bridge # 建立自定義 brdige 網路
docker run -d --network=ng-bridge --name app1 --hostname app1 nginx # 其實不用特別指定 hostname(默認可以直接使用 container name)
docker network ls
docker network rm ng-bridge
```


### Volume 操作

```sh
docker volume create mydata
docker run -d -v mydata:/data nginx
docker volume ls
```




---

## 6. 資源管理

### 容器資源限制

```sh
docker run --cpus="1.5" --memory="512m" nginx
docker run --blkio-weight=500 nginx
```

### 監控工具

- `docker stats`
- cAdvisor
- Prometheus + Grafana

### Docker Compose 進階用法

```yaml
services:
  web:
    image: nginx
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    depends_on:
      db:
        condition: service_healthy
  db:
    image: mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 3
```

### Swarm / Kubernetes 基礎

```sh
# Swarm
docker swarm init
docker service create --name web --replicas 3 nginx

# Kubernetes
minikube start
kubectl create deployment web --image=nginx
kubectl scale deployment web --replicas=3
kubectl get pods
```

### 資源分配架構（ASCII 圖）

```
+-------------------+
|      Host         |
+-------------------+
|   Container 1     |  CPU: 1.0  MEM: 512M
|   Container 2     |  CPU: 0.5  MEM: 256M
|   ...             |
+-------------------+
```

---

## 7. 開發 / CI/CD

### 本地開發流程

- 多服務協作（Docker Compose）
- 熱重載（volume 掛載 + nodemon/webpack）

```yaml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - .:/app
    command: npx nodemon index.js
    depends_on:
      - db
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: example
    ports:
      - "5432:5432"
```

### 測試自動化

```sh
docker run --rm -v $(pwd):/app -w /app python:3.11 bash -c "pip install pytest && pytest"
docker run --rm -v $(pwd):/app -w /app node:20 bash -c "npm install && npm test"
```

### CI/CD 整合

- GitHub Actions、GitLab CI、Jenkins
- 自動建置、測試、部署

#### GitHub Actions 範例

```yaml
name: CI
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build & Test
        run: |
          docker compose up -d
          docker compose exec web npm test
```

### 多環境管理

- 以多個 compose 檔案、環境變數切換 dev/staging/prod

---

## 8. 安全

### 映像檔安全

- 僅從可信 Registry 拉取
- 精簡映像（如 alpine）
- 啟用 Docker Content Trust

### 權限管理

- 避免 root 執行，指定一般用戶
- 移除不必要的 Capability
- 可採用 rootless Docker

### 網路隔離

- 僅開放必要 port
- 結合主機防火牆
- Overlay 網路建議加密

### 漏洞掃描

- Trivy、Clair
- 建議整合至 CI/CD

### 安全性最佳實踐（清單）

- 僅用可信映像
- 定期更新
- 避免 root
- 移除多餘 Capability
- 僅開放必要 port
- 定期漏洞掃描
- 啟用映像簽章
- 實施資源限制
- 監控行為與日誌
- 自動化安全檢查

### 安全架構（ASCII 圖）

```
+-------------------+
|   Registry        |<---[簽章/驗證]---+
+-------------------+                 |
        |                             |
        v                             |
+-------------------+                 |
|   Image           |                 |
+-------------------+                 |
        |                             |
        v                             |
+-------------------+                 |
| Container (非root)|                 |
+-------------------+                 |
        |                             |
        v                             |
+-------------------+                 |
|   Host Firewall   |-----------------+
+-------------------+
```

---

## 9. 維運

### 日誌管理

```sh
docker logs -f <container_id>
docker logs --tail 100 <container_id>
docker logs <container_id> > container.log
```

- 集中式日誌（ELK、Fluentd、Loki）
- 設定 log rotation
- 日誌結構化（JSON）

### 監控指標

- CPU、記憶體、磁碟、網路
- 容器重啟次數、健康狀態

### 常見排錯技巧

1. 檢查容器狀態：`docker ps -a`
2. 查看日誌：`docker logs <container_id>`
3. 進入容器：`docker exec -it <container_id> /bin/sh`
4. 檢查網路/Volume
5. 檢查資源限制

### 維運自動化

- Watchtower 自動更新
- `--restart=always` 自動重啟
- HEALTHCHECK 健康檢查

### 排錯流程（ASCII 流程圖）

```
[容器異常]
     |
     v
[檢查狀態] -> [查日誌] -> [檢查資源/網路] -> [修正]
```

---

## 10. FAQ

### 安裝與權限

- 權限不足：用 `sudo` 或加入 `docker` 群組
- Docker Desktop vs CLI：Desktop 有圖形介面，CLI 適合自動化

### 容器啟動與管理

- port 已被佔用：查詢佔用程式或更換埠口
- 查運行中容器：`docker ps`

### 網路與連線

- 容器無法互通：確認是否同一 network
- 容器無法連外：檢查主機網路或重啟 Docker

### 檔案存取

- 無法存取掛載目錄：檢查主機目錄權限

### 常見錯誤訊息

| 錯誤訊息                            | 排查建議                                   |
| ----------------------------------- | ------------------------------------------ |
| permission denied                   | 檢查權限或用 sudo                          |
| Cannot connect to the Docker daemon | 啟動 Docker 服務                           |
| no space left on device             | 清理未用映像/容器 `docker system prune -a` |
| network not found                   | 先建立 network                             |
| image not found                     | 檢查名稱或執行 `docker pull`               |

### FAQ 排查流程（ASCII 流程圖）

```
[啟動容器失敗]
     |
     v
[檢查錯誤訊息]
     |
     v
[查詢 log]
     |
     v
[檢查網路/埠口/映像/目錄]
```

### 參考資料

- [Docker 官方文件](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Docker 社群論壇](https://forums.docker.com/)
- [Awesome Docker](https://github.com/veggiemonk/awesome-docker)
- [Docker网络模式Linux - Bridge | Host | None](https://www.bilibili.com/video/BV1Aj411r71b/?spm_id_from=333.337.search-card.all.click&vd_source=81381cbff2baf8dfd1b22050d029f496)

---

（本筆記已整合所有內容，所有圖例均以 ASCII 呈現，章節結構依指定順序，內容已去重、彙整、優化。）
