# Log 與監控

## 目錄
- [主題簡介](#主題簡介)
- [原理與架構](#原理與架構)
- [常用指令與範例](#常用指令與範例)
- [常見錯誤與排查](#常見錯誤與排查)
- [最佳實踐與安全性](#最佳實踐與安全性)
- [實戰案例](#實戰案例)

---

## 主題簡介

Log（日誌）用於記錄系統、應用、網路與安全事件，是問題追蹤、審計與合規的基礎。監控則即時收集、分析系統與應用指標，主動發現異常、觸發告警，協助維運、容量規劃與效能優化。兩者結合可大幅提升運維效率、降低故障風險，為現代企業、雲端、DevOps/SRE 團隊不可或缺的基礎設施。

---

## 原理與架構

### Log 類型與流程

- **系統 Log**：如 `/var/log/syslog`、`/var/log/messages`
- **應用程式 Log**：Web/DB/自訂 log
- **安全 Log**：如 `/var/log/auth.log`
- **審計 Log**：存取、操作行為記錄

#### Log 流程（以 Linux 為例）

1. 應用/系統產生 log（`logger`、應用內建）
2. 傳送至 `/dev/log`（socket）
3. 由 syslogd/rsyslogd 收集、分類、寫入 `/var/log/*`
4. 可設定 log forwarding 至遠端（集中管理）

#### Syslog 與 Journald

- **Syslog**：標準 log 協定，支援本地/遠端收集，格式統一
- **Journald**：systemd 內建，二進位格式，支援 metadata、查詢、持久化

#### 監控架構

- **資料收集**：agent/exporter（如 node_exporter、filebeat）
- **資料儲存**：時序資料庫（Prometheus）、全文檢索（ELK）
- **視覺化**：Grafana、Kibana
- **告警**：Alertmanager、email、webhook

#### 監控指標

- **主機層級**：CPU、記憶體、磁碟、網路
- **應用層級**：QPS、延遲、錯誤率
- **自訂指標**：業務/服務相關數據

#### 告警設計

- 條件式規則（如 CPU > 90% 持續 5 分鐘）
- 多層級通知（警告、嚴重）
- 告警抑制、收斂、降噪

---

## 常用指令與範例

### 1. Log 查詢與管理

#### journalctl

```shell
# 查看所有 log
journalctl
# 查看特定服務 log
journalctl -u nginx
# 查看最近 100 行
journalctl -n 100
# 持續監控 log（類似 tail -f）
journalctl -f
```
**範例輸出：**
```text
Sep 17 10:00:01 server systemd[1]: Started nginx.service.
Sep 17 10:00:02 server nginx[1234]: nginx started.
```

| 欄位  | 說明        |
| --- | --------- |
| 時間  | log 發生時間  |
| 主機  | 來源主機名稱    |
| 服務  | 產生日誌的服務名稱 |
| 訊息  | 事件內容      |

#### logger

```shell
logger "這是一則測試 log"
```
> 新增一則自訂訊息至 syslog，常用於腳本記錄。

#### tail / grep / less

```shell
tail -n 50 /var/log/syslog
tail -f /var/log/messages
grep "error" /var/log/nginx/error.log
less /var/log/auth.log
```
> 快速查詢、過濾、瀏覽大量 log。

#### logrotate

- 配置檔：`/etc/logrotate.conf`
```shell
logrotate -d /etc/logrotate.conf
```
> 模擬 log 輪替流程，避免磁碟爆滿。

---

### 2. 監控指令

#### top / htop

```shell
top
htop
```
**範例輸出（top）：**
```text
%Cpu(s):  5.0 us,  1.0 sy,  0.0 ni, 93.0 id,  1.0 wa,  0.0 hi,  0.0 si,  0.0 st
```
| 欄位   | 說明           |
| ------ | -------------- |
| us     | 使用者空間 CPU  |
| sy     | 核心空間 CPU    |
| id     | 閒置比例        |
| wa     | 等待 I/O 時間   |

#### vmstat

```shell
vmstat 1 5
```
**範例輸出：**
```text
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 123456  7890 123456    0    0     1     2    3    4  5  1 93  1  0
```
| 欄位    | 說明                   |
| ------- | ---------------------- |
| r       | 可執行程序數           |
| swpd    | swap 使用量            |
| free    | 空閒記憶體             |
| si/so   | swap in/out            |
| us/sy/id| CPU 使用/核心/閒置     |

#### iostat

```shell
iostat -x 1 3
```
| 欄位      | 說明           |
| --------- | -------------- |
| tps       | 每秒 I/O 次數   |
| %util     | 裝置利用率      |
| await     | 平均等待時間    |

#### sar

```shell
sar -u 1 3
```
> 收集 CPU 使用率歷史數據，便於趨勢分析。

---

## 常見錯誤與排查

### 1. log 滿溢

- 現象：系統異常、應用無法寫入 log
- 排查：
  ```shell
  df -h
  du -sh /var/log/*
  ```
- 處理：調整 logrotate 設定、壓縮/刪除舊 log

### 2. 權限錯誤

- 現象：應用無法寫入 log
- 排查：
  ```shell
  ls -l /var/log/
  ```
- 處理：調整檔案/目錄權限與擁有者（建議僅管理員可寫）

### 3. 監控數據異常

- 現象：監控圖表斷線、數據不更新
- 排查：
  ```shell
  systemctl status node_exporter
  curl http://localhost:9100/metrics
  ```
- 處理：重啟 agent/exporter，檢查網路、防火牆

### 4. log 格式錯誤

- 現象：log 收集/分析異常
- 排查：確認應用 log 格式、收集規則一致

---

## 最佳實踐與安全性

- **Log 輪替**：定期壓縮、刪除舊 log，避免磁碟爆滿
- **權限管理**：log 檔案應設為僅管理員可讀寫，防止洩漏
- **監控指標選擇**：聚焦關鍵資源與業務指標，避免過度監控
- **告警設計**：避免誤報、重複通知，設計抑制與收斂機制
- **集中管理**：建議 log 與監控資料集中儲存，便於查詢與分析
- **結構化 log**：建議應用輸出 JSON/結構化 log，利於自動分析
- **安全監控**：結合 SIEM、IDS/IPS 進行威脅偵測
- **審計與合規**：保留關鍵 log，定期備份，符合法規要求

---

## 實戰案例

### 案例 1：log 滿溢導致服務異常

1. 發現服務異常，檢查磁碟空間
   ```shell
   df -h
   du -sh /var/log/*
   ```
2. 清理/壓縮舊 log，調整 logrotate 設定
   ```shell
   sudo rm /var/log/nginx/access.log.1
   sudo logrotate -f /etc/logrotate.conf
   ```

### 案例 2：log forwarding 集中管理

1. 設定 rsyslog 將 log 傳送至遠端
   `/etc/rsyslog.d/50-default.conf`
   ```
   *.* @@logserver.example.com:514
   ```
2. 重新啟動 rsyslog
   ```shell
   sudo systemctl restart rsyslog
   ```

### 案例 3：監控數據異常排查

1. 監控圖表斷線，檢查 exporter 狀態
   ```shell
   systemctl status node_exporter
   curl http://localhost:9100/metrics
   ```
2. 檢查防火牆、網路連線
   ```shell
   sudo iptables -L
   ping logserver.example.com
   ```

### 案例 4：自訂 log 格式提升分析效率

1. 應用輸出 JSON 格式 log
   ```json
   {"time":"2025-09-17T10:00:00Z","level":"info","msg":"user login","user":"alice"}
   ```
2. 利用 ELK/Kibana 搜尋、分析 log

---

## 參考

- [my/02.系統監測.md](my/02.%E7%B3%BB%E7%B5%B1%E7%9B%A3%E6%B8%AC.md:1)
- [my/04.主機性能測試.md](my/04.%E4%B8%BB%E6%A9%9F%E6%80%A7%E8%83%BD%E6%B8%AC%E8%A9%A6.md:1)
- [ELK Stack 官方文件](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-elastic-stack.html)
- [Prometheus 官方文件](https://prometheus.io/docs/introduction/overview/)
- [Linux 日誌管理與監控最佳實踐](https://wiki.archlinux.org/title/Log_management)
