# 地址
---
## 種類
### sockaddr
為舊式的地址結構體，大小與`socketaddr_in` 與`socketaddr_in6` 相同，可以直接轉換，所以所有的網路函數都是使用其作為參數。大小為16 bytes
#### 結構
```cpp
struct socketaddr {
	unsigned short sa_family; // 地址種類 : socket address family
	char sa_data[14]; // 用14 bytes的端口與地址
};
```
#### 地址類型
* `AF_INET` : ipv4，佔用4個bytes
* `AF_INET6` : ipv6, 佔用16個bytes
> 端口佔用2bytes

我們不會直接操作socketaddr而是會使用以下兩者。

### sockaddr_in
為ipv4地址結構體，大小也為16bytes。
#### 結構
```cpp
struct sockaddr_in {
	short int sin_family; // 地址種類
	unsigned short int sin_port; // 端口
	struct in_addr sin_addr; // 地址
	unsigned char sin_zero[8]; // 對齊
};
```
##### sin_addr定義
```cpp
struct in_addr { // in表示ipv4, in6表示ipv6 
	unsigned long s_addr; // 地址 
};
```

### socketaddr_in6
為ipv6地址結構體，大小為28 bytes。
##### 為什麼可以比socketaddr大，那這樣參數傳遞是否有問題？
ans: 還是可以傳遞，同樣是把socketaddr指針指向socketaddr_in6，但是要函數傳遞sizeof(socketaddr_in6)。如下
```cpp
bind(sockfd, reinterpret_cast<sockaddr*>(&addr6), sizeof(addr6));
```
#### 結構
```cpp
struct sockaddr_in6 { 
	u_int16_t sin6_family; // 地址族 
	u_int16_t sin6_port; // 端口号 
	u_int32_t sin6_flowinfo; // IPv6 流信息 
	struct in6_addr sin6_addr; // IPv6 地址 
	u_int32_t sin6_scope_id; // 范围 ID 
};
```

## 使用案例
### ipv4
```cpp
#inlcude <netinet/in.h> // sockaddr, sockaddr_in, sockaddr_in6
#include <arpa/inet.h> // htons, inet_addr
#include <sys/socket.h> // AF_INET
#include <cstring> // memset

int main() {
    sockaddr_in addr;
    // 清空
    memset(&addr, 0, sizeof(addr));
    // 指定地址
    addr.sin_family = AF_INET;
    // 指定端口
    addr.sin_port = htons(8080);  // host to net short
    // 地址
    // 方法一：
    // addr.sin_addr.s_addr = inet_addr("192.168.0.1");
    // 方法二：與ipv6相同
    inet_pton(AF_INET, "192.168.0.1", &(addr.sin_addr))
    // ... 完成初始化
}
```
> inet_pton : internet presentation to network
### ipv6
```cpp
#inlcude <netinet/in.h> // sockaddr, sockaddr_in, sockaddr_in6
#include <arpa/inet.h> // htons, inet_addr
#include <sys/socket.h> // AF_INET
#include <cstring> // memset

int main() {
    sockaddr_in6 addr;
    // 清空
    memset(&addr, 0, sizeof(addr));
    // 指定地址
    addr.sin_family = AF_INET;
    // 指定端口
    addr.sin_port = htons(8080);  // host to net short
    // 地址
    inet_pton(AF_INET6, "::1", &(addr.sin6_addr));
    // ... 完成初始化
}
```

# 本機
---
