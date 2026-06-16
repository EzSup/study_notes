---
tags: [observability, logging, serilog, opentelemetry, aspnetcore, backend]
aliases: [Logging, Serilog, Structured Logging, OpenTelemetry, Health Checks]
---

> Логування — не просто рядки в консолі. Structured logging дозволяє логи фільтрувати і агрегувати як дані. Observability = Logs + Metrics + Traces.

---

## 1. Microsoft.Extensions.Logging — абстракція

`ILogger<T>` — вбудована абстракція .NET, не прив'язана до конкретної реалізації.

```csharp
public class OrderService(ILogger<OrderService> logger)
{
    public async Task ProcessOrderAsync(Guid orderId)
    {
        // structured logging: {} — placeholder, не string interpolation
        logger.LogInformation("Processing order {OrderId}", orderId);

        try
        {
            // ...
            logger.LogInformation("Order {OrderId} processed successfully in {ElapsedMs}ms",
                orderId, stopwatch.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Failed to process order {OrderId}", orderId);
            throw;
        }
    }
}
```

> ⚠️ **Ніколи не використовуй string interpolation в логах:**
> ```csharp
> logger.LogInformation($"Order {orderId} processed"); // ❌ втрачається structured data
> logger.LogInformation("Order {OrderId} processed", orderId); // ✅
> ```
> З string interpolation `orderId` стає частиною рядка. З placeholders — окремим структурованим полем доступним для пошуку.

### Рівні логування

| Рівень | Коли використовувати |
|---|---|
| `Trace` | Найдетальніша діагностика, зазвичай тільки в dev |
| `Debug` | Діагностична інформація для розробника |
| `Information` | Нормальний потік додатку (запит прийшов, замовлення оброблено) |
| `Warning` | Несподівана ситуація але додаток продовжує працювати |
| `Error` | Помилка в конкретній операції, але сервіс живий |
| `Critical` | Критична помилка, сервіс може впасти |

---

## 2. Serilog — structured logging

```
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Sinks.Seq           // локальний лог-сервер
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Thread
```

### Налаштування

```csharp
// Program.cs
builder.Host.UseSerilog((ctx, services, config) =>
{
    config
        .ReadFrom.Configuration(ctx.Configuration)  // з appsettings.json
        .ReadFrom.Services(services)                 // enrichers з DI
        .Enrich.FromLogContext()                     // з LogContext.PushProperty()
        .Enrich.WithMachineName()
        .Enrich.WithEnvironmentName()
        .Enrich.WithThreadId()
        .WriteTo.Console(new JsonFormatter())        // JSON в консоль
        .WriteTo.File(
            path: "logs/log-.txt",
            rollingInterval: RollingInterval.Day,
            retainedFileCountLimit: 30)
        .WriteTo.Seq("http://localhost:5341");       // Seq UI
});
```

```json
// appsettings.json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "System": "Warning"
      }
    }
  }
}
```

### Enrichers та LogContext

```csharp
// додати властивість до всіх логів в scope
using (LogContext.PushProperty("CorrelationId", correlationId))
using (LogContext.PushProperty("UserId", userId))
{
    logger.LogInformation("Processing started");  // буде містити CorrelationId і UserId
    await ProcessAsync();
    logger.LogInformation("Processing finished");
}
```

### Request logging middleware

```csharp
// логування HTTP запитів (замість дефолтного Microsoft Request Logging)
app.UseSerilogRequestLogging(opt =>
{
    opt.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
        diagnosticContext.Set("UserId", httpContext.User.FindFirstValue(ClaimTypes.NameIdentifier));
    };
    opt.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000}ms";
});
```

---

## 3. Correlation ID — відстеження запиту

Correlation ID — унікальний ідентифікатор що проходить через всі сервіси разом із запитом.

```csharp
// middleware для додавання Correlation ID
public class CorrelationIdMiddleware(RequestDelegate next)
{
    private const string Header = "X-Correlation-Id";

    public async Task InvokeAsync(HttpContext ctx)
    {
        var correlationId = ctx.Request.Headers[Header].FirstOrDefault()
                            ?? Guid.NewGuid().ToString();

        ctx.Response.Headers[Header] = correlationId;

        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            await next(ctx);
        }
    }
}

// реєстрація
app.UseMiddleware<CorrelationIdMiddleware>();
```

```csharp
// при виклику зовнішнього сервісу — передати correlation ID далі
httpClient.DefaultRequestHeaders.Add("X-Correlation-Id", correlationId);
```

---

## 4. OpenTelemetry — стандарт спостережуваності

OpenTelemetry (OTel) — vendor-neutral стандарт для Logs + Metrics + Traces.

```
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.SqlClient
dotnet add package OpenTelemetry.Exporter.Jaeger     // або OTLP для Grafana/Tempo
```

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()   // HTTP requests
            .AddHttpClientInstrumentation()   // outgoing HTTP
            .AddSqlClientInstrumentation()    // SQL queries
            .AddEntityFrameworkCoreInstrumentation()
            .AddSource("MyApp.*")             // custom ActivitySource
            .AddJaegerExporter(opt =>
            {
                opt.AgentHost = "localhost";
                opt.AgentPort = 6831;
            });
    })
    .WithMetrics(metrics =>
    {
        metrics
            .AddAspNetCoreInstrumentation()
            .AddRuntimeInstrumentation()      // GC, thread pool
            .AddPrometheusExporter();         // /metrics endpoint
    });
```

### Custom Spans (Activity)

```csharp
private static readonly ActivitySource _activitySource = new("MyApp.Orders");

public async Task ProcessOrderAsync(Guid orderId)
{
    using var activity = _activitySource.StartActivity("ProcessOrder");
    activity?.SetTag("order.id", orderId);

    try
    {
        var result = await _repo.GetOrderAsync(orderId);
        activity?.SetTag("order.status", result.Status.ToString());
        activity?.SetStatus(ActivityStatusCode.Ok);
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        activity?.RecordException(ex);
        throw;
    }
}
```

### Три стовпи спостережуваності

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│  Logs   │  │ Metrics │  │ Traces  │
│(events) │  │(numbers)│  │ (spans) │
└─────────┘  └─────────┘  └─────────┘
     ↓              ↓            ↓
 Що сталось   Скільки/як   Де це сталось
  і чому?     швидко?      (chain calls)
```

| Тип | Питання | Інструменти |
|---|---|---|
| **Logs** | Що сталось? Деталі помилки? | Serilog → Seq, Elasticsearch |
| **Metrics** | Скільки запитів/сек? Latency P99? Error rate? | Prometheus + Grafana |
| **Traces** | Через які сервіси пройшов запит? Де затримка? | Jaeger, Zipkin, Tempo |

---

## 5. Health Checks

```csharp
builder.Services
    .AddHealthChecks()
    .AddSqlServer(connectionString, name: "database")
    .AddRedis(redisConnectionString, name: "redis")
    .AddRabbitMQ(rabbitConnString, name: "rabbitmq")
    .AddCheck<CustomHealthCheck>("custom");

// endpoints
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // тільки перевірити що додаток живий (без зовнішніх залежностей)
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")  // додаток готовий приймати трафік
});

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse  // детальний JSON
});
```

```csharp
// кастомна перевірка
public class CustomHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext ctx, CancellationToken ct)
    {
        var isHealthy = CheckSomething();
        return Task.FromResult(isHealthy
            ? HealthCheckResult.Healthy("All good")
            : HealthCheckResult.Unhealthy("Something is wrong"));
    }
}
```

**Liveness vs Readiness (Kubernetes):**

```
Liveness probe:  /health/live  — чи живий pod (якщо ні → restart)
Readiness probe: /health/ready — чи готовий приймати трафік (якщо ні → вийняти з LB)
```

---

## 6. Типові помилки

```csharp
// ❌ string interpolation — втрачається structured data
logger.LogError($"Failed for user {userId}");

// ✅ structured logging
logger.LogError("Failed for user {UserId}", userId);

// ❌ логувати exception.Message — втрачається stack trace
logger.LogError("Error: {Message}", ex.Message);

// ✅ передавати exception першим параметром
logger.LogError(ex, "Failed to process order {OrderId}", orderId);

// ❌ sensitive дані в логах
logger.LogInformation("User logged in with password {Password}", password);

// ❌ логувати на рівні вище ніж потрібно
logger.LogError("User not found"); // це Warning або Info, не Error

// ❌ логувати в циклі на Information рівні (flood)
foreach (var item in millionItems)
    logger.LogInformation("Processing {Item}", item); // → Trace або Debug
```

---

## 7. Типові питання на співбесіді

**Q: Яка різниця між structured і plain text logging?** Plain text — рядок що людина читає. Structured — лог як об'єкт з полями (JSON). Structured дозволяє фільтрувати по конкретних полях (`WHERE OrderId = '...'`), агрегувати, будувати дашборди. Serilog з JSON sink — structured за замовчуванням.

**Q: Навіщо Correlation ID?** Ідентифікатор що присвоюється на вході в систему і передається між сервісами в заголовку. Дозволяє зібрати всі логи і трейси для одного запиту навіть якщо він пройшов через 10 мікросервісів.

**Q: Яка різниця між Logs, Metrics і Traces?** Logs — дискретні події з деталями. Metrics — числові агрегати в часі (RPS, latency percentiles, error rate). Traces — span tree що показує як запит проходив через сервіси і де затримка. Разом = повна observability.

**Q: Що таке liveness і readiness probe?** Liveness — чи живий процес (якщо ні → Kubernetes restart pod). Readiness — чи готовий pod приймати трафік (якщо ні → вийняти з load balancer). Readiness перевіряє зовнішні залежності (БД, Redis), liveness — тільки сам процес.

**Q: Чому не можна логувати через string interpolation?** При `$"User {id}"` значення `id` вбудовується в рядок і втрачається як окреме структуроване поле. При `"User {UserId}", id` — Serilog зберігає `{ UserId: 123, Message: "User 123" }` і можна шукати по `UserId = 123`.
