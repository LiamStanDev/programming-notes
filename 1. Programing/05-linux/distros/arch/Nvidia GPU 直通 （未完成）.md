### 設備建議
1. 副熒幕：接 CPU 內顯
2. 主熒幕：接 Nvidia 獨顯
3. 虛擬機設定：
	1. 需使用 UEFI（OVMF）開機
	2. 機器選用 Q35

### 操作流程

#### 1. 設定 Kernel 開機參數啓用 IOMMU
編輯 `/etc/default/grub`：
```bash
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt video=efifb:off"
```

| 參數                     | 功能                                             |
| ---------------------- | ---------------------------------------------- |
| intel_iommu=on         | 開啟 Intel 的 IOMMU 支援（必要）                        |
| iommu=pt               | 使用 passthrough 模式（只針對被直通裝置啟用虛擬化，其他正常）          |
| video=efifb:off        | 停用 EFI framebuffer，防止它佔用 iGPU 導致不能 passthrough |

```bash
# 更新 grub
sudo grub-mkconfig -o /boot/grub/grub.cfg
# 重啓
reboot

# 查找 IOMMU Group
shopt -s nullglob
for g in /sys/kernel/iommu_groups/*; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

找到
```text
IOMMU Group 14:
        0000:01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD106 [GeForce RTX 4060 Ti] [10de:2803] (rev a1)
        0000:01:00.1 Audio device [0403]: NVIDIA Corporation AD106M High Definition Audio Controller [10de:22bd] (rev a1)
```

#### 2. 編輯 Kernel 核心參數 ivfo 綁定
編輯 `/etc/default/grub`：
```shell
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt video=efifb:off vfio-pci.ids=10de:2803,10de:22bd kvm.ignore_msrs=1 kvm.report_ignored_msrs=0"
```
> 相同的 IOMMU 群組要一起被綁定

| 參數                                        | 功能                                                |
| ----------------------------------------- | ------------------------------------------------- |
| vfio-pci.ids=10de:2803,10de:22bd8086:4680 | 指定 vfio-pci 模組要接管的裝置（iGPU 的 Vendor:Device ID）     |
| kvm.ignore_msrs                           | 避免 Nvidia 顯卡初始化時存取 MSR (宿主奇才能使用的 Register) KVM 報錯 |
| kvm.report_ignored_msrs=0                 | 同上述原因，這邊就是不要回報 msrs 錯誤                            |

接下來新增禁止核心模組再開幾