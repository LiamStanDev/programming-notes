1. (Bug 尚未解決) wezterm 啟動錯誤
發生原因：可能來自於系統更新或者 hyprland 更新，導致 wezterm 找不到 display protocal
解決方式：
1. 在官網下載 [nightly binary](https://wezfurlong.org/wezterm/install/linux.html#raw-linux-binary)，選擇 raw linux binary
2. 解壓縮之後將 usr 目錄下的 bin 與 share 放入 .local 目錄裡面的相應目錄。
3. 建立 .desktop 文件
```desktop
[Desktop Entry]
Name=WezTerm
Comment=Wez's Terminal Emulator
Keywords=shell;prompt;command;commandline;cmd;
Icon=/home/liam/.local/share/icons/hicolor/128x128/apps/org.wezfurlong.wezterm.png
StartupWMClass=org.wezfurlong.wezterm
Exec=/home/liam/.local/bin/wezterm
Type=Application
Categories=System;TerminalEmulator;Utility;
Terminal=false
```

2. 設定 fontconfig
* 系統級別： /etc/fonts/fonts.conf 與 /etc/fonst/conf.d 裡面的文件，前面的數字表示載入順序
* 用戶級別： ~/.config/fontconfig/fonts.conf
```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fotnts.dtd">
<fontconfig>
	<alias>
		<family>sans-serif</family>
		<prefer>
            <family>Noto Sans CJK TC</family>
			<family>Noto Sans CJK JP</family>
			<family>Noto Sans CJK SC</family>
		</prefer>
	</alias>
	<alias>
		<family>monospace</family>
		<prefer>
            <family>Yozai</family>
            <family>Comic Mono</family>
            <family>CaskaydiaCove Nerd Font</family>
            <family>Noto Sans CJK TC</family>
			<family>Noto Sans Mono CJK JP</family>
			<family>Noto Sans Mono CJK SC</family>
		</prefer>
	</alias>
</fontconfig>
```

3. Timeshift 無法起動
* 原因：運行 timeshift-launcher 時，xhost not found
* 解決： paru xorg-xhost

3. 建立 gpg key
```shell
gpg --full-gen-key
```


4. Arch linux 安裝系統選項
```shell
pacstrap -K /mnt base linux linux-firmware
```
* pacstrap: 這是一個腳本，用於將基本系統安裝到 /mnt 指定的新安裝位置。它是 Arch Linux 安裝過程的一部分，類似於其他發行版中的安裝器。
* -K: 這個選項告訴 pacstrap 保留已下載包的 PGP 簽名。這主要用於安全目的。
* /mnt: 這是你將安裝 Arch Linux 的目標目錄。在安裝過程中，你通常會先將硬碟分區掛載到這個目錄。
* base: 這是一個包組，包含了安裝最基本的 Arch Linux 系統所需的核心包。它包括了必要的系統工具和庫。
	* 其他選項：base-devel
* linux: 這是 Arch Linux 的默認核心包，包含了 Linux 核心自己。這是系統的核心部分，負責管理硬體和提供系統呼叫。
	* 其他選項：linux-lts (長期支持版)，linux-zen (特定硬體優化)
* linux-firmware: 這個包包含了用於各種硬體設備的固件文件，如顯卡、網卡等。這些固件是許多硬體設備正常運作所必需的。
* 其他韌體推薦
	* intel-ucode: 包含 Intel 處理器的微代碼更新，安裝微代碼更新可維護系統穩定性、安全性和性能
	* amd-ucode
	