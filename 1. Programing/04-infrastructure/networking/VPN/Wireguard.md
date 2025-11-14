### 簡介
---
Wireguard VPN 是一個快速穩定且小巧的 VPN，已經合併在 Linux 內核 5.6 以的版本中
> 這邊有建議請先將所有防火牆都關閉，測試通過後在開啟防火牆。
> 另外 Fedora 預設的 firewall 不要使用 (大坑)

### 使用
##### 安裝工具
```shell
sudo dnf install wireguard-tools systemd-resolved
```
	
##### 準備
```shell
# 檢查是否載入
lsmod | grep wireguard 
sudo -i # 進入 root

# 以下只有公網那台要
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf # 永久開啟 ip 轉發
sysctl -p # reload
```

##### 配置: 一公網 IP + 多內網 IP
* server 1 (具有公網 IP)
```shell
# 修改權限
cd /etc/wireguard
chmod 777 /etc/wireguard
umask 077

# 生成服務器密鑰
wg genkey > server.key
# 生成客戶端密鑰
(wg pubkey < server.key) > server.key.pub

# (更快) 
wg genkey | tee privatekey | wg pubkey > publickey

# 開啟防火牆
firewall-cmd --add-port=50814/udp --permanent
firewall-cmd --reload

# 開啟 DNS 
sudo systemctl enable systemd-resolved.service
sudo systemctl start systemd-resolved.service

echo "
[Interface]
Address = 10.0.0.1
PrivateKey = $(privatekey)
ListenPort = 50814

[Peer]
PublicKey = <server2_public_key>
AllowedIPs = 10.0.0.2/32

[Peer]
PublicKey = <server3_public_key>
AllowedIPs = 10.0.0.3/32
" > wg0.conf

# 啟用
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
systemctl status wg-quick@wg0
wg # 可以查看目前連接
#wg-quick up wg0 # 開啟
#wg-quick down wg0 # 關閉
```
> 註解
> 1. firewalld 使用的時候要注意現在網卡的 zone 是剛好對應到
> 2. 可以使用 iptables
> `PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE`
> `PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE`
> 但是 eth0 要改成自己的網卡

* server 2
```shell
# 修改權限
cd /etc/wireguard
chmod 777 /etc/wireguard
umask 077

# 生成服務器密鑰對
wg genkey | tee privatekey | wg pubkey > publickey

# 開啟防火牆
firewall-cmd --add-port=50814/udp --permanent
firewall-cmd --reload

echo "
[Interface]
Address = 10.0.8.2
PrivateKey = $(privatekey)
ListenPort = 50814

[Peer]
PublicKey = <server1_public_key>
AllowedIPs = 10.0.8.0/24 
Endpoint = 118.233.146.60:50814
PersistentKeepalive = 25
" > wg0.conf

# 啟用
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
systemctl status wg-quick@wg0
```
> 註解：
> 在內網主機中，需要將局域網所有的 IP 打到具有公網 IP 的主機上，所以網段為 /24，且也要添加上 Endpoint
> 其中 PersistentKeepalive 用於在 NAT 網路或者防火牆下使用，因為 wireguard 是非常安靜的協議，所以在以上兩個環境下，他們在看到一個連接上都沒有進行通訊，就會自動斷開，此時就無法再次連接了，故啟用該屬性表示在指定秒數發送心跳包維持連接，建議 20-30秒。

* server 3
```shell
# 修改權限
cd /etc/wireguard
chmod 777 /etc/wireguard
umask 077

# 生成服務器密鑰對
wg genkey | tee privatekey | wg pubkey > publickey

# 開啟防火牆
firewall-cmd --add-port=50814/udp --permanent
firewall-cmd --reload

echo "
[Interface]
Address = 10.0.8.3
PrivateKey = $(privatekey)
ListenPort = 50814

[Peer]
PublicKey = <server1_public_key>
AllowedIPs = 10.0.8.0/24 
Endpoint = 118.233.146.60:50814
PersistentKeepalive = 25
" > wg0.conf

# 啟用
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
systemctl status wg-quick@wg0
wg
```

reference: https://www.wireguard.com/papers/wireguard.pdf#page7


# Wireguard-easy
---
```shell
docker run --detach \
  --name wg-easy \
  --env LANG=en \
  --env WG_HOST=18.183.14.21 \
  --env PASSWORD_HASH='$2a$12$PjyNn8rVTW97gYmNQuZgYuudPi6LbkbXUg9JWsg7bW5frac6UbZXK' \
  --env PORT=51821 \
  --env WG_PORT=51820 \
  --volume ~/.wg-easy:/etc/wireguard \
  --publish 51820:51820/udp \
  --publish 51821:51821/tcp \
  --cap-add NET_ADMIN \
  --cap-add SYS_MODULE \
  --sysctl 'net.ipv4.conf.all.src_valid_mark=1' \
  --sysctl 'net.ipv4.ip_forward=1' \
  --restart unless-stopped \
  ghcr.io/wg-easy/wg-easy
```