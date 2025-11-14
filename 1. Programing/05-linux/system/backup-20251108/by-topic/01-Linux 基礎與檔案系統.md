# Linux 基礎與檔案系統

---

## 一、主題簡介

Linux 是現代伺服器、雲端、嵌入式系統的主流作業系統。其檔案系統設計嚴謹，強調「一切皆檔案」理念，支援多用戶、多任務，並以穩定性、安全性著稱。熟悉 Linux 檔案系統結構與操作，能有效提升系統管理、故障排查與自動化能力，是開發、維運、資安等領域的核心基礎。

---

## 二、原理與架構說明

### 1. Linux 系統架構

- **核心（Kernel）**：負責硬體管理、記憶體、檔案系統、網路等底層資源。
- **Shell**：命令列介面，提供與核心互動的橋樑（如 bash、zsh）。
- **應用程式**：各類工具與服務（如 vim、nginx、docker）。

### 2. 檔案系統結構

- 採用階層式目錄樹（root `/` 為頂層）。
- 主要目錄說明：
    - `/bin`：基本指令
    - `/etc`：系統設定檔
    - `/home`：用戶家目錄
    - `/var`：可變動資料（如 log）
    - `/tmp`：暫存檔
    - `/usr`：使用者應用程式與資源
    - `/dev`：裝置檔案
    - `/mnt`、`/media`：掛載點

### 3. 路徑與目錄

- **絕對路徑**：由 `/` 開頭（如 `/etc/passwd`）。
- **相對路徑**：相對於目前目錄（如 `../logs`）。
- `.` 代表目前目錄，`..` 代表上一層目錄。

### 4. inode 與檔案屬性

- **inode**：儲存檔案的中繼資料（如大小、權限、擁有者、時間戳）。
- 每個檔案/目錄皆有唯一 inode 編號。
- 檔案內容與名稱分離，名稱存在目錄結構，內容由 inode 指向。

### 5. 權限與擁有者

- 權限分為「擁有者（user）」、「群組（group）」、「其他人（other）」。
- 權限類型：讀（r）、寫（w）、執行（x）。
- 權限表示法：`rwxr-xr--` 或八進位（如 `755`）。
- 擁有者與群組可用 `chown`、`chgrp` 調整。

---

## 三、常用命令範例（含完整輸出）

### 1. 檔案與目錄查詢

#### ls

```bash
$ ls -l /etc
```
範例輸出：
```
drwxr-xr-x  2 root root 4096  4月 10 12:00 network
-rw-r--r--  1 root root  296  4月 10 12:00 passwd
```

#### tree

```bash
$ tree /etc -L 2
```
範例輸出：
```
/etc
├── passwd
├── network
│   └── interfaces
└── ssh
    └── sshd_config
```

#### find

```bash
$ find /var/log -name "*.log" -size +10M
```
範例輸出：
```
/var/log/syslog.log
/var/log/nginx/access.log
```

### 2. 目錄操作

#### cd & pwd

```bash
$ cd /var/log
$ pwd
/var/log
$ cd ../tmp
$ pwd
/tmp
```

### 3. 檔案屬性查詢

#### stat

```bash
$ stat /etc/passwd
```
範例輸出：
```
  File: /etc/passwd
  Size: 296        Blocks: 8          IO Block: 4096   regular file
Device: 803h/2051d Inode: 1234567    Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/   root)   Gid: (    0/   root)
Access: 2025-09-17 12:00:00.000000000 +0800
Modify: 2025-09-17 12:00:00.000000000 +0800
Change: 2025-09-17 12:00:00.000000000 +0800
```

---

## 四、常見錯誤與排查方式

- **找不到檔案**
    - 檢查路徑拼寫、檔案是否存在。
    - 工具：`ls`、`find`。
    - 範例：
      ```bash
      $ ls /not/exist
      ls: cannot access '/not/exist': No such file or directory
      $ find / -name "passwd"
      ```
- **權限錯誤**
    - 檢查檔案權限與擁有者。
    - 工具：`ls -l`、`chmod`、`chown`。
    - 範例：
      ```bash
      $ cat /etc/shadow
      cat: /etc/shadow: Permission denied
      $ sudo chown user:user file.txt
      $ sudo chmod 644 file.txt
      ```
- **磁碟空間不足**
    - 工具：`df -h`、`du -sh`。
    - 範例：
      ```bash
      $ df -h
      Filesystem      Size  Used Avail Use% Mounted on
      /dev/sda1        20G   19G   0G 100% /
      ```
    - 處理：清理不必要檔案或擴充磁碟。
- **inode 用盡**
    - 工具：`df -i`
    - 範例：
      ```bash
      $ df -i
      Filesystem      Inodes  IUsed   IFree IUse% Mounted on
      /dev/sda1      1310720 1310720      0  100% /
      ```
    - 處理：刪除過多小檔案。

---

## 五、最佳實踐與安全性建議

- 目錄結構規劃清晰，便於維護與自動化。
- 權限最小化原則，避免不必要的寫入或執行權限。
- 定期備份重要資料（建議用 `rsync`、`tar`）。
- 使用軟連結管理共用資源，減少重複。
- 監控磁碟空間與 inode，避免服務中斷。
- 關鍵檔案（如 `/etc/passwd`、`/etc/shadow`）嚴格限制權限。
- 重要操作建議搭配 `sudo`，並審查指令內容。
- 建議啟用審計（auditd）追蹤敏感檔案異動。

---

## 六、實戰案例

### 案例 1：快速查找並清理大檔案

**情境**：系統磁碟空間不足，需找出佔用空間最大的檔案並清理。

```bash
$ du -ah /var | sort -rh | head -n 10
```
- 解析：`du -ah` 計算所有檔案大小，`sort -rh` 由大到小排序，`head -n 10` 取前 10 名。

### 案例 2：修正權限錯誤導致服務啟動失敗

**情境**：Nginx 無法啟動，log 顯示權限錯誤。

```bash
$ sudo systemctl status nginx
$ sudo ls -l /var/www/html
$ sudo chown -R www-data:www-data /var/www/html
$ sudo chmod -R 755 /var/www/html
$ sudo systemctl restart nginx
```
- 解析：確認目錄擁有者與權限，修正後重啟服務。

### 案例 3：inode 用盡導致無法建立新檔案

**情境**：無法建立新檔案但磁碟空間充足。

```bash
$ df -i
$ find /var/log -type f -name "*.log" -delete
```
- 解析：檢查 inode 使用狀況，刪除大量小檔案釋放 inode。

---

## 七、進階技巧（資深工程師適用）

- **硬連結（Hard Link）/軟連結（Symbolic Link）**
    - 建立硬連結：`ln 原檔案 新硬連結`
    - 建立軟連結：`ln -s 原檔案 新軟連結`
    - 差異：硬連結共用 inode，軟連結為獨立檔案指向原檔案路徑。
- **掛載（Mount）**
    - 掛載檔案系統：`mount /dev/sdb1 /mnt/data`
    - 查看掛載狀態：`mount | grep /mnt`
    - 卸載：`umount /mnt/data`
- **檔案系統調整**
    - 擴充檔案系統：`resize2fs /dev/sdb1`
    - 檢查檔案系統：`fsck /dev/sdb1`
- **ACL（Access Control List）**
    - 設定 ACL 權限：`setfacl -m u:username:rwx 檔案`
    - 查詢 ACL：`getfacl 檔案`

---

## 八、參考資料

- [Linux Filesystem Hierarchy Standard](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html)
- [Arch Wiki - File systems](https://wiki.archlinux.org/title/File_systems)
- [Linux man pages](https://man7.org/linux/man-pages/)
