# 安裝
---
#### 1. 建立 RAID
```shell
sudo mkfs.btrfs -f -d raid0 -m raid0 -L arch_root /dev/nvme0n1p3 /dev/nvme1n1p1
```
* `-f`: 強制
* `-d`: 指定資料區格式，e.g. `raid0`, `raid1`, ...
* `-m`: 指定元數據區格式，e.g. `raid0`, `raid1`, ...
* `-L`: label

#### 2. 掛載
```shell
sudo mkdir /mnt
sudo mount /dev/nvme0n1p3 /mnt # raid 只要掛載其中一個就可以
```

#### 3. 建立子卷
```shell
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@var
btrfs su cr /mnt/@tmp
```
> su, cr 分別是 subvolume 與 create 縮寫
> 不用建立 `@snapshots` 子卷若沒有使用 `snapper` 工具

#### 4. 重新掛載
```shell
# 卸載
umount /mnt
# 建立目錄
mkdir -p /mnt/{home,var,tmp}

# 掛載
mount -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@ /dev/nvme0n1p2 /mnt
mount -o noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@home /dev/nvme0n1p2 /mnt/home
mount -o noatime,nodatacow,ssd,discard=async,space_cache=v2,subvol=@tmp /dev/nvme0n1p2 /mnt/tmp
mount -o noatime,compress=zstd:1,ssd,discard=async,space_cache=v2,subvol=@var /dev/nvme0n1p2 /mnt/var
```

#### 5. 永久掛載
```shell
sudo genfstab -U /mnt >> /mnt/etc/fstab
sudo vim /nmt/etc/fstab # 檢查
```

#### 6. 禁用 COW
因爲 COW 嚴重影響寫入效能故建議在以下幾個目錄進行調整
```shell
sudo chattr +C /var/lib/docker
sudo chattr +C /home/liam/Documents # 通常有大文件，以及 VM
```

# Snapshot
---
### 快照
##### 建立與刪除
```shell
# 建立快照
sudo mkdir /.snapshots
# 1. root
sudo btrfs su snap / /snapshots/root-$(date +%Y%m%d-%H%M%S)
# 2. home
sudo btrfs su snap /home /snapshots/home-$(date +%Y%m%d-%H%M%S)

# 刪除快照
sudo btrfs su del /snapshots/root-20250724-1330
```
> 快照就是子卷

##### 還原
###### Root 建議在 Live USB 中操作
```shell
# 1. 掛載
sudo mount /dev/nvme0n1p3 /mnt

# 2. 復原
sudo btrfs su del /mnt/@ 刪除 @ 子卷
sudo btrfs su snap /mnt/.snapshots/root-20250724-1330 /mnt/@ # 複製到 @ 子卷

# 3. 驗証
sudo btrfs su li /mnt
```

###### Home
```shell
sudo mv /home /home_old # 備份
sudo btrfs su snap /snapshots/home-20250724-1330 /home
```

### 維運
#### 查看狀態
```shell
sudo btrfs filesystem show
# 查看使用空間與 RAID 類型
sudo btrfs filesystem df /
# 硬碟 I/O 錯誤檢查
sudo btrfs device stats / 
```


# Reference
---
https://gist.github.com/dante-robinson/fdc55726991d3f17e0dbef1701d343ef