# 容量使用
---
### df : diskfree
用於列出整個文件系統的使用情況。
```shell
df -hT 
```
* -h : human readable, 使用G, M表示
* -T : 顯示分區格式 e.g. ext4
🔥 可以使用 `duf` 工具
### du : disk usage
用於列出某個資料夾或者文件的佔用情況。
```shell
du -hs ~/Document
```
* -h : human readable
* -s : summary, 列出目錄的使用
* 默認不輸入路徑，表示當前目錄

# 磁盤分割與格式化
---
## 磁盤分割
### fdisk
用於磁盤分割命令
#### 查看磁盤現在分區情況
```shell
fdisk -l
```
#### 分割指定磁區
會進入終端交互模式。
```shell
fdisk /dev/sdb
```
* 按下 g : 表示採用GPT分區
* 按下 q : 表示什麼都不做後退出
* 按下 w : 表示寫入剛做的所有操作（小心）

## 文件系統
### mkfs
支持cramfs, ext3, ext4, msdos, vfat
```shell
mkfs.ext4 /dev/sdX # 建立 ext4
mkfs.fat -F 32 /dev/sdX # 建立 fat32 常用於製作 linux boot
mkfs.btrfs /dev/sdX
```



### 製作多系統可用 usb
> 建立 NTFS, exFAT 文件系統，他更適合在 windows, macOS, linux 下使用
#### 下載工具
```shell
sudo pacman -S exfat-utils # exFAT
sudo pacman -S ntfs-3g
```
1. 選 d : 刪除原始分區
2. 選 n : 建立新的分區
3. 選 p : 建立主分區
4. 都選默認更新整個扇區
5. 選 t : 選擇 07 建立 HPFS/NTFS/exFAT 
```shell
# exfat
sudo mkfs.nfts /dev/sda
# ntfs
sudo mkfs.exfat /dev/sda
```

# 掛載與卸除
---
### 查詢掛載
```shell
mount -l | bat
```

### 掛載
```shell
mount /dev/sdb3 /mnt/windows
```
* 將sdb3 掛載到 /mnt/windows
### 卸除
```shell
unmount /dev/sdb3
```

### 自動掛載硬碟 (靜態)
> 這種方式會使 linux 開機時尋找硬碟，若硬碟不存在則無法開機。
1. 添加掛載目錄
```shell
sudo mkdir -p /media/Backup
```
2. 查詢外部硬碟 uuid
```shell
sudo fdisk -l # 找到指定硬碟分區
lsblkid # 取得該分區uuid
```
3. 更改 static file system table
```shell
sudo -e /etc/fstab
# 添加以下內容
# /dev/sda1 (Backup)
UUID=2f26bfbc-192f-4b4d-8e6a-bacb6a556380	/media/Backup	ext4		defaults	0 0
```
* option 選 default 就好，裡面已經包含多數常用的 options 了。
* dump 選 1 表示該分區使用 dump 程序來進行備份。
* pass 選 1 表示 fsck 會對其進行檢查。
4. 進行掛載
```shell
sudo mount -a
```
* 表示對所有的 fstab 中的 fs 進行掛載

### 自動掛載(動態)
* 下載 udiskie
* 開機啓動背景運行
```shell
udiskie -t & 
```


# 文件系統操作
---
## Btrfs
Btrfs 是一個先進的文件系統，有如下幾個特性：
1. **COW (Copy-On-Write)**：Btrfs使用複製寫入技術，這意味著當需要修改文件時，它會首先創建一個新的副本，而不是直接在原始文件上進行修改。這有助於減少數據損壞的風險，並使數據恢復變得更容易。
2. **快照**：Btrfs支持快照，可以在不複製實際數據的情況下創建文件系統狀態的快照。這對於數據備份和版本控制非常有用。您可以隨時恢復到先前的快照狀態。
3. **數據校驗和修復**：Btrfs使用校驗和來檢測數據損壞，以確保數據的完整性。如果發現數據損壞，Btrfs可以自動從鏡像或RAID配置中的其他副本修復數據。
4. **多設備支持**：Btrfs允許您創建RAID配置，將數據存儲在多個物理硬碟上，以提高可用性和容錯性。它支持RAID 0、RAID 1、RAID 5、RAID 6等級。
5. **在線調整大小**：Btrfs允許您在線擴展或收縮文件系統，而無需卸載它。這使得調整文件系統大小變得更加方便和靈活。
6. **透明壓縮**：Btrfs支持透明壓縮，可以自動進行讓使用者沒有感覺壓縮與解壓縮過程，可以減少磁盤空間的使用，並提高存儲效率。
7. **子卷**: 可以在一個分區建立多個子卷，子卷不用指定大小，可以方便對於各個子卷進行設置。

#### 參考資料
* [Arch linux 安裝 btrfs](https://gist.github.com/dante-robinson/fdc55726991d3f17e0dbef1701d343ef)

### btrfs 操作
> 注意： btrfs 的 snapshot 功能只能在同一個 filesystem 中，所以不能跨硬碟
> 快照功能實際上不會增加太多空間，有點像是 git 版本控制的概念增量存儲。
```shell
# 建立子卷
btrfs subvolume create @subvol

# 掛載子卷
# e.g. 掛載剛建立在 /dev/nvme0n1p3 的子卷 @subvol 到 /mnt/point/subvol 上
mount -o subvol=@subvol /dev/nvme0n1p3 /mnt/point/subvol 

# 查看掛載點下的子卷 e.g. 主分區的
btrfs subvolume list /

# 刪除子卷
btrfs subvolume delete /mnt/point/subvol

# 建立快照，快照慣例也是使用 @ 開頭
btrfs subvolume snapshot / /snapshots/@snap_20241224_1223 # 建立 / 下的快照
# 刪除快照
btrfs subvolume delete /snapshots/@snap_20241224_1223

# 顯示文件系統使用情況，整個 brts 不是 subvol
btrfs filesystem df /

# 顯示該文件系統的硬體設備
btrfs filesystem show /

# 文件系統修復
btrfs check --repair /dev/nvme0n1p3

# 平衡文件系統：使空間利用率提高
btrfs balance start /path/to/mountpoint
btrfs balance status /path/to/mountpoint
```
* 注意 subvolume, create, delete, filesystem 可以分別使用短寫 su, cr, de, fi
#### 快照還原
本例爲 rollback 整個系統
```shell
# 查看子卷包含快照
btrfs subvolume list /
# 切換到目標快照
btrfs subvolume set-default <快照ID> /
# 驗證是否生效
btrfs subvolume get-default /
```
> 只要 rollback 某個子卷就將 /mnt/point 改成對應 mount point

### 建立主分區 btrfs
1. 建立 btrfs 文件系統
```shell
# 將 linux 分區製作 btrfs
mkfs.btrfs /dev/nvme0n1p3

# 建立子卷，@只是約定俗成代表子卷
btrfs su cr /mnt/@
btrfs su cr /mnt/@var
btrfs su cr /mnt/@opt
btrfs su cr /mnt/@tmp
btrfs su cr /mnt/@snapshots
```
2. 客製化 mount
```shell
mount -o noatime,commit=120,space_cache=v2,subvol=@ /dev/nvme0n1p2 /mnt

mkdir /mnt/{boot,home,var,opt,tmp,snapshots}

mount -o noatime,commit=120,space_cache=v2,subvol=@opt /dev/nvme0n1p2 /mnt/opt
mount -o noatime,commit=120,space_cache=v2,subvol=@tmp /dev/nvme0n1p2 /mnt/tmp
mount -o noatime,commit=120,space_cache=v2,subvol=@snapshots /dev/nvme0n1p2 /mnt/snapshots
mount -o noatime,commit=120,space_cache=v2,subvol=@home /dev/nvme0n1p2 /mnt/home
mount -o subvol=@var /dev/nvme0n1p2 /mnt/var
```
3. 生成 fs table
* 會依據指定的 mount point 建立該分區的 fs table
```shell
genfstab -U /mnt >> /mnt/etc/fstab
```
> -U 表示使用 UUID 唯一識別

4. 細部修改 fstab：添加透明壓縮、寫入時間等
	1. `rw`：表示文件系統將以可讀寫模式掛載，允許對文件系統進行讀取和寫入操作。
	2. `noatime`：此選項指示系統在讀取文件時不更新文件的訪問時間戳，這可以減少對磁碟的寫入操作，提高性能。
	3. `ssd`：這個選項通知文件系統分區位於SSD（固態硬碟）上，以便優化性能。
	4. `discard=async`：這個選項啟用TRIM支援，可以增加SSD的壽命，並提高寫入性能。可以只道 ssd 中哪些是可直接寫入，避免先刪除在寫入。
	5. `space_cache=v2`：啟用Btrfs文件系統的空間緩存（space_cache）功能，以提高性能。
	6. `commit=120`：此選項指定文件系統在每次寫入操作之後將數據同步到磁盤的時間間隔（以秒為單位）。
	7. `compress=zstd`：使用zstd壓縮算法來壓縮文件系統上的數據，以節省磁盤空間，其他選項[參考](https://www.reddit.com/r/btrfs/comments/hyra46/benchmark_of_btrfs_decompression/)
	1. `subvolid` 和 `subvol`：這兩個選項用於指定Btrfs子卷的號碼（subvolume ID）和子卷的名稱。Btrfs允許您將文件系統劃分為多個子卷，這些選項用於指定子卷的位置和名稱。
```text
# <file system> <dir> <type> <options> <dump> <pass>
# /dev/nvme0n1p3
UUID=947476a8-c983-4ed5-be27-6a12840efc07	/         	btrfs     	rw,noatime,ssd,discard=async,space_cache=v2,commit=120,compress=zstd,subvolid=256,subvol=/@	0 0

# /dev/nvme0n1p3
UUID=947476a8-c983-4ed5-be27-6a12840efc07	/opt      	btrfs     	rw,noatime,ssd,discard=async,space_cache=v2,commit=120,subvolid=258,subvol=/@opt	0 0

# /dev/nvme0n1p3
UUID=947476a8-c983-4ed5-be27-6a12840efc07	/tmp      	btrfs     	rw,noatime,nodatacow,ssd,discard=async,space_cache=v2,commit=120,subvolid=259,subvol=/@tmp	0 0

# /dev/nvme0n1p3
UUID=947476a8-c983-4ed5-be27-6a12840efc07	/home     	btrfs     	rw,noatime,ssd,discard=async,space_cache=v2,commit=120,compress=zstd,subvolid=261,subvol=/@home	0 0

# /dev/nvme0n1p3
UUID=947476a8-c983-4ed5-be27-6a12840efc07	/var      	btrfs     	rw,noatime,ssd,discard=async,space_cache=v2,compress=zstd,commit=60,subvolid=257,subvol=/@var	0 0

# /dev/nvme0n1p3
UUID=947476a8-c983-4ed5-be27-6a12840efc07	/snapshots 	btrfs     	rw,noatime,ssd,discard=async,space_cache=v2,commit=120,compress=zstd,subvolid=260,subvol=/@snapshots	0 0

# /dev/nvme0n1p1
UUID=A70B-A48A      	/boot     	vfat      	rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro	0 2

# /dev/nvme0n1p2
UUID=0901db6f-57ca-420c-b1e2-b275d20417f4	none      	swap      	defaults  	0 0
```



