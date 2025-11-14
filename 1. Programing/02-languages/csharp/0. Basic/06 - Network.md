# Socket
---
建議複習 [[Function for Web|C++ Web]] 知識，流程與其相同。[[Computer Networking#TCP|TCP 基礎]]也可以看一下。

### TCP 實作
* AddressFamily: 有很多選項，ipv4 使用 InterNetwork, ipv6 選擇 InterNetworkV6
* SocketType: 有很多選項 Tcp 選擇 Stream， Udp 選擇 Datagram
#### Server Side 
流程：
1. 建立 Socket 實例
2. Bind 
3. Listen: 開啟 connection queue，客戶端可以開始連接
4. Accept: 從 connection queue 取出連接 handler
```c#
// 設定 ip 與端口
IPAddress address = IPAddress.Parse("127.0.0.1");
int port = 5567;
IPEndPoint endPoint = new IPEndPoint(address, port);
// 建立 socket 實例
using Socket socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
// Bind
socket.Bind(endPoint);
// Listen
socket.Listen(10); // 設定connection queue 連接數
Console.WriteLine("Waiting for a connection...");
Socket handler = null;
do {
	// Accept
    using handler = await socket.AcceptAsync();

    byte[] buffer = new byte[1024];
    int length = await handler.ReceiveAsync(buffer);

    string dataReceive = Encoding.UTF8.GetString(buffer, 0, length);
    Console.WriteLine($"Receive: {dataReceive}");

    string dataSend = "Hello from Server";
    byte[] bytes = Encoding.UTF8.GetBytes(dataSend);

    int result = await handler.SendAsync(bytes);
    Console.WriteLine($"Result: {result}");
} while (handler != null);
```
#### Client Side
流程：
1. 建立 Socket 實例
2. Connect
3. Send
4. Receive
```c#
using Socket socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);

IPAddress address = IPAddress.Parse("127.0.0.1");
int port = 5567;

IPEndPoint endPoint = new IPEndPoint(address, port);

await socket.ConnectAsync(endPoint);
Console.WriteLine("Connected to the server.");

string data = "Hello from Client";
byte[] bytesRend = Encoding.UTF8.GetBytes(data);

int result = await socket.SendAsync(bytesRend);
Console.WriteLine($"Result: {result}");


byte[] buffer = new byte[1024];
int length = await socket.ReceiveAsync(buffer);
string dataReceive = Encoding.UTF8.GetString(buffer, 0, length);
Console.WriteLine($"Received: {dataReceive}");
```


# TcpClient & TcpListener
---
相較於使用 Socket 從頭實作，使用更高抽象的 `TcpListner` 與 `TcpClient` 類，可以更簡單的實作 TCP 網路傳輸。
不同點：
* Accept 的是 TcpClient: 不是 Socket，隨然也可以
* 使用 NetworkStream 進行交互：不使用 Socket 的 Receive 與 Send
* Listen -> Start
* Send -> Write
* Receive -> Read
### Server Side
```c#
IPAddress address = IPAddress.Parse("127.0.0.1");
int port = 5543;
IPEndPoint endPoint = new IPEndPoint(address, port);

using TcpListener listener = new TcpListener(endPoint);

listener.Start(10);
Console.WriteLine("Waiting for connection...");

TcpClient client = null;

do {
    client = await listener.AcceptTcpClientAsync();
    Console.WriteLine("Server connected");
    NetworkStream stream = client.GetStream();
    // Receive
    byte[] buffer = new byte[1024];
    int length = await stream.ReadAsync(buffer);
    string dataReceive = Encoding.UTF8.GetString(buffer, 0, length);
    Console.WriteLine($"Receive: {dataReceive}");

    // Send
    string dataSend = "Hello from Server";
    byte[] bytes = Encoding.UTF8.GetBytes(dataSend);
    await stream.WriteAsync(bytes, 0, bytes.Length);

} while (client != null);
```

### Client Side
```c#
IPAddress address = IPAddress.Parse("127.0.0.1");
int port = 5543;
IPEndPoint endPoint = new IPEndPoint(address, port);

TcpClient client = new TcpClient();

await client.ConnectAsync(address, port);

NetworkStream stream = client.GetStream();
// Send 
string dataSend = "Hello from Client";
byte[] bytes = Encoding.UTF8.GetBytes(dataSend);
await stream.WriteAsync(bytes, 0, bytes.Length);

// Receive
byte[] buffer = new byte[1024];
int lenght = await stream.ReadAsync(buffer);
string dataReceive = Encoding.UTF8.GetString(buffer, 0, lenght);
Console.WriteLine($"Receive: {dataReceive}");

client.Close();

```


# HTTP
---
HTTP 是基於 TCP 之上的更高級別的協議，在 TCP 基礎上添加網路請求與響應的規則和結構。.NET 原生提供 `HTTPClient` 與 `HTTPListener` 兩個類，但 Server 建議使用更高級的 ASP.NET Core 其擁有更豐富完備的功能 。

### Server
```c#
HttpListener listener = new HttpListener();

listener.Prefixes.Add("http://localhost:8080/");

listener.Start();
Console.WriteLine("Listening...");

// 取得連接上下文，很像取得 tcpclinet or handler
HttpListenerContext? context = null;

do {
    context = await listener.GetContextAsync();
    Console.WriteLine("Client connected");
    if (context.Request.Url?.AbsolutePath == "/Hi") {

        string responseString = "<HTML><BODY> Hello world!</BODY></HTML>";
        byte[] bytes = Encoding.UTF8.GetBytes(responseString);
        HttpListenerResponse response = context.Response;
        response.ContentLength64 = bytes.Length;
        response.StatusCode = 200;
        Stream stream = response.OutputStream;
        await stream.WriteAsync(bytes, 0, bytes.Length);
    }
} while (context != null);
```

### Client
```c#
HttpClient client = new HttpClient();

HttpResponseMessage response = await client.GetAsync("http://localhost:8080/Hi");
string content = await response.Content.ReadAsStringAsync();
Console.WriteLine(response.StatusCode);
Console.WriteLine("Content: ");
Console.WriteLine(content);
```
> 裡面有 client 中有 Get, Post, Put 等方法 

# APS.Net Core Minimal 
---
不基於 HttpListener，而是從頭搭建的，默認使用 kestrel 作為 Web 服務器，也就是 ASP.Net 使用 Kestrel 的代碼庫，提供更多高級的功能 e.g. RESTful API, MVC, Middleware, DI 等。性能表現相較 HTTPListner 高。

Todo...