# Serilog
---
### 簡介
Serilog 提供檔案與控制台與其他地方的日誌紀錄，具有簡單易用的特性受 .Net 社區所喜愛。提供純文字、json 與 XML 幾種輸出格式。
> 另外一個原因是 Microsoft.Extensions.Logging 沒有文件日誌功能。

提供以下幾種 Sinks (Destinations):
* [Serilog.Sinks.File](https://www.nuget.org/packages/Serilog.Sinks.File/5.0.1-dev-00972)
* [Serilog.Sinks.Console](https://www.nuget.org/packages/Serilog.Sinks.Console/5.1.0-dev-00943)
* [Serilog.Sinks.Debug](https://www.nuget.org/packages/Serilog.Sinks.Debug)
* [Serilog.Sinks.Elasticsearch](https://www.nuget.org/packages/Serilog.Sinks.Elasticsearch)
* [Serilog.Sinks.MSSqlServer](https://www.nuget.org/packages/Serilog.Sinks.MSSqlServer/6.6.1-dev-00077)
* 等等

### 安裝
* 添加 Serilog
* 添加想要使用的 Sink
```shell
dotnet add package Serilog
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
```

### 在 Console 中使用
```c#
static async Task Main()
{
	Log.Logger = new LoggerConfiguration()
		.MinimumLevel.Debug()
		.WriteTo.Console()
		.WriteTo.File("logs/myapp.txt", rollingInterval: RollingInterval.Day)
		.CreateLogger();

	Log.Information("Hello, world!");

	int a = 10, b = 0;
	try
	{
		Log.Debug("Dividing {A} by {B}", a, b);
		Console.WriteLine(a / b);
	}
	catch (Exception ex)
	{
		Log.Error(ex, "Something went wrong");
	}
	finally
	{
		await Log.CloseAndFlushAsync();
	}
}
```

### 在 ASP.net 中使用 (重點)
我們在 API 中多添加 `Serilog.AspNetCore`。
然後在 Program.cs 中配置
```c#
// 這是 bootstrap logger，他只會在項目啟動時使用，之後會被替換
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateBootstrapLogger();

Log.Information("Starting up!");

try
{
    var builder = WebApplication.CreateBuilder(args);
	builder.Services.AddSerilog(logCfg => logCfg
		.ReadFrom.Configuration(builder.Configuration) // 讀取配置文件
	    .Enrich.FromLogContext()
	);

	// ...

    var app = builder.Build();
    
	// 這會取代框架自帶的 Logger，用於 log Request 相關的資訊
    app.UseSerilogRequestLogging();
    
    app.UseAuthorization();

	// ...
}
catch (Exception ex)
{
    Log.Fatal(ex, "An unhandled exception occurred during bootstrapping");
}
finally
{
    Log.CloseAndFlush();
}
```


在 AppSetting.json 中加上以下配置
```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore.Mvc": "Warning",
        "Microsoft.AspNetCore.Routing": "Warning",
        "Microsoft.AspNetCore.Hosting": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console"
      },
      {
        "Name": "File",
        "Args": {
          "path": "Logs/applog-.txt", // 會在啟動項目的根目錄，不會在 bin 目錄
          "rollingInterval": "Day"
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName"],
    "Properties": {
      "ApplicationName": "Your ASP.NET Core App"
    }
  }
}
```

在其他 Layer 需要先添加 `Microsoft.Extensions.Logging.Abstraction` 包，就能夠使用 `ILogger<T>` 構造器注入 serilog 了。

