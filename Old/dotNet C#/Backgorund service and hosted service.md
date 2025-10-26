Background service - це сервіс, який виконується постійно на фоні не блокуючий основний потік програми. Він може бути реалізований через інтерфейс IHostedService чи клас BackgroundService
#### IHostedService
The `IHostedService` interface defines two methods:
- **`StartAsync(CancellationToken cancellationToken)`**: Called when the application host starts.
- **`StopAsync(CancellationToken cancellationToken)`**: Called when the application host is performing a graceful shutdown.
``` C#
public class TimedHostedService : IHostedService, IDisposable
{
    private readonly ILogger<TimedHostedService> _logger;
    private Timer _timer;

    public TimedHostedService(ILogger<TimedHostedService> logger)
    {
        _logger = logger;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Timed Hosted Service running.");

        _timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromSeconds(5));

        return Task.CompletedTask;
    }

    private void DoWork(object state)
    {
        _logger.LogInformation("Timed Hosted Service is working.");
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Timed Hosted Service is stopping.");
        _timer?.Change(Timeout.Infinite, 0);
        return Task.CompletedTask;
    }

    public void Dispose()
    {
        _timer?.Dispose();
    }
}
```

Важливою особливістю також є те, що незважаючи на те, що StartAsync і StopAsync методи є асинхронними ([[Async, Multithreading and concurrency]]), програма всеодно буде очікувати їх завершення для продовження роботи. 

Щоб налаштувати це, при додавання сервісів в DI контейнер потрібно налаштувати таке:
``` C#
builder.Service.Configure<HostOptions>(x => {
	x.ServicesStartConcurrently = [true/false],
	x.ServicesStopConcurrently = [true/false]
})
```
##### IHostedLifecycleService
В .Net 8 з'явився іще додатковий інтерфейс, який розширяє звичайний IHostedService тим, що окрім `StartAsync` і `StopAsync` в ньому також додались методи `StartingAsync` і `StoppingAsync`, `StartedAsync` і `StoppedAsync`.  Вони виконуються у такому порядку: `StartingAsync` => `StartAsync` =>`StartedAsync` => `StoppingAsync` => `StopAsync` => `StoppedAsync`

#### BackgroundService

``` C#
public class TimedBackgroundService : BackgroundService
{
    private readonly ILogger<TimedBackgroundService> _logger;

    public TimedBackgroundService(ILogger<TimedBackgroundService> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Timed Background Service running.");

        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Timed Background Service is working.");
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }

        _logger.LogInformation("Timed Background Service is stopping.");
    }
}
```

### Key Differences

- **Level of Abstraction**:
    - **`IHostedService`**: Requires manual implementation of starting and stopping logic.
    - **`BackgroundService`**: Simplifies the implementation by providing a base class with a single method to override.
- **Use Cases**:
    - **`IHostedService`**: Suitable for more complex scenarios where you need fine-grained control over the service lifecycle.
    - **`BackgroundService`**: Ideal for simpler, long-running tasks that benefit from reduced boilerplate code.
#### Pipeline injection of the background services:
`builder.Services.AddHostedService<ExampleService>();`

### Quartz.NET
Quartz.NET - це потужний планувальник задач з відкритим кодом, який може бути альтернативою до базових background services коли потрібна більш складна логіка планування.

Приклад налаштування і виконання джоби:
``` C#
/// SimpleJob.cs
public class SimpleJob : IJob
{
    public async Task Execute(IJobExecutionContext context)
    {
        await Console.Out.WriteLineAsync("Running SimpleJob!");
    }
}

/// Program.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddQuartz(q =>
    {
        q.UseMicrosoftDependencyInjectionJobFactory();
        
        var jobKey = new JobKey("SimpleJob");
        q.AddJob<SimpleJob>(opts => opts.WithIdentity(jobKey));

        q.AddTrigger(opts => opts
            .ForJob(jobKey)
            .WithIdentity("SimpleJob-trigger")
            .WithCronSchedule("0 0/5 * * * ?")); // Кожні 5 хвилин
    });

    services.AddQuartzHostedService(q => q.WaitForJobsToComplete = true);
}
```

Також Quartz дозволяє зберігати заплановані задачі в БД.