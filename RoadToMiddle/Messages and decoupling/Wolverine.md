## 1. Що таке Wolverine?

Wolverine — це фреймворк для .NET, що поєднує **in-process медіатор** (як MediatR) та **distributed messaging** (як NServiceBus/MassTransit) в одному пакеті. Ключова відмінність від аналогів — Wolverine використовує **кодогенерацію** під час старту замість runtime reflection, що дає значно ефективніші пайплайни та чистіші stack traces.

```
Wolverine = Mediator (in-process) + Message Broker integration (RabbitMQ, SQS, Azure Service Bus...)
```

Фреймворк розглядає HTTP як "ще один різновид повідомлення" — тобто та сама логіка обробників працює і для HTTP-запитів, і для черг.

### Ключова термінологія

| Термін | Опис |
|---|---|
| **Envelope** | Обгортка навколо повідомлення, що містить метадані (ID, заголовки, час, tenant тощо) |
| **Transport** | Підтримка зовнішньої інфраструктури (RabbitMQ, SQS, Azure Service Bus, TCP) |
| **Endpoint** | Конфігурація з'єднання до зовнішнього ресурсу (черга, топік, порт) |
| **Node** | Запущений екземпляр Wolverine-додатку |
| **Message Store** | База даних для зберігання повідомлень (inbox/outbox, саги) |
| **Local Queue** | Внутрішня черга всередині одного процесу (in-memory або durable) |

---

## 2. Повідомлення (Messages)

Повідомлення в Wolverine — це звичайні .NET-класи або C# records. Wolverine **не вимагає** реалізовувати конкретні інтерфейси чи наслідувати базові класи.

```csharp
// Command — навмисна дія
public record DebitAccount(long AccountId, decimal Amount);

// Event — факт що щось відбулось
public record AccountOverdrawn(long AccountId);
```

> ⚠️ Wolverine не робить структурної різниці між Command та Event — це концептуальне розрізнення лише для розробника.

**Вимоги до повідомлень:**
- Тип повинен бути `public`
- Без обов'язкових базових класів або інтерфейсів

### Ідентифікація повідомлень

За замовчуванням Wolverine використовує `FullName` типу як ідентифікатор. Можна перевизначити:

```csharp
[MessageIdentity("person-born")]
public class PersonBorn
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

### Версіонування

```csharp
[MessageIdentity("person-born", Version = 2)]
public class PersonBornV2
{
    public string FirstName { get; set; }
    public DateTime Birthday { get; set; }
}
```

Wolverine генерує content type: `application/vnd.person-born.v2+json`.

### Message Forwarding (трансформація старих версій)

```csharp
public class PersonBorn : IForwardsTo<PersonBornV2>
{
    public string FirstName { get; set; }

    public PersonBornV2 Transform()
        => new PersonBornV2 { FirstName = FirstName, Birthday = DateTime.MinValue };
}
```

> Починаючи з версії 6.0, forwarder потрібно реєструвати явно:
> `opts.RegisterMessageForwarder<PersonBorn, PersonBornV2>()`

---

## 3. Обробники повідомлень (Message Handlers)

Wolverine генерує код для виклику обробників під час старту — **без runtime reflection**. Обробники — це звичайні .NET методи без зобов'язання реалізовувати конкретні інтерфейси.

### Конвенції іменування

Wolverine знаходить обробники автоматично за суфіксом типу та назвою методу:

| Що шукає | Приклади |
|---|---|
| Суфікс класу | `...Handler`, `...Consumer` |
| Назва методу | `Handle()`, `Consume()` |

```csharp
public class DebitAccountHandler
{
    public void Handle(DebitAccount command)
    {
        Console.WriteLine($"Debiting {command.Amount} from {command.AccountId}");
    }
}
```

Всі типи (клас, метод, повідомлення) повинні бути `public`.

### Варіанти сигнатур

```csharp
// Sync
public void Handle(MyMessage msg) { }

// Async
public async Task Handle(MyMessage msg) { }

// Static (краща продуктивність — без алокації об'єкту)
public static void Handle(MyMessage msg) { }

// З поверненням cascading message
public MyResponse Handle(MyMessage msg) => new MyResponse();
```

### Ін'єкція параметрів у метод

Крім самого повідомлення, обробник може приймати будь-що з IoC-контейнера або з Wolverine runtime:

```csharp
public async Task Handle(
    DebitAccount command,
    IAccountRepository repo,         // IoC сервіс
    Envelope envelope,               // метадані повідомлення
    IMessageBus bus,                 // для публікації зсередини обробника
    CancellationToken cancellationToken,
    DateTime now)                    // поточний час (ін'єктується Wolverine)
{
    // ...
}
```

### Compound Handlers — розподіл відповідальності

Wolverine підтримує спеціальні методи всередині одного класу для організації пайплайну:

```
Before / Load / Validate  →  Handle  →  After / PostProcess  →  Finally
```

```csharp
public class PlaceOrderHandler
{
    // Завантажує дані та може зупинити виконання
    public async Task<(HandlerContinuation, Order?)> LoadAsync(PlaceOrderCommand cmd, IOrderDb db)
    {
        var order = await db.FindAsync(cmd.OrderId);
        return order is null
            ? (HandlerContinuation.Stop, null)
            : (HandlerContinuation.Continue, order);
    }

    // Основна бізнес-логіка
    public OrderConfirmed Handle(PlaceOrderCommand cmd, Order order)
    {
        order.Confirm();
        return new OrderConfirmed(order.Id);
    }

    // Виконується після Handle
    public void After(PlaceOrderCommand cmd)
    {
        // пост-обробка, логування
    }
}
```

### Декілька обробників на одне повідомлення

За замовчуванням кілька обробників для одного типу повідомлення об'єднуються в один логічний ланцюг. Щоб зробити їх незалежними:

```csharp
opts.MultipleHandlerBehavior = MultipleHandlerBehavior.Separated;
```

---

## 4. IMessageBus — публікація та відправка

`IMessageBus` — основний сервіс для ініціювання обробки та публікації повідомлень. Реєструється як **scoped** в IoC.

```
IMessageBus
  ├── InvokeAsync()         — виконати локально та дочекатись результату
  ├── InvokeAsync<T>()      — Request/Reply (повернути відповідь)
  ├── SendAsync()           — відправити (мінімум 1 підписник, інакше Exception)
  ├── PublishAsync()        — опублікувати (тихо якщо немає підписників)
  └── ScheduleAsync()       — відкладена відправка
```

### InvokeAsync — локальне виконання

```csharp
public async Task invoke_debit_account(IMessageBus bus)
{
    // Виконується в поточному потоці, синхронно (in-process)
    await bus.InvokeAsync(new DebitAccount(2222, 250));
}
```

### SendAsync vs PublishAsync

```csharp
// SendAsync — кидає Exception якщо немає жодного підписника
await bus.SendAsync(new PlaceOrder(orderId));

// PublishAsync — мовчки ігнорує якщо немає підписників (ідеально для events)
await bus.PublishAsync(new OrderPlaced(orderId));
```

### Request/Reply

```csharp
// Обробник повертає відповідь
public class GetOrderHandler
{
    public OrderDetails Handle(GetOrder query, IOrderRepository repo)
        => repo.FindById(query.OrderId);
}

// Виклик з очікуванням відповіді
var details = await bus.InvokeAsync<OrderDetails>(new GetOrder(orderId));
```

> Якщо тип відповіді збігається з requested response type — він **не** відправляється як cascading message, а повертається напряму (Wolverine 3.0+).

### Планування повідомлень

```csharp
// Відправити через 5 хвилин
await bus.ScheduleAsync(new SendReminder(userId), TimeSpan.FromMinutes(5));

// Відправити в конкретний момент
await bus.ScheduleAsync(new SendReminder(userId), DateTimeOffset.UtcNow.AddHours(1));
```

### DeliveryOptions — кастомізація доставки

```csharp
await bus.PublishAsync(new OrderPlaced(orderId), new DeliveryOptions
{
    DeliverWithin = TimeSpan.FromMinutes(10), // TTL
    Headers = { ["x-tenant"] = "acme" }
});
```

---

## 5. Каскадні повідомлення (Cascading Messages)

Замість ручного виклику `IMessageBus` всередині обробника можна просто **повернути** повідомлення як результат. Wolverine опублікує їх автоматично після успішної обробки.

```csharp
// Без каскадних повідомлень
public class OrderHandler
{
    public async Task Handle(PlaceOrder cmd, IMessageBus bus)
    {
        // ...
        await bus.PublishAsync(new OrderConfirmed(cmd.OrderId));
    }
}

// З каскадними повідомленнями — чистіше, легше тестувати
public class OrderHandler
{
    public OrderConfirmed Handle(PlaceOrder cmd)
    {
        // ...
        return new OrderConfirmed(cmd.OrderId);
    }
}
```

> ✅ Каскадні повідомлення відправляються лише після **успішного** завершення обробника.

### Кілька повідомлень

```csharp
// Через tuple — самодокументований підхід
public (OrderConfirmed, SendEmail) Handle(PlaceOrder cmd)
{
    return (new OrderConfirmed(cmd.OrderId), new SendEmail(cmd.CustomerEmail));
}

// Через IEnumerable — динамічна кількість
public IEnumerable<object> Handle(PlaceOrder cmd)
{
    yield return new OrderConfirmed(cmd.OrderId);
    if (cmd.SendEmail)
        yield return new SendEmail(cmd.CustomerEmail);
}
```

### Кастомізація доставки каскадних повідомлень

```csharp
public IEnumerable<object> Handle(PlaceOrder cmd)
{
    yield return new OrderConfirmed(cmd.OrderId);
    yield return new SendReminder(cmd.UserId).DelayedFor(TimeSpan.FromDays(1));
    yield return new AuditLog(cmd.OrderId).ScheduledAt(DateTime.UtcNow.AddMinutes(5));
}
```

### ISendMyself — кастомна маршрутизація

```csharp
public class SpecialMessage : ISendMyself
{
    public ValueTask ApplyAsync(IMessageBus bus)
        => bus.SendAsync(this, new DeliveryOptions { /* ... */ });
}
```

---

## 6. Middleware — Russian Doll Model

Wolverine реалізує middleware через **кодогенерацію** (не через runtime об'єктні алокації). Згенерований код нагадує "матрьошку":

```
middleware.Before()
  → handler.Handle()
middleware.After()
finally { middleware.Finally() }
```

### Конвенції lifecycle-методів

| Фаза | Назви методів |
|---|---|
| До обробника | `Before`, `BeforeAsync`, `Load`, `LoadAsync`, `Validate`, `ValidateAsync` |
| Після обробника | `After`, `AfterAsync`, `PostProcess`, `PostProcessAsync` |
| Завжди (finally) | `Finally`, `FinallyAsync` |

### Приклад middleware

```csharp
public class TransactionMiddleware
{
    private readonly IDbSession _session;

    public TransactionMiddleware(IDbSession session) => _session = session;

    public Task BeforeAsync() => _session.BeginTransactionAsync();

    public Task AfterAsync() => _session.CommitAsync();

    public Task FinallyAsync() => _session.DisposeAsync().AsTask();
}
```

### Зупинка обробки через HandlerContinuation

`Before`-методи можуть повертати `HandlerContinuation` щоб зупинити пайплайн:

```csharp
public class AuthMiddleware
{
    public HandlerContinuation Before(Envelope envelope)
    {
        if (!envelope.Headers.ContainsKey("api-key"))
            return HandlerContinuation.Stop;

        return HandlerContinuation.Continue;
    }
}
```

Можна повертати tuple щоб одночасно передати дані та контролювати потік:

```csharp
public async Task<(HandlerContinuation, Customer?)> LoadAsync(OrderCommand cmd, ICustomerRepo repo)
{
    var customer = await repo.FindAsync(cmd.CustomerId);
    return customer is null
        ? (HandlerContinuation.Stop, null)
        : (HandlerContinuation.Continue, customer);
}
```

### Застосування middleware

```csharp
// Глобально через policy
opts.Policies.AddMiddleware<TransactionMiddleware>();

// До всіх обробників конкретного типу повідомлень
opts.Policies.ForMessagesOfType<IRequiresTransaction>()
    .AddMiddleware<TransactionMiddleware>();

// Через атрибут на обробнику
[Middleware(typeof(TransactionMiddleware))]
public class PlaceOrderHandler { ... }
```

---

## 7. Обробка помилок (Error Handling)

### Доступні стратегії

| Стратегія | Опис |
|---|---|
| `RetryNow` | Негайна повторна спроба |
| `ScheduleRetry` | Повторна спроба через вказаний час |
| `Requeue` | Повернути повідомлення в кінець черги |
| `Discard` | Записати в лог і видалити |
| `MoveToErrorQueue` | Перемістити в Dead Letter Queue |
| `PauseListener` | Тимчасово зупинити обробку endpoint |

### Конфігурація через атрибути

```csharp
[RetryNow(typeof(SqlException), 50, 100, 250)]       // 3 повтори з затримкою в мс
[MoveToErrorQueueOn(typeof(InvalidMessageException))] // відразу в DLQ
public void Handle(ProcessPayment cmd) { ... }
```

### Конфігурація через Fluent API

```csharp
opts.Policies
    .OnException<TimeoutException>()
    .ScheduleRetry(5.Seconds(), 30.Seconds(), 5.Minutes()); // 3 спроби з наростаючою затримкою

opts.Policies
    .OnException<InvalidOperationException>()
    .Discard();
```

### Jitter — захист від "thundering herd"

При одночасному відновленні багатьох Consumer-ів додавання випадковості запобігає перевантаженню:

```csharp
opts.Policies
    .OnException<TimeoutException>()
    .ScheduleRetry(5.Seconds())
    .WithExponentialJitter(); // затримка в діапазоні [d, d × (1 + 2·attempt)]
```

| Метод | Ефективна затримка |
|---|---|
| `WithFullJitter` | `[d, 2·d]` |
| `WithBoundedJitter` | `[d, d × (1 + percent)]` |
| `WithExponentialJitter` | `[d, d × (1 + 2·attempt)]` |

### Circuit Breaker

Тимчасово зупиняє обробку endpoint коли відсоток помилок перевищує поріг:

```csharp
opts.ListenToRabbitQueue("orders")
    .CircuitBreaker(cb =>
    {
        cb.MinimumThreshold = 10;           // мінімум 10 спроб перед оцінкою
        cb.FailurePercentageThreshold = 10; // 10% помилок → відкрити circuit
        cb.PauseTime = 1.Minutes();         // пауза перед повторною спробою
        cb.TrackingPeriod = 5.Minutes();    // вікно відстеження
    });
```

```
[CLOSED] → failure rate > threshold → [OPEN] → pause elapsed → [HALF-OPEN] → success → [CLOSED]
                                                                             → failure → [OPEN]
```

### Fault Events

При термінальному збої Wolverine може автоматично публікувати `Fault<T>` з деталями:

```csharp
// Зареєструватись на Fault-події
public class OrderFaultHandler
{
    public void Handle(Fault<ProcessOrder> fault)
    {
        // fault.Message, fault.Exception, fault.Attempts, fault.CorrelationId
    }
}
```

---

## 8. Durable Messaging — Inbox / Outbox

### Проблема без Outbox

При відправці повідомлення разом із записом в БД можливі три сценарії втрати:

```
1. БД ✅  →  повідомлення відправлено ❌  →  подія втрачена
2. повідомлення відправлено ✅  →  БД ❌  →  подія без даних
3. БД ✅  →  повідомлення не доставлено  →  подія втрачена
```

### Transactional Outbox — рішення

Повідомлення зберігається **в тій самій транзакції** що і зміни в БД. Окремий фоновий процес читає з outbox і відправляє в брокер:

```
[Handler] → [DB Transaction] → { бізнес-дані + outbox record }
[Background relay] → reads outbox → sends to broker → marks as sent
```

```csharp
// Увімкнути durable outbox для конкретного endpoint
opts.PublishAllMessages()
    .ToRabbitExchange("orders")
    .UseDurableOutbox();

// Або глобально
opts.Policies.UseDurableOutboxOnAllSendingEndpoints();
```

### Durable Inbox

Вхідні повідомлення зберігаються в БД **до** успішної обробки. Якщо обробник впав — повідомлення не загубиться:

```
[Broker] → [Inbox record saved] → [Handler] → [Success] → [Inbox record deleted]
                                            → [Failure] → [Inbox record stays, retry later]
```

### Local Queue Durability

```csharp
opts.LocalQueue("critical-orders")
    .UseDurableInbox(); // кожне повідомлення в цій черзі зберігається в БД
```

### Stale Message Recovery

```csharp
opts.Durability.InboxStaleTime = TimeSpan.FromMinutes(5);  // відновлювати завислі inbox
opts.Durability.OutboxStaleTime = TimeSpan.FromMinutes(5); // відновлювати завислі outbox
```

---

## 9. Серіалізація

### Доступні формати

| Формат | Пакет | Коли використовувати |
|---|---|---|
| **System.Text.Json** (default) | вбудований | більшість сценаріїв |
| **Newtonsoft.Json** | `WolverineFx.Newtonsoft` | потрібна гнучкість |
| **MessagePack** | `WolverineFx.MessagePack` | high-throughput, бінарний |
| **MemoryPack** | `WolverineFx.MemoryPack` | максимальна швидкість (.NET) |
| **Protobuf** | `WolverineFx.Protobuf` | cross-language сумісність |

### Конфігурація

```csharp
// Глобально
opts.UseNewtonsoftForSerialization();

// Для конкретної черги
opts.LocalQueue("fast-path").UseMessagePackSerialization();
```

### Self-Serializing Messages

```csharp
public class BinaryEvent : ISerializable
{
    public byte[] Write() => /* власна серіалізація */;

    public static object Read(byte[] bytes) => /* власна десеріалізація */;
}
```

### Envelope ID Generation

```csharp
// За замовчуванням: NewId (machine-identity based sequential GUID)
// Ризик дублікатів у cloud де інстанції діляться network identity

// Рекомендовано для cloud/containers (.NET 9+)
opts.EnvelopeIdGeneration = EnvelopeIdGeneration.GuidV7; // RFC 9562, time-ordered, cryptographically random
```

---

## 10. Типові питання на співбесіді

**Q: Чим Wolverine відрізняється від MediatR?** Wolverine об'єднує in-process медіатор і distributed messaging в одному фреймворку. На відміну від MediatR, Wolverine використовує кодогенерацію замість reflection (швидше), підтримує транспорти (RabbitMQ, SQS тощо) та має вбудований Outbox/Inbox паттерн.

**Q: Що таке cascading messages і яка їх перевага?** Cascading messages — це повідомлення що повертаються як значення з обробника. Вони автоматично публікуються після успішного завершення. Перевага: обробник залишається чистою функцією без залежності від `IMessageBus`, що спрощує юніт-тести.

**Q: Яка різниця між `SendAsync` і `PublishAsync`?** `SendAsync` кидає Exception якщо немає жодного підписника — підходить для команд де обробник обов'язковий. `PublishAsync` мовчки ігнорує відсутність підписників — підходить для подій де підписники опціональні.

**Q: Як влаштований Outbox паттерн у Wolverine?** Повідомлення зберігаються в тій самій транзакції що і бізнес-дані. Фоновий relay процес читає неотправлені записи з outbox і доставляє їх в брокер. Це гарантує що або і зміни в БД, і повідомлення успішні, або обидва відкочуються.

**Q: Що таке compound handler і навіщо він потрібен?** Compound handler — це клас з кількома lifecycle-методами: `Load/Before/Validate` → `Handle` → `After` → `Finally`. Дозволяє розподілити відповідальність: окремо завантажити дані, валідувати, виконати бізнес-логіку, зробити пост-обробку.

**Q: Як Wolverine обробляє помилки? Які є стратегії?** Wolverine підтримує: негайний retry, retry з затримкою (+ jitter для запобігання thundering herd), requeue, discard, переміщення в DLQ, тимчасову паузу listener. Все конфігурується через атрибути або Fluent API. Для endpoint-рівня — Circuit Breaker.

**Q: Що таке HandlerContinuation і де він використовується?** `HandlerContinuation` — enum з `Continue` і `Stop`. Повертається з `Before`/`Load`/`Validate` методів middleware або compound handler для зупинки пайплайну. Наприклад: якщо сутність не знайдена — зупинити обробку без помилки.

**Q: Навіщо GuidV7 для EnvelopeIdGeneration?** `NewId` (дефолт) генерує sequential GUID на основі network identity машини. У контейнерах де інстанції можуть мати однаковий network interface — є ризик дублікатів. `GuidV7` використовує криптографічно випадкову time-ordered генерацію (RFC 9562) і безпечний у будь-якому середовищі.
