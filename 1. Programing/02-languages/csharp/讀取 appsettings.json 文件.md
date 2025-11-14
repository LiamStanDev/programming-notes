### 基本
* 添加項目依賴
```text
Microsoft.Extensions.Configuration -- 基礎包
Microsoft.Extensions.Configuration.Json -- 用於解析 json
Microsoft.Extensions.Configuration.Binder -- 可以將 Section 使用 Get 解析出類 (Optional)
Microsoft.Extensions.Configuration.Abstractions -- 只包含抽象，用於與其他接口進行交互 (Optional)
Microsoft.Extensions.Options.ConfigurationExtensions -- 用於依賴注入 (Optional)
```

* 建立 appsettings.json 文件
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "MySettings": {
    "Setting1": "Value1",
    "Setting2": "Value2"
  }
}
```

* 配置 .csproj 將 appsettings.json 拷貝到輸出目錄
```html
<ItemGroup>
  <Content Include="appsettings.json">
    <CopyToOutputDirectory>Always</CopyToOutputDirectory>
  </Content>
</ItemGroup>
```

* 在 Program.cs 中讀取
```cs
var builder = new ConfigurationBuilder()
			.SetBasePath(AppContext.BaseDirectory)
           .AddJsonFile("appsettings.json", optional: false);  // optional 為 false 表示若沒有找到 appsettings.json 會拋錯
           
IConfiguration configuration = builder.Build();

// 讀取值
string? setting1 = configuration["MySettings:Settkng1"];
string? setting1 = configuration.GetValue<string>("MySettings:Setting1");

// 讀取嵌套
LoggingConfig? loggingConfig = configuration.GetSection("Logging").Get<LoggingConfig>(); // 使用　Get 需要依賴 Binder 包
```


### Option 包使用方式
用於解析 configuration 成為一個實例後可以加到依賴注入中
```text
Microsoft.Extensions.Options.ConfigurationExtensions
```

* 添加到依賴注入
```cs
services.Configure<MySettings>(Configuration.GetSection("MySettings"));
```

* 使用服務
```cs
public class MyClass
{
    private readonly MySettings _settings; // 需要 MySetting

    public MyClass(IOptions<MySettings> settings) // 構造器依賴注入
    {
        _settings = settings.Value; // 使用 Value 取值，可能不存在
    }

    public void SomeMethod()
    {
        Console.WriteLine($"Setting1: {_settings.Setting1}");
        Console.WriteLine($"Setting2: {_settings.Setting2}");
    }
}
```