### 簡介
---
Flatpak 是一个为 Linux 发行版设计的软件部署、应用虚拟化和包管理系统。它为应用程序提供了一个沙盒环境，允许应用程序在与其他系统组件隔离的环境中运行。

### 使用
---
* 下載軟體 [Flathub](https://flathub.org/)
```shell
flatpak install flathub org.js.nuclear.Nuclear
```

* 下載位置
	* 軟體運行： /var/lib/flatpak/exports/bin 目錄下
	* 軟體圖示：/var/lib/flatpak/exports/share/icons/hicolor/512x512/apps 目錄下

#### 讓 rofi 可以啟動
桌面軟體需要建立 .desktop 文件，裡面描述要如何執行該應用，與一些應用的描述信息，flatpak 下載完軟體之後就會在 /var/lib/flatpak/exports/share/applications。

rofi 啟動的應用位置在 
1. /bin 目錄下的二進制文件
2. /usr/share/applications 目錄下的 .desktop 文件
3. $HOME/.local/share/applications 目錄下的 .desktop 文件

```shell
cp /var/lib/flatpak/exports/share/applications/* ~/.local/share/applications
```
> 圖片顯示不出來要自己修改 .desktop 文件將 Icon 指向 /var/flatpak/exprts/share/icon 目錄下的圖片