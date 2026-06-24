# Resilience в .NET: від концепції до Polly

## Що таке Resilience?

**Resilience** (стійкість, відмовостійкість) — здатність системи витримувати та відновлюватися після збоїв, деградацій або несподіваних умов, продовжуючи при цьому надавати прийнятний рівень сервісу.

Стійка система не просто "не падає" — вона **деградує поступово** і **відновлюється автоматично**.

### Чому це важливо?

У монолітному додатку збій одного компоненту — це локальна проблема. У мікросервісній архітектурі або при інтеграції з зовнішніми API — це каскадний ефект, який може покласти всю систему.

```
Сервіс A → Сервіс B → Сервіс C (лежить)
               ↓
    B чекає відповіді від C
               ↓
    A чекає відповіді від B
               ↓
    Всі потоки заблоковані → OutOfMemory → A падає теж
```

Це називається **каскадний збій** (cascading failure).

---

## Проблеми розподілених систем

При роботі з мережею та зовнішніми сервісами виникають специфічні класи помилок:

| Тип проблеми | Приклад | Поведінка |
|---|---|---|
| **Transient failures** | Тимчасова втрата мережі | Пройде сам через кілька секунд |
| **Timeout** | Сервіс завис, не відповідає | Нескінченне очікування |
| **Overload** | Сервіс перевантажений | 503 Service Unavailable |
| **Partial failure** | Деякі ендпоінти падають | Непередбачувана поведінка |
| **Slow callers** | Запити тривають 30с замість 200ms | Блокування ресурсів |

### Проблема "Thundering Herd"

Якщо сервіс впав і всі клієнти починають ретраїти одночасно — вони перевантажують його в момент відновлення. Це вбиває сервіс вдруге.

**Рішення:** додавати випадковий **jitter** до затримок між повторами.

---

## Основні патерни стійкості

### 1. Retry (Повтор)

Автоматично повторити операцію при збої. Підходить для **transient failures**.

- **Exponential backoff:** кожна наступна спроба чекає вдвічі довше
- **Jitter:** додає випадковий розкид до затримки

```
Спроба 1 → збій → чекати 1с
Спроба 2 → збій → чекати 2с
Спроба 3 → збій → чекати 4с
Спроба 4 → успіх ✓
```

### 2. Circuit Breaker (Автоматичний вимикач)

Запобігає зверненню до явно несправного сервісу. Аналогія — запобіжник у електриці.

**Стани:**
```
[Closed] ──(N збоїв)──→ [Open] ──(timeout)──→ [Half-Open]
    ↑                                                |
    └──────────────── успіх ────────────────────────┘
```

- **Closed:** все нормально, запити проходять
- **Open:** сервіс "вимкнений", запити відхиляються одразу (fail fast)
- **Half-Open:** пробний запит — якщо успішний, повертається до Closed

### 3. Timeout

Обмежує максимальний час очікування відповіді. Запобігає блокуванню потоків.

### 4. Bulkhead (Переборка)

Ізолює ресурси для різних операцій. Якщо одна операція займає всі потоки — інші продовжують працювати.

Назва — від морських переборок в кораблях, які запобігають затопленню всього судна при пробоїні в одному відсіку.

### 5. Fallback (Запасний варіант)

Повертає заздалегідь визначений результат при збої замість помилки.

### 6. Cache

Кешує успішні відповіді, щоб при збої сервісу повернути закешоване значення.

---

## Polly — бібліотека resilience для C#

**Polly** — це .NET бібліотека з відкритим кодом, яка реалізує всі вищезгадані патерни стійкості у вигляді **декларативних політик** (policies).

- GitHub: https://github.com/App-vNext/Polly
- NuGet: [Polly](https://www.nuget.org/packages/Polly)
- Ліцензія: BSD 3-Clause

### Встановлення

```bash
# Основна бібліотека
dotnet add package Polly

# Інтеграція з HttpClient (Microsoft.Extensions)
dotnet add package Microsoft.Extensions.Http.Polly

# Новий API (.NET 8+)
dotnet add package Microsoft.Extensions.Resilience
dotnet add package Polly.Extensions
```

### Концепція

Polly будується навколо трьох речей:

1. **Policy** — правило того, що робити при збої
2. **Execute** — виконати код з застосуванням цієї policy
3. **Handle** — які винятки (або результати) вважати збоєм

```csharp
// Загальна структура
var policy = Policy
    .Handle<SomeException>()       // що вважати збоєм
    .WaitAndRetry(3, ...);         // що робити при збої

await policy.ExecuteAsync(async () =>
{
    // твій код тут
});
```

---

## Основні політики Polly

### Retry

```csharp
// Простий ретрай
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .RetryAsync(3);

// З затримкою (exponential backoff + jitter)
var retryWithBackoff = Policy
    .Handle<HttpRequestException>()
    .Or<TimeoutRejectedException>()
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt =>
            TimeSpan.FromSeconds(Math.Pow(2, attempt))
            + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 100)), // jitter
        onRetry: (exception, timeSpan, attempt, context) =>
        {
            Console.WriteLine($"Retry {attempt} after {timeSpan.TotalSeconds:F1}s: {exception.Message}");
        });

// Нескінченний ретрай (обережно!)
var foreverRetry = Policy
    .Handle<TransientException>()
    .WaitAndRetryForeverAsync(attempt => TimeSpan.FromSeconds(attempt));
```

### Circuit Breaker

```csharp
// Класичний
var circuitBreaker = Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (exception, duration) =>
            Console.WriteLine($"Circuit OPEN for {duration.TotalSeconds}s: {exception.Message}"),
        onReset: () =>
            Console.WriteLine("Circuit CLOSED — service recovered"),
        onHalfOpen: () =>
            Console.WriteLine("Circuit HALF-OPEN — probing service")
    );

// Advanced — з відсотком збоїв (AdvancedCircuitBreaker)
var advancedCB = Policy
    .Handle<Exception>()
    .AdvancedCircuitBreakerAsync(
        failureThreshold: 0.5,              // 50% збоїв
        samplingDuration: TimeSpan.FromSeconds(10),
        minimumThroughput: 8,               // мін. кількість запитів для аналізу
        durationOfBreak: TimeSpan.FromSeconds(30)
    );
```

### Timeout

```csharp
// Optimistic — покладається на CancellationToken
var optimisticTimeout = Policy
    .TimeoutAsync(TimeSpan.FromSeconds(5));

// Pessimistic — для коду без підтримки CancellationToken
var pessimisticTimeout = Policy
    .TimeoutAsync(
        seconds: 5,
        timeoutStrategy: TimeoutStrategy.Pessimistic,
        onTimeoutAsync: async (context, timespan, task) =>
        {
            Console.WriteLine($"Timeout after {timespan.TotalSeconds}s");
        });
```

### Bulkhead

```csharp
var bulkhead = Policy
    .BulkheadAsync(
        maxParallelization: 10,    // макс. паралельних виконань
        maxQueuingActions: 20,     // макс. запитів у черзі
        onBulkheadRejectedAsync: async context =>
        {
            Console.WriteLine("Bulkhead rejected — too many concurrent calls");
        });
```

### Fallback

```csharp
// Для типу з результатом
var fallback = Policy<string>
    .Handle<Exception>()
    .FallbackAsync(
        fallbackValue: "default cached response",
        onFallbackAsync: async (exception, context) =>
        {
            Console.WriteLine($"Fallback triggered: {exception.Exception?.Message}");
        });

// З динамічним fallback-значенням
var dynamicFallback = Policy<UserDto>
    .Handle<HttpRequestException>()
    .FallbackAsync(
        fallbackAction: async (result, context, ct) =>
        {
            return await localCacheService.GetUserAsync(context["userId"].ToString());
        });
```

### Cache

```csharp
// Потрібен NuGet: Polly.Caching.Memory
var memoryCache = new MemoryCache(new MemoryCacheOptions());
var cacheProvider = new MemoryCacheProvider(memoryCache);

var cachePolicy = Policy
    .CacheAsync<UserDto>(
        cacheProvider: cacheProvider.AsyncFor<UserDto>(),
        ttl: TimeSpan.FromMinutes(5),
        onCacheGet: (context, key) => Console.WriteLine($"Cache HIT: {key}"),
        onCacheMiss: (context, key) => Console.WriteLine($"Cache MISS: {key}"),
        onCachePut: (context, key) => Console.WriteLine($"Cache PUT: {key}"),
        onCacheGetError: (context, key, ex) => Console.WriteLine($"Cache GET error: {ex.Message}"),
        onCachePutError: (context, key, ex) => Console.WriteLine($"Cache PUT error: {ex.Message}")
    );
```

---

## PolicyWrap — комбінування політик

Кілька політик можна об'єднати в одну. **Виконання відбувається ззовні всередину** — як матрьошка.

```csharp
var retryPolicy = Policy.Handle<Exception>().WaitAndRetryAsync(3, _ => TimeSpan.FromSeconds(1));
var circuitBreaker = Policy.Handle<Exception>().CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));
var timeout = Policy.TimeoutAsync(TimeSpan.FromSeconds(5));
var bulkhead = Policy.BulkheadAsync(10, 20);

// Порядок: зовнішній → внутрішній (найближчий до коду)
var combinedPolicy = Policy.WrapAsync(
    retryPolicy,     // 1. зовнішній: повторює при збої
    circuitBreaker,  // 2. захищає від зверненнь до "мертвого" сервісу
    bulkhead,        // 3. обмежує паралельність
    timeout          // 4. внутрішній: обмежує час виконання
);

await combinedPolicy.ExecuteAsync(async cancellationToken =>
{
    await CallExternalServiceAsync(cancellationToken);
}, CancellationToken.None);
```

### Рекомендований порядок

```
[Retry] → [CircuitBreaker] → [Bulkhead] → [Timeout] → (твій код)
```

**Чому саме так?**
- Retry — зовні, щоб повторювати при будь-якому внутрішньому збої
- CircuitBreaker — перевіряє стан сервісу до спроби
- Timeout — внутрі, щоб Retry міг повторити після timeout

---

## Інтеграція з HttpClient та DI

Найчастіший use-case — HTTP-запити. `Microsoft.Extensions.Http.Polly` робить інтеграцію елегантною.

```csharp
// Program.cs
builder.Services.AddHttpClient<IWeatherService, WeatherService>(client =>
{
    client.BaseAddress = new Uri("https://api.weather.com");
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy())
.AddPolicyHandler(GetTimeoutPolicy());

// Політики
static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy() =>
    HttpPolicyExtensions
        .HandleTransientHttpError()  // HttpRequestException, 5xx, 408
        .OrResult(r => r.StatusCode == HttpStatusCode.TooManyRequests)
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: (attempt, response, context) =>
            {
                // Читаємо Retry-After з заголовка якщо є
                if (response?.Result?.Headers?.RetryAfter?.Delta is TimeSpan retryAfter)
                    return retryAfter;
                return TimeSpan.FromSeconds(Math.Pow(2, attempt));
            },
            onRetryAsync: async (outcome, timespan, attempt, context) =>
            {
                var logger = context.GetLogger();
                logger?.LogWarning("Retry {attempt} after {delay}s", attempt, timespan.TotalSeconds);
                await Task.CompletedTask;
            });

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy() =>
    HttpPolicyExtensions
        .HandleTransientHttpError()
        .AdvancedCircuitBreakerAsync(
            failureThreshold: 0.5,
            samplingDuration: TimeSpan.FromSeconds(10),
            minimumThroughput: 10,
            durationOfBreak: TimeSpan.FromSeconds(30));

static IAsyncPolicy<HttpResponseMessage> GetTimeoutPolicy() =>
    Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(10));
```

### PolicyRegistry — централізоване зберігання

```csharp
// Реєстрація
builder.Services.AddPolicyRegistry(new PolicyRegistry
{
    { "RetryPolicy", GetRetryPolicy() },
    { "CircuitBreaker", GetCircuitBreakerPolicy() },
});

// Використання з іменованим HttpClient
builder.Services.AddHttpClient("MyClient")
    .AddPolicyHandlerFromRegistry("RetryPolicy")
    .AddPolicyHandlerFromRegistry("CircuitBreaker");
```

---

## Polly v8 / .NET 8 — новий API

З Polly v8 (2023) з'явився новий API через `ResiliencePipeline`. Він є частиною `Microsoft.Extensions.Resilience` і є рекомендованим підходом для нових .NET 8+ проектів.

### Ключові відмінності від v7

| v7 (старий) | v8 (новий) |
|---|---|
| `Policy` | `ResiliencePipeline` |
| `.ExecuteAsync()` | `.ExecuteAsync()` |
| `PolicyWrap` | Builder pipeline |
| Окремі пакети | `Microsoft.Extensions.Resilience` |
| Менше телеметрії | OpenTelemetry з коробки |

### Базовий приклад

```csharp
// Без DI
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(1),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        ShouldHandle = new PredicateBuilder()
            .Handle<HttpRequestException>()
            .Handle<TimeoutRejectedException>()
    })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(10),
        MinimumThroughput = 10,
        BreakDuration = TimeSpan.FromSeconds(30),
        OnOpened = args =>
        {
            Console.WriteLine("Circuit opened");
            return ValueTask.CompletedTask;
        }
    })
    .AddTimeout(TimeSpan.FromSeconds(5))
    .Build();

await pipeline.ExecuteAsync(async ct =>
{
    await CallServiceAsync(ct);
}, cancellationToken);
```

### Реєстрація через DI

```csharp
// Program.cs
builder.Services.AddResiliencePipeline("my-api-pipeline", builder =>
{
    builder
        .AddRetry(new RetryStrategyOptions { MaxRetryAttempts = 3, UseJitter = true })
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            SamplingDuration = TimeSpan.FromSeconds(10),
            MinimumThroughput = 10,
            BreakDuration = TimeSpan.FromSeconds(30)
        })
        .AddTimeout(TimeSpan.FromSeconds(10))
        .ConfigureTelemetry(LoggerFactory.Create(b => b.AddConsole())); // вбудована телеметрія
});

// Використання в сервісі
public class MyService
{
    private readonly ResiliencePipeline _pipeline;

    public MyService(ResiliencePipelineProvider<string> provider)
    {
        _pipeline = provider.GetPipeline("my-api-pipeline");
    }

    public async Task<Data> GetDataAsync(CancellationToken ct)
    {
        return await _pipeline.ExecuteAsync(async token =>
        {
            return await _httpClient.GetFromJsonAsync<Data>("/api/data", token);
        }, ct);
    }
}
```

### Типізований pipeline (з результатом)

```csharp
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddFallback(new FallbackStrategyOptions<HttpResponseMessage>
    {
        FallbackAction = args => ValueTask.FromResult(
            new HttpResponseMessage(HttpStatusCode.OK)
            {
                Content = new StringContent("{\"cached\": true}")
            }),
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
    })
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        MaxRetryAttempts = 3,
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .HandleResult(r => (int)r.StatusCode >= 500)
    })
    .Build();
```

---

## Context — передача даних між шарами

`Context` дозволяє передавати довільні дані між шарами policy та кодом, що виконується.

```csharp
// Старий API (v7)
var context = new Context
{
    ["userId"] = "user-123",
    ["correlationId"] = Guid.NewGuid().ToString(),
    [ContextExtensions.LoggerKey] = logger
};

await retryPolicy.ExecuteAsync(async ctx =>
{
    var userId = ctx["userId"]?.ToString();
    logger.LogInformation("Executing for user {UserId}", userId);
    await DoWorkAsync(userId);
}, context);

// Новий API (v8) — через ResilienceContext
var resilienceContext = ResilienceContextPool.Shared.Get(cancellationToken);
resilienceContext.Properties.Set(new ResiliencePropertyKey<string>("userId"), "user-123");

try
{
    await pipeline.ExecuteAsync(async ctx =>
    {
        ctx.Properties.TryGetValue(new ResiliencePropertyKey<string>("userId"), out var userId);
        await DoWorkAsync(userId);
    }, resilienceContext);
}
finally
{
    ResilienceContextPool.Shared.Return(resilienceContext); // повертаємо в пул
}
```

---

## Реальні приклади використання

### Виклик зовнішнього Payment API

```csharp
public class PaymentService
{
    private readonly IAsyncPolicy<HttpResponseMessage> _policy;
    private readonly HttpClient _httpClient;

    public PaymentService(HttpClient httpClient)
    {
        _httpClient = httpClient;

        _policy = Policy.WrapAsync(
            // Retry: 3 спроби з jitter
            HttpPolicyExtensions
                .HandleTransientHttpError()
                .WaitAndRetryAsync(3,
                    attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt))
                             + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 200))),

            // Circuit Breaker: відкривається після 50% збоїв
            HttpPolicyExtensions
                .HandleTransientHttpError()
                .AdvancedCircuitBreakerAsync(0.5, TimeSpan.FromSeconds(10), 5, TimeSpan.FromSeconds(60)),

            // Timeout: максимум 5 секунд
            Policy.TimeoutAsync<HttpResponseMessage>(5)
        );
    }

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        try
        {
            var response = await _policy.ExecuteAsync(() =>
                _httpClient.PostAsJsonAsync("/payments", request));

            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<PaymentResult>();
        }
        catch (BrokenCircuitException ex)
        {
            // Circuit відкритий — payment gateway недоступний
            throw new ServiceUnavailableException("Payment service is temporarily unavailable", ex);
        }
    }
}
```

### Читання з бази даних з fallback на кеш

```csharp
var readPolicy = Policy<UserProfile>
    .Handle<SqlException>()
    .FallbackAsync(
        fallbackAction: async (ctx, ct) =>
        {
            var userId = ctx.OperationKey;
            return await _redisCache.GetAsync<UserProfile>(userId)
                ?? UserProfile.Guest(); // ще один запасний варіант
        })
    .WrapAsync(
        Policy<UserProfile>
            .Handle<SqlException>()
            .WaitAndRetryAsync(2, _ => TimeSpan.FromMilliseconds(100))
    );

var profile = await readPolicy.ExecuteAsync(
    async ctx => await _database.GetUserProfileAsync(ctx.OperationKey),
    new Context(userId)
);
```

---

## Підводні камені

### 1. Неправильний порядок у PolicyWrap

```csharp
// ❌ НЕПРАВИЛЬНО — Circuit Breaker зовні Retry
// Retry вже не буде пробувати, якщо CB відхилить
Policy.WrapAsync(circuitBreaker, retry, timeout);

// ✅ ПРАВИЛЬНО
Policy.WrapAsync(retry, circuitBreaker, timeout);
```

### 2. Перестворення Circuit Breaker

```csharp
// ❌ НЕПРАВИЛЬНО — кожен запит отримує новий CB без збереженого стану
services.AddTransient<IMyService>(sp =>
    new MyService(Policy.Handle<Exception>().CircuitBreakerAsync(3, TimeSpan.FromSeconds(30))));

// ✅ ПРАВИЛЬНО — CB є синглтоном або зареєстрований у PolicyRegistry
services.AddSingleton<IAsyncPolicy>(
    Policy.Handle<Exception>().CircuitBreakerAsync(3, TimeSpan.FromSeconds(30)));
```

### 3. Retry без jitter (Thundering Herd)

```csharp
// ❌ НЕПРАВИЛЬНО — всі інстанції ретраять одночасно
.WaitAndRetryAsync(3, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)));

// ✅ ПРАВИЛЬНО — з jitter
.WaitAndRetryAsync(3, attempt =>
    TimeSpan.FromSeconds(Math.Pow(2, attempt))
    + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 200)));
```

### 4. Handle занадто широко

```csharp
// ❌ НЕПРАВИЛЬНО — ретраїмо навіть 404 або 401
Policy.Handle<Exception>().RetryAsync(3);

// ✅ ПРАВИЛЬНО — тільки transient помилки
HttpPolicyExtensions.HandleTransientHttpError()
    .RetryAsync(3);
// або
Policy
    .Handle<HttpRequestException>()
    .OrResult<HttpResponseMessage>(r => (int)r.StatusCode >= 500)
    .RetryAsync(3);
```

### 5. Timeout без CancellationToken у внутрішньому коді

```csharp
// ❌ НЕПРАВИЛЬНО — Optimistic timeout не спрацює
var timeout = Policy.TimeoutAsync(5); // Optimistic за замовчуванням

await timeout.ExecuteAsync(async () =>
{
    await SomeLegacyMethodThatIgnoresCancellation(); // не приймає CancellationToken!
});

// ✅ ПРАВИЛЬНО — Pessimistic для legacy коду
var timeout = Policy.TimeoutAsync(5, TimeoutStrategy.Pessimistic);

// АБО передавати CancellationToken
await timeout.ExecuteAsync(async ct =>
{
    await ModernMethodAsync(ct); // приймає і перевіряє CancellationToken
});
```

### 6. Ретраїти некоректні запити

```csharp
// ❌ НЕПРАВИЛЬНО — ретраїмо 400 Bad Request (марно)
HttpPolicyExtensions.HandleTransientHttpError()
    .OrResult(r => r.StatusCode == HttpStatusCode.BadRequest) // не треба!
    .RetryAsync(3);

// Ретраїти варто тільки:
// 500, 502, 503, 504 — серверні помилки
// 408 — Request Timeout
// 429 — Too Many Requests (з Retry-After)
// HttpRequestException — мережеві помилки
```

---

## Шпаргалка: коли що використовувати

| Ситуація | Патерн | Polly API |
|---|---|---|
| Тимчасова мережева помилка | Retry | `WaitAndRetryAsync` |
| Сервіс часто недоступний | Circuit Breaker | `CircuitBreakerAsync` / `AdvancedCircuitBreakerAsync` |
| Повільні відповіді | Timeout | `TimeoutAsync` |
| Забагато паралельних запитів | Bulkhead | `BulkheadAsync` |
| Graceful degradation | Fallback | `FallbackAsync` |
| Повторні однакові запити | Cache | `CacheAsync` |
| Кілька проблем одночасно | Комбінація | `PolicyWrap` / `ResiliencePipeline` |
| .NET 8+, новий проект | Новий API | `ResiliencePipelineBuilder` |
| HttpClient в DI | Інтеграція | `AddPolicyHandler` / `AddResilienceHandler` |

### Мінімальний резонний набір для HTTP-клієнта

```csharp
builder.Services.AddHttpClient<IMyService, MyService>()
    .AddPolicyHandler(
        HttpPolicyExtensions
            .HandleTransientHttpError()
            .WaitAndRetryAsync(3, attempt =>
                TimeSpan.FromSeconds(Math.Pow(2, attempt))
                + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 100))))
    .AddPolicyHandler(
        HttpPolicyExtensions
            .HandleTransientHttpError()
            .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)))
    .AddPolicyHandler(
        Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(10)));
```

---

*Документ охоплює Polly v7 та v8. Для нових проектів на .NET 8+ рекомендується `Microsoft.Extensions.Resilience` з `ResiliencePipelineBuilder`.*
