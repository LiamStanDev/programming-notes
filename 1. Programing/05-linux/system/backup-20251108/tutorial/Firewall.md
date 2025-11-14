## 簡介
---
防火牆（Firewall）在Linux系統中是重要的安全工具，**用於控製網路流量，防止未經授權的存取和攻擊**。 Linux中有多種防火牆解決方案可供選擇，其中最常用的是iptables和更現代的nftables。
* iptables
	* 用於配置和管理 IPv4 套接字過濾規則。
	* 使用規則集（ruleset）來控制網絡流量，允許您定義哪些流量應該被接受、拒絕或重定向。
* nftables
	* 是 iptables 的繼承者，它是一個更現代的、功能更強大的包過濾和防火牆框架。
	* nftables 允許您配置和管理 IPv4 和 IPv6 的過濾規則，以及更多的高級網絡功能。
	* 它更容易配置和管理，並提供更好的性能和擴展性。
* firewalld
	* 是一個動態的防火牆管理工具，通常在 Linux 分發版（如Fedora、CentOS、RHEL）中使用。
	* 基於iptables 或 nftables (default)，並提供了一個更高級別的界面來配置和管理防火牆規則。
	* firewalld 支持不同的服務、區域和接口配置，並允許根據需要自動調整防火牆規則。

## nftables
---
### 安裝
```shell
sudo pacman -S nftables
```
skip 因爲主要深入學習 firewalld 就可以了

## firewalld
---
Firewalld（Firewall Daemon）為用戶提供了更簡單、更直觀的界面，用於設置網絡安全規則，而無需深入了解 iptables 或 nftables 的複雜語法。特點如下：
1. **動態更新**：Firewalld 允許您在運行時動態添加、刪除或修改防火牆規則，而不需要重新啟動防火牆服務。這使得配置更加靈活，不會中斷現有的網絡連接。
2. **區域和服務**：Firewalld 使用概念上的「區域」（zone）和「服務」（service）來簡化配置。每個區域可以有自己的防火牆規則，並且您可以將不同的網絡接口分配給不同的區域。同時，服務定義了特定應用程序或服務的規則，例如 SSH、HTTP 或 FTP。這樣，您可以更容易地將規則應用到不同的應用程序或服務。
3. **預設規則**：Firewalld 附帶一組預設的安全性規則，這些規則可幫助保護您的系統免受未經授權的訪問。例如，它會阻止嘗試訪問未打開的端口或危險的網絡服務。
4. **圖形用戶界面支持**：Firewalld 支持圖形用戶界面工具，如 firewall-config，使用戶可以透過可視化界面輕鬆配置防火牆規則。
5. **簡單的命令行接口**：Firewalld 還提供了命令行工具，允許用戶使用簡單的命令來配置防火牆，這使得自動化和腳本化配置成為可能。

### 概念
##### Rule
防火牆規則是一組設置，它們決定了哪些數據包可以通過防火牆，哪些被阻止。這些規則通常基於源IP地址、目標IP地址、端口、協議等條件來定義。
* 源 (Source)：源是指數據包的來源，通常使用 IP 地址來識別。你可以基於源 IP 地址來設置防火牆規則，例如，允許或拒絕特定源 IP 地址的數據包。
* 目標 (Destination)：目標是指數據包的目標，通常使用目標 IP 地址來識別。你可以基於目標 IP 地址來設置防火牆規則，以控制數據包的流向。
* 協議 (Protocol)：協議指的是數據包使用的通信協議，例如 TCP、UDP 或 ICMP。你可以根據協議類型來設置防火牆規則，以允許或拒絕特定類型的數據包。
* 動作 (Action)：在防火牆規則中，你需要指定一個動作，即允許 (Allow) 或拒絕 (Deny) 數據包。如果規則條件符合，則執行相應的動作。
* 匹配 (Match)：匹配是指當一個數據包到達防火牆時，防火牆通過檢查數據包的特定屬性（例如源 IP 地址、目標端口、協議等）來確定是否應用特定的防火牆規則。
* 連接追蹤 (Connection Tracking)：防火牆可以跟蹤數據包的連接狀態，以確保返回數據包的正確路徑。這對於維護連接的狀態非常重要，特別是對於協議如 TCP，它需要跟蹤連接的狀態。

##### Zone
區域是 firewalld 中的一個重要概念，它代表**不同的網絡環境**或安全區域。每個區域**有不同的防火牆規則集**，用於控制該區域的流量。firewalld 預先定義了一些常見的區域，如：
* public（公共）：用於不受信任的公共網絡，通常有最嚴格的安全設置。
* internal（內部）：用於內部網絡，通常有較寬鬆的安全設置。
* trusted（信任）：用於受信任的網絡，通常允許最多的流量。
##### Service
服務是一個定義了應用程序或服務所使用的端口和協議的命名集合。可以通過指定服務來簡化防火牆規則的設置，而不必手動指定端口和協議。

> Zone 與 Service 是用來簡化 nftables 中繁雜的端口、協議細節設定。


### 安裝與啓用
```shell
sudo pacman -S firewalld
systemctl enable firewalld
systemctl start firewall
```

### CLI 使用
```shell
# 防火牆狀態
sudo firewall-cmd --state
# 取得當前區域
sudo firewall-cmd --get-active-zones

# 添加允許某個端口的規則
# e.g. 允許SSH連接，SSH默認使用22端口
sudo firewall-cmd --zone=public --add-port=22/tcp --permanent

# 添加允許某個服務的規則到某個 zone
sudo firewall-cmd --zone=public --add-service=http --permanent

# 移除規則
sudo firewall-cmd --zone=public --remove-port=22/tcp --permanent
# 移除服務
sudo firewall-cmd --zone=public --remove-service=http --permanent

# 更換 zone (注意 zone 是綁在網卡的) 
firewall-cmd --zone=dmz --change-interface=eth0

# 更新防火牆
sudo firewall-cmd --reload

# 查詢指定區域的所有規則
sudo firewall-cmd --zone=public --list-all

# 查詢指定區域的所有規則
sudo firewall-cmd --list-all-zones
```

### 自定義服務
* 要將服務存放在 `/etc/firewalld/services/` 目錄下 
1. 建立 /etc/firewall/services/my-custom-svc.xml
2. 寫入規則
```xml
<service>
    <short>My Custom Service</short>
    <description>A custom service for my application</description>
    <!-- 我的 service 運行的端口與協議 -->
    <port protocol="tcp" port="8080"/>
    <port protocol="udp" port="1234"/>

		<!-- Rule 1: Allow traffic from specific source IP address -->
    <rule family="ipv4">
        <source address="192.168.1.100"/>
        <accept/>
    </rule>

    <!-- Rule 2: Block traffic to a specific destination IP address -->
    <rule family="ipv4">
        <destination address="10.0.0.5"/>
        <drop/>
    </rule>

    <!-- Rule 3: Allow traffic on TCP port 8080 -->
    <rule family="ipv4">
        <port protocol="tcp" port="8080"/>
        <accept/>
    </rule>

    <!-- Rule 4: Log all traffic on UDP port 1234 -->
    <rule family="ipv4">
        <port protocol="udp" port="1234"/>
        <log/>
    </rule>

    <!-- Rule 5: Limit the rate of incoming traffic -->
    <rule family="ipv4">
        <limit value="5/m" burst="10"/>
        <accept/>
    </rule>

    <!-- Rule 6: Audit matching packets -->
    <rule family="ipv4">
        <audit/>
        <accept/>
    </rule>
</service>
```
3. 啓用服務
```shell
sudo firewall-cmd --zone=public --add-service=my-custom-svc --permanent
sudo firewall-cmd --reload
```

#### 各種規則
* source 和 destination：用來限制來自特定源IP地址或訪問特定目標IP地址的數據包。
* port 和 protocol：你可以使用 port 屬性指定規則的目標端口，使用 protocol 屬性指定協議（例如，TCP、UDP）。這可以用來限制特定端口上的數據包。
* accept 和 drop：你可以使用 <accept/> 標記來允許匹配的數據包通過，使用 <drop/> 標記來拒絕匹配的數據包。
* limit：你可以使用 limit 屬性來限制某個時間段內匹配數據包的數量。這可以用於防止洪水攻擊。
* log：你可以使用 log 屬性來啟用日誌記錄，以便記錄匹配的數據包。
* audit：你可以使用 audit 屬性來將匹配的數據包報告給 SELinux，以便進一步的安全審核。
* masquerade 和 forward-port：這些屬性用於配置NAT（Network Address Translation）規則，允許數據包在防火牆上進行地址轉換。
* ipv4 和 ipv6：你可以使用 ipv4 和 ipv6 屬性來指定規則適用於IPv4或IPv6數據包。

