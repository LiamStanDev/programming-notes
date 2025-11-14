source: https://docs.duendesoftware.com/identityserver/v7/quickstarts/
# Intro
---
Duende IdentityServer（前身為 IdentityServer4）是一個針對 .NET 開發的開源身份驗證和授權框架。它允許您在應用程序中實現標準的 OpenID Connect 和 OAuth 2.0 驗證協議，以便安全地保護您的應用程序和 API。

1. **OpenID Connect 和 OAuth 2.0：** Duende IdentityServer 支持 OpenID Connect 和 OAuth 2.0 標準，這兩者是用於身份驗證和授權的常見協議。
2. **支援各種客戶端：** 客戶端可以是 Web 應用程序、單頁應用程序（SPA）、原生移動應用程序、後端服務等，IdentityServer 都提供了適當的協議和流程。
3. **多種驗證方式：** 支援密碼、授權碼、隱式、混合等多種授權流程，以滿足不同應用程序和使用情境的需求。
4. **用戶和身份管理：** Duende IdentityServer 支援用戶和身份的管理，包括基本的用戶名密碼驗證、外部身份提供者（如 Google、Facebook 等）。
5. **客戶端和範圍管理：** 管理客戶端和範圍，定義授權策略和設置權限。
6. **自定義擴展點：** 提供了豐富的擴展點，使開發者可以根據實際需求自定義行為，例如自定義存儲、自定義登錄頁面等。
7. **強大的日誌和監視：** Duende IdentityServer 提供了詳細的日誌和監視功能，有助於追踪和調試身份驗證和授權流程。
8. **開源和社區支援：** Duende IdentityServer 是一個開源項目，它在 GitHub 上進行開發，並擁有一個活躍的社區，提供了廣泛的文檔和支援。

# Quick Start
---
### Stage 1: API Access (Client Credentials)
這個部分我們將 API 保護起來，客戶端需要向 Identity Server 去得 Access Token 才能訪問該 API，使用的方式爲 OAuth 2.0 中的 Client Credentials flow，這種方式就是發放 Access token 給註冊的客戶端。
#### 1. 下載 Duende 腳手架
```csharp
dotnet new install Duende.IdentityServer.Templates
```

#### 2. 建立 Project 
```csharp
mkdir -p quickstart/src
cd quickstart
dotnet new sln -n QuickStart
dotnet new isempty -o src/IdentityServer // 這不會提供 UI
dotnet new webapi -o src/Api --no-openapi // 需要保護的 API
dotnet new console -o src/Client // 請求受保護 API Client
dotnet sln add src/IdentityServer
dotnet sln add src/Api
dotnet sln add src/Client
```

#### 3. 建立 Api Scope
scope 是用於 OAuth 2.0 用來區分能授權的範圍，Client 會向 Identity Server 請求 scope，得到 access_token 後，在向壽保護的 API 驗證 access_token 與其中的 scope，來檢驗是否是授權的用戶。
```csharp
public static IEnumerable<ApiScope> ApiScopes =>
    new ApiScope[]
    {
        new ApiScope(name: "api1", displayName: "My API") 
    };
```
#### 4. 建立 Client
這邊只得是向 Identity Server 註冊用戶，該用戶可能爲 webapp, native app 等。需要有 ClientId 唯一識別用戶，與 Secret 來驗證是否是合法的用戶，還有該用戶允許申請的 scope。
```csharp
public static IEnumerable<Client> Clients =>
    new Client
    
    {
        new Client
        {
            ClientId = "client",

            // no interactive user, use the clientid/secret for authentication
            AllowedGrantTypes = GrantTypes.ClientCredentials,

            // secret for authentication
            ClientSecrets =
            {
                new Secret("secret".Sha256())
            },

            // scopes that client has access to
            AllowedScopes = { "api1" }
        }
    };
```

#### 5. 配置 Identity Server 
將 Config.cs 中的內容配置到 Ideneity Server 中。
```csharp
public static WebApplication ConfigureServices(this WebApplicationBuilder builder)
{
    builder.Services.AddIdentityServer()
        .AddInMemoryApiScopes(Config.ApiScopes) // 添加 ApiScope
        .AddInMemoryClients(Config.Clients); // 添加 Client

    return builder.Build();
}
```
運行之後就可以查看 ` https://localhost:5001/.well-known/openid-configuration` 這個頁面，這頁面也就是 Discovery Document，他是標準的 OAuth 2.0 與 OIDC 連接端點，裏面提供要向哪個 url 聲請 token 等等。 結果如下：
```json
{
   "issuer":"http://localhost:5001",
   "jwks_uri":"http://localhost:5001/.well-known/openid-configuration/jwks",
   "authorization_endpoint":"http://localhost:5001/connect/authorize",
   "token_endpoint":"http://localhost:5001/connect/token",
   "userinfo_endpoint":"http://localhost:5001/connect/userinfo",
   "end_session_endpoint":"http://localhost:5001/connect/endsession",
   "check_session_iframe":"http://localhost:5001/connect/checksession",
   "revocation_endpoint":"http://localhost:5001/connect/revocation",
   "introspection_endpoint":"http://localhost:5001/connect/introspect",
   "device_authorization_endpoint":"http://localhost:5001/connect/deviceauthorization",
   "backchannel_authentication_endpoint":"http://localhost:5001/connect/ciba",
   "pushed_authorization_request_endpoint":"http://localhost:5001/connect/par",
   "require_pushed_authorization_requests":false,
   "frontchannel_logout_supported":true,
   "frontchannel_logout_session_supported":true,
   "backchannel_logout_supported":true,
   "backchannel_logout_session_supported":true,
   "scopes_supported":[
      "openid",
      "profile",
      "api1",
      "offline_access"
   ],
   "claims_supported":[
      "sub",
      "name",
      "family_name",
      "given_name",
      "middle_name",
      "nickname",
      "preferred_username",
      "profile",
      "picture",
      "website",
      "gender",
      "birthdate",
      "zoneinfo",
      "locale",
      "updated_at"
   ],
   "grant_types_supported":[
      "authorization_code",
      "client_credentials",
      "refresh_token",
      "implicit",
      "password",
      "urn:ietf:params:oauth:grant-type:device_code",
      "urn:openid:params:grant-type:ciba"
   ],
}
```


#### 6. 保護 API
* 在 Api 項目中，添加 Microsoft.AspNetCore.Authentication.JwtBearer，有以下功能
	* 找到並解析 `Authorization: Bearer` header
	* 驗證 JWT 簽名，並驗證 issuer
	* 確保 JWT 沒有過期
* src/Api/Program.cs
```csharp
builder.Services.AddAuthentication()
    .AddJwtBearer(cfg =>
    {
        cfg.Authority = "http://localhost:5001"; // identity server
				cfg.TokenValidationParameters.ValidateAudience = false; // OAuth 不使用 Audience，而是使用 scope 來確認
        // 在測試中若使用issuer 使用 http，要使用以下屬性因爲 Authentication 爲確保安全默認使用 Https
				cfg.RequireHttpsMetadata = false;
    });
builder.Services.AddAuthorization();

...

app.MapGet("identity", (ClaimsPrincipal user) => user.Claims.Select(c => new { c.Type, c.Value }))
    .RequireAuthorization();
```

#### 7. 開始測試
* 在 Client 項目中，添加 IdentityModel，提供 OIDC 與 Auth 2.0 的客戶端功能。

* src/Client/Program.cs
```csharp
using IdentityModel.Client;

var client = new HttpClient();
// 取得 Discovery Document
var disco = await client.GetDiscoveryDocumentAsync("https://localhost:5001");
if (disco.IsError)
{
    Console.WriteLine(disco.Error);
    return;
}

// 使用 Client Credential 方式，取得 access token
var tokenResponse = await client.RequestClientCredentialsTokenAsync(new ClientCredentialsTokenRequest
{
    Address = disco.TokenEndpoint,
    ClientId = "client",
    ClientSecret = "secret",
    Scope = "api1" // 想要申請的 scope
});

if (tokenResponse.IsError)
{
    Console.WriteLine(tokenResponse.Error);
    Console.WriteLine(tokenResponse.ErrorDescription);
    return;
}

Console.WriteLine(tokenResponse.AccessToken);

// 訪問受保護 API
var apiClient = new HttpClient();
apiClient.SetBearerToken(tokenResponse.AccessToken!); // 添加 Bearer Token

var response = await apiClient.GetAsync("https://localhost:6001/identity"); // 受保護資源url
if (!response.IsSuccessStatusCode)
{
    Console.WriteLine(response.StatusCode);
}
else
{
    var doc = JsonDocument.Parse(await response.Content.ReadAsStringAsync()).RootElement;
    // 這邊先解析成 JsonDocument，相較直接讀 string 更好讀
    Console.WriteLine(JsonSerializer.Serialize(doc, new JsonSerializerOptions { WriteIndented = true }));
}
```

#### 8. 讓 Api 驗證 scope
* src/Api/Program.cs
```csharp
builder.Services.AddAuthorization(opt =>
{
	opt.AddPolicy("ApiScope", policy =>
		{
			policy.RequireAuthenticatedUser();
			policy.RequireClaim("scope", "api1"); // 要求有 api1 scope

		});
});

...

app.MapGet("identity", (ClaimsPrincipal user) => user.Claims.Select(c => new { c.Type, c.Value }))
    .RequireAuthorization("ApiScope");
```


### Stage 2: User Authentication (Code Type)
前面的內容完全只有 OAuth 2.0 也就是授權的部分，所以客戶端只是想要獲取 access_token 後訪問資源，本章要讓用戶能進行登入並且受保護的 Api 能獲得用戶資訊，也就是要實現 OIDC。OIDC 需要實現：
* interactive UI
* 配置 OIDC scope
* 配置 OIDC client
* 提供可登入的用戶

#### 添加 UI
在 src/IdentityServer 項目中使用 
```shell
dotnet new isui # 這只會生成 ui 不會覆蓋其他文件
```
會生成 `Page` 與 `wwwroot` 兩個文件夾，並提供默認界面。

#### 配置 OIDC scope
* Config.cs
```csharp
public static IEnumerable<IdentityResource> IdentityResources =>
    new IdentityResource[]
    {
        new IdentityResources.OpenId(), // 支持 standard openid，也就是提供 subject id
        new IdentityResources.Profile(), // 支持 profile，包含 fisrt name, last name, email, etc.
    };
```
* HostingExtension.cs
```csharp
builder.Services.AddIdentityServer()
    .AddInMemoryIdentityResources(Config.IdentityResources) // 添加 Identity 資源
    .AddInMemoryApiScopes(Config.ApiScopes)
    .AddInMemoryClients(Config.Clients);
```


#### 添加可登入的測試用戶
* HostingExtension.cs
```csharp
builder.Services.AddIdentityServer()
    .AddInMemoryIdentityResources(Config.IdentityResources)
    .AddInMemoryApiScopes(Config.ApiScopes)
    .AddInMemoryClients(Config.Clients);
    .AddTestUsers(TestUsers.Users);
```
`TestUsers.Users` 提供用 bob 與 alice 兩個用戶，兩者的密碼都是用戶名，可用來測試。

#### 註冊 Client
* Config.cs
```cs
public static IEnumerable<Client> Clients =>
	new Client[]
	{
		new Client
		{
			ClientId = "web",
			AllowedGrantTypes = GrantTypes.Code, // Code Type
			ClientSecrets = { new Secret("secret".Sha256())}, 
			AllowedScopes =
			{
					IdentityServerConstants.StandardScopes.OpenId, // "openid"
					IdentityServerConstants.StandardScopes.Profile, // "profile"
			},
			// 登入後重定向位置 (由客戶端使用的框架而定，這邊使用 Microsoft.AspNetCore.Authentication.OpenIdConnect)
			// 若前端爲 nextauth 請見 https://next-auth.js.org/configuration/providers/oauth
			RedirectUris = { "http://localhost:5002/signin-oidc" },
			// 登出之後重定向位置 (由客戶端使用的框架而定，這邊使用 Microsoft.AspNetCore.Authentication.OpenIdConnect)
			PostLogoutRedirectUris = { "http://localhost:5002/signout-callback-oidc" },
		}
	};
```

#### 添加客戶端
```shell
dotnet new webapp -o src/WebClient # this is razor page
dotnet sln add src/WebClient
```
* 添加 `Microsoft.AspNetCore.Authentication.OpenIdConnect` 包，提供 Asp.net core openid 權限驗證中間件。

* src/WebClient/Program.cs
```cs
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddRazorPages();
builder.Services.AddAuthentication(opt =>
{
	// 默認驗證方式 
	opt.DefaultScheme = "Cookies";  // 這邊的策略名稱不重要，需要後續添加實際策略時綁定用

	// 當默認驗證失敗會採取的策略
	opt.DefaultChallengeScheme = "oidc"; // 這邊的策略名稱不重要，需要後續添加實際策略時綁定用 

})
.AddCookie("Cookies") // 將 Cookies 綁定在 cookies 驗證策略

// OpenIdConnection 會將爲驗證用戶 redirect 到 IdentityServer，完成登入後
// 會建立 Cookies 供後續驗證。
.AddOpenIdConnect("oidc", cfg =>
{
	cfg.Authority = "http://localhost:5001";
	cfg.RequireHttpsMetadata = false;
	
	cfg.ClientId = "web";
	cfg.ClientSecret = "secret";
	cfg.ResponseType = "code";

	cfg.Scope.Clear();
	cfg.Scope.Add("openid");
	cfg.Scope.Add("profile");

	cfg.MapInboundClaims = false; // use microsoft claim type, not jwt
	cfg.SaveTokens = true; // Save access token and refresh token
});
var app = builder.Build();

// Configure the HTTP request pipeline.
if (!app.Environment.IsDevelopment())
{
	app.UseExceptionHandler("/Error");
	// The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
	app.UseHsts();
}

app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapRazorPages().RequireAuthorization(); // 全局授權，整個 Razor page 訪問都要驗證
app.Run();
```

##### 大坑: Chrome 在點擊登入後沒有跳轉
我首先使用 firefox 測式，是可以跳轉，因爲 firefox 的報錯信息是最完整的，所以我在 console 中發現中顯示一個 warning，表示 cookie 並沒有 same-site policy。
我懷疑 Chrome 就是因爲這個 Cookie 導致無法跳轉回去，那甚麼事 Same-site cookie？ 這表示當網站訪問第三方網站時，第三方網站會使用 Cookie 或設置 Cookie，若要返回原網站這些 Cookie 可能會導致 CSRF (跨域請求僞造)，也就是源網站獲取第三方網站的 Cookie 進行僞造，但我們在使用 Identity Server 希望這些 Cookie 能被原網站使用，這就需要配置 same-site Cookie。
* src/IdentityServer/Program.cs
```cs
// 添加以下這行
app.UseCookiePolicy(new CookiePolicyOptions
{
	MinimumSameSitePolicy = SameSiteMode.Lax
});

app.UseStaticFiles();
app.UseRouting();
...
```
以上做了什麼事情？
1. CookiePolicy 中間件： app.UseCookiePolicy 用於添加 ASP.NET Core 中的 CookiePolicy 中間件。這個中間件可以幫助您管理網站中的 cookie 策略，確保它們符合瀏覽器的安全性要求。
2. CookiePolicyOptions： 這是用於配置 CookiePolicy 中間件行為的選項對象。在這裡，您設置了 MinimumSameSitePolicy 屬性。
3. MinimumSameSitePolicy： 這個屬性指定了最小的 SameSite 策略，即在什麼情況下允許 cookie 在跨站請求中發送。SameSiteMode.Lax 表示最小的 SameSite 策略為 "Lax"，這意味著允許在導航到第三方站點時發送 cookie，但僅限於 GET 請求。這有助於防止 CSRF 攻擊。
也就是說會將尚未標記 same-site cookie 或者 same-site 策略小於 SameSiteMode.Lex 的 cookie 都標記成爲 Lex


#### 登出功能
* 在 src/WebClient/Pages 執行 `dotnet new page -n Signout`。

* src/WebClient/Pages/Signout.cshtml.cs
```cs
public class SignoutModel : PageModel
{
    public IActionResult OnGet()
    {
		    // 參數對應 AddAuthentication() 配置
        return SignOut("Cookies", "oidc");
    }
}
```

* 在 `src/WebClient/Pages/Shared/_Layout.cshtml` 中添加導航，或者你想添加按鈕也行
```html
<ul class="navbar-nav flex-grow-1">
    <li class="nav-item">
        <a class="nav-link text-dark" asp-area="" asp-page="/Index">Home</a>
    </li>
    <li class="nav-item">
        <a class="nav-link text-dark" asp-area="" asp-page="/Privacy">Privacy</a>
    </li>
    <li class="nav-item">
        <a class="nav-link text-dark" asp-area="" asp-page="/Signout">Signout</a>
    </li>
</ul>
```


#### 取得用戶資訊
我們發現 Cliam 裏面除了 sub 就沒有多的東西，但我們想要用戶的資訊，所以要 Client 添加以下這段
* src/WebClient/Program.cs
```cs
	cfg.Scope.Clear();
	cfg.Scope.Add("openid");
	cfg.Scope.Add("profile");
	cfg.GetClaimsFromUserInfoEndpoint = true; // 告訴 Client 要從 userinfo 端點互取用戶資訊
```

#### 添加更多 Cliam，讓 Client 取得更多用戶資訊
* src/IdentityServer/Config.cs
```cs
	public static IEnumerable<IdentityResource> IdentityResources =>
		new IdentityResource[]
		{
			new IdentityResources.OpenId(),
			new IdentityResources.Profile(),
			new IdentityResource
			{
				Name = "moreinfo", // scope name
				UserClaims = { JwtClaimTypes.Email, JwtClaimTypes.EmailVerified } // 添加以上 Claim
			}
		};
		public static IEnumerable<Client> Clients =>
		new Client[]
			{
				new Client
				{
					ClientId = "web",
					AllowedGrantTypes = GrantTypes.Code,
					ClientSecrets = { new Secret("secret".Sha256())},
					AllowedScopes =
					{
						IdentityServerConstants.StandardScopes.OpenId,
                        IdentityServerConstants.StandardScopes.Profile,
                        "moreinfo" // 添加這個 scope
					},
					RedirectUris = { "http://localhost:5002/signin-oidc" },
					PostLogoutRedirectUris = { "http://localhost:5002/signout-callback-oidc" },
				}
			};

```

* src/WebClient/Program.cs
```cs
  .AddOpenIdConnect("oidc", options =>
  {
      // ...
      options.Scope.Add("verification");
      // email_verified 不會自動 map 需要手動 map
      // 第一個參數爲 ClaimType，第二個參數是 Json 的 key
      options.ClaimActions.MapJsonKey("email_verified", "email_verified"); 
      // ...
  }
```


### Stage 3: 取得 refresh token 與授權訪問受保護的 Api
#### 添加 scope
* src/IdentityServer/Config.cs
```cs
new Client
{
    ClientId = "web",
    ClientSecrets = { new Secret("secret".Sha256()) },

    AllowedGrantTypes = GrantTypes.Code,
    RedirectUris = { "https://localhost:5002/signin-oidc" },
    PostLogoutRedirectUris = { "https://localhost:5002/signout-callback-oidc" },
    

    AllowedScopes =
    {
        IdentityServerConstants.StandardScopes.OpenId,
        IdentityServerConstants.StandardScopes.Profile,
        "verification",
        "api1" // 添加這個 scope
    }

    AllowOfflineAccess = true, // 表示會取得 refresh token，Offline 表示不用再次與 ID Server 交互，也就是說不用重新登入
}
```

#### WebClient 獲得 refresh token 與 api scope
* src/WebClient/Program.cs
```cs
	cfg.Scope.Add("openid");
	cfg.Scope.Add("profile");
	cfg.Scope.Add("moreinfo");
	cfg.Scope.Add("api1");
	cfg.Scope.Add("offline_access"); // refresh token
	cfg.GetClaimsFromUserInfoEndpoint = true;
	cfg.ClaimActions.MapJsonKey("email_verified", "email_verified");
```

#### 訪問 Api
* src/WebClient/CallApi.cshtml.cs
```cs
using System.Net.Http.Headers;
using System.Text.Json;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace MyApp.Namespace
{

	public class CallApiModel : PageModel
	{
		public string Json = string.Empty;

		public async Task OnGet()
		{
			var accessToken = await HttpContext.GetTokenAsync("access_token");
			var client = new HttpClient();
			client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);
			var content = await client.GetStringAsync("http://localhost:6001/identity");

			var parsed = JsonDocument.Parse(content);
			var formatted = JsonSerializer.Serialize(parsed);

			Json = formatted;
		}
	}
}
```

* src/WebClient/CallApi.cshtml
```cs
@page
@model MyApp.Namespace.CallApiModel

<pre>@Model.Json</pre>
```

之後訪問 `http://localhost:5002/CallApi`

### Stage 4: 添加 EF core

### Stage 5: 使用 Asp.net core identity
