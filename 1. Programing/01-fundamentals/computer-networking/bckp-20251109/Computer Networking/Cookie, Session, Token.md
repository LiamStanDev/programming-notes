## Cookie
---
Cookie 是用於解決 HTTP 無狀態，但又需要保持該用戶的狀態而產生的技術，過程如下:
1. 客戶端發起 HTTP 請求 e.g. 登入
2. 服務端 Set-Cookie，使客戶端將 Cookie 存放於瀏覽器
3. 客戶端後續的請求都會附上 Cookie
> Cookie 中含有 Name, Value, Max-age, HTTP only 等內容

> HTTP only 屬性可以使客戶端的腳本語言無法獲取該 Cookie，用來防止跨網站腳本攻擊(XSS)
## Session
---
但是用戶資訊使用 Cookie 是有安全疑慮的，故衍伸出 Session，Session 會存放在服務器中，並會向客戶端設定 Cookie，該 Cookie 中包含 SessionID 與 Max-age。流程如下:
1. 客戶端發起 HTTP 請求 e.g. 登入
2. 服務端生成 SessionID，為一個字符串並且有進行簽名，故修改 SessionID 無法識別，並 Set-Cookie。
4. 客戶端後續的請求都會附上 Cookie，直到超過 Max-age

問題:
1. 會話數據存儲
	* 描述：在伺服器端存儲大量會話數據可能導致性能下降。
	- 解決方案：使用高效的會話管理策略，如會話資料分布式存儲或僅將必要數據存儲於會話中。
> Session 中服務端要存儲所有用戶的資訊

2. 會話劫持（Session Hijacking)
	- 描述：攻擊者盜取會話ID，並以用戶身份進行操作。
	- 解決方案：使用HTTPS保證傳輸加密，將cookie設為HttpOnly和Secure，定期更換會話ID。

## JWT (Json Web Token)
---
JWT 由三部分組成，以 `.` 分開   [JWT.io](https://jwt.io/)
1. header: 使用甚麼算法來生成簽名
2. payload: 數據
3. signature
服務端首先將 header 與 payload 經由 Base64 進行編碼 (不是加密)，得出來的兩個結果結合算法 + secret key 生成 signature (服務端可以用 secret key 解密 signature 確認內容是否有修改)。客戶端就可以經由 cookie, Bearer Header 或者 localStorage 設定給客戶端了。

好處:
1. JWT 包含全部信息，不需要在服務端進行存儲
	* session ID 只是 ID 還是需要在數據庫進行查詢，才知道是哪個用戶。
2. JWT 信息不會被修改
3. Mobile 軟體友好：因為 Mobile 沒有 Cookie 功能。
4. 性能好：不用查詢資料庫，資料庫查詢相較於 JWT 解密來的更快。

傳輸方式：
1. **Cookie**:
    - 瀏覽器向服務端發送請求時，會**自動携带**對應域名的所有cookie，其中就包括Token。
    - 但可能受到跨腳本攻擊（XSS）的威脅，因为惡意脚本可能读取cookie。
2. **Authorization Header**:
    - 将 Token 放在 HTTP 请求的 Authorization 头部中。这种方法通常与Bearer Token一起使用，格式为`Authorization: Bearer <token>`。
    - 不易受到XSS攻击的影响，但它需要**显式地在每次请求中设置**。
3. **LocalStorage或SessionStorage**:
    - Web应用也可以选择将Token存储在浏览器的LocalStorage或SessionStorage中。
    - 使得JavaScript代码可以容易地访问Token，但同样容易受到XXS攻击的威胁。

問題:
1. 同樣與Session一樣可以能會有 XSS 攻擊。

> 以上三個方式都建議使用 HTTPS 協議進行交互，以確保安全。


### 實作
```c#
public string GenerateJwtToken() 
{
    var tokenHandler = new JwtSecurityTokenHandler();
    var key = Encoding.ASCII.GetBytes("YourSecretKey");
    var tokenDescriptor = new SecurityTokenDescriptor
    {
        Subject = new ClaimsIdentity(new[] 
        {
            new Claim("id", "用户ID"),
            new Claim(JwtRegisteredClaimNames.Sub, "用户名或其他"),
            // 可以根据需要添加更多的Claim
        }),
        Expires = DateTime.UtcNow.AddDays(7),
        SigningCredentials = new SigningCredentials(new SymmetricSecurityKey(key), SecurityAlgorithms.HmacSha256Signature),
        Issuer = "YourIssuer",
        Audience = "YourAudience"
    };

    var token = tokenHandler.CreateToken(tokenDescriptor);
    return tokenHandler.WriteToken(token);
}
```

## 總結
---
* Cookie 是一種數據載體，在每次請求攜帶
* Session 誕生在服務端，保存在服務端
* JWT 誕生在服務端，保存在客戶端


