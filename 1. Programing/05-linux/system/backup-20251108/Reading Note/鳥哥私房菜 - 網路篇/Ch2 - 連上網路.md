# Linux 網卡
---
##### 網路裝置代號
Linux 裝置都是以文件名稱來取代，*網路介面卡 (Network Interface Card, NIC)* 是以模組名稱代替 eth0 (第一張網卡)、eth1 (第二張網卡) 以此類推。

##### 網卡模組 
網卡模組就是驅動，因為網卡為硬體所以需要核心才能驅動，基本上 Linux 發行版預設支持的 Intel 與初階 RealTek 都可以支援，若 Linux 核心不支援或者驅動閉源只能透過:
1. 編譯更新的 Linux 核心
2. 編譯網路卡的核心模組
3. 自行由網路卡開發商的官方網站下載
4. 買支援的網卡 (推薦!)

##### 捕捉網卡資訊
```shell
dmesg | grep -in eth0 # -i 表示忽略 case, -n 表示顯示數字
```

> dmesg 表示 diagnostic message，用來輸出系統啟動時內核的日誌、運行驅動設備和模塊的日誌。
> e.g. 設備連接、內核模塊加載。

顯示如下
```shell
377:e1000: eth0: e1000_probe: Intel(R) PRO/1000 Network Connection
383:e1000: eth1: e1000_probe: Intel(R) PRO/1000 Network Connection
418:e1000: eth0 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX
```
* `e1000` 為模組名，使用 Intel 網卡，此網卡達到 1000Mbps 的全雙工網路速度。

詳細資訊可以使用
```shell
lspci eth | grep -in eth
```
顯示如下
```shell
00:03.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet
```

##### 網卡模組
檢查 e1000 模組是否有被順利載入
```shell
lsmod | grep -in 1000
```
顯示如下
```shell
52:e1000                 119381  0
```

##### 補充: 模組操作
* 添加移除模組操作
```shell
rmmode e1000 # 移除 e1000 模組
modprobe e1000 # 添加模組
modinfo e1000 # 模組資訊
```
* 自啟動網路卡模組
```shell
# /etc/modprobe.d/ether.conf
alias eth0 e1000e # 第一張網卡
alias eth1 e1000e # 第二張網卡
```

