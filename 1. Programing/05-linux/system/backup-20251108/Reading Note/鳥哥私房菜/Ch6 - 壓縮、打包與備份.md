# 打包壓縮
---
### tar
##### 解壓縮
```shell
tar -xzvf xxx.tar.gz -C ~/Document # 解壓縮
```
- -x : 表示拆包
- -z : 使用gzip壓縮(壓縮能力差)
- -j : 使用bzip壓縮(壓縮能力中)
- -J : 使用xz壓縮(壓縮能力高)
- -v : verbose
- -f : 待建立的文件
- -C : 輸出目錄位置

##### 壓縮
```shell
tar -czvf  xxx.tar.gz xxx
```
* -c : create表示打包
###### 更新壓縮
```shell
tar -czvf xxx.newer.tar.gz --newer-mtime="2015/06/17" /dist/*
```

> 甚麼是 tarball ?
> 表示 tar 命令後並進行壓縮的檔案統稱之

##### 練習: 使用管道打包解包
```shell
tar -czvf - /etc | tar -xzvf -
```
* 第一個 `-` 表示標準輸出，第二個 `-` 表示標準輸入

### zip, unzip
##### 解壓縮
```shell
unzip xxx.zip
```

##### 壓縮
```shell
zip -r -v xxx.zip ~/Document/xxx
```
* -r : recursive,表示打包壓縮



# 備份
---
### xfs
```shell
sudo dnf in xfsdump # it will install: xfsdump, xfsrestore
```
#### xfs 系統備份
xfs 支持增量備份。
```shell
xfsdump [-l Level] [-f backupfile] target
xfsdump -I # 顯示備份資訊
```

```shell
# 完整備份 Level 0 (必須先完整備份才能進行後續增量備份)
xfsdump -l 0 -f /srv/boot.dump /boot # 會進入交互模式，要你取名子

# 增量備份，level 有 1-9，每次增量備份都要比前一次值大，
# 1 表示與 0 進行比較後增量，2 表示與 1 進行比較後增量
xfsdump -l 1 -f /srv/boot.dump1 /boot

# 查看 
xfsdump -I
```

#### xfs 系統還原
```shell
# 查看
xfsrestore -I

# 復原 Level 0
xfsrestore -f /srv/boot.dump /boot
# 需要一步一步來，先復原 level 0 在來 level 1 繼續下去
xfsrestore -f /srv/boot.dump1 /boot
```

#### rsync 備份
```shell
rsync -avzh --progress /mnt/point /home/liam/Documents/backup
```
* `-a`: archive
* `-v`: verbose
* `-z`: compress when transfer
* `-h`: human readable
* `--progress`: show progress

# 光碟寫入
---
#### 第一步: 建立 iso 檔案 (TODO)
skip...
```shell
mkisofs -r -v -o -V 'linux' /tmp/system.img /root /home /etc
```
* -r: 表示適用於 Linux/Unix 系統，換成 -J 表示使用 windows 系統
* -V: 表示讀取後顯示的名稱 (不一定要加)


#### 第二步: 燒入設備
這邊使用 `dd` 命令，但 `dd` 命令功能非常強大，本質就是拷貝。`cp` 用於單個文件，而 `dd` 可以對磁碟內容進行拷貝，其效率更高。
```shell
# 將光碟 iso 取下來
dd if='/dev/sdb' of='/tmp/system.iso'

# 將 iso 寫到 usb 中。
dd if='/tmp/system.img' of='/dev/sda' 

# 案例將 Fedora40 寫入到 usb 中
sudo umount /dev/sda
sudo dd if=/home/liam/Downloads/Fedora-Server-dvd-x86_64-40-1.14.iso of=/dev/sda bs=4M status=progress oflag=sync
sudo eject /dev/sda
```
> status=progress 會顯示進度，oflag=sync 確保寫入是同步進行

##### 其他 dd 操作
```shell
# 從 /dev/zero 讀取 1M 共讀取 20 次，故大小為 20M
dd if=/dev/zero of=/tmp/123 bs=1M count=20
```