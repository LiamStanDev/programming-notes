### 目標
將整合顯卡（例如 Intel UHD Graphics）**直通給 QEMU 虛擬機使用**，達到以下目的：
- VM 可以使用 iGPU 進行桌面顯示或硬體加速（如 QSV 編解碼）；
- 主機（Host）不再使用該 iGPU，而交由 VM 控制。

### 流程
#### 1. BIOS 設定

| 設定                    | 設定  | 說名                     |
| --------------------- | --- | ---------------------- |
| VT-x                  | 開啓  | cpu虛擬化必須               |
| VT-d / Pre boot IOMMU | 開啓  | 在 bios 階段就加載 IOMMU     |
| iGPU Multi-Monitor    | 開啓  | 避免開機時 iGPU 沒用到因省電考慮不啓用 |

#### 2. 查詢裝置
```bash
# 查看當前顯卡有哪些
lspci | grep VGA
# 找出 ID
lspci -nnks 00:02.0
```
> `-s`: 表示指定裝置, `-k`: 顯示 driver 與 module

#### 3. Kernel 開機參數
編輯 `/etc/default/grub`：
```bash
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt video=efifb:off vfio-pci.ids=8086:4682" # ids 前一條指令可以看到多個請用逗號分隔
```

| 參數                     | 功能                                             |
| ---------------------- | ---------------------------------------------- |
| intel_iommu=on         | 開啟 Intel 的 IOMMU 支援（必要）                        |
| iommu=pt               | 使用 passthrough 模式（只針對被直通裝置啟用虛擬化，其他正常）          |
| video=efifb:off        | 停用 EFI framebuffer，防止它佔用 iGPU 導致不能 passthrough |
| vfio-pci.ids=8086:4680 | 指定 vfio-pci 模組要接管的裝置（iGPU 的 Vendor:Device ID）  |

```bash
# 更新 grub
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

#### 4. 產生 intramfs 時預載入 VFIO 模組
編輯 `/etc/mkinitcpio.conf`：
```bash
MODULES=(vfio_pci vfio vfio_iommu_type1)
```

| 模組               | 說明與作用                     |
| ---------------- | ------------------------- |
| vfio             | 核心 VFIO 模組，負責隔離與管理直通裝置的存取 |
| vfio_pci         | 處理 PCI 裝置直通的主要模組          |
| vfio_iommu_type1 | 提供與 IOMMU 相關的記憶體隔離功能      |


#### 5. 黑名單內建驅動（防止搶佔 iGPU）
建立 `/etc/modprobe.d/blacklist-i915.conf`：
```bash
blacklist i915
```

```bash
# 更新 initramfs
sudo mkinitcpio -P 
```

#### 6. 重啓之後驗証是否成功
```bash
# 取得 iGPU ID
lspci -nn | grep VGA

# 確認是否被綁定 vfio
lspci -nnk -d 8086:4682
```

### 失敗原因
最後遇到 code 43 with GPU passthrough 問題。


### 參考
ref: https://ivonblog.com/posts/archlinux-integrated-gpu-passthrough/
