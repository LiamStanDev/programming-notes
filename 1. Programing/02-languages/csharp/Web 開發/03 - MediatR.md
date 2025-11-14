# 介紹
---
我們在實現 CORS 時會將 Service 區分成 QueryService 與 CommandService，在依賴注入時就需要注入很多服務，這樣並不能實現 OCP 原則，故我們需要在 API 層與 Application 層中使用中介模式來實現解偶，這就是 MediatR 要解決的問題。

* 原本
![[Pasted image 20240514093312.png]]
```c#
public UserController(ICustomerCommand customerCommand, ICustomerQuery customerQuery)
{
	_customerCommand = customerCommand;
	_customerQuery = customerQuery;
}
```

* 使用 MediatR
![[Pasted image 20240514093322.png]]
```c#
public UserController(IMediatR mediator) 
{
	_mediator = mediator;
}
```
> 若只使用 `Send` 方法可以使用 `ISender` 取代 `IMediatR`


# 使用
---
### 安裝
* 在 nuget 中添加 `MediatR` 包

#### 添加進 IServiceCollection
```c#
services.AddMediatR(cfg => services.AddMediatR(cfg =>
	cfg.RegisterServicesFromAssemblies(AppDomain.CurrentDomain.GetAssemblies())
);
```
會註冊以下:
- `IMediator` as transient
- `ISender` as transient
- `IPublisher` as transient
- `IRequestHandler<,>` concrete implementations as transient
- `IRequestHandler<>` concrete implementations as transient
- `INotificationHandler<>` concrete implementations as transient
- `IStreamRequestHandler<>` concrete implementations as transient
- `IRequestExceptionHandler<,,>` concrete implementations as transient
- `IRequestExceptionAction<,>)` concrete implementations as transient

### 我在抽象類 Controller 中的使用
```c#
[ApiController]
public class BaseApiController : ControllerBase
{
	private ISender _sender = null!;
	protected ISender Sender => _sender 
		??= HttpContext.RequestServices.GetRequiredSerivce<ISender>();
}
```


### 模式一: Request/response messages
表示只會有一個 handler 處理。其中又分為 Request 與 Streaming 兩種處理方式。

#### Request 處理方式
##### Request Types
* `IRequest<TResponse>`: 有返回值的請求
* `IRequest`: 沒有返回值的請求

```c#
// LoginQuery.cs
// Request
public record LoginQuery(
    string Email,
    string Password
) : IRequest<ErrorOr<AuthenticationResult>>;


// LoginQueryHandler.cs
// RequestHandler
public class LoginQueryHandler : IRequestHandler<LoginQuery, ErrorOr<AuthenticationResult>>
{
    private readonly IJwtTokenGenerator _jwtTokenGenerator;
    private readonly IUserRepository _userRepository;

    public LoginQueryHandler(IJwtTokenGenerator jwtTokenGenerator, IUserRepository userRepository)
    {
        _jwtTokenGenerator = jwtTokenGenerator;
        _userRepository = userRepository;
    }
    public async Task<ErrorOr<AuthenticationResult>> Handle(LoginQuery query, CancellationToken cancellationToken)
    {
        if (_userRepository.GetUser(query.Email) is not User user)
        {
            return Errors.Authentication.InvalidCredentials;
        }

        if (user.Password != query.Password)
        {
            return Errors.Authentication.InvalidCredentials;
        }

        string token = _jwtTokenGenerator.GenerateToken(user);
        return new AuthenticationResult(user, token);
    }
}


// API 中使用
AuthenticationResult res = Sender.Send(new LoginQuery { Email = "geffc1454@gmail.com" Password = "Pa$$w0rd" });
```


#### Streaming 處理方式
當處理是一個 Enumerable 時，可以使用 Streaming 的方式逐一處理，但我還想不到甚麼場景可以使用， skip...

### 模式二: Notification messages
表示可以有多個 handler 處理。


```c#
// NotificationRequest
public record Ping: INotification  // 沒有泛型，Notifaction 沒有返回值
{}

// 可以建立多個 INotificationHander 監聽 Ping
public class Pong : INotificationHandler<Ping>
{
    public Task Handle(Ping notification, CancellationToken cancellationToken)
    {
        Debug.WriteLine("Pong");
        return Task.CompletedTask;
    }
}


// 在 API 中使用記得添加 IPublish
await Publisher.Publish(new Ping());
```