---
tags: [security, auth, jwt, oauth, aspnetcore, backend]
aliases: [JWT, OAuth 2.0, OpenID Connect, Authentication, Authorization]
---

> **Автентифікація** — хто ти є (identity). **Авторизація** — що тобі дозволено (permissions). Спочатку завжди автентифікація, потім авторизація.

---

## 1. JWT — JSON Web Token

### Структура

JWT складається з трьох base64url-encoded частин розділених крапкою:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   ← Header
.eyJzdWIiOiJ1c2VyMSIsInJvbGUiOiJhZG1pbiIsImV4cCI6MTcxNTAwMDAwMH0  ← Payload
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature
```

**Header** — алгоритм підпису:
```json
{ "alg": "HS256", "typ": "JWT" }
```

**Payload** — claims (твердження про користувача):
```json
{
  "sub": "user123",        // subject — ID користувача
  "name": "Alice",
  "role": "admin",
  "iat": 1715000000,       // issued at
  "exp": 1715003600,       // expiration
  "iss": "myapp.com",      // issuer
  "aud": "myapp-api"       // audience
}
```

**Signature** — HMAC-SHA256 або RSA підпис:
```
HMACSHA256(base64(header) + "." + base64(payload), secret)
```

> ⚠️ JWT **не шифрується** (якщо не використовується JWE). Payload читається будь-ким. Ніколи не клади в JWT чутливих даних (паролі, CVV, PII).

> ⚠️ JWT **неможливо відкликати** до закінчення терміну дії — якщо треба logout/revoke, потрібний blacklist у Redis або короткий TTL + refresh token.

### Налаштування в ASP.NET Core

```csharp
// Program.cs
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opt =>
    {
        opt.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = "myapp.com",
            ValidateAudience = true,
            ValidAudience = "myapp-api",
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Secret"]!)),
            ClockSkew = TimeSpan.FromSeconds(30) // допуск на різницю годинників
        };
    });

builder.Services.AddAuthorization();

// після builder.Build():
app.UseAuthentication(); // хто ти?
app.UseAuthorization();  // що тобі можна? (завжди після UseAuthentication)
```

### Генерація токена

```csharp
public string GenerateToken(User user)
{
    var claims = new[]
    {
        new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
        new Claim(ClaimTypes.Email, user.Email),
        new Claim(ClaimTypes.Role, user.Role),
        new Claim("tenant", user.TenantId)  // custom claim
    };

    var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["Jwt:Secret"]!));
    var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

    var token = new JwtSecurityToken(
        issuer: "myapp.com",
        audience: "myapp-api",
        claims: claims,
        expires: DateTime.UtcNow.AddHours(1),
        signingCredentials: creds
    );

    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

---

## 2. Refresh Tokens

Access token має короткий TTL (15 хв – 1 год). Refresh token — довгий (7-30 днів), зберігається в HttpOnly cookie або БД.

```
[Login] → Access Token (15 min) + Refresh Token (30 days)
[Request] → Authorization: Bearer <access_token>
[Access expired] → POST /auth/refresh { refreshToken } → new Access Token + new Refresh Token
[Logout] → invalidate Refresh Token в БД
```

```csharp
// генерація refresh token
public string GenerateRefreshToken()
{
    var bytes = RandomNumberGenerator.GetBytes(64);
    return Convert.ToBase64String(bytes);
}

// зберігати в БД разом з userId і expiration
// при /auth/refresh — перевіряти що токен існує, не протермінований, не використаний
```

> Rotation: при кожному refresh — видавати **новий** refresh token і інвалідувати старий. Якщо старий використали ще раз — сигнал про крадіжку, відкликати всі токени юзера.

---

## 3. Claims, Roles, Policies

### Claims — твердження про користувача

```csharp
// читання claims у контролері/хендлері
var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
var email = User.FindFirstValue(ClaimTypes.Email);
var tenantId = User.FindFirstValue("tenant");
var isAdmin = User.IsInRole("admin");
```

### Roles — проста авторизація

```csharp
[Authorize(Roles = "admin")]
public IActionResult AdminOnly() { ... }

[Authorize(Roles = "admin,moderator")]  // OR
public IActionResult AdminOrMod() { ... }
```

### Policies — гнучка авторизація

Roles не вистачає для складних правил. Policies дають повний контроль.

```csharp
// реєстрація
builder.Services.AddAuthorization(opt =>
{
    opt.AddPolicy("MinAge18", policy =>
        policy.RequireClaim("age")
              .AddRequirements(new MinAgeRequirement(18)));

    opt.AddPolicy("AdminOrOwner", policy =>
        policy.RequireAssertion(ctx =>
            ctx.User.IsInRole("admin") ||
            ctx.User.FindFirstValue(ClaimTypes.NameIdentifier) == resourceOwnerId));

    opt.AddPolicy("PremiumUser", policy =>
        policy.RequireClaim("subscription", "premium", "enterprise"));
});

// кастомний requirement
public class MinAgeRequirement(int minAge) : IAuthorizationRequirement { public int MinAge => minAge; }

public class MinAgeHandler : AuthorizationHandler<MinAgeRequirement>
{
    protected override Task HandleRequirementAsync(AuthorizationHandlerContext ctx, MinAgeRequirement req)
    {
        var ageClaim = ctx.User.FindFirstValue("age");
        if (ageClaim is not null && int.Parse(ageClaim) >= req.MinAge)
            ctx.Succeed(req);
        return Task.CompletedTask;
    }
}

// використання
[Authorize(Policy = "MinAge18")]
public IActionResult AdultContent() { ... }
```

---

## 4. OAuth 2.0

OAuth 2.0 — протокол **авторизації** (не автентифікації). Дозволяє застосунку отримати доступ до ресурсів від імені користувача, не знаючи його пароля.

### Ролі

| Роль | Хто |
|---|---|
| **Resource Owner** | Користувач (власник даних) |
| **Client** | Твій застосунок |
| **Authorization Server** | Google, GitHub, Keycloak (видає токени) |
| **Resource Server** | API яке захищає дані |

### Flows (Grant Types)

**Authorization Code** — для web/mobile застосунків від імені користувача:

```
User → Client → Authorization Server (login/consent) → redirect з code
Client → exchanges code for tokens (з client_secret, через back-channel)
```

**Authorization Code + PKCE** (Proof Key for Code Exchange) — для SPA і mobile де немає безпечного client_secret:

```
Client генерує code_verifier (random) + code_challenge = SHA256(code_verifier)
Відправляє code_challenge при authorize запиті
При обміні code → tokens відправляє code_verifier (сервер перевіряє)
```

**Client Credentials** — server-to-server, без користувача:

```
Service A → Authorization Server (client_id + client_secret) → Access Token
Service A → Service B API (з токеном)
```

**Device Flow** — для пристроїв без браузера (TV, CLI):

```
Device → Auth Server → отримує device_code і user_code
User → переходить на сторінку в браузері, вводить user_code
Device → polling Auth Server → Access Token
```

| Flow | Коли використовувати |
|---|---|
| Authorization Code + PKCE | Web SPA, Mobile app — від імені користувача |
| Authorization Code | Traditional web app (є back-end для client_secret) |
| Client Credentials | M2M (service-to-service), background jobs |
| Device | TV apps, CLI tools, IoT |

---

## 5. OpenID Connect (OIDC)

OAuth 2.0 вирішує **авторизацію**. OIDC — надбудова над OAuth 2.0, що додає **автентифікацію**.

OIDC додає:
- **ID Token** (JWT) — інформація про автентифікованого користувача (`sub`, `name`, `email`, `picture`)
- **UserInfo Endpoint** — отримати додаткові claims
- **Discovery Document** (`/.well-known/openid-configuration`) — метадані провайдера

```
OAuth 2.0 видає:  Access Token (що можна робити)
OIDC додає:       ID Token     (хто ти є)
```

```csharp
// підключення OIDC в ASP.NET Core
builder.Services
    .AddAuthentication(opt =>
    {
        opt.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        opt.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
    })
    .AddCookie()
    .AddOpenIdConnect(opt =>
    {
        opt.Authority = "https://accounts.google.com";
        opt.ClientId = config["Google:ClientId"];
        opt.ClientSecret = config["Google:ClientSecret"];
        opt.ResponseType = "code";
        opt.Scope.Add("openid");
        opt.Scope.Add("email");
        opt.Scope.Add("profile");
        opt.CallbackPath = "/signin-google";
    });
```

---

## 6. Збереження токенів на клієнті

| Місце | Захист від XSS | Захист від CSRF | Рекомендація |
|---|---|---|---|
| **localStorage** | ❌ вразливий | ✅ | Не для access token |
| **sessionStorage** | ❌ вразливий | ✅ | Краще ніж localStorage, але не ідеально |
| **HttpOnly Cookie** | ✅ недоступний JS | ❌ потрібен CSRF token | Рекомендовано для refresh token |
| **Memory (JS var)** | ✅ | ✅ | Рекомендовано для access token (SPA) |

Найкращий підхід для SPA:
- Access token — в пам'яті (змінна), короткий TTL
- Refresh token — в HttpOnly cookie, з `SameSite=Strict`

---

## 7. Типові питання на співбесіді

**Q: Яка різниця між автентифікацією і авторизацією?** Автентифікація — перевірка ідентичності ("хто ти?"). Авторизація — перевірка прав ("що тобі можна?"). Спочатку auto, потім authz.

**Q: Чому JWT неможливо відкликати?** JWT — stateless токен, сервер не зберігає стан. Після видачі токен валідний до закінчення терміну незалежно від дій. Рішення: дуже короткий TTL (15 хв) + refresh token в Redis blacklist, або зберігати jti (JWT ID) відкликаних токенів.

**Q: Яка різниця між OAuth 2.0 і OpenID Connect?** OAuth 2.0 — протокол авторизації (доступ до ресурсів). OIDC — надбудова над OAuth 2.0 для автентифікації, додає ID Token (JWT з інформацією про користувача) і стандартизований UserInfo Endpoint.

**Q: Коли використовувати Client Credentials flow?** Для server-to-server (M2M) комунікації без участі користувача. Наприклад: background job отримує токен і звертається до внутрішнього API, або два мікросервіси спілкуються між собою.

**Q: Яка різниця між Roles і Policies в ASP.NET Core?** Roles — проста авторизація за роллю. Policies — гнучкі правила на основі claims, складних умов, кастомних requirement handlers. Policy може перевіряти комбінацію claims, права на конкретний ресурс, зовнішні дані.

**Q: Що таке PKCE і навіщо він потрібен?** PKCE (Proof Key for Code Exchange) — захист Authorization Code flow для клієнтів де неможливо безпечно зберегти client_secret (SPA, mobile). Клієнт генерує випадковий code_verifier, відправляє його хеш при authorize, і оригінал при обміні code → token. Захищає від перехоплення authorization code.
