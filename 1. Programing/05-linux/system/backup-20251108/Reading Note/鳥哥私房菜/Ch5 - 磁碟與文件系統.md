# File System
---
磁碟分區完之後還需要進行格式化，原因是要格式化成文件系統的格式，而一個分割槽就是一個能被格式化成為一種文件系統，但是現在因為 LVM 技術可以將一個分割槽格式化成多個文件系統，故現在稱**可以被掛載的資料稱為一個文件系統而不是一個分割槽**。

#### 甚麼是文件系統 ?
通常文件系統會將屬性與實際資料分別放在不同的區塊，如下:
* inode: 裡面存放屬性，如文件權限(rwx)、owner、group 與時間等等，每個 inode 都有編號，一個文件占用一個 inode，也會記錄該文件所在的 block 編號
* data block: 存放實際文件資料，具有編號，若檔案太大會有多個 block。
* super block: 紀錄檔案系統的整體資訊，如inode 與 block 的總量、使用量與剩餘量等。

透過文件系統，我們找到 inode 就能找到文件的資料，相對於在整個磁盤中尋找來的更有效率。


#### 文件系統與目錄樹
##### 目錄
Linux 建立目錄時會分配一個 inode 與至少一個 block，
* inode: 存放目錄權限與屬性
* block: 存放該目錄下的檔名與檔名占用的 innode 編號。
```shell
ls -li # 會顯示 inode 編號
```

##### 讀取流程
e.g. `/etc/passwd`
1. `/` 的 inode: 透過掛載點找到根目錄的 inode，查詢權限並驗證是否能繼續取得 block 編號
2. `/` 的 block: 透過 block 查詢 `etc/` 目錄的名稱後取得 inode 編號
3. `etc/` 的 inode: 查詢與驗證權限，成功後取得 `etc/` 的 block 編號
4. `etc/` 的 block: 查詢 `passwd` 的名稱後去得 inode 編號
...

#### filesystem 大小與讀取性能
一個文件系統太大，當使用久了文件來來去去，會導致 block 很分散 (越大實際物理位置可能分散越遠)，會引響讀取效率，可以進行文件系統反碎片畫(若有提供的化)操作。

#### Linux 支援的文件系統
* 傳統文件系統: ext2、FAT
* 日誌式文件系統: ext3/ext4、XFS、ZFS
* 網路文件系統: NFS、SMBFS
查看 linux 支援的文件系統
```shell
ls -l /lib/modules/$(uanem -r)/kernel/fs
```
* `uname -r`: 打印目前內核版本

#### Virtual Filesystem Switch (VFS)
因為一個 Linux 系統下同時需要運行多個文件系統，故使用 VFS 進行一層抽象之後，對 Linux 來說就是操作 VFS，VFS 分別對不同文件系統進行操作。

#### XFS 簡介
XFS 也是日誌文件系統，幾乎所有 Ext4 的功能都支援，專門用於處理高容量的磁碟與高效能文件系統的用途。
分為以下區塊:
1. data section: 包含 ext 中的 inode/data block/superblock，都放置在這個區塊。
2. log section: 存放日誌資料，當時既寫入到硬碟後才會清理，這個區塊頻繁變動。
3. realtime section: 資料會先在這邊進行創建，等待分配完畢後再放入 data section 中。


```shell
xfs_info /dev/sda # method1: 裝置名稱
xfs_info /dev/ # method2: 掛載點
```

# 文件系統操作
---
### 容量
##### df : diskfree
用於列出整個文件系統的使用情況。
```shell
df -hT 
```
* -h : human readable, 使用G, M表示
* -T : 顯示分區格式 e.g. ext4
* -i : 改為使用 inode 的數量來顯示
🔥 可以使用 `duf` 工具

##### du : disk usage
用於列出某個資料夾或者文件的佔用情況。
```shell
du -hs ~/Document
```
* -h : human readable
* -s : summary, 列出目錄的使用
* 默認不輸入路徑，表示當前目錄

### Link
##### Hard Link
* 每個文件都會占用一個 inode，文件內容由 inode 來指向。
* 目錄的 block 會記錄檔名與檔名對應的 inode。
> 檔名指與目錄有關，檔案內容與 inode 有關，所以一個 inode 可能有多個檔名指向。

Hard link 指的就是在某一個目錄下新增一個檔名對應到某個 inode 的號碼。
```shell
ls -li /etc/crontab # -i 列出 inode number
ln /etc/crontab . # 建立 hard link
ls -li /etc/crontab crontab
```

###### 優點
* 安全: 因為刪除只是刪除檔名，不會涉及 inode 與 block
* inode 與 block 數量都不會改變。 (除了目錄的 data block)
###### 缺點
* 不能跨 Filesystem
* 不能用在目錄
	* 因為目錄要記錄 . 與 .. 故會有循環問題。

##### Soft Link
Soft link 就是建立一個文件，有自己的 inode 與 block，而 block 紀錄的是 link 的檔名，所以原本檔案刪除 Soft link 會顯示找不到檔名。
相較於 Hard link，Soft link 沒有限制也很安全，所以使用比較廣泛。
```shell
ls -li /etc/crontab # -i 列出 inode number
ln -s /etc/crontab . # 建立 soft link
ls -li /etc/crontab crontab
```

# 分割、格式化、檢驗與掛載
---
新增一顆新的磁碟時所需要的動作，
1. 磁區分割，建立可用的 partition。
2. 對磁區進行格式化，建立可用的 filesystem。
3. 對 filesystem 進行檢測。
4. 建立掛載點，掛載到目錄樹的某個目錄。

### 磁碟分割
##### 列出所有磁碟
```shell
lsblk
```
* RO: 表示是否唯讀
* TYPE: 顯示是 disk, partition, 還是 rom
* RM: 表示是否可卸載，e.g. 光碟、usb
##### 列出磁碟的 UUID
```shell
blkid
```
會列出每個分區的 UUID 與文件系統類型。
##### 列出分割表資訊
```shell
sudo fdisk -l /dev/nvme0n1
```
會顯示 MBR 還是 GPT 等資訊。
##### 磁碟分割
使用 MBR 使用 `fdisk`，若使用 GPT 則使用 `gdisk`。統一使用 `cfdisk` 這個超好用! 另外 type 其實只是標誌用，其實根本不用管。


### 格式化
格式化其實就是建構文件系統。
```shell
mkfs.xfs -f /dev/nvme0n1p1 
```
* 裡面有很多參數可以調整(e.g. data section 配置、多核心讀取等配置)，需要在自己看，但默認就很快了。

### 檢驗
##### 修復
這只有在文件系統損壞時或者確認剛格式化的文件系統才使用，平時最好不要使用。
```shell
xfs_repair /dev/nvme0n1p1
fsck.ext4 # ext4 請用這個
```

### 掛載
設定掛載點時需要確認以下幾點:
1. 單一文件系統不應掛載在多個目錄中
2. 單一目錄不應掛載多個文件系統
3. 掛載點目錄應該為空目錄: 這樣原文資料會被遮蓋，等待卸載之後才會顯示。

```shell
mount -l # 顯示所有掛載資訊
lsblk # 列出存儲設備
mount /dev/sda1 /mnt/myusb # 掛載
```
* mount 實用參數
```text
-a  ：依照設定檔 /etc/fstab 的內容將所有未掛載的磁碟都掛載上來
-l  ：單純的輸入 mount 會顯示目前掛載的資訊。加上 -l 可增列 Label 名稱！
-t  ：可以加上檔案系統種類來指定欲掛載的類型。常見的 Linux 支援類型有：xfs, ext3, ext4,
      reiserfs, vfat, iso9660(光碟格式), nfs, cifs, smbfs (後三種為網路檔案系統類型)
-n  ：在預設的情況下，系統會將實際掛載的情況即時寫入 /etc/mtab 中，以利其他程式的運作。
      但在某些情況下(例如單人維護模式)為了避免問題會刻意不寫入。此時就得要使用 -n 選項。
-o  ：後面可以接一些掛載時額外加上的參數！比方說帳號、密碼、讀寫權限等：
      async, sync:   此檔案系統是否使用同步寫入 (sync) 或非同步 (async) 的
                     記憶體機制，請參考[檔案系統運作方式](https://linux.vbird.org/linux_basic/centos7/0230filesystem.php#harddisk-filerun)。預設為 async。
      atime,noatime: 是否修訂檔案的讀取時間(atime)。為了效能，某些時刻可使用 noatime
      ro, rw:        掛載檔案系統成為唯讀(ro) 或可讀寫(rw)
      auto, noauto:  允許此 filesystem 被以 mount -a 自動掛載(auto)
      dev, nodev:    是否允許此 filesystem 上，可建立裝置檔案？ dev 為可允許
      suid, nosuid:  是否允許此 filesystem 含有 suid/sgid 的檔案格式？
      exec, noexec:  是否允許此 filesystem 上擁有可執行 binary 檔案？
      user, nouser:  是否允許此 filesystem 讓任何使用者執行 mount ？一般來說，
                     mount 僅有 root 可以進行，但下達 user 參數，則可讓
                     一般 user 也能夠對此 partition 進行 mount 。
      defaults:      預設值為：rw, suid, dev, exec, auto, nouser, and async
      remount:       重新掛載，這在系統出錯，或重新更新參數時，很有用！
```

#### 特殊掛載 (loop 掛載)
用於想要讀取 iso 檔案。
```shell
mount -o loop /tmp/CentOS.iso /data/centos

unmount /data/centos
```

### 卸載
```shell
unmount /dev/sdb3
```

### 開機掛載
`/etc/fstab` 中添加，會在開機自動掛載。
```text
UUID=2f26bfbc-192f-4b4d-8e6a-bacb6a556380	/media/Backup	ext4		defaults	0 0
```
* option 選 default 就好，裡面已經包含多數常用的 options 了。
* dump 選 1 表示該分區使用 dump 程序來進行備份，備份方式有其他的現在都添 0
* pass 選 1 表示 fsck 會對其進行檢查，xfs 不用 fsck 所以都填 0。