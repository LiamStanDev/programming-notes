### 安裝系統
* 虛擬磁碟建立
```shell
qemu-img create -f qcow2 /path/to/windows11.qcow2 100G
qemu-img info ./windows11.qcow2 # 查看訊息
```
> 建議使用 qcow2 或者 raw
* 增加磁盤大小，不會格式化
```shell
qemu-img resize /path/to/windows10.qcow2 +50G
```
* 安裝 OS
```shell
qemu-system-x86_64 \
  -machine q35 \
  -cpu host \
  -enable-kvm \
  -smp cores=6 \
  -m 16384 \
  -drive file=/home/liam/Documents/VMs/windows10.qcow2,if=virtio \
  -vga qxl \
  -device qemu-xhci \
  -device usb-tablet \
  -device virtio-mouse-pci \
  -netdev user,id=net0 -device e1000,netdev=net0 \
  -audiodev pa,id=snd0 -device usb-audio,audiodev=snd0 \
  -drive file=/home/liam/Documents/VMs/Win10_22H2_English_x64v1.iso,media=cdrom \
  -drive file=/home/liam/Documents/VMs/virtio-win-0.1.240.iso,media=cdrom
```
* -hda /path 可以取代 -drive file=/path,format=qcow2，但這邊還有做 virtIO 與 cache 優化
* usb-tablet 可以使滑鼠在界面上更精準，沒有的話 host 跟 guest 滑鼠會衝突。
* -vga 有很多選項，運行最順的是 qxl
* -device e1000 這個網卡才 win10 讀得到， virtio-net-pci 據說有優化但讀不到
* -audiodev 中的 pa 表示 palseaudio, 也可以使用 alsa等，然後 usb-audio 讀的到其他的如 AC97 則 windows 讀不到。
* -cdrom 第一個為 win10 鏡像，第二個為 fedora project 中下載的 virtIO 驅動
> win10 / win11 激活命令：`irm https:massgrave.dev/get | iex`

### 運行環境
```shell
qemu-system-x86_64 \
  -machine q35 \
  -cpu host \
  -enable-kvm \
  -smp cores=6 \
  -m 16384 \
  -drive file=/home/liam/Documents/VMs/windows10.qcow2,if=virtio \
  -vga qxl \
  -device qemu-xhci \
  -device usb-tablet \
  -device virtio-mouse-pci \
  -netdev user,id=net0 -device e1000,netdev=net0 \
  -audiodev pa,id=snd0 -device usb-audio,audiodev=snd0 \
```

### 免費 windows
irm https://massgrave.dev/get | iex


# Virt-Manager
---
因爲直接使用 qemu 做細部調整實在太痛苦了，後來我發現使用 virt-manager 超級方便。

* win10 + VirtIO 安裝： https://www.youtube.com/watch?v=nIln3G3N3m0
* 優化: https://ivonblog.com/posts/qemu-kvm-vfio-gaming/
* Looking glass 安裝: https://blandmanstudios.medium.com/tutorial-the-ultimate-linux-laptop-for-pc-gamers-feat-kvm-and-vfio-dee521850385
## 介紹
### libvirt
* 定義：libvirt是一個用於管理不同虛擬化技術的抽象虛擬化管理庫。它提供了一個統一的API，用於管理QEMU/KVM、Xen、LXC、OpenVZ等多種虛擬化技術。
* 功能：libvirt允許用戶通過命令行工具、API或GUI工具來創建、配置、啟動、停止、監視和管理虛擬機器和容器。
* 使用場景：libvirt通常用於虛擬化服務器環境，使管理者能夠輕松地處理多個虛擬機器實例。

### virt-manager
* 定義：virt-manager是一個用於**圖形化虛擬機器管理工具**。它是libvirt庫的前端界面，通過提供直觀的GUI，使用戶可以更容易地管理虛擬機器。
* 功能：virt-manager允許用戶創建、配置、啟動、停止、複製、移動和監視虛擬機器，而無需使用命令行。它還提供了虛擬機器的性能監控和細節查看。
* 特點：它是一個跨平台的應用程序，可以在Linux和Windows上運行。它支持多種虛擬化技術，包括QEMU/KVM、Xen和LXC。
* 使用場景：virt-manager通常用於桌面虛擬化環境或小型虛擬化部署，例如測試和開發環境，或者需要圖形界面的場景。
### 下載
* win10 鏡像
* virtIO: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/

```shell
sudo pacman -S virt-manager dnsmasq dmidecode

# 啓用 libvirtd 服務
systemctl enable libvirtd
systemctl status libvirtd
```
* `dnsmasq`:  Lightweight, easy to configure DNS forwarder and DHCP server
* `dmidecode`: Desktop Management Interface table related utilities

* 詳細使用： https://ivonblog.com/posts/install-windows-11-qemu-kvm-on-linux/

#### 解決 windows 解析度無法調整問題
* 請查看：https://www.spice-space.org/download.html
* 下載 spice-guest-tools 提供解析度調整 copy 共通等功能


#### 解決 NAT default inactive 問題

1. 建立以下文件 `default-net.xml`

```xml
<network>
  <name>default</name>
  <uuid>9a05da11-e96b-47f3-8253-a3a482e445f5</uuid>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:0a:cd:21'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
```

2. 執行

```shell
# 定義網路
sudo virsh net-define --file default-net.xml

# 啓動默認
sudo virsh net-start default

# 自啓動設定
sudo virsh net-autostart --network default
```



### Qemu 橋接網路
reference: [安装qemu-kvm以及配置桥接网络](https://zhou-yuxin.github.io/articles/2018/%E5%AE%89%E8%A3%85qemu-kvm%E4%BB%A5%E5%8F%8A%E9%85%8D%E7%BD%AE%E6%A1%A5%E6%8E%A5%E7%BD%91%E7%BB%9C/index.html)
1. 建立 tuntap 橋接設備
```shell
sudo ip tuntap add dev tap0 mode tap user liam
```
建立 tuntap 設備名稱為 tap0 模式為 tap 用戶為 liam

2. 設置 tap 設備權限
```shell
sudo chmod 0666 /dev/net/tun ```

3. 為 tap0 網卡添加 IP 地址，不要與真實網卡相同網段，如 192.168.100.3/24 表示 192.168.100.xxx 都是相同網段，故可以設成 192.168.200.1/24
```shell
sudo ip addr add 192.168.200.1/24 dev tap0
sudo ip link set dev tap0 up
```

4. 啟用 IP 轉發，把宿主機當成路由器
```shell
sudo sysctl -w net.ipv4.ip_forward=1
```
`-w` 表示寫入

5. 開啟 firewalld GUI 界面，添加 Masqureeade (添加到此 zone 的設備會進行 IPv4 轉發)

6. 啟動 qemu

### 硬體 serial number 欺騙 (常用於 windows 系統)
在 xml 中添加
```xml
  <sysinfo type="smbios">
    <bios>
      <entry name="vendor">LENOVO</entry>
    </bios>
    <system>
      <entry name="version">0.9.4</entry>
      <entry name="serial">8CG8161SH7</entry>
    </system>
    <baseBoard>
      <entry name="manufacturer">LENOVO</entry>
      <entry name="product">20BE0061MC</entry>
      <entry name="version">0B98401 Pro</entry>
      <entry name="serial">W1KS427111E</entry>
    </baseBoard>
    <chassis>
      <entry name="manufacturer">Dell Inc.</entry>
      <entry name="version">2.12</entry>
      <entry name="serial">65X0XF2</entry>
      <entry name="asset">40000101</entry>
      <entry name="sku">Type3Sku1</entry>
    </chassis>
    <oemStrings>
      <entry>myappname:some arbitrary data</entry>
      <entry>otherappname:more arbitrary data</entry>
    </oemStrings>
  </sysinfo>
  <os firmware="efi">
    ...
v    <smbios mode="sysinfo"/>
  </os>
```



## 性能優化

以下性能優化用於 windowns 虛擬機

#### 1. Hyper-V enlightenments

```xml
<hyperv>
  <relaxed state='on'/>
  <vapic state='on'/>
  <spinlocks state='on' retries='8191'/>
  <vpindex state='on'/>
  <synic state='on'/>
  <stimer state='on'>
    <direct state='on'/>
  </stimer>
  <reset state='on'/>
  <frequencies state='on'/>
  <reenlightenment state='on'/>
  <tlbflush state='on'/>
  <ipi state='on'/>
</hyperv>
```


- `<relaxed state='on'/>`：啟用 relaxed timer，減少虛擬機與主機的時鐘同步負擔。
- `<vapic state='on'/>`：啟用虛擬 APIC，提升中斷處理效能。
- `<spinlocks state='on' retries='8191'/>`：優化多核心下的 spinlock 行為，減少 CPU 資源浪費。
- `<vpindex state='on'/>`：啟用虛擬處理器索引，提升多核心調度效率。
- `<synic state='on'/>`：啟用合成中斷控制器，改善中斷管理。
- `<stimer state='on'><direct state='on'/></stimer>`：啟用合成計時器，提升計時精度與效能。
- `<reset state='on'/>`：支援 Hyper-V 的重置功能。
- `<frequencies state='on'/>`：讓虛擬機能取得主機 CPU 頻率資訊。
- `<reenlightenment state='on'/>`：支援 reenlightenment，提升動態調整效能。
- `<tlbflush state='on'/>`：優化 TLB flush 行為，提升記憶體管理效能。
- `<ipi state='on'/>`：啟用虛擬化的 IPI（Inter-Processor Interrupts）。

#### 2. 關閉計時器

```xml
<clock offset='localtime'>
  <timer name='rtc' present='no' tickpolicy='catchup'/>
  <timer name='pit' present='no' tickpolicy='delay'/>
  <timer name='hpet' present='no'/>
  <timer name='kvmclock' present='no'/>
  <timer name='hypervclock' present='yes'/>
</clock>
```


- `<clock offset='localtime'>`：設定虛擬機的時鐘與主機本地時間同步（常用於 Windows）。
- `<timer name='rtc' present='no' .../>`：關閉傳統 RTC（實時時鐘）計時器。
- `<timer name='pit' present='no' .../>`：關閉 PIT（可程式化間隔計時器）。
- `<timer name='hpet' present='no'/>`：關閉 HPET（高精度事件計時器）。
- `<timer name='kvmclock' present='no'/>`：關閉 KVM 專用時鐘。
- `<timer name='hypervclock' present='yes'/>`：僅啟用 Hyper-V 時鐘。

#### 3. CPU 綁定

```xml
<vcpu placement='static'>6</vcpu>
<iothreads>1</iothreads>
<cputune>
  <vcpupin vcpu='0' cpuset='1'/>
  <vcpupin vcpu='1' cpuset='5'/>
  <vcpupin vcpu='2' cpuset='2'/>
  <vcpupin vcpu='3' cpuset='6'/>
  <vcpupin vcpu='4' cpuset='3'/>
  <vcpupin vcpu='5' cpuset='7'/>
  <emulatorpin cpuset='0,4'/>
  <iothreadpin iothread='1' cpuset='0,4'/>
</cputune>
```


#### 完整配置

```xml
<domain type="kvm">
  <name>win10</name>
  <uuid>fee3a9bc-853c-4dd0-8cac-3c4546ed68a8</uuid>
  <metadata>
    <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
      <libosinfo:os id="http://microsoft.com/win/10"/>
    </libosinfo:libosinfo>
  </metadata>
  <memory unit="KiB">12582912</memory>
  <currentMemory unit="KiB">12582912</currentMemory>
  <vcpu placement="static">6</vcpu>
  <iothreads>2</iothreads>
  <cputune>
    <vcpupin vcpu="0" cpuset="0"/>
    <vcpupin vcpu="1" cpuset="1"/>
    <vcpupin vcpu="2" cpuset="2"/>
    <vcpupin vcpu="3" cpuset="3"/>
    <vcpupin vcpu="4" cpuset="4"/>
    <vcpupin vcpu="5" cpuset="5"/>
  </cputune>
  <sysinfo type="smbios">
    <bios>
      <entry name="vendor">LENOVO</entry>
    </bios>
    <system>
      <entry name="version">0.9.4</entry>
      <entry name="serial">8CG8161SH7</entry>
    </system>
    <baseBoard>
      <entry name="manufacturer">LENOVO</entry>
      <entry name="product">20BE0061MC</entry>
      <entry name="version">0B98401 Pro</entry>
      <entry name="serial">W1KS427111E</entry>
    </baseBoard>
    <chassis>
      <entry name="manufacturer">Dell Inc.</entry>
      <entry name="version">2.12</entry>
      <entry name="serial">65X0XF2</entry>
      <entry name="asset">40000101</entry>
      <entry name="sku">Type3Sku1</entry>
    </chassis>
    <oemStrings>
      <entry>myappname:some arbitrary data</entry>
      <entry>otherappname:more arbitrary data</entry>
    </oemStrings>
  </sysinfo>
  <os firmware="efi">
    <type arch="x86_64" machine="pc-q35-9.2">hvm</type>
    <firmware>
      <feature enabled="no" name="enrolled-keys"/>
      <feature enabled="no" name="secure-boot"/>
    </firmware>
    <loader readonly="yes" type="pflash" format="raw">/usr/share/edk2/ovmf/OVMF_CODE.fd</loader>
    <nvram template="/usr/share/edk2/ovmf/OVMF_VARS.fd" templateFormat="raw" format="raw">/var/lib/libvirt/qemu/nvram/win10_VARS.fd</nvram>
    <boot dev="hd"/>
    <smbios mode="sysinfo"/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <hyperv mode="custom">
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vpindex state="on"/>
      <runtime state="on"/>
      <synic state="on"/>
      <stimer state="on"/>
      <frequencies state="on"/>
      <tlbflush state="on"/>
      <ipi state="on"/>
      <evmcs state="off"/>
      <avic state="off"/>
    </hyperv>
    <vmport state="off"/>
  </features>
  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" clusters="1" cores="6" threads="1"/>
  </cpu>
  <clock offset="localtime">
    <timer name="rtc" present="no"/>
    <timer name="pit" present="no"/>
    <timer name="hpet" present="no"/>
    <timer name="hypervclock" present="yes"/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled="no"/>
    <suspend-to-disk enabled="no"/>
  </pm>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type="file" device="disk">
      <driver name="qemu" type="qcow2"/>
      <source file="/home/liam/Documents/virt/storage/win10.qcow2"/>
      <target dev="vda" bus="virtio"/>
      <address type="pci" domain="0x0000" bus="0x04" slot="0x00" function="0x0"/>
    </disk>
    <controller type="usb" index="0" model="qemu-xhci" ports="15">
      <address type="pci" domain="0x0000" bus="0x02" slot="0x00" function="0x0"/>
    </controller>
    <controller type="pci" index="0" model="pcie-root"/>
    <controller type="pci" index="1" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="1" port="0x10"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="2" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="2" port="0x11"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x1"/>
    </controller>
    <controller type="pci" index="3" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="3" port="0x12"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x2"/>
    </controller>
    <controller type="pci" index="4" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="4" port="0x13"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x3"/>
    </controller>
    <controller type="pci" index="5" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="5" port="0x14"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x4"/>
    </controller>
    <controller type="pci" index="6" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="6" port="0x15"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x5"/>
    </controller>
    <controller type="pci" index="7" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="7" port="0x16"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x6"/>
    </controller>
    <controller type="pci" index="8" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="8" port="0x17"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x7"/>
    </controller>
    <controller type="pci" index="9" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="9" port="0x18"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="10" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="10" port="0x19"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x1"/>
    </controller>
    <controller type="pci" index="11" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="11" port="0x1a"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x2"/>
    </controller>
    <controller type="pci" index="12" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="12" port="0x1b"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x3"/>
    </controller>
    <controller type="pci" index="13" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="13" port="0x1c"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x4"/>
    </controller>
    <controller type="pci" index="14" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="14" port="0x1d"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x5"/>
    </controller>
    <controller type="sata" index="0">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1f" function="0x2"/>
    </controller>
    <controller type="virtio-serial" index="0">
      <address type="pci" domain="0x0000" bus="0x03" slot="0x00" function="0x0"/>
    </controller>
    <interface type="network">
      <mac address="52:54:00:27:aa:25"/>
      <source network="default"/>
      <model type="virtio"/>
      <address type="pci" domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
    </interface>
    <serial type="pty">
      <target type="isa-serial" port="0">
        <model name="isa-serial"/>
      </target>
    </serial>
    <console type="pty">
      <target type="serial" port="0"/>
    </console>
    <channel type="spicevmc">
      <target type="virtio" name="com.redhat.spice.0"/>
      <address type="virtio-serial" controller="0" bus="0" port="1"/>
    </channel>
    <input type="tablet" bus="usb">
      <address type="usb" bus="0" port="1"/>
    </input>
    <input type="mouse" bus="ps2"/>
    <input type="keyboard" bus="ps2"/>
    <graphics type="spice" autoport="yes">
      <listen type="address"/>
      <image compression="off"/>
    </graphics>
    <sound model="ich9">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1b" function="0x0"/>
    </sound>
    <audio id="1" type="spice"/>
    <video>
      <model type="qxl" ram="65536" vram="65536" vgamem="16384" heads="1" primary="yes"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0"/>
    </video>
    <redirdev bus="usb" type="spicevmc">
      <address type="usb" bus="0" port="2"/>
    </redirdev>
    <redirdev bus="usb" type="spicevmc">
      <address type="usb" bus="0" port="3"/>
    </redirdev>
    <watchdog model="itco" action="reset"/>
    <memballoon model="virtio">
      <address type="pci" domain="0x0000" bus="0x05" slot="0x00" function="0x0"/>
    </memballoon>
  </devices>
</domain>
```

### Looking Glass

* 教學：https://ivonblog.com/posts/looking-glass-host-for-windows/
* nixos 設定：https://www.reddit.com/r/NixOS/comments/1kf674l/has_anyone_set_up_looking_glass_on_nixos/