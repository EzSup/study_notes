# Dependency Injection in .NET

**Dependency Injection (DI)** — це патерн проєктування, який полягає у передаванні залежностей об'єкта ззовні, а не створенні їх усередині класу. Це дозволяє зменшити зв'язність компонентів, полегшити тестування та сприяє кращій розширюваності коду.

### **Який принцип реалізує DI?**
DI реалізує принцип **Inversion of Control (IoC)** (інверсія управління). Замість того, щоб клас сам керував створенням своїх залежностей, це завдання передається зовнішньому контейнеру (DI-контейнеру).

### **Основна ідея IoC**
Клас не повинен створювати свої залежності самостійно — їх потрібно передавати йому через конструктор, методи або властивості.


### **Типи Dependency Injection (DI) у різних типах .NET-додатків**

У різних типах .NET-додатків механізм DI може дещо відрізнятися за способом використання та реалізації. Однак загальна концепція однакова — передача залежностей через IoC-контейнер.

---

## 🔹 **1. DI в ASP.NET Core (WebApplication)**

🔸 **Використовує вбудований DI-контейнер**  
🔸 **Реєстрація сервісів у `Program.cs`:**

csharp

КопіюватиРедагувати

`var builder = WebApplication.CreateBuilder(args); builder.Services.AddScoped<IMyService, MyService>();  var app = builder.Build(); app.Run();`

🔸 **Отримання залежностей через конструктор (Constructor Injection):**

csharp

КопіюватиРедагувати

`public class HomeController : Controller {     private readonly IMyService _service;     public HomeController(IMyService service)     {         _service = service;     } }`

✅ **Особливість:**

- Автоматичне впровадження залежностей у контролери, middleware, фільтри тощо.
- Життєвий цикл `Scoped` прив'язаний до HTTP-запиту.

---

## 🔹 **2. DI у Worker Service (Фонові сервіси)**

🔸 **Використовує `HostBuilder`** (аналог `WebApplicationBuilder`, але без веб-сервера).  
🔸 **Реєстрація сервісів:**

csharp

КопіюватиРедагувати

`var host = Host.CreateDefaultBuilder(args)     .ConfigureServices(services =>     {         services.AddSingleton<IMyService, MyService>();         services.AddHostedService<MyBackgroundService>();     })     .Build();`

🔸 **Впровадження залежностей у фоновий сервіс:**

csharp

КопіюватиРедагувати

`public class MyBackgroundService : BackgroundService {     private readonly IMyService _service;     public MyBackgroundService(IMyService service)     {         _service = service;     }      protected override async Task ExecuteAsync(CancellationToken stoppingToken)     {         while (!stoppingToken.IsCancellationRequested)         {             _service.DoWork();             await Task.Delay(1000, stoppingToken);         }     } }`

✅ **Особливість:**

- `Scoped` не має сенсу, оскільки немає HTTP-запитів.
- Використовують `Singleton` або `Transient`.

---

## 🔹 **3. DI у консольних додатках**

🔸 **За замовчуванням DI немає**, але можна додати через `HostBuilder`.  
🔸 **Реєстрація сервісів:**

csharp

КопіюватиРедагувати

`var host = Host.CreateDefaultBuilder(args)     .ConfigureServices(services =>     {         services.AddTransient<IMyService, MyService>();     })     .Build();  var service = host.Services.GetRequiredService<IMyService>(); service.DoWork();`

✅ **Особливість:**

- Використання DI **не є обов’язковим**, але може бути корисним.
- Можна використовувати **Generic Host** для логування, конфігурації та DI.

---

## 🔹 **4. DI у WPF та Windows Forms**

🔸 **За замовчуванням DI не інтегрований**, але можна налаштувати вручну.  
🔸 **Реєстрація сервісів у `App.xaml.cs`:**

csharp

КопіюватиРедагувати

`public partial class App : Application {     private readonly IServiceProvider _serviceProvider;          public App()     {         var services = new ServiceCollection();         services.AddSingleton<IMainViewModel, MainViewModel>();         _serviceProvider = services.BuildServiceProvider();     }          protected override void OnStartup(StartupEventArgs e)     {         var mainWindow = new MainWindow         {             DataContext = _serviceProvider.GetRequiredService<IMainViewModel>()         };         mainWindow.Show();     } }`

✅ **Особливість:**

- DI використовується для `ViewModel` (MVVM-підхід).
- Використання `Singleton` або `Scoped`, якщо потрібно зберігати стан.

---

## 🔹 **5. DI у .NET MAUI (мобільні додатки)**

🔸 **Вбудований DI-контейнер, схожий на ASP.NET Core**  
🔸 **Реєстрація сервісів у `MauiProgram.cs`:**

csharp

КопіюватиРедагувати

`public static MauiApp CreateMauiApp() {     var builder = MauiApp.CreateBuilder();     builder.Services.AddSingleton<IMyService, MyService>();      return builder.Build(); }`

✅ **Особливість:**

- Використовується для `ViewModel` у MVVM-патерні.
- `Scoped` не використовується, оскільки немає HTTP-запитів.

---

## **📌 Висновок: яка різниця між DI у різних типах .NET-додатків?**

|Тип додатку|Чи є вбудований DI?|Спосіб використання DI|Особливості|
|---|---|---|---|
|**ASP.NET Core**|✅ Так|`WebApplicationBuilder`|Автоматична ін'єкція у контролери, middleware, фільтри|
|**Worker Service**|✅ Так|`HostBuilder`|Використовують `Singleton` або `Transient`|
|**Консольний додаток**|❌ Ні (але можна додати)|`HostBuilder`|DI не є обов’язковим, але спрощує управління залежностями|
|**WPF / WinForms**|❌ Ні (але можна додати)|`ServiceCollection`|Використовують для `ViewModel` у MVVM-підході|
|**.NET MAUI**|✅ Так|`MauiApp.CreateBuilder`|Аналог ASP.NET Core DI, працює у мобільних додатках|

### 🔥 **Ключові висновки:**

- **У ASP.NET Core та Worker Service DI вбудований та працює через `HostBuilder`.**
- **У консольних, WPF і MAUI-додатках DI можна налаштувати вручну через `ServiceCollection`.**
- **Життєвий цикл сервісів (Scoped, Singleton, Transient) залежить від типу додатку.**

## Див. також
- [[Design patterns]]
- [[Middlewares]]
- [[Filters]]
- [[Backgorund service and hosted service]]
- [[Clean Architecture]]

---
#dotnet #csharp #dependency-injection #ioc #aspnet
