# GunPG
---
## 介紹

GnuPG (通常簡稱為 GPG) 是一個用於**數據加密和數字簽名**的免費工具，用於保護通信和文件的安全。它使用 PGP (Pretty Good Privacy) 加密技術。

#### 特點
1. 公鑰加密：每一個用戶都有一對密鑰（公鑰與私鑰），公鑰用於加密私鑰用於解密或創建簽名。
2. 數字簽名：用戶可以使用私鑰對文件或消息進行簽名，期他人可以使用公鑰驗證簽名。
3. Web of Trust: 不同於 CA 的信任模式，GPG 使用的是信任網路模式，用戶間可以相互簽名建立信任
#### 使用場景
1. 電子郵件加密
2. 文件與數據加密
3. 數字簽名

## 使用
### 生成密鑰對
```shell
gpg --gen-key
gpg --full-gen-key # 比較好用
```
> 注：gpg key 會使用 email 作為該密鑰對的唯一識別

### 查看密鑰列表
```shell
gpg --list-keys
```

### 導入與導出公鑰
```shell
gpg --armor --export your-email@example.com
gpg --import publickeyfile
```
> --armor 表示將二進制轉為文本，用於郵件等傳輸

### 加密文件
```
gpg --encrypt --recipient 'recipient-email@example.com' file-to-encrypt
```

### 解密文件
```shell
gpg --decrypt encrypted-file.gpg
```

### 簽名文件
```
gpg --sign file
```

### 驗證簽名
```
gpg --verify signed-file
```

# OpenSSL
----
## 介紹
OpenSSL 是一個開源工具，實現 SSL （Secure Sockets Layer）層和 TLS（Transport Layer Security） 協議，在網路通信中保證數據安全的關鍵組建。
#### 特點
1. 加密庫：
	1. 對稱加密：AES, DES
	2. 非對稱加密: RSA, ECDSA, SHA-256
2. SSL/TLS 實現
3. 證書處理
#### 使用場景
1. 保護網路通信
2. 證書管理
3. 加密與解密數據
4. 生成數字簽名

## 使用
### 生成密鑰
```shell
openssl genpkey -algorithm RSA -out private_key.pem -pkeyopt rsa_keygen_bits:2048
```
* `genkey`: 生成密鑰
* `-algorithm`: 指定算法
* `-out`: 輸出文件
* `-pkeyopt rsa_keygen_bits:2048`: 設置密鑰長度爲 2048

### 提取公鑰
```shell
openssl rsa -in private_key.pem -pbuout -out public_key.pem
```
* `rsa`: 處理 rsa 算法
* `-in`: 指定 input
* `pubout`: 表示生成公鑰

### 創建證書簽名請求 (CSR)
是向證書頒發機構 (CA) 申請數字簽名的第一步。
```shell
openssl req -new -key private_key.pem -out request.csr
```
- `req`: 用於處理 CSR 的命令
- `-new`: 表示創建一個新的請求。
- `-key private_key.pem`: 指定私鑰。
- `-out request.csr`: 将生成的 CSR 保存到 `request.csr` 文件。

### 創建字簽名證書
自簽名證書是沒有通過 CA 認證的，常用於測試目的。
```shell
openssl req -new -x509 -days 365 -key private_key.pem -out selfsigned.crt
```
- `req`: 用於處理 CSR 的命令
* `-x509`: 表示生成 X.509 格式的證書。
- `-new`: 表示創建一個新的請求。
- `-key private_key.pem`: 指定私鑰。
- `-days 365`: 證書有效期爲 365 天。

### 生成一串密碼
```shell
openssl rand -base64 32
```


# mkcert
---
在開發過程中會需要生成字簽名證書用來測試 HTTPS。可以使用 openssl 工具但是有點太麻煩了，因爲 openssl 是用於生成正式的證書，我們只是想要測試所以可以使用 mkcert。
### 使用
我們想生成域名爲 app.carsties, api.carsties, id.carsties 的字簽名證書。
```shell
mkcert -key-file carsties.com.key -cert-file carsties.com.crt app.carsties.com api.carsties.com id.carsties.com  
```

#### 解決 chrome 不接收 self-signed certicate 問題
請參考： https://stackoverflow.com/questions/7580508/getting-chrome-to-accept-self-signed-localhost-certificate
