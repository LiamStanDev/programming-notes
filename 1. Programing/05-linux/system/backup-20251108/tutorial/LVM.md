### 介紹
---
LVM (Logical Valume Manager)，主要目的是*解決傳統使用分區的方式無法擴容*的問題，透過將物硬碟進行抽象，以邏輯卷的方式表現給上層，而底層就交給 LVM 自動管理。

> 傳統分區問題:
> 因為一個分區完成之後就無法再修改，而每個分區都有自己獨立的文件系統，故擴容時需要備份 -> 給新硬碟分區 -> 寫入文件系統 -> 將數據移入。

#### LVM 的邏輯卷
![[Pasted image 20240507091832.png]]
* PV (Physical Volumes): 也就是將原始**分區**，格式化後稱之
* PE (Physical Extend): 磁碟切成以 4 MB 為大小的小區塊稱之，是 LVM 中最小的管理單位，PV 中會有很多個 PE。
* Volume Group (VG): 所有被加入的 PV 中的 PE 的集合，可以理解為空間池。
* Logical Volumes (LV): 從 VG 中拿出指定大小的空間所建立，由多個 PE 組成，是最終暴露給上層的抽象。

最後會生成 `/dev/vg_name/lv_name` 的文件。
> 它本質上是 soft link 到 /dev 中的文件。


### 操作
---
#### 創建 LVM 流程
1. 將物理磁碟初始化為 PV
```shell
pvcreate /dev/sdb /dev/sdc
```
2. 創建 VG 並將 PV 添加進去
```shell
vgcreate liamvg /dev/sdb /dev/sdc
```
3. 基於 VG 創建建立 LV
```shell
lvcreate -n liamlv -L 2G liam
```
4. 對 LV 建立文件系統
```shell
mkfs.xfs /dev/liamvg/liamlv
```
5. 掛載
```shell
mount /dev/liamvg/liamlv /mnt
```

#### 查看 LVM
```shell
# 查看 pv
pvdisplay
pvs # 簡略
# 查看 vg
vgdisplay
vgs # 簡略
# 查看 lv
lvdisplay
lvs # 簡略
```

#### 刪除 LVM
以下操作請先 `unmount`
```shell
# 刪除 LV
lvremove /dev/liamvg/liamlv
# 刪除 VG
vgremove liamvg
# 刪除 PV
pvremove /dev/sdb
```

#### LVM 拉伸與縮小
可以在線操作不用 `unmount`。

##### 擴充 LV
```shell
# 1. 確保 vg 有足夠空間
vgs

# 2. 擴充邏輯卷
lvextend -L +1G /dev/liamvg/liamlv

# 3. 確認
lvdisplay

# 4. 更新文件系統
resize2fs /dev/liamvg/liamlv # for ext2/3/4
xfs_growfs # for xfs

# 5. 檢查
df -h
```

##### 擴充 VG
```shell
# 1. 添加新的硬碟後，刷成 PV
pvcreate /dev/sdd
# 2. 添加到 VG
vgextend liamvg /dev/sdd
#3. 檢查
vgs
```

##### 縮小 (VG 與 LV)
這些操作都需要 `unmount`，這邊就不介紹了。
使用命令 `lvreduce`, `vgreduce` 這兩個命令，注意文件系統可能會需要 fsck。
