Фільтри – це компоненти, які дозволяють виконувати код _до або після_ певних етапів виконання дії (action) в контролері.
Щоб оголосити фільтр, потрібно створити клас, успадкований від [[Attributes]] і реалізуючий один із інтерфейсів фільтрів.
Залежно від деяких умов фільтрів, є різні інтерфейси, але головний це `IActionFilter`.
`IActionFilter` - Найбільш часто використовуваний тип фільтрів. Виконується безпосередньо перед та після виконання дії контролера.
	Його методи:
	 `OnActionExecuting(ActionExecutingContext context)` – виконується перед виконанням дії.	
	 `OnActionExecuted(ActionExecutedContext context)` – виконується після виконання дії.

Синтаксис:
``` C#
using Microsoft.AspNetCore.Mvc.Filters;
using System.Diagnostics;

[AttributeUsage(AttributeTargets.Method)]  
public class OnlymyFilter : Attribute, IActionFilter  
{  
    private readonly Stopwatch _stopwatch = new Stopwatch();  
  
    public void OnActionExecuting(ActionExecutingContext context)  
    {        _stopwatch.Start();  
        Console.WriteLine($"EXECUTING {context.HttpContext.Request.Path}");  
    }  
    public void OnActionExecuted(ActionExecutedContext context)  
    {        _stopwatch.Stop();  
        Console.WriteLine($"EXECUTEDDDDD {context.HttpContext.Request.Path} in {_stopwatch.ElapsedMilliseconds} ms");  
    }}

// Застосування фільтра:
[OnlymyFilter]
public class HomeController : Controller
{
    public IActionResult Index()
    {
        return View();
    }
}
```

Є ще асинхронний варіант фільтра, `IAsyncActionFilter`, в ньому тільки один метод, але дії до і після виконання контролерів і наступних фільтрів в пайплайні розділяються викликом `next()`:
``` C#
public class OnlymeAsyncFilter : Attribute, IAsyncActionFilter  
{  
    private readonly Stopwatch _stopwatch = new Stopwatch();  
  
    public Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)  
    {        Console.WriteLine($"ASYNC EXECUTING {context.HttpContext.Request.Path}");  
        next();        
        Console.WriteLine($"ASYNC EXECUTEDDDDD {context.HttpContext.Request.Path} in {_stopwatch.ElapsedMilliseconds} ms");  
        return Task.CompletedTask;  
    }  
}
```

Таким чином, якщо викликати наші фільтри так:
``` C#
[HttpGet("myaction")]  
[OnlymyFilter]  
[OnlymeAsyncFilter]  
public IActionResult TEST()  
{  
    _logger.Log(LogLevel.Information, "CONTROLLER EXECTUTING");  
    return Ok();  
}
```

То матимемо такий вивід в консолі:
``` C#
EXECUTING /mymy/myaction
ASYNC EXECUTING /mymy/myaction
info: FiltersAndMiddlewares.MyController[0]
      CONTROLLER EXECTUTING
ASYNC EXECUTEDDDDD /mymy/myaction in 0 ms
EXECUTEDDDDD /mymy/myaction in 7 ms
```

Фільтри, які застосовуються, утворюють вже свій пайплайн, який, так як виконується вже коли запит дойшов до контролерів, виконується після мідлвеєрів.
Фільтри виконуються в такому порядку:
1. `IAuthorizationFilter`
2. `IResourceFilter.OnResourceExecuting`
3. `IActionFilter.OnActionExecuting`
4. *Виконання дії контролера*
5. `IActionFilter.OnActionExecuted`
6. `IResultFilter.OnResultExecuting`
7. *Виконання результату дії*
8. `IResultFilter.OnResultExecuted`
9. `IResourceFilter.OnResourceExecuted`
10. `IExceptionFilter` (виконується тільки у випадку необробленого винятку)

