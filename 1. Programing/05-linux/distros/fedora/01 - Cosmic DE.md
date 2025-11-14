### Installation
```shell
sudo dnf copr enable ryanabx/cosmic-epoch
sudo dnf install cosmic-desktop
```
* 記得登出再登入。

### Configuration
Cosmic DE 的配置文件在 `/etc/cosmic-comp/config.ron` 中 
```shell
sudo mkdir /etc/cosmic-comp
sudo cp cosmic-comp/config.ron /etc/cosmic-comp
sudo -e /etc/cosmic-comp/config.ron
```
> cosmic-comp 是 cosmic compositor 為該 DE 的核心。



