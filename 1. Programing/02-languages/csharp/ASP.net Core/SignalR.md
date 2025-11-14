# Intro
---
SignalR 用於在 ASP.NET 應用中開發 real-time 功能的庫，提供簡單地 API 用於建立 server-to-client RPC 來調用 browser 的 javascript 庫或者其他 .NET 程序。提供 connectoin management, grouping connection 等功能。

### WebSocket and fallbacks
SignalR 會先使用 Websocket 進行連接，若不行就會退回到 long polling。
使用 websocket 的好處如下:
* The most efficient use of server memory.
* The lowest latency.
* The most underlying features, such as full duplex communication between client and server.

### Models for communicatoin
#### Model 1: Persistent Connection
表示客戶端和伺服器之間的通訊通道。允許實時雙向通訊，客戶端和伺服器之間可以傳送消息。Connection 可以用於傳送給單一收件人、分組的收件人或廣播消息。使用 Persistent Connection API 可以更細致地控制通訊的低層細節

#### Model 2: Hubs
Hub 是建立在 Connection API 之上的更高層次的通道。它**允許客戶端和伺服器直接呼方法來與對方進行通信**。Hub 提供了一個更抽象和易於使用的方式，讓客戶端和伺服器可以像呼叫本地方法一樣呼叫對方的方法。


# 實作
---
### Server
#### Config
* Program.cs
```cs
var builder = WebApplication.CreateBuilder(args);

// 添加 signalR 中間件
builder.Services.AddSignalR();

...
// 路由到指定的 Hub
app.MapHub<ChatHub>("/chat");
```
#### Create and use hub
```cs
public class ChatHub : Hub 
{
		public ansyc Task SendMessage(string user, string message)
		{
			await Clients.All.SendAsync("ReceiveMessage", user, message);
		}
}
```
##### 注意
* Hub 是 transient scope，所以不要存放狀態在 Hub 中
* 不要實例化 Hub 類，請使用 `IHubContext` 進行操做

##### Client Object
* All: 該方法調用所有 clients 的
* Caller: 該方法調用請求該 hub 的 client
* Others
### Client

