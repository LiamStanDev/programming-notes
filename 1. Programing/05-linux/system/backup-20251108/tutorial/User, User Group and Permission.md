## User 
---
### 新增
```shell
useradd -G docker wheel -s zsh liam
```
* -G : 添加組(允許多個)
* -s : 指定用戶登入shell
🔥 也可以之後使用 `chsh -s /usr/zsh`

### 刪除
```shell
userdel liam
```
* -r : 將用戶的家目錄也刪了(要小心使用)

### 更改
```shell
usdermod -a -G docker wheel -s zsh liam
```
* 比新增多了一個-a，表示append

### 新增與修改密碼
```shell
passwd
```
* root 可以在後面添加任何用戶的名稱，用來修改密碼
* 其他用戶只能修改自己的密碼

### 切換用戶
```shell
su liam 
su # 切換到root
```
* `sudo` 就是這個概念


## Group
---
### 新增
```shell
groupadd docker # 添加docker組
```

### 刪除
```shell
groupdel docker # 刪除docker組
```

### 修改
```shell
groupmod -g 101 -n newdocker docker
```
* -g : 修改組別識別碼
* -n : 修改名稱
### 查看
#### 列出所有組
```shell
groups
```
#### 列出該用戶所有的組
```shell
groups liam
```
#### 用戶組信息存放位置
在 `etc/group` 文件中。

## Permision
---
### 修改單一文件
```shell
chmod a+r file_name
```
* a u g o : 分別表示all, user(owner), group, other
* +/- r w x : 表示添加或刪除read, write, execute權限

### 所有子目錄文件
```shell
chmod -r a+r dir_name
```
* -r : recursive
