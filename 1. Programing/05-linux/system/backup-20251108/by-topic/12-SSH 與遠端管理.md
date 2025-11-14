# SSH 與遠端管理

## 目錄
- [簡介與核心概念](#簡介與核心概念)
- [原理與架構](#原理與架構)
- [常用命令與完整範例](#常用命令與完整範例)
- [常見錯誤與排查流程](#常見錯誤與排查流程)
- [最佳實踐與安全建議](#最佳實踐與安全建議)
- [實戰案例](#實戰案例)

---

## 簡介與核心概念

SSH（Secure Shell）是現代 Linux/UNIX 系統遠端管理的標準協定，提供加密的終端連線、身份驗證與資料傳輸。常見用途包括：
- 遠端登入與命令執行
- 檔案傳輸（SCP/SFTP）
- 內網穿透與端口轉發
- 跳板機串接
- 自動化部署

**核心特色：**
- 端對端加密，防止竊聽與中間人攻擊
- 支援密碼與金鑰認證
- 可擴展性強，適用於自動化與大規模管理

---

## 原理與架構

### SSH 協定流程
1. **TCP 連線建立**（預設 port 22）
2. **協商加密演算法**（如 AES、ChaCha20）
3. **身份驗證**（密碼或金鑰）
4. **建立安全通道**，進行命令、檔案、轉發等操作

### 認證方式
- **密碼認證**：簡單但不安全，建議停用
- **金鑰認證**：以公開金鑰（public key）與私密金鑰（private key）配對，支援 RSA、ECDSA、ED25519 等格式
- **多因素認證**：可結合 Google Authenticator、Yubikey

### 端口轉發（Port Forwarding）
- **本地轉發（-L）**：將本地 port 綁定到遠端服務
- **遠端轉發（-R）**：將遠端 port 綁定到本地服務
- **動態轉發（-D）**：SOCKS 代理，適合全流量代理

### 跳板機（Jump Host）
- 用於隔離內外網，強化安全
- 支援 ProxyJump/ProxyCommand 多重跳板

### SSH 配置檔
- `~/.ssh/config` 可集中管理多主機設定、跳板、金鑰、別名等

---

## 常用命令與完整範例

### 1. 遠端登入

```bash
ssh user@host
```
**範例輸出：**
```
Welcome to Ubuntu 22.04 LTS (GNU/Linux 5.15.0-60-generic x86_64)
user@host:~$
```

指定金鑰與自訂 port：
```bash
ssh -i ~/.ssh/id_ed25519 user@host -p 2222
```

### 2. 檔案傳輸

上傳檔案：
```bash
scp file.txt user@host:/path/
```
下載檔案：
```bash
scp user@host:/path/file.txt ./
```
**SFTP 互動式操作：**
```bash
sftp user@host
# 常用指令：ls、cd、put、get、exit
```

### 3. 金鑰管理

產生金鑰對：
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```
啟用金鑰代理並載入私鑰：
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```
複製公鑰到遠端主機：
```bash
ssh-copy-id user@host
```

### 4. 端口轉發

本地端口轉發（將本地 8080 綁定到遠端 3306）：
```bash
ssh -L 8080:127.0.0.1:3306 user@host
```
動態代理（SOCKS5）：
```bash
ssh -D 1080 user@host
```

### 5. 跳板機連線

單一跳板：
```bash
ssh -J jumphost user@target
```
多重跳板：
```bash
ssh -J jump1,jump2 user@target
```
`~/.ssh/config` 範例：
```
Host myserver
  HostName 192.168.1.10
  User admin
  IdentityFile ~/.ssh/id_ed25519
  ProxyJump jumphost
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h:%p
  ControlPersist 10m
```

### 6. 批次遠端操作

```bash
for host in host1 host2; do
  ssh $host "uptime"
done
```

---

## 常見錯誤與排查流程

| 錯誤現象             | 可能原因與解法                                                                 |
|----------------------|------------------------------------------------------------------------------|
| 連線失敗/Timeout     | 檢查網路、防火牆、主機名稱、port 是否正確；可用 `ping`、`telnet` 測試          |
| 金鑰權限錯誤         | 私鑰權限需為 600：`chmod 600 ~/.ssh/id_*`                                     |
| Host key mismatch    | 主機指紋變更，刪除 `~/.ssh/known_hosts` 對應條目                             |
| Permission denied    | 用戶名錯誤、金鑰未加入、遠端未允許該金鑰、金鑰格式不符                         |
| Too many authentication failures | 嘗試金鑰過多，建議用 `IdentitiesOnly yes` 限定金鑰                    |

**排查指令：**
```bash
ssh -v user@host
```
- 顯示詳細連線過程，協助定位問題
- 關鍵字：`Permission denied`、`No route to host`、`Connection refused`

---

## 最佳實踐與安全建議

- **金鑰管理**：私鑰嚴格權限（600），勿外洩；建議每台主機/用途分開金鑰
- **停用密碼登入**：`/etc/ssh/sshd_config` 設定 `PasswordAuthentication no`
- **限制允許用戶**：`AllowUsers`、`AllowGroups`
- **更換預設 port**：降低被掃描風險（但非根本防禦）
- **啟用 Fail2Ban**：防暴力破解
- **定期更換金鑰**：降低金鑰洩漏風險
- **跳板機隔離**：內外網分離，跳板機嚴格控管
- **自動化配置**：善用 `~/.ssh/config`、Ansible 等工具
- **日誌監控**：定期檢查 `/var/log/auth.log`、`journalctl -u sshd`

---

## 實戰案例

### 案例一：自動化批次部署

需求：同時在多台主機執行更新腳本

```bash
for host in web1 web2 db1; do
  ssh $host "sudo systemctl restart myapp"
done
```
> 建議搭配金鑰認證與 ssh-agent，避免密碼重複輸入。

---

### 案例二：安全存取內部資料庫

需求：本地開發機安全連線內網 MySQL

```bash
ssh -L 3307:127.0.0.1:3306 user@bastion
# 本地連線 127.0.0.1:3307 即可安全存取內網 DB
```

---

### 案例三：多重跳板串接

需求：跨多層防火牆連線內部主機

```bash
ssh -J bastion1,bastion2 user@target
```
或於 `~/.ssh/config` 設定多層 ProxyJump。

---

### 案例四：權限錯誤排查

現象：`Permission denied (publickey)`

排查步驟：
1. 檢查私鑰權限：`ls -l ~/.ssh/id_*`
2. 檢查公鑰是否已加入遠端 `~/.ssh/authorized_keys`
3. 檢查 sshd 設定 `PubkeyAuthentication yes`
4. 使用 `ssh -v` 觀察詳細錯誤訊息

---

### 案例五：SSH 配置最佳化

需求：頻繁多主機連線，提升效率

於 `~/.ssh/config` 設定：
```
Host *
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h:%p
  ControlPersist 10m
```
> 可大幅減少多次連線時的驗證延遲。

---
