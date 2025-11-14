# systemctl
---
`systemctl` 是一個用於管理 Linux 系統中的**服務（services）、單位（units）和套件（packages）的命令行工具**。它是用於控制 Systemd 系統初始化和服務管理的主要工具之一

### 使用方式
```shell
# 啓動服務
systemctl start <服務名稱>
# 停止服務
systemctl stop <服務名稱>
# 重啓服務
systemctl restart <服務名稱>
# 開啓開機自啓動，可以添加 --user 啓動該 user 的自啓動服務
systemctl enable <服務名稱> 
# 停止開機自啓動服務
systemctl disable <服務名稱>
# 檢查服務狀態
systemctl status <服務名稱>
# 查看服務文件是否啓用
systemctl list-unit-files --state=enabled
# 查看運行的服務
systemctl list-unit --type=service
# 重新加載，以便更新服務
systemctl daemon-reload
# 查看服務日誌
journalctl -u <服務名稱> # -u 表示 unit
journalctl -u <服務名稱> -b # -b 表示只看這次啟動的 log

```

# sysctl
---
`sysctl` 是一個用於在 Unix 和 Unix-like 系統上動態調整系統核心參數和運行時配置的命令。通過 `sysctl`，系統管理員可以查詢、修改和管理許多不同的內核和運行時參數，以優化系統性能、安全性和行為。

### 使用方式
```shell
# 查詢參數，例子：
sysctl kernel.version
# 臨時設置參數：系統從起後失效
sysctl -w vm.swappiness=10
# 重新加載配置
sysctl -p 
```

### 修改 sysctl.conf
修改位置爲 `/etc/sysctl.conf` 文件或者 `/etc/sysctl.d/` 目錄下。常見參數與默認值如下：
1. TCP/IP
```shell
# 用於調整最大連接數。
net.core.somaxconn=65535 
# 用於調整網絡設備接收緩衝區的大小
net.core.netdev_max_backlog=65535 
# 調整 TCP 連接的保持活動時間，可以減少該值以更快地檢測失效的連接。
net.ipv4.tcp_keepalive_time=300 # 300 秒之後會檢查連接是否有效
```

2. 文件
```shell
# 文件描述符限制
fs.file-max=65535
# 調整 inotify 監聽器的最大數量
fs.inotify.max_user_watches=524288
```

3. 內存
```shell
# 控制系統的交換空間使用，0 表示靜可能不使用 swap
vm.swappiness=10
# 調整 hugepages 
vm.nr_hugepages=10 # 提供 10 個 hugepage 給應用申請，很像是 hugepages pool
```
> page: 表示操作系統讀取 ram 最小的單位，普通 page 大小爲 4KB
> hugepage: 比 page 更大，通常爲 2MB 或者 1GB (由物理設備決定)，是否使用 huge page 需要應用程序主動請求 `mmap()`。

4. CPU
```shell
# 配置系统支持的最大進程ID數量
kernel.pid_max=32768
```
