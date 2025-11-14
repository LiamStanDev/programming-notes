# 架構
---
### Natives Modules
由 JS 實現提供應用程序可以直接調用的庫，如 fs、path、http 等，但 JS 語言是無法直接操作底層硬件的，故 nodejs 是透過 C++ 編寫的模塊來操作，如下圖所示：
![[Pasted image 20240820183847.png|800]]
* V8: 提供 JS 的運行環境
* libuv: 事件循環、事件隊列、異步 IO
* 第三方功能模塊：http、zlib
> 以上表示 Nodejs 架構而不是 JS 本身

# Nodejs 異步

### libuv 庫
![[Pasted image 20240820184400.png|800]]
libuv 庫作為異步操作的抽象層，在 linux 會使用 epoll、windows 下會使用 IOCP 等，會依照平台使用對應的異步方法。

### 事件驅動
