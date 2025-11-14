在 ASP.NET Core 中，ModelState 是一個包含所有提交的表單值以及與這些值相關的驗證信息的集合，爲 `ControllerBase` 的一個屬性，因此用於所有 Controller 中。每當用戶提交表單數據，`ModelState` 就會被填充和更新。
### 功能
1. 存放表單所有字段值
2. 驗證信息：當使用數據註解（如 `[Required]`、`[Range]` 等）來裝飾模型屬性時，ASP.NET Core 會自動驗證這些字段並將任何驗證錯誤添加到 `ModelState` 中。
### 使用
#### 驗證是否合法並添加自定義錯誤信息
```csharp
public ActionResult GetValidationError()
{
	if (!ModelState.IsValid) 
	{
		ModelState.AddModelError("Problem1", "this is the fisrt problem");
		ModelState.AddModelError("Problem2", "this is the second problem");
	}
		return ValidationProblem();
}
```






