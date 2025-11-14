# IP 命令
---
ip 命令集大成，功能操作 ifconfig + route 兩個指令。
```shell
ip [-option] [action] [instruction]
```
* options: 
	* `-s`: 表示列出統計數據 (statistics)
* actions:
	* `link`: 對裝置(device) 的設定，有 MTU, MAC 位置等。
	* `addr/address`: 對 IP 的設定
	* `route`: 路由設定

#### ip link
```shell
ip [-s] link show # 查閱裝置資訊
```

```shell
ip link set [device] [actions] # device 表示 eth0, eth1 這樣的代號
```
* actions: 
	* up/down: 啟動 (up) 或者關閉 (down) 某個網卡。
	* address: 修改 MAC 地址 (若設備可以的話)
	* name: 給裝置一個名字
	* mtu: 最大傳輸單元
釋例如下，
```shell
ip link set eth0 up
ip link set eth0 down

ip link set eth0 mtu 1000 # 更改 mtu 為 1000 bytes

ip link set eth0 name vbird # 需要先 down 設備才行

ip link set eth0 address 00:0C:29:E1:1E:23
```

#### ip addr
```shell
ip addr show
```

```shell
ip addr [add|del] [IP參數] [device 名稱] [其他參數]
```

操作如下，
* 添加 ip 地址
```shell
ip addr add 192.168.0.100/24 broadcast + scope global dev ens160
```
首先 add 面要加上 ip 與子網掩碼，broadcast 後面要指定廣播 ip，`+` 表示自動設定，也可以改為 `broadcast 192.168.0.255`，scope 後面有 `global`(允許所有連線), `site`(僅支援 ipv6連線), `link`(僅允許本機的裝置們相互的連線), `host`(僅允許本機內部連線)，通常使用 `global`，dev 後面接網卡名。
* 刪除 ip 地址
```shell
ip addr del 192.168.0.100/24 dev ens160
```

#### ip route
```shell
ip route show
```

* 添加默認路由
```shell
ip route add default via 192.168.1.254 dev eth0
```
* 刪除路由
```shell
ip route del 192.168.10.0/24
```

# 網路命令
---
### 網路層數據傳輸: curl
全名 client URL，常用於做為下載器或測試服務。
* `-X` 指定 HTTP 請求方法，e.g. POST, GET, PUT...
* `-v` 詳細輸出通信過程
* `-s` 不顯示進度
* `-S` 錯誤時才輸出，常與 `-s` 一起使用
* `-O` 將回應保存成文件，等同 `wget`
* `-I` 打印 Header，默認只會打印 body
	* `-i` 表示都打印
* `-d` 發送數據用於 POST 請求
* `-H` 添加 HTTP Header
* `-b` 添加 cookie
```shell
# GET
curl -b 'foo=bar' https://google.com

# POST
# 發送 json 數據並添加標頭 json 標頭
curl -d '{"login": "emma", "pass": "123"}' -H 'Content-Type: application/json' https://google.com/login

# Download
curl -O https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
```

### 連通測試: ping

### 節點分析: traceroute

### 端口監聽: ss, netstat

### 域名解析: nslookup

### 封包擷取: wireshark

### TCP/UDP 服務檢測: nc
netcat 命令用於 TCP, UDP 或 unit socket 的數據流操作，有以下功能：
* 打開 tcp 連接
* 發送 udp 數據包
* 監聽 TCP, UDP 端口

#### 測試 TCP/UDP
```shell
nc -z -v 192.168.10.12 22 #tcp
nc -z -v -u 192.168.10.12 123 # udp


nc -z -w 10 -v  192.168.10.12 22 #tcp
```
* `-z` 表示不發送任何數據，常用來測試
* `-u` 表示使用 UDP，默認為 TCP
* `-w` 設置超時秒數
#### 監聽
```shell
nc -l localhost 8888 # 讓 nc 監聽 8888 端口，之後可以輸出，用於測試 client
```
