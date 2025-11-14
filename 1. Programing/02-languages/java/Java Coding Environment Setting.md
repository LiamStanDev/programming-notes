# JDK 與 Maven
---
### JDK 
#### 下載
JDK有很多版本，我這邊使用免費的openJDK
source: https://jdk.java.net/archive/
```shell
# 下載 openJDK 17
cd ~/.local
wget https://download.java.net/java/GA/jdk17.0.2/dfd4a8d0985749f896bed50d7138ee7f/8/GPL/openjdk-17.0.2_linux-x64_bin.tar.gz
tar -xzvf openjdk-17.0.2_linux-x64_bin.tar.gz
mv openjdk-17.0.2_linux-x64_bin openjdk-17.0.2
```
#### 環境變量
```shell
export JAVA_HOME="$HOME/.local/openjdk-17.0.2"
export PATH="$PATH:$JAVA_HOME/bin"
```

### Maven
source: https://maven.apache.org/download.cgi
#### 下載
```shell
cd ~/.local
wget https://dlcdn.apache.org/maven/maven-3/3.9.4/binaries/apache-maven-3.9.4-bin.tar.gz
tar -xzvf apache-maven-3.9.4-bin.tar.gz
mv apache-maven-3.9.4-bin maven
```
#### 環境變量
```shell
export MAVEN_HOME="$HOME/.local/maven"
export PATH="$PATH:$MAVEN_HOME/bin"
```


# Intellij + WSL 配置
---
### Requirement
1. 不要在windows系統中配置Maven環境到環境變量中
> 因為不知道為甚麼 wsl 中 `maven --version` 會將　 MAVEN_HOME　設定為windows　中的

### Steps
1. 進入主菜單設置。
![[Screenshot 2023-08-21 022011.png]]
2. 將Maven配置成wsl中的路徑，很重要不然會indexing很久(因為會在windows與wsl有大量文件拷貝)
![[Screenshot 2023-08-21 022133.png]]

# Intellij　使用與插件
---
### 插件
1. One Dark Theme

### 快捷鍵
1. 自動補全：Alt + Inster
2. 格式化代碼：Ctrl + Alt + L
3. 查找文件：Shift + Shift

