### 簡介
---
CORS 政策為瀏覽器為避免非同源網站殖入惡意的 JS 程序，對服務器發起惡意的請求而收到惡意的資源，所以要求服務器要標明對於不同的域 (domain) 能執行什麼方法 (GET, POST...)。

重點：
* CORS 限制的是瀏覽器的 JS 代碼能否接收跨域的資源。

### 細節
---
1. 前端發起請求時，瀏覽器會先發送一個**預檢請求** (使用 OPTIONS 方法)，根據服務器的返回來得知是否允許跨域訪問。
2. 預檢請求有 Access-Control-Max-Age，並且會在瀏覽器進行緩存，並不會每次訪問都進行
3. 服務器開啟允許跨域訪問，但 JS 代碼還是無法取得該響應的 Header 全部內容，只能取得 body，若希望某些 Header 可以被取得，要在響應的首部添加 "Access-Control-Expose-Headers"，後面添加可以接收的首部

### 流程
---
#### 後端
* Program.cs
```cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddCors(); // 添加服務

var app = builder.Build();
app.UseCors(policy => {
	policy.AllowAnyHeader() // 不指定頭部需要什麼，與 Access-Control-Expose-Header 無關
	.AllowAnyMethod()
	.AllowCredentials() // 使 cookie 可以 pass 到前端，反之亦然
	.WithOrigins("localhost:3000");
});
```

