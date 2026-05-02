---
tags: [dotnet, csharp, aspnet, middlewares, pipeline]
aliases: [Middlewares, Middleware]
---

# Middlewares

Middleware — це компоненти, які формують пайплайн обробки HTTP-запитів в ASP.NET Core. Кожен middleware отримує запит, може виконати логіку до і після виклику наступного middleware, і передає запит далі через `next()`. Це реалізація патерна [[Design patterns#Ланцюжок обов'язків (*Chain of Responsibility*)|Chain of Responsibility]].

```
  HTTP Request ──► Middleware 1 ──► Middleware 2 ──► ... ──► Endpoint
  HTTP Response ◄── Middleware 1 ◄── Middleware 2 ◄── ... ◄──┘
```

## Як працює пайплайн
Middleware додаються в `Program.cs` через методи `Use...()`. Порядок додавання визначає порядок виконання. Запит проходить через кожен middleware зверху вниз, а відповідь — знизу вверх.

```C#
var app = builder.Build();

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.UseRequestLogging();    // наш кастомний middleware

app.MapControllers();
app.Run();
```

### Стандартні middleware в ASP.NET Core
Порядок додавання має значення. Рекомендований порядок:
1. `UseExceptionHandler` — глобальна обробка помилок
2. `UseHsts` — HTTP Strict Transport Security
3. `UseHttpsRedirection` — перенаправлення на HTTPS
4. `UseStaticFiles` — віддача статичних файлів
5. `UseRouting` — маршрутизація
6. `UseCors` — Cross-Origin Resource Sharing
7. `UseAuthentication` — автентифікація
8. `UseAuthorization` — авторизація
9. `UseEndpoints` / `MapControllers` — ендпоінти

## Створення кастомного middleware

### Middleware class
Кастомний middleware створюється як клас з методом `InvokeAsync`, який приймає `HttpContext` і викликає `_next(context)` для передачі запиту далі.

``` C#
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;

    public RequestLoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Log information before processing the request
        var startTime = DateTime.Now;
        Console.WriteLine($"Request started at {startTime}: {context.Request.Method} {context.Request.Path}");

        // Call the next middleware in the pipeline
        await _next(context);

        // Log information after processing the request
        var endTime = DateTime.Now;
        var duration = endTime - startTime;
        Console.WriteLine($"Request completed at {endTime}. Duration: {duration.TotalMilliseconds}ms");
    }
}

// Extension method to make it easier to add the middleware to the pipeline
public static class RequestLoggingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RequestLoggingMiddleware>();
    }
}
```

### Inline middleware (через lambda)
Для простих випадків можна використовувати `app.Use()` без створення окремого класу:

```C#
app.Use(async (context, next) =>
{
    Console.WriteLine($"Before: {context.Request.Path}");
    await next(context);
    Console.WriteLine($"After: {context.Response.StatusCode}");
});
```

### Terminal middleware
`app.Run()` створює термінальний middleware — він не викликає `next()` і завершує пайплайн:

```C#
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello from terminal middleware");
});
```

### Map — розгалуження пайплайну
`app.Map()` дозволяє розгалужити пайплайн за шляхом запиту:

```C#
app.Map("/api/health", appBuilder =>
{
    appBuilder.Run(async context =>
    {
        await context.Response.WriteAsync("OK");
    });
});
```

## Middleware vs [[Filters]]
|                | Middleware                            | Filters                                      |
| -------------- | ------------------------------------- | -------------------------------------------- |
| **Рівень**     | Весь HTTP пайплайн                    | Тільки MVC/контролери                        |
| **Порядок**    | Визначається в `Program.cs`           | Визначається атрибутами або глобально        |
| **Доступ**     | `HttpContext`                         | `ActionExecutingContext`, моделі, результати |
| **Коли юзати** | Логування, CORS, auth, error handling | Валідація, кешування на рівні action         |

Middleware виконуються **до** того, як запит потрапляє до контролерів. Фільтри виконуються **всередині** MVC пайплайну, коли контролер вже визначений.

## Див. також
- [[Filters]]
- [[Design patterns]]
- [[Di .net applications]]
- [[Backgorund service and hosted service]]
