# 建置高可用集群
---
### 1. 建立 HAProxy
* docker-compose.yaml
```yaml
services:
  haproxy:
    container_name: proxy
    image: haproxy:lts
    restart: always
    ports:
      - 8100:8100 # management tool
    environment:
      - TZ=Asia/Taipei
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro # ro means container readonly
```

* haproxy.cfg
```text
global
        maxconn         32768
        log             127.0.0.1 local0 info

defaults
        log             global
        mode            tcp
        option          tcplog
        option          dontlognull
        retries         3
        option          redispatch
        maxconn         32768

listen  states
        bind            *:8100
        mode            http
        stats           enable
        stats           realm Haproxy\ Statistics
        stats           uri /
        stats           auth admin:admin
        timeout         connect 5000
        timeout         server 5s
        timeout         client 5s
```

### 2. 使用 Keepalive
#### 簡介
keepalived 通過 VRRP 協議，在相同 subnet 下中多個 keepalived 會建立一個虛擬 ip，該虛擬 ip 也需要在相同的 subnet 中，我們需要指定那個 keepalived 實例爲 Master 其他爲 Backup，當其他主機訪問該虛擬 ip 時，會發送 arp 請求，只有 Master 會回應 arp，故虛擬 ip 會被綁定到 Master 主機的 mac 地址，流向就會導向 Master。
> 虛擬 IP 會綁定在 master 主機的網卡上
##### 若 Master 掛掉呢？
keepalived 的 master 會定時的向所有的 keepalived 實例發送心跳包，若 Master 出問題 backup 會收不到心跳包 backup 節點們就會選舉出新的 Master，並回應 arp 請求刷新 ip table，將流量導向新的 master 節點。

#### 配置 HAproxy 高可用
本例將虛擬 ip 設定爲 192.168.124.124
##### vm1
```shell
sudo apt install keepalived
```
* 添加 `/etc/keepalived/keepalived.conf`
```shell
# 指定腳本
vrrp_script chk_proxy { 
    script "/home/liam/rabbitmq/check-proxy.sh" # 執行腳本位置，注意需要給 x 執行權限
    interval 2
    weight -2
}

vrrp_instance VI_PROXY {
    state MASTER # 指定 master
    interface enp1s0  # 使用您的網絡接口，透過 ip a 命令查看
    virtual_router_id 51 # 所有 keepalived 實例需要相同
    priority 101 # master 需要大於其他
    virtual_ipaddress {
        192.168.124.124  # 虛擬 IP 地址
    }
    track_script {
        chk_proxy # 指定要追蹤的腳本
    }
	authentication {
        auth_type PASS
        auth_pass qwe865123
    }
}
```

* check-proxy.sh
```shell
#!/bin/bash

nc -zv localhost 8100 # 使用 net cat 確認 haproxy 的管理工具是否能通

if [ $? != "0" ]; then
        killall keepalived # 若失敗殺掉 keepalived 就會停止心跳包
fi
```
> 注意：要使用 `chmod a+x check-proxy.sh` 添加權限

##### vm2
```shell
sudo apt install keepalived
```
* 添加 `/etc/keepalived/keepalived.conf`
```shell
# 指定腳本
vrrp_script chk_proxy { 
    script "/home/liam/rabbitmq/check-proxy.sh" # 執行腳本位置，注意需要給 x 執行權限
    interval 2
    weight -2
}

vrrp_instance VI_PROXY {
    state BACKUP # 指定爲 Backup
    interface enp1s0
    virtual_router_id 51 
    priority 100 
    virtual_ipaddress {
        192.168.124.124  
    }
    track_script {
        chk_proxy
    }
	authentication {
        auth_type PASS
        auth_pass qwe865123
    }
}
```

* check-proxy.sh
```shell
#!/bin/bash

nc -zv localhost 8100 # 使用 net cat 確認 haproxy 的管理工具是否能通

if [ $? != "0" ]; then
        killall keepalived # 若失敗殺掉 keepalived 就會停止心跳包
fi
```
> 注意：要使用 `chmod a+x check-proxy.sh` 添加權限