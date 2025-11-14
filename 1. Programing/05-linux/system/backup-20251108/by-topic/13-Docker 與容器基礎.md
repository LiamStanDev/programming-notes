# Docker 與容器基礎

---

## 目錄
- [主題簡介](#主題簡介)
- [原理與架構](#原理與架構)
- [常用命令與範例](#常用命令與範例)
- [常見錯誤與排查](#常見錯誤與排查)
- [最佳實踐與安全性](#最佳實踐與安全性)
- [實戰案例](#實戰案例)
- [參考資源](#參考資源)

---

## 主題簡介

Docker 是一套開源容器平台，能將應用程式及其依賴打包於輕量級、可攜式的容器中，實現一致的部署與運行環境。常見應用場景包括微服務架構、CI/CD、測試隔離、雲端原生等。核心優勢為快速啟動、資源隔離、易於移植與自動化。

---

## 原理與架構

### 容器技術核心
- **Namespace**：隔離進程、網路、檔案系統等資源。
- **Cgroups**：限制與分配 CPU、記憶體、I/O 等資源。
- **UnionFS**：分層檔案系統，支援映像快取與重用。

### Docker 架構
- **Docker Engine**：核心服務，包含 Daemon、REST API、CLI。
- **Client/Server 架構**：用戶端透過 CLI 或 API 操作 Daemon。
- **Registry**：映像檔儲存與分發（如 Docker Hub、Harbor）。

### 關鍵物件
| 名稱       | 說明                                   |
| ---------- | -------------------------------------- |
| Image      | 只讀模板，描述應用與環境，分層設計         |
| Container  | 映像檔的執行實例，具獨立檔案系統與資源限制 |
| Volume     | 持久化資料，與主機或多容器共享            |
| Network    | 容器間通訊，支援 bridge、overlay 等      |
| Compose    | YAML 定義多容器應用，簡化部署             |

---

## 常用命令與範例

### 1. 映像檔操作

```shell
# 下載映像檔
docker pull nginx:alpine
# 輸出：
# alpine: Pulling from library/nginx
# Digest: sha256:...
# Status: Downloaded newer image for nginx:alpine

# 建立映像檔
docker build -t myapp:1.0 .
# 輸出：
# Successfully built 1a2b3c4d5e6f
# Successfully tagged myapp:1.0

# 查看本地映像
docker images
# 輸出：
# REPOSITORY   TAG     IMAGE ID       CREATED         SIZE
# myapp        1.0     1a2b3c4d5e6f   2 minutes ago   120MB

# 上傳映像檔
docker push myrepo/myapp:1.0
# 輸出：
# The push refers to repository [docker.io/myrepo/myapp]
# 1.0: digest: sha256:... size: 1573
```

### 2. 容器操作

```shell
# 執行容器（前景/背景）
docker run --rm -it --name test-nginx -p 8080:80 nginx:alpine
# 輸出：
# ...（啟動日誌）

# 背景執行
docker run -d --name web -p 8080:80 nginx
# 輸出：
# 123456789abc

# 查看執行中容器
docker ps
# 輸出：
# CONTAINER ID   IMAGE     COMMAND                  STATUS          PORTS                  NAMES
# 123456789abc   nginx     "/docker-entrypoint.…"   Up 10 seconds   0.0.0.0:8080->80/tcp   web

# 進入容器
docker exec -it web /bin/sh
# 輸出：
# /#

# 停止/移除容器
docker stop web
docker rm web
```

### 3. 資料卷與網路

```shell
# 建立資料卷
docker volume create mydata

# 掛載資料卷
docker run -d --name db -v mydata:/var/lib/mysql mysql:5.7

# 建立自訂網路
docker network create mynet

# 指定網路啟動容器
docker run -d --name app --network mynet myapp:1.0
```

### 4. 日誌與監控

```shell
# 查看容器日誌
docker logs web

# 即時監控資源
docker stats
# 輸出：
# CONTAINER ID   NAME   CPU %   MEM USAGE / LIMIT   NET I/O   BLOCK I/O   PIDS
# 123456789abc   web    0.07%   2.5MiB / 1GiB       ...       ...         2
```

### 5. Compose 操作

```shell
# 啟動多容器服務
docker compose up -d
# 輸出：
# [+] Running 2/2
#  ✔ Network myapp_default  Created
#  ✔ Container myapp_web    Started

# 查看服務日誌
docker compose logs
```

---

## 常見錯誤與排查

| 錯誤現象             | 排查步驟與解法                                                                 |
|----------------------|------------------------------------------------------------------------------|
| 映像檔拉取失敗       | 檢查網路、Registry 權限、名稱拼寫。`docker login` 驗證帳號。                  |
| 埠衝突               | `docker ps`、`ss -tlnp` 查詢埠占用，調整 `-p` 參數或釋放埠口。                |
| 資源不足             | `docker stats` 檢查資源，調整主機/容器限制。                                 |
| 權限問題             | 用戶需加入 `docker` 群組：`sudo usermod -aG docker $USER`，重登生效。         |
| 容器啟動失敗         | `docker logs <容器名>` 查日誌，檢查映像設定與依賴。                           |
| Volume 權限錯誤      | 檢查目錄權限、SELinux/AppArmor 設定。                                         |
| DNS 解析失敗         | 檢查主機 `/etc/resolv.conf`，或用 `--dns` 指定 DNS。                          |
| 無法連外網           | 檢查防火牆（iptables/nftables）、主機網路、Docker bridge 設定。                |

### 常用排查命令

```shell
# 查看所有容器狀態
docker ps -a

# 查看容器詳細資訊
docker inspect <容器名或ID>

# 查看網路設定
docker network ls
docker network inspect bridge

# 查看 Volume 狀態
docker volume ls
docker volume inspect mydata
```

---

## 最佳實踐與安全性

- **映像檔管理**：定期清理未用映像/容器/Volume
  ```shell
  docker system prune -a
  ```
- **資源限制**：建議所有容器加上 `--memory`、`--cpus` 限制
- **最小權限原則**：避免容器內以 root 執行，善用 `USER` 指令
- **定期更新**：映像檔與基底映像需定期更新修補漏洞
- **網路隔離**：自訂網路、僅開放必要埠口
- **資料持久化**：重要資料務必用 Volume，避免資料遺失
- **多階段建構**：減少映像體積、降低攻擊面
- **安全掃描**：建議用 `docker scan` 或 CI 工具檢查映像漏洞
- **日誌與監控**：善用 `docker logs`、`docker stats`、外部監控工具（如 Prometheus）

---

## 實戰案例

### 案例1：快速部署 Nginx 靜態網站

```shell
# 建立網站目錄
mkdir -p ~/webroot
echo "Hello Docker" > ~/webroot/index.html

# 啟動 Nginx 並掛載目錄
docker run -d --name web -p 8080:80 -v ~/webroot:/usr/share/nginx/html:ro nginx:alpine

# 驗證
curl http://localhost:8080
# 輸出：
# Hello Docker
```

### 案例2：資料庫備份自動化

```shell
# 啟動 MySQL 並掛載資料卷
docker volume create dbdata
docker run -d --name db -e MYSQL_ROOT_PASSWORD=pass -v dbdata:/var/lib/mysql mysql:5.7

# 備份資料庫
docker exec db mysqldump -uroot -ppass --all-databases > backup.sql
```

### 案例3：排查容器無法連外

```shell
# 1. 檢查容器網路
docker exec -it web ping 8.8.8.8

# 2. 檢查主機防火牆
sudo iptables -L -n

# 3. 檢查 Docker bridge
docker network inspect bridge
```

---

## 參考資源

- [Docker 官方文件](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Awesome Docker](https://github.com/veggiemonk/awesome-docker)
- [my/03.防火牆.md](03.防火牆.md)（防火牆與 NAT 實務）
