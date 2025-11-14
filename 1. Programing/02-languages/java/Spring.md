# 基本概念
---
### Servlet
Servlet 運行在 Web 伺服器或應用伺服器上的 Java 程序，它可以**接收**客戶端的請求，**處理**這些請求，並產生和**返回**響應給客戶端。每當有新的請求到達 Servlet，容器可能會為該請求**啟動一個新的線程**(相較於以前開啟新進程的方式更低的消耗)。
### Tomcat
Tomcat 是一個流行的開源 Java Servlet 容器，Tomcat 提供了一個運行環境，讓開發者能夠執行基於 Servlet 的 Web 應用程序。
1. 當一個 HTTP 請求到達 Tomcat，它將該請求路由到相應的 Servlet 進行處理
2. Tomcat 管理 Servlet 的生命週期
3. 角色的訪問控制、HTTPS 支持、認證和授權
4. 在一個 Tomcat 實例上部署多個 Web 應用程序
5. Tomcat 提供了資源和連接池管理功能

### 請求到響應流程
![[Screenshot 2023-08-18 132156.png]]
* Thyemeleaf: 用來將數據轉換成HTML

### Maven (Gradle) Standard Directory Layout
|  dir  |  details |
|---|---|
|**`src/main/java`**|Application/Library sources|
|`src/main/resources`|Application/Library resources|
|`src/main/filters`|Resource filter files|
|`src/main/webapp`|Web application sources|
|`src/test/java`|Test sources|
|`src/test/resources`|Test resources|
|`src/test/filters`|Test resource filter files|
|`src/it`|Integration Tests (primarily for plugins)|
|`src/assembly`|Assembly descriptors|
|`src/site`|Site|
|`LICENSE.txt`|Project's license|
|`NOTICE.txt`|Notices and attributions required by libraries that the project depends on|
|`README.txt`|Project's readme|
