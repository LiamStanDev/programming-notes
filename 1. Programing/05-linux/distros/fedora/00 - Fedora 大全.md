# 基礎設定
---
### 設定 dnf
```toml
# /etc/dnf/dnf.conf

fastestmirror=True
max_parallel_downloads=10
```

### RPM Fushion and Copr
這兩個 package 都是提供官方主動提供的 packages。
* RPM Fusion: 因為 licences 的問題無法由官方提供的包，包含 free 與 non-free 兩種來源。
* Copr: 是由社區驅動的目標是提供一個平台給開發人員將他們的軟體發布在這，也就是 Fedora 的 AUR。
#### RPM Fushion
###### 安裝
```shell
# enable RPM Fushion
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
sudo dnf config-manager setopt fedora-cisco-openh264.enabled=1

# 提供在軟體商店可以看到 RPM Fushion 提供的軟件
sudo dnf groupupdate core

# ref: https://rpmfusion.org/Howto
# Multimedia
sudo dnf swap ffmpeg-free ffmpeg --allowerasing # ffmpeg
sudo dnf groupupdate multimedia --setopt="install_weak_deps=False" --exclude=PackageKit-gstreamer-plugin
sudo dnf groupupdate sound-and-video

# hardware accelerated codec
sudo dnf install intel-media-driver # intel (amd 請上官網看)
sudo dnf install nvidia-vaapi-driver # nvidia
```


##### nvidia
```shell
# for 555
sudo dnf --releasever=41 install akmod-nvidia xorg-x11-drv-nvidia nvidia-gpu-firmware xorg-x11-drv-nvidia-cuda kernel-devel --nogpgcheck
sudo dnf install kernel-devel # 因為要 rebuild kernel

# compiling and installing kernel modules
akmods

# regenerate initramfs
dracut -f --regenerate-all
```
* akmod-nvidia: Automatic Kernel Module for nvidia
* 以上執行 almods 之後出現 Fail 內容，通常是少了某一個版本的 kernel-devel 使用 dnf 載下來就行。



#### Copr
以下是一些例子。
```shell
# spotify
sudo dnf copr enable gregw/spotify
sudo dnf install spotify-client
# google-chrome
sudo dnf copr enable spot/google-chrome
sudo dnf install google-chrome-stable
```

### Flatpak
```shell
# Fedora 預裝 flatpak，以下命令添加 flathub 來源
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```

### dnf 使用

| 指令                             | 描述                                                                                                 |
| ------------------------------ | -------------------------------------------------------------------------------------------------- |
| `dnf install PACKAGE`          | 安裝軟件，<br>`dnf --releasever=41 in` :指定 fedora 版本<br>`--enablerepo=updates-testing in`: 下載 testing 包 |
| `dnf remove PACKAGE`           | 移除軟件                                                                                               |
| `dnf search SUBSTRING`         | 查找套件                                                                                               |
| `dnf info PACKAGE`             | 套件資訊                                                                                               |
| `dnf provides PACKAGE`         | 查找指定文件在哪可 packages<br>e.g. `dnf provides /bin/psql`, `dnf provides *gdb*` 會找到名稱有gdb的其他套件           |
| `dnf repoquery -l`             | 查找 packages 安裝哪些文件<br>e.g. `dnf repoquery -l postgresql`                                           |
| `dnf list installed`           | 列出已經安裝的套件                                                                                          |
| `dnf group list`               | 列出所有的 groups                                                                                       |
| `dnf group info GROUP_NAME`    | 套件組信息                                                                                              |
| `dnf group install GROUP_NAME` | 安裝套件組 (可以使用 `dnf install @GROUP_NAME`)                                                             |
| `dnf group remove GROUP_NAME`  | 移除套件組                                                                                              |
| `dnf upgrade`, `dnf update`    | 更新所有軟件包                                                                                            |
| `dnf check-update`             | 查看更新，但不會實際更新軟體                                                                                     |
| `duf autoremove`               | 列出所有因為依賴而安裝的 leaf packages                                                                         |

### rpm 命令使用

| 指令                   | 描述                |
| -------------------- | ----------------- |
| `rpm -i PACKAGE.rpm` | 安奘指定的 rpm package |
| `rpm -e PACKAGE.rpm` | 卸載軟件              |

 

### dnf repo 管理
```shell
sudo dnf repolist # 列出全部
sudo dnf repolist enabled # 列出 enabled 的
sudo dnf config-manager --set-enabled <repo_id>
sudo dnf config-manager --set-disabled <repo_id>
# 全局 testing package
sudo dnf config-manager --set-enable updates-testing
sudo dnf config-manager --set-disable updates-testing
# 全局 rawhide
sudo dnf config-manager --set-enabled rawhide
sudo dnf config-manager --set-disable rawhide
```
* [Update Testing Repo details](https://fedoraproject.org/wiki/QA:Updates_Testing)
> 可以在 /etc/yum.repos.d 路徑下找到各個 repo 的配置文件。

##### 補充：查詢包在哪個 repo
* fedora package: https://packages.fedoraproject.org/pkgs/hyprland/hyprland/
* rpm fushion: https://koji.rpmfusion.org/koji/packageinfo?packageID=365
	* 下載指定版本 `sudo dnf --releasever=41 in akmod-nvidia`

### 回到穩定版本
1. (非常重要) 使用 timeshift 進行整機備份
2. 執行退版命令
```shell
sudo dnf config-manager --set-disable updates-testing
sudo dnf clean all
sudo dnf list installed | rg update-testing
sudo dnf distro-sync # 會以穩定版為主，與 update 不同可能導致回退，不會影響其他 repos
```

# Hyprland
---
```shell
sudo dnf install hyprland hyprpicker rofi-wayland swaybg copyq waybar google-roboto-fonts udiskie blueman network-manager-applet pavucontrol swaylock polkit-gnome adwaita-icon-theme fcitx5-chewing firewall-applet

# notification center
#dnf copr enable erikreider/SwayNotificationCenter
#dnf install SwayNotificationCenter

# 下載 gtk theme
從 https://www.gnome-look.org/p/1661959 網頁中下載，後解壓縮
mv Colloid-Dark-Nord ~/.local/share/themes

# hyprlock
sudo dnf config-manager --set-enable updates-testing # 需要開啟 testing
sudo dnf install hyprlock hypridle
```
* 一定要使用 rofi-wayland 版，不然不會自動 focus (ref: https://www.reddit.com/r/hyprland/comments/1blh865/issue_with_rofi_not_holding_focus)
* 若在 `dnf update` 後 crash，可以先檢查 `nvidia-smi` 是否有正確回應，沒有表示可能更新了 kernel，需要下載最新的 kernel-devel 後，進行 `akmods` 與 `dracut`

### EWW bar
```shell
git clone https://github.com/elkowar/eww
cd eww
sudo dnf in gtk3-devel gtk-layer-shell-devel pango-devel gdk-pixbuf2-devel libdbusmenu-gtk3-devel cairo-devel glib2 libgcc glibc atk-devel
cd target/release
chmod +x ./eww
./eww daemon
./eww open <window_name>
```


### Fcitx5
```shell
# 下載主題
git clone https://github.com/escape0707/fcitx5-adwaita-dark.git ~/.local/share/fcitx5/themes/adwaita-dark

# 配置
# Vertical Candidate List
Vertical Candidate List=True
# Use mouse wheel to go to prev or next page
WheelForPaging=True
# Font
Font="Sans 10"
# Menu Font
MenuFont="Yozai 10"
# Tray Font
TrayFont="Yozai Bold 10"
# Tray Label Outline Color
TrayOutlineColor=#000000
# Tray Label Text Color
TrayTextColor=#ffffff
# Prefer Text Icon
PreferTextIcon=False
# Show Layout Name In Icon
ShowLayoutNameInIcon=True
# Use input method language to display text
UseInputMethodLanguageToDisplayText=True
# Theme
Theme=adwaita-dark
# Dark Theme
DarkTheme=default-dark
# Follow system light/dark color scheme
UseDarkTheme=False
# Follow system accent color if it is supported by theme and desktop
UseAccentColor=True
# Use Per Screen DPI on X11
PerScreenDPI=False
# Force font DPI on Wayland
ForceWaylandDPI=0
# Enable fractional scale under Wayland
EnableFractionalScale=True
```

### Chrome HIDPI 模糊
https://wiki.archlinuxcn.org/zh/Chromium 的 Wayland 原生運行，並注意輸入法部份

# KDE Plasma
---
### 配置系統
* 在 icons 選 tela-icon-theme: 使用 [github](https://github.com/vinceliuice/Tela-icon-theme) 下載完之後運行 `./install.sh` 
* 在 Login Screen 下載一個自己喜歡的 sddm theme
* 在 Splash Screen 選一個自己喜歡的
#### fcitx5 配置 
* 在 virtual keyboard 選擇 fcitx5 wayland
* 下載 [gitlab](https://gitlab.com/scratch-er/fcitx5-breeze)後下載 `sudo dnf install inkscape`，先運行 `./build.py` 後運行 `./install.sh`
* 在右下角輸入裡面的 config 選擇 breeze-theme
###### 解決 chrome wayland 無法使用問題
* chrome 啟動命令
```shell
GTK_IM_MODULE=fcitx google-chrome --enable-features=UseOzonePlatform --ozone-platform=wayland --enable-wayland-ime
```
> 注意這邊不要把 GTK_IM_MODULE 放在全局環境變數中，Fctix5 會一直跳建議視窗
* 將這條命令添加在 settings -> Keyboard -> Shortcuts 中
* 透過快捷鍵啟動後，把圖示 pin 在 pannel 上 (這樣圖標也會透過該命令啟動)
> 為什麼不改 .desktop 文件？
> 因為他媽的改了不會生效！！！！！！！！

#### Wezterm 配置
當我們在啟用 wezterm 的 wayland 模式的情況下，`window_decorations = "NONE"`, 這個配配置會無效，但我們可以透過設定 window rule 來設定
1. 右鍵點擊窗口標題欄
2. 選擇“more actions” > “configure special application rule”
3. 透果 Detect window properties 按鈕來設定當前的 Properties
4. (重點) 添加 property 為 No titlebar and frame，並且打勾 yes
![[Pasted image 20240825102856.png|600]]



# 開發環境
---
```shell
sudo dnf group install development-tools
# 註解: @virtualizatoin 就是 qemu
sudo dnf install zsh git stow eza fd-find ripgrep neovim @virtualization git-delta btop htop fastfetch zoxide gcc gcc-c++ tbb luarocks

# npm
mkdir ~/.nvm
# installs nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
# download and install Node.js (you may need to restart the terminal)
nvm install 22
# verifies the right Node.js version is in the environment
node -v # should print `v22.12.0`
# verifies the right npm version is in the environment
npm -v # should print `10.9.0`

# bun
curl -fsSL https://bun.sh/install | bash

# wezterm 
sudo dnf copr enable wezfurlong/wezterm-nightly
sudo dnf install wezterm # 一定要使用 copr 的版本，rpm 包有問題，clipboard 會無法使用

# should manually install
cd ~/.local

# mini conda
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod a+x Miniconda3-latest-Linux-x86_64.sh
./Miniconda3-latest-Linux-x86_64.sh
rm -f ./Miniconda3-latest-Linux-x86_64.sh

conda config --set changeps1 False
conda config --set auto_activate_base true

# go
wget https://go.dev/dl/go1.22.2.linux-amd64.tar.gz
tar -xzvf go1.22.2.linux-amd64.tar.gz
rm -f ./go1.22.2.linux-amd64.tar.gz

# rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# fzf 需要自己編譯 , lazygit, wezterm, starship
git clone --depth 1 https://github.com/junegunn/fzf.git ~/Download/fzf
~/.fzf/install

curl -sS https://starship.rs/install.sh | sh

# lazygit
sudo dnf copr enable atim/lazygit -y
sudo dnf install lazygit

# 下載 lazydocker
go install github.com/jesseduffield/lazydocker@latest # lazydocker

# visual studio code
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" | sudo tee /etc/yum.repos.d/vscode.repo > /dev/null
sudo dnf check-update
sudo dnf install code # or code-insiders
```

> devel 包與一般包查別是什麼？ e.g. tbb vs. tbb-devel
> tbb 包只會安裝運行時依賴如動態鏈接庫，而當你要開發的時候是需要靜態鏈接庫以及頭文件，這是由 devel 包提供的。
> * 默認這些頭文件會放在 `/usr/include` 中，庫會放在 `/usr/lib64` 
> * cuda 這種比較獨立的專有軟件不會遵守 FHS 標準，會放在 `/usr/local` 中

### zsh 配置
```shell
chsh -s /bin/zsh
# zap
zsh <(curl -s https://raw.githubusercontent.com/zap-zsh/zap/master/install.zsh) --branch release-v1
```
* 記得先登出在運行
* 去看 ~/.config/zsh 中有沒有多出的 .zshrc 文件，請查看是否是原本的被更名

### dotfiles
```shell
ssh-keygen -t rsa
cd ~
git clone git@github.com:LiamStanDev/dots.git
cd dots
stow  */
```

### container
```shell
# 下載 podman
sudo dnf install podman

# 以下目前 fedora 不推薦
# remove old version
sudo dnf remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine

# install
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo

sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# enable docker
sudo systemctl enable --now docker
```

### postgres 工具
```shell
sudo dnf install postgresql 
# add server (only local install postgres)
sudo dnf install postgresql-server postgresql-contrib
```


### Qemu Virtualization
```shell
# user mode emulator
sudo dnf install qemu-user # (semi-emulator based on linux) also include riscv 

# x86_64
sudo dnf install qemu-system-x86_64

# riscv system emulator
sudo dnf install qemu-system-riscv 
```


### RISC-V 環境
![[00 - 環境配置#Fedora 安裝方式]]

### GTK-rs
```shell
sudo dnf install gtk4-devel gcc
```

# 疑難雜症
---
### spotify 更新後打不開
```shell
flatpak run com.spotify.Client
```
會顯示以下信息
```text
/app/extra/bin/spotify: /usr/lib/x86_64-linux-gnu/libcurl.so.4: no version information available (required by /app/extra/bin/spotify)
```
這時候要執行 `rm -rf ~/.var/app/com.spotify.Client/cache`

ref: https://forums.opensuse.org/t/spotify-libcurl-error/174722/2
> 他指出 flatpak 大多數問題都可以用清理 cache 來解決


# Nvidia 
---
### 重裝
```shell
sudo dnf rm akmod-nvidia xorg-x11-drv-nvidia nvidia-gpu-firmware xorg-x11-drv-nvidia-cuda
sudo dnf distro-sync
sudo dnf autoremove
sudo dnf clean all
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia nvidia-gpu-firmware xorg-x11-drv-nvidia-cuda --nogpgcheck
su
akmods --rebuild
dracut -f --regenerate-all
nvidia-smi
```

### 最新版本 (不一定好)
```shell
sudo dnf --releasever=41 install akmod-nvidia xorg-x11-drv-nvidia nvidia-gpu-firmware xorg-x11-drv-nvidia-cuda --nogpgcheck
su # 切換為 root
akmods --rebuid # 建構 nvidia dirver 作為動態 kernel module
dracut -f --regenerate-all # 重新生成 initramdisk
# 檢查
nvidia-smi
```