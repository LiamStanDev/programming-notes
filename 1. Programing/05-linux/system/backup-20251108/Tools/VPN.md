# Zerotier
---
ref: https://ivonblog.com/posts/install-zerotier-on-linux/
1. 下載客戶端
```shell
curl -s https://install.zerotier.com | sudo bash
# or
sudo dnf in zerotier-one
```
2. 登入後 Create A Network，選擇想要的虛擬網段
3. 啟動 zerotier-one
```shell
sudo firewall-cmd --add-port=9993/udp
sudo firewall-cmd --reload

sudo systemctl enable --now zerotier-one
sudo zerotier-cli join "Network ID" # 請查看 zerotier 網頁
sudo zerotier-cli status

```