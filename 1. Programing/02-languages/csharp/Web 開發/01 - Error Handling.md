# Global
---
Global 錯誤處理是用於處理一些下層未處裡的 Exception，我們需要將他打印成 ProblemDetail 的形式提供給調用者知道。

在 `API/Middleware/ExceptionMiddleware.cs` 中建立
```c#
public class ExceptionMiddleware
{
	private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionMiddleware> _logger;
    private readonly IHostEnvironment _env;
    
    public ExceptionMiddleware(
        RequestDelegate next, // receive a HttpContext return a Task. Use to pass to the next middleware.
        ILogger<ExceptionMiddleware> logger,
        IHostEnvironment env
    )
    {
        _next = next;
        _logger = logger;
        _env = env;
    }

	public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, ex.Message);
            context.Response.ContentType = "application/json";
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            
            var prob = new ProblemDetails()
			{
				Status = StatusCodes.Status500InternalServerError,
				Title = ex.Message,
				Detail = _env.IsDevelopment() ? ex.StackTrace?.ToString() : null,
			};
			
            var options = new JsonSerializerOptions
            {
                PropertyNamingPolicy = JsonNamingPolicy.CamelCase
            };
            
            string json = JsonSerializer.Serialize(response, options);
            
            await context.Response.WriteAsync(json);
        }
    }

}
```

# Domain Error
---
若我們都使用 Exception 來處理錯誤有以下幾個問題:
1. 開銷較大
2. 只能有一個 Exception 類型
3. 調用者不一定要處理
所以使用像是 Rust 一樣的 Result Enum 是一個很好的方案，而社區提供了一個 C# 的實現叫做 [ErrorOr](https://github.com/amantinband/error-or)，我覺得非常好用!

### 基本
##### Error Type
Error 將錯誤分為以下幾類:
```c#
public enum ErrorType
{
    Failure,
    Unexpected,
    Validation,
    Conflict,
    NotFound,
    Unauthorized,
    Forbidden,
}
```
* Constructor 參數有:
	* `code`: 用於定位錯誤類型
	* `description`: 描述細節
	* `metaData` (optional): 為 `Dictionary<string, object>` 存放一些想傳遞的訊息
也有自定義的錯誤
```c#
var error = Error.Custom(
    type: MyErrorTypes.ShouldNeverHappen, // 多了一個 Type
    code: "User.ShouldNeverHappen",
    description: "A user error that should never happen"
);
```

##### 多 Error 返回
```c#
public class User(string _name)
{
    public static ErrorOr<User> Create(string name)
    {
        List<Error> errors = []; // 存放 Error

        if (name.Length < 2)
        {
	        // 添加
            errors.Add(Error.Validation(description: "Name is too short"));
        }

        if (name.Length > 100)
        {
	        // 添加
            errors.Add(Error.Validation(description: "Name is too long"));
        }

        if (errors.Count > 0)
        {
            return errors; // 可以很方便直接返回
        }

        return new User(name);
    }
}
```

##### `ErrorOr<T>` 中實用的 Properties
* `IsError`: 確認是否有 Error
* `Value`: 返回成功的 Result，沒有會返回 null，記得先確認 IsError
* `Errors`: 返回 `List<Error>` 同上，記得先確認 IsError

##### `ErrorOr<T>` 中實用的方法
注意都有 Async 版本
###### Match
```c#
string foo = result.Match(
    value => value,
    errors => $"{errors.Count} errors occurred.");
```

###### MatchFirst
```c#
string foo = result.MatchFirst(
    value => value,
    firstError => firstError.Description);
```

###### Then
當為正確的時候做進一步處理，返回值仍然為 ErrorOr
```c#
ErrorOr<int> foo = result
    .Then(val => val * 2); // val 為正確的值
```

###### Else
同 Then 但用於處理錯誤，
```c#
ErrorOr<string> foo = result
    .Else(errors => $"{errors.Count} errors occurred.");
```

### 項目中使用
##### Step1 : 建立錯誤
在 `Domain/Common/Errors/Errors.Authentication.cs` 中建立
```c#
public static partial class Errors
{
    public static class User
    {
        public static Error DuplicatedEmail => Error.Conflict(
            code: "User.DuplicateEmail",
            description: "Email is already in use."
        );
    }
}
```
> 你也可以根據錯誤類型放在 Application Layer 中

##### Step2: 在 Handler 中觸發錯誤 (MediatR)
```c#
// 返回值被包裹在 ErrorOr
public async Task<ErrorOr<AuthenticationResult>> Handle(RegisterCommand command, CancellationToken cancellationToken)
    {
        // Check user exist
        if (_userRepository.GetUser(command.Email) is not null)
        {
            return Errors.User.DuplicatedEmail;
        }

		// ...
    }
```


##### Step3: 在 API 中取值並處理
```c#
[HttpPost("register")]
public async Task<IActionResult> Register(RegisterRequest request)
{
	RegisterCommand command = new(request.FirstName, request.LastName, request.Email, request.Password);
	// 這邊會返回 ErrorOr<T>
	ErrorOr<AuthenticationResult> authResult = await Sender.Send(command);

	// 要使用就必須進行處理
	return authResult.Match( // 使用 Match 進行處理
		authResult => Ok(MapAuthResult(authResult)), // 這邊是成功的返回值
		errors => Problem(errors) // errors 為 List<Error> 類型，這邊將他放到 Probem 中，你也可以迭帶它
	);
}
```

