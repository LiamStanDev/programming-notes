### 安裝ssh
* ubuntu
```shell
sudo apt-get install openssh-server
```

### 查看狀態運行
```shell
systemctl status sshd
systemctl restart sshd
systemctl enable sshd
```

### 連接遠端主機
```shell
ssh username@remotehost -p 22
```
* -p (optional): 表示ssh監聽端口，不輸入默認為22

### 免密碼登入
主要是將`~/.ssh/authorized_key`追加本機的public key
* ssh-copy-id是一個工具：`brew install ssh-copy-id`
```shell
ssh-copy-id username@remotehost -p 22
```
> 默認端口爲 22 故以上命令不用添加 22 

### 配置別名
每次都要輸入`ssh username@remotehost`很麻煩，可以使用別名
在本地的`~/.ssh/config`中追加
```text
Host ubuntu
    HostName 172.16.44.129
    User liam
    Port 22
```

### 文件傳輸 : scp (ssh copy)
文件傳輸可以使用scp使用方式幾乎與ssh一樣，同樣也支持別名。
#### 傳輸文件
```shell
scp -P 22 /path/local/file user@remote:/path/remote/file 
scp dir/file ubuntu:dir/file # 傳送到~/dir/file
```
* 使用相對路徑默認為`$HOME`
* **remote傳送到local只要將順序交換就行，先輸入remote在輸入local**

#### 傳輸資料夾
```shell
scp -r dir ubuntu:Document/dir
```

### 保持進程運行
要注意ssh結束後，會自動殺掉所有進程，所以要保持連線才有辦法持續運作。正常來說是使用tmux來做到。

### 掛載遠端目錄到本機
todo...



