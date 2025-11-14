### S 系列: Sync
```shell
# 下載
pacman -S emac 
# y: 表示 apt-update (update databse), u: 表示 apt-upgrade
pacman -Syu 
# s: search
pacman -Ss emac 
```

### R系列： Remove
```shell
# 移除該軟體 (極度不建議)
pacman -R vidir 
# 移除該軟體與 dependencies, s: recursive
pacman -Rs vidir 
# 移除該軟體與dependencies 與 system config, n: nosave
pacman -Rns vidir  # 最乾淨
```

### Q 系列: Local Query
```shell
# 列出所有包, 包含 dependency
pacman -Q | wc -l 
# 只列出我們實際輸入後才下載的包
pacman -Qe | wc -l 
# only name, q: quiet 
pacman -Qeq  # 最常用
# pkgs from main repo
pacman -Qn 
# pkgs from AUR
pacman -Qm 
# 列出非必要的依賴, d: dependencies
pacman -Qdt 
# 查詢該軟件下載的文件
pacman -Ql <pkg>
# 查詢所有名稱包含的本地 pkgs
pacman -Qs <substr>
```
> wc -l: word count with list

### F 系列：Files
```shell
# 查詢該文件在哪個包下載
pacman -Fy /usr/include/stdio.h  # -y 只有第一次要使用，用於更新資料庫
```

