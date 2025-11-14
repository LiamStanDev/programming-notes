### 網路連接
```shell
systemctl restart iwd
pacman -S dhcpcd
systemctl start dhcpcd

# 連接 wifi
iwctl # 進入
station list 
station wlan0 show
station wlan0 scan
station wlan0 get-networks
station wlan0 connect WIFINAME

# 測試網路
ping www.google.com
```

### 分區
```shell
# 查看設備與分區
fdisk -l

# 分區有多個要進行多次, 目標建立: EFI, Swap, Root filesystem
cfdisk /dev/nvme0n1

# 格式化文件系統
mkfs.fat -F 32 /dev/nvme0n1p1
mkswap /dev/nvme0n1p2
mkfs.btrfs -f -L arch_root /dev/nvme0n1p3 

# 確認
lsblk --fs
```
> btrfs raid 配置請參考 [[Btrfs 使用]]

### 掛載與建立子卷
```shell
# 掛載設備
mount /dev/nvme0n1p3 /mnt

# 建立子卷
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@var
btrfs su cr /mnt/@tmp
btrfs su cr /mnt/@snapshots
# 檢查
btrfs su list /mnt

# 卸載
umount /mnt # 加上 -R 會遞迴卸載

# 重新掛載
mount -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@ /dev/nvme0n1p3 /mnt
mkdir -p /mnt/{home,var,tmp,.snapshots}
mount -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@ /dev/nvme0n1p3 /mnt
mount -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@home /dev/nvme0n1p3 /mnt/home
mount -o noatime,nodatacow,ssd,discard=async,space_cache=v2,subvol=@tmp /dev/nvme0n1p3 /mnt/tmp
mount -o noatime,compress=zstd:1,ssd,discard=async,space_cache=v2,subvol=@var /dev/nvme0n1p3 /mnt/var
mount -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@snapshots /dev/nvme0n1p3 /mnt/.snapshots

# 掛載 efi
mkdir -p /mnt/efi # 我以前都用 /mnt/boot
mount /dev/nvme0n1p1 /mnt/efi

# 啟用交換區
swapon /dev/nvme0n1p2
```

### 安裝
```shell
# 跟新目前安裝鏡像站
pacman -Sy
pacman -S reflector
reflector --country TW --protocol https --sort rate --latest 5 > /etc/pacman.d/mirrorlist

# 安裝系統
pacstrap -K /mnt base base-devel linux linux-firmware git btrfs-progs grub efibootmgr vim networkmanager pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber reflector openssh man sudo archlinux-keyring intel-ucode
```

### 系統基礎設定
```shell
# Fstab
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab # check seriously

# Chroot
arch-chroot /mnt

# Time
ln -sf /usr/share/zoneinfo/Asia/TW /etc/localtime
hwclock --systohc

# local
vim /etc/locale.gen
locale-gen
echo 'LANG=en_US.UTF-8' > /etc/locale.conf

# network
vim /etc/hostname
```

* `/etc/hosts`
```text
127.0.0.1 localhost
::1 localhost
127.0.1.1 <hostname>
```

### User
```shell
passwd

useradd -mG wheel liam # -m: create home dir
passwd liam

vim /etc/sudoers
```

### Grub config
```shell
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

### Before reboot
```shell
systemctl enable NetworkManager
systemctl enable sshd
exit
umount -R /mnt

reboot
```

### After reboot
```shell
# ntp 服務設定
sudo timedatectl set-ntp true

# 重建 keyring，避免過期導致 gpg 問題
sudo pacman-key --init
sudo pacman-key --populate archlinux
sudo pacman -Sy archlinux-keyring

# 更新 mirror
sudo pacman -Sy
su
reflector --country TW --protocol https --sort rate --latest 5 > /etc/pacman.d/mirrorlist
exit

# 安裝 paru
cd ~
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si # 選 rustup 要先執行 `rustup default stable`

sudo vim /etc/pacman.conf
```