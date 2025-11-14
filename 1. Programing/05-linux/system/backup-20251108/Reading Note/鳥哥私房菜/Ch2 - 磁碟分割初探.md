# Partition table
### MSDOS 分割表
---
早期為了相容 Windows 磁碟，採用 MSDOS 分割表，放置於第一個磁區，大小為 512 bytes，有兩個資料:
1. Master Boot Record (MBR): 安裝 boot loader 的地方 (446 bytes)
2. Partition table: 記錄整個硬碟的分割狀態 (64 bytes)

##### Partition Table
每組紀錄開始與結束的磁柱號碼，只有 64 bytes 故只能有 4 筆資料。

##### MSDOS 分割表問題
* MBR 空間太小
* 無法抓取 2.2T 以上的容量磁碟，因為 Partitial table 空間太小。

### GUID partition table, GPT 磁碟分割表
---
###### Logical Block Address, LBA
為了間容所有硬碟，在磁區的定義上會使用邏輯區塊位置來處理(增加一層抽象)，GPT 預設 LBA 大小為 512 bytes。

### GPT
![[Pasted image 20240423093355.png|600]]
* LBA 0: 與 MSDOS 間容存放了 boot loader (446 bytes)，但是原本的 Partition table 空間作為標示表明為 GPT 格式。
* LBA 1: 作為 GPT 表頭，記錄了分割表位置與大小，也紀錄備份用的 GPT 分割 (最後的 34 個 LBA 區塊)。
* LBA 2-33: 每個 LBA 可以記錄 4 個分割紀錄，總共可以有 128 筆分割。
	* 每筆紀錄使用 128 bytes 空間。

# 開機檢測程式
---
### BIOS
主機板上的 CMOS 會記錄各項硬體參數，而 BIOS 就是寫入到主機板的第一個程式 (韌體)，開機時就會自動執行，也是第一個執行的程式。

BIOS 到硬碟中讀取 MBR，然後就會執行 MBR 中的 boot loader。

### UEFI
因為 BIOS 本身是不兼容 GPT，只是 GPT 有提供 MBR 兼容區間才能讀取，故 UEFI 取代了 BIOS。
現在我們說的 BIOS 其實就是 UEFI，他有漂亮的介面，能進行硬體檢測，但功能一樣就是為了載入 boot loader。


# 目錄樹與掛載
---
### Directory tree, 目錄樹結構
Linux 所有資料都是以文件方式存在，而 Linux 的目錄樹指的就是以根目錄 (/) 為主，向下成呈現分支狀的目錄結構的一種文件架構。
![[Pasted image 20240423104255.png]]

### Mount, 掛載
Linux 系統使用的是目錄樹，是一種抽象結構，而資料是存放在磁盤中的分割槽中，我們就需要進行掛載。

##### 定義
利用一個目錄當成進入點，將磁盤分割槽放置在該目錄下，也就是進入該目錄就是讀取該分割槽。
我們會將某些目錄分別掛載到不同的分割槽，這樣需要清理與修復都可以單獨處理，有以下目錄:
* `/boot`: 這個目錄放了內核鏡像、boot loader 等。
* `/`
* `/home`: 用戶資料
* `/var`: 資料庫等會常使用這個空間
* `/tmp`: 暫存空間
* `swap`: 交換區

> 最簡單只需要 `/boot` `/` 與 `swap` 就好。


