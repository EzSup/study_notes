`IEndpointRouteBuilder`

``` C#
// Custom middleware class
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