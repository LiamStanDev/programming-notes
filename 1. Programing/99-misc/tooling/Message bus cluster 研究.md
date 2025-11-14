
# RabbitMQ 集群
---
ref: https://blog.csdn.net/u010797364/article/details/129379987

主機資訊與配置如下：

| 主機  | IP              | 備註                        |
| --- | --------------- | ------------------------- |
| vm1 | 192.168.124.66  | 作為 cluster node。          |
| vm2 | 192.168.124.149 | 配置 haproxy，並設定爲 MASTER 節點 |
| vm3 | 192.168.124.142 | 配置 haproxy，並設定爲 BACKUP 節點 |
![[Pasted image 20240527100102.png]]

### 建立普通 cluster
#### vm1 配置
在目錄 rabbitmq 中建立 rabbitmq.yaml
```yaml
services:
  krabbitmq:
    container_name: rabbit
    image: rabbitmq:3-management-alpine
    restart: always
    ports:
      - 4369:4369 # Erlang 通訊端口，初始化 cluster 必須
      - 5671:5671 # AMPD　協定 ssl 版本
      - 5672:5672 # AMPD　協定
      - 15672:15672 # 管理工具
      - 25672:25672 # RabbitMQ cluster 通信同步端口
    environment:
      - TZ=Asia/Taipei # Time zone
      - RABBITMQ_ERLANG_COOKIE=qwerty123456qwe # cookie 所有 node 需要一致
      - RABBITMQ_DEFAULT_USER=admin # 管理工具的帳號
      - RABBITMQ_DEFAULT_PASS=admin # 管理工具的密碼
    hostname: rabbit1 # 我們將 vm1 設定爲 rabbit1，每臺主機要不同
    extra_hosts:
      - rabbit1:127.0.0.1 # 自己的要寫上 loop 地址，因為是在 docker 內部不是 docker 的宿主
      - rabbit2:192.168.124.149
      - rabbit3:192.168.124.142
```
啟動
```shell
docker compose -f rabbitmq.yaml up -d
```

#### vm2 配置
在目錄 rabbitmq 中建立 rabbitmq.yaml
```yaml
services:
  krabbitmq:
    container_name: rabbit
    image: rabbitmq:3-management-alpine
    restart: always
    ports:
      - 4369:4369 
      - 5671:5671 
      - 5672:5672 
      - 15672:15672 
      - 25672:25672 
    environment:
      - TZ=Asia/Taipei
      - RABBITMQ_ERLANG_COOKIE=qwerty123456qwe # cookie 所有 node 需要一致
      - RABBITMQ_DEFAULT_USER=admin 
      - RABBITMQ_DEFAULT_PASS=admin 
    hostname: rabbit2 # 修改這邊 
    extra_hosts:
      - rabbit1:192.168.124.66
      - rabbit2:127.0.0.1 # loop
      - rabbit3:192.168.124.142
```

啟動並配置 cluster
```shell
docker compose -f rabbitmq.yaml up -d
docker exec -it rabbit /bin/bash

# 進入 container 後
rabbitmqctl stop_app # 不要使用 stop，會停止整個前臺程式，會導致 container 退出
rabbitmqctl reset
rabbitmqctl join_cluster rabbit@rabbit1 # 這邊的 rabbit1 會吃 compose file env 的配置
rabbitmqctl start_app
```

#### vm3 配置
在目錄 rabbitmq 中建立 docker-compose.yaml
```yaml
services:
  krabbitmq:
    container_name: rabbit
    image: rabbitmq:3-management-alpine
    restart: always
    ports:
      - 4369:4369 
      - 5671:5671 
      - 5672:5672 
      - 15672:15672 
      - 25672:25672 
    environment:
      - TZ=Asia/Taipei
      - RABBITMQ_ERLANG_COOKIE=qwerty123456qwe # cookie 所有 node 需要一致
      - RABBITMQ_DEFAULT_USER=admin 
      - RABBITMQ_DEFAULT_PASS=admin 
    hostname: rabbit3 # 修改這邊 
    extra_hosts:
      - rabbit1:192.168.124.66
      - rabbit2:192.168.124.149 
      - rabbit3:127.0.0.1 # loop
```

啟動並配置 cluster
```shell
docker compose -f rabbitmq.yaml up -d
docker exec -it rabbit /bin/bash

# 進入 container 後
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster rabbit@rabbit1
rabbitmqctl start_app
```

查看 mamagement tool 頁面是否配置 cluster 成功，如下圖
![[Pasted image 20240521133037.png]]
> 若出問題請將所有開啟端口的防火牆都關閉

### 配置鏡像模式
對於 Rabbitmq cluster 有幾種模式:
1. 普通模式：只會有一個 queue 會有真實的數據，其他 queue 只會存放指針指向存放數據的 queue
2. 鏡像模式：每一個 queue 都會有數據，速度較慢但是在生產環境下能做到備援
... 還有其他模式
#### 方式一：使用管理介面
![[Pasted image 20240521133432.png]]
添加完之後同樣可以在管理界面中查看 Exchange 與 qeue 有沒有顯示 ha-all policy。

### 配置 HAProxy
HAProxy 能作為 Loadbalancer 將流量進行分流，也可以將流量傳給正常服務中。

#### vm2
* haproxy.yaml
```yaml
services:
  haproxy:
    container_name: proxy
    image: haproxy:lts
    restart: always
    ports:
      - 8100:8100
      - 5670:5670 # for rabbitmq
      - 5673:5673 # for ssl rabbitmq
      - 15670:15670 # for rabbitmq management tool
    environment:
      - TZ=Asia/Taipei
    extra_hosts:
      - rabbit1:192.168.124.66 # 注意這邊不是 loop 地址，我們要到容器外面也就是 docker 的宿主
      - rabbit2:192.168.124.149
      - rabbit3:192.168.124.142
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro # ro means container readonly
```
* haproxy.cfg 
```cfg
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

listen  rabbitmq-balancer
        mode            tcp
        bind            *:5670
        balance         roundrobin
        server          rabbit1 rabbit1:5672 check inter 5s rise 2 fall 3
        server          rabbit2 rabbit2:5672 check inter 5s rise 2 fall 3
        server          rabbit3 rabbit3:5672 check inter 5s rise 2 fall 3
        timeout         connect 5000
        timeout         server 5s
        timeout         client 5s


listen  rabbitmq-manager-balancer
        mode            http
        bind            *:15670
        balance         roundrobin
        server          rabbit1 rabbit1:15672 check inter 5s rise 2 fall 3
        server          rabbit2 rabbit2:15672 check inter 5s rise 2 fall 3
        server          rabbit3 rabbit3:15672 check inter 5s rise 2 fall 3
        timeout         connect 5000
        timeout         server 5s
        timeout         client 5s


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
以上配置了 management tool 與無 ssl 的 rabbitmq 還有 haproxy 本身的工具。

#### vm3
* haproxy.yaml
```yaml
services:
  haproxy:
    container_name: proxy
    image: haproxy:lts
    restart: always
    ports:
      - 8100:8100
      - 5670:5670 # for rabbitmq
      - 5673:5673 # for ssl rabbitmq
      - 15670:15670 # for rabbitmq management tool
    environment:
      - TZ=Asia/Taipei
    extra_hosts:
      - rabbit1:192.168.124.66 # 注意這邊不是 loop 地址，我們要到容器外面也就是 docker 的宿主
      - rabbit2:192.168.124.149
      - rabbit3:192.168.124.142
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro # ro means container readonly
```
* haproxy.cfg 
```cfg
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

listen  rabbitmq-balancer
        mode            tcp
        bind            *:5670
        balance         roundrobin
        server          rabbit1 rabbit1:5672 check inter 5s rise 2 fall 3
        server          rabbit2 rabbit2:5672 check inter 5s rise 2 fall 3
        server          rabbit3 rabbit3:5672 check inter 5s rise 2 fall 3
        timeout         connect 5000
        timeout         server 5s
        timeout         client 5s


listen  rabbitmq-manager-balancer
        mode            http
        bind            *:15670
        balance         roundrobin
        server          rabbit1 rabbit1:15672 check inter 5s rise 2 fall 3
        server          rabbit2 rabbit2:15672 check inter 5s rise 2 fall 3
        server          rabbit3 rabbit3:15672 check inter 5s rise 2 fall 3
        timeout         connect 5000
        timeout         server 5s
        timeout         client 5s


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
以上配置了 management tool 與無 ssl 的 rabbitmq 還有 haproxy 本身的工具。

### 配置 keepalived
#### 簡介
keepalived 通過 VRRP 協議，在相同 subnet 下中多個 keepalived 會建立一個虛擬 ip，該虛擬 ip 也需要在相同的 subnet 中，我們需要指定那個 keepalived 實例爲 Master 其他爲 Backup，當其他主機訪問該虛擬 ip 時，會發送 arp 請求，只有 Master 會回應 arp，故虛擬 ip 會被綁定到 Master 主機的 mac 地址，流向就會導向 Master。

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

#### 結果
訪問 `192.168.124.124:8100` 如下
![[Pasted image 20240521142537.png]]
訪問 `192.168.124.124:1570` 如下
