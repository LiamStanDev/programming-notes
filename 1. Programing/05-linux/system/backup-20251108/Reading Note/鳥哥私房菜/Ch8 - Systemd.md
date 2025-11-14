# 簡介
---
### daemon 與 service
service 表示常駐於記憶體中的程序，並且可以提供其他應用所需的功能(服務)，而 daemon 是用來運行(管理) service 的進程。

### systemd
##### 優勢
* 平行處理所有服務，開機加速
* 只需要 systemctl 指令就能操作
* 能處理服務相依性
* 依 daemon 功能分類
* 將多個 daemons 組成群組
* 向下兼容 init (以前的)

##### 設定檔位置
* `/usr/lib/systemd/system/`: 服務啟動腳本設定檔案放置位置
* `/etc/systemd/system/`: 實際要啟動的服務腳本，裡面都是 Link，連結到 `/usr/lib/systemd/system/` 中。
> 要修改設定檔要去 `/etc/system/system` 中修改


##### systemd unit 分類

| Extensoin                                      | Description                                                                                              |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| .service<br>(service unit)                     | 一般服務類型，是最常見的類型                                                                                           |
| .socket<br>(socket unit)                       | socket 作為 Inter-process communication 的功能，透過這個 unit 讓 systemd 監控指定的 socket，當有信息傳入會啟動指定的 service，(實現自動啟用) |
| .target<br>(target unit)                       | 為一群 unit 的集合。                                                                                            |
| .mount<br>.automount<br>(mount/automount unit) | 檔案系統的掛載服務                                                                                                |
| .path<br>(path unit)                           | 偵測到特定檔案或者目錄運行某些服務                                                                                        |
| .timer<br>(timer unit)                         | 循環執行的任務，相較於 crontab 更有彈性。                                                                                |


# Unit 的設定
---
```toml
[Unit]           # 此 unit 的解釋、執行服務相依性有關
Description=OpenSSH server daemon
After=network.target sshd-keygen.service
Wants=sshd-keygen.service

[Service]        # 這個項目與實際執行的指令參數有關
EnvironmentFile=/etc/sysconfig/sshd
ExecStart=/usr/sbin/sshd -D $OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]        # 這個項目說明此 unit 要掛載哪個 target 底下
WantedBy=multi-user.target
```

#### Unit 參數

| 參數            | 含意                    |
| ------------- | --------------------- |
| Description   | 描述                    |
| Documentation | 說明文檔的路徑               |
| After         | 說明服務啟動前需要啟動的服務，並沒有強制性 |
| Before        | 說明服務啟動前後啟要動的服務，沒有強制性  |
| Requires      | 明確指定服務啟動前要啟動的服務，有強制性  |
| Wants         | 明確指定服務啟動後要啟動的服務，沒有強制性 |
| Conflicts     | 服務衝突檢查，有強制性。          |

#### Service 參數 (其他Socket, Timer 就不講了)

| 參數              | 含意                                                                                                                                                                                                                                                                                   |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Type <br>       | 說明 daemon 啟動方式，有以下幾種類型:<br>1. simple: 由 ExecStart 啟動，並常駐在內存中。<br>2. forking: 由 ExecStart 啟動透過 spawns 延伸出子進程，原父進程就會終止，這樣做有好處因為一個服務運行太久可能會累積過多的未釋放資源，重啟性能可能會比較好。<br>3. oneshot: 與 simple 類似，但是程序會結束。<br>4. dbus: 與 simple 類似，但需要取得 D-Bus 名稱。<br>5. idle: 與 simple 類似，但是最後才執行，也就是空閒才啟動。 |
| EnvironmentFile | 可以設定啟動前需要執行的腳本，通常是配製一些變數                                                                                                                                                                                                                                                             |
| ExecStart       | 就是實際執行此 daemon 的指令或腳本程式。僅接受 `指令 參數 參數 ...` 不支持所有 bash 語法。                                                                                                                                                                                                                            |
| ExecStop        | 與 systemctl stop 的執行有關，關閉此服務時所進行的指令。                                                                                                                                                                                                                                                 |
| ExecReload      | 與 systemctl reload 有關的指令行為。                                                                                                                                                                                                                                                          |
| Restart         | 當設定 Restart=1 時，則當此 daemon 服務終止後，會再次的啟動此服務。                                                                                                                                                                                                                                          |
| TimeOutSec      | 等待服務正常啟動的時間。                                                                                                                                                                                                                                                                         |
| RestartSec      | 重啟的間隔。                                                                                                                                                                                                                                                                               |

#### Install 參數

| 參數       | 含意                                                          |
| -------- | ----------------------------------------------------------- |
| WantedBy | 後面接的大部分是 `*.target unit`，表示這個 unit 本身是附掛在哪一個 target unit 底下 |
| Also     |  當目前這個 unit 本身被 enable 時，Also 後面接的 unit 也請 enable           |
| Alias    |                                                             |


# 操作
---
```shell
systemctl status 
systemctl start
systemctl stop
systemctl disable 
systemctl enable 
systemctl enable --now
systemctl list-unit-files --state=enabled
systemctl list-dependencies docker.service
```
> 加上 --user 表示使用 user 的設定

### journalctl
用來看系統 log
```shell
journalctl # 查看所有 log （時間由舊到新） 
journalctl -e # 查看所有 log （時間由新到舊）

journalctl -u docker.service # -u 表示 unit
```
