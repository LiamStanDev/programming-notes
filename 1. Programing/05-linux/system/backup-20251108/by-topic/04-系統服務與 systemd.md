# 系統服務與 systemd

## 目錄
- [主題簡介](#主題簡介)
- [原理與架構](#原理與架構)
- [常用指令與完整範例](#常用指令與完整範例)
- [常見錯誤與排查](#常見錯誤與排查)
- [最佳實踐與安全性建議](#最佳實踐與安全性建議)
- [實戰案例](#實戰案例)

---

## 主題簡介

systemd 是現代 Linux 發行版的初始化系統與服務管理器，負責：
- 系統啟動流程（PID 1）
- 服務/資源/日誌管理
- 依賴與自動化控制

**核心特色：**
- 並行啟動、依賴管理、單元（Unit）描述、豐富日誌、資源限制
- 廣泛應用於伺服器、桌面、嵌入式

---

## 原理與架構

### systemd 運作流程

| 階段         | 說明                                   |
| ------------ | -------------------------------------- |
| 開機         | systemd 以 PID 1 啟動，載入 target     |
| 依賴解析     | 依據單元檔案解析依賴與啟動順序         |
| 並行啟動     | 支援多服務同時啟動，提升速度           |
| 監控         | 服務異常自動重啟、狀態監控             |
| 日誌收集     | 整合 journal，集中管理所有服務日誌      |

### Unit（單元）類型

| 類型      | 用途說明                       | 範例                |
| --------- | ------------------------------ | ------------------- |
| service   | 服務單元                       | nginx.service       |
| socket    | 通訊端單元，支援 socket 啟動    | sshd.socket         |
| target    | 啟動階段群組                   | multi-user.target   |
| mount     | 掛載點                         | home.mount          |
| timer     | 定時器                         | logrotate.timer     |
| path      | 路徑監控                       | rsync.path          |
| device    | 設備監控                       | dev-sda.device      |
| swap      | 交換空間                       | swap.swap           |
| slice     | 資源分組                       | user.slice          |
| scope     | 臨時資源群組                   | session-2.scope     |

### systemd vs SysVinit

| 項目         | systemd                         | SysVinit                |
| ------------ | ------------------------------ | ----------------------- |
| 啟動方式     | 並行、依賴解析                  | 單線性、序列腳本        |
| 管理單元     | Unit 檔案（.service 等）        | /etc/init.d/ 腳本       |
| 日誌         | journalctl 集中管理             | /var/log/messages       |
| 監控/重啟    | 內建自動重啟、資源限制          | 需自行腳本處理          |

---

## 常用指令與完整範例

### 服務管理（systemctl）

```bash
# 啟動/停止/重啟/重新載入
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx

# 啟用/停用開機自動啟動
sudo systemctl enable nginx
sudo systemctl disable nginx

# 查看服務狀態
systemctl status nginx
```
**範例輸出：**
```text
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled)
   Active: active (running) since Wed 2025-09-17 10:00:00 CST; 1h 23min ago
 Main PID: 1234 (nginx)
    Tasks: 2 (limit: 4915)
   Memory: 5.2M
   CGroup: /system.slice/nginx.service
           ├─1234 nginx: master process /usr/sbin/nginx
           └─1235 nginx: worker process
```

### 日誌查詢（journalctl）

```bash
# 查詢所有 systemd 日誌
journalctl

# 查詢特定服務日誌
journalctl -u nginx

# 實時監控日誌
journalctl -f -u nginx

# 查詢本次開機所有日誌
journalctl -b
```

### 服務單元驗證與除錯

```bash
# 驗證單元檔案語法
systemd-analyze verify myapp.service

# 查看依賴樹
systemctl list-dependencies nginx

# 查看服務啟動時間
systemd-analyze blame
```

### 舊指令相容

```bash
sudo service nginx start
sudo service nginx status
sudo chkconfig --list
sudo chkconfig nginx on
```

---

## 常見錯誤與排查

| 錯誤現象         | 排查步驟與命令                                                                 | 常見原因                       |
| ---------------- | ---------------------------------------------------------------------------- | ------------------------------ |
| 服務啟動失敗     | `systemctl status myapp`<br>`journalctl -xeu myapp`                           | 單元檔案錯誤、依賴未滿足、埠佔用 |
| 單元檔案語法錯   | `systemd-analyze verify myapp.service`                                        | 格式錯誤、區塊缺失              |
| 無法自動啟動     | `systemctl is-enabled myapp`<br>檢查 `[Install]` 與 `WantedBy` 設定           | 未 enable、target 設定錯誤      |
| 日誌查詢異常     | `journalctl -u myapp`<br>`journalctl -b`                                      | 日誌輪替、權限不足              |
| 服務異常重啟迴圈 | `systemctl status` 觀察 `Restart=` 設定                                       | 啟動指令錯誤、依賴未就緒        |

---

## 最佳實踐與安全性建議

- **服務管理**：統一用 `systemctl`，避免混用舊指令。
- **單元檔案管理**：自訂單元建議放 `/etc/systemd/system/`，勿直接改 `/lib/systemd/system/`。
- **依賴設定**：善用 `After=`, `Requires=`, `Wants=`，避免啟動順序錯亂。
- **資源限制**：於 `[Service]` 設定 `MemoryLimit=`, `CPUQuota=`，防止資源濫用。
- **日誌管理**：定期檢查 journal，設定輪替與容量限制，避免磁碟爆滿。
- **安全性**：
  - 服務啟動用專屬帳號（`User=`、`Group=`）
  - 限制權限（`ProtectSystem=`, `PrivateTmp=`, `NoNewPrivileges=`）
  - 僅開放必要服務於開機自動啟動
- **覆寫設定**：用 `systemctl edit` 建立 override，避免直接修改原單元檔。

---

## 實戰案例

### 1. 建立自訂服務並自動重啟

建立 `/etc/systemd/system/myapp.service`：

```ini
[Unit]
Description=My Custom App
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myapp
Restart=on-failure
RestartSec=3
User=appuser
MemoryLimit=256M
CPUQuota=30%
ProtectSystem=full
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

啟用並啟動：
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
```

### 2. 服務異常自動修復

- `Restart=on-failure` 可自動重啟異常服務
- `RestartSec=秒數` 設定重啟間隔
- `StartLimitIntervalSec=60`、`StartLimitBurst=3` 防止無限重啟

### 3. 日誌輪替與容量限制

設定 `/etc/systemd/journald.conf`：
```ini
SystemMaxUse=500M
SystemKeepFree=100M
MaxRetentionSec=7day
```
重啟 journal：
```bash
sudo systemctl restart systemd-journald
```

### 4. socket activation 實例

建立 `myapp.socket` 與 `myapp.service`，服務僅在有連線需求時啟動，節省資源。

---

## 參考連結

- [systemd 官方文件](https://www.freedesktop.org/wiki/Software/systemd/)
- [Arch Wiki: systemd](https://wiki.archlinux.org/title/Systemd)
- [Red Hat systemd Best Practices](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_basic_system_settings/using-systemd-to-manage-services_configuring-basic-system-settings)
