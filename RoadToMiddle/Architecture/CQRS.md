---
tags: [architecture, cqrs, mediator, patterns, backend]
aliases: [CQRS, Command Query Responsibility Segregation, MediatR]
---

> CQRS (Command Query Responsibility Segregation) — принцип розділення операцій **запису** (Commands) і **читання** (Queries) на різні моделі. Виникає з CQS Бертрана Мейєра: "метод або повертає дані, або змінює стан — але не одночасно."

---

## 1. Проблема без CQRS

Традиційна модель: один сервіс і одна модель для CRUD.

```
UserService
  ├── GetUser(id)          → повертає User зі всіма полями
  ├── GetUserList(filter)  → повертає той самий User DTO
  ├── CreateUser(...)      → валідація + запис
  └── UpdateUser(...)      → валідація + запис
```

**Конфлікти:**
- Read-модель оптимальна для UI (денормалізована, з join-ами) — write-модель оптимальна для бізнес-логіки (нормалізована, з інваріантами)
- Один і той самий Entity завантажується і для читання, і для запису → зайве навантаження на write-path
- Масштабування: reads >> writes, але масштабуємо одночасно обидва

---

## 2. Ідея CQRS

```
                    ┌─────────────┐
Commands (write) →  │ Write Model │ → БД (нормалізована)
                    └─────────────┘
                                          ↓ (синхронно або через events)
                    ┌─────────────┐
Queries  (read)  →  │ Read Model  │ → Read Store (денормалізована / кеш)
                    └─────────────┘
```

- **Command** — намір змінити стан. Не повертає дані (або мінімум — ID нового ресурсу). Може відхилити зміну.
- **Query** — запит даних без побічних ефектів. Ніколи не змінює стан.

---

## 3. Рівні застосування

**Логічне розділення (найчастіше)** — один сервіс, один стор, але розділені handler-и:

```
CreateOrderCommand → CreateOrderHandler → Orders table
GetOrderQuery      → GetOrderHandler    → Orders table (той самий, але різна проекція)
```

**Фізичне розділення** — окрема read replica або read store:

```
CreateOrderCommand → Write DB (PostgreSQL, normalized)
                         ↓ events/replication
GetOrderQuery      → Read DB (denormalized view / Redis / Elasticsearch)
```

---

## 4. MediatR — реалізація в .NET

MediatR — медіатор: відправляєш запит/команду → він знаходить handler і викликає.

```
Request → IMediator.Send() → Handler → Response
```

### Встановлення

```csharp
dotnet add package MediatR
```

```csharp
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));
```

### Command

```csharp
// команда — реалізує IRequest<TResponse>
public record CreateOrderCommand(Guid CustomerId, List<OrderItem> Items) : IRequest<Guid>;

// handler
public class CreateOrderHandler(IOrderRepository repo, IEventBus bus)
    : IRequestHandler<CreateOrderCommand, Guid>
{
    public async Task<Guid> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.CustomerId, cmd.Items);
        await repo.AddAsync(order, ct);
        await bus.PublishAsync(new OrderCreated(order.Id), ct);
        return order.Id;
    }
}

// виклик
var orderId = await mediator.Send(new CreateOrderCommand(customerId, items));
```

### Query

```csharp
// запит
public record GetOrderQuery(Guid OrderId) : IRequest<OrderDto?>;

// handler — читає з read-оптимізованої проекції
public class GetOrderHandler(IReadDb db) : IRequestHandler<GetOrderQuery, OrderDto?>
{
    public async Task<OrderDto?> Handle(GetOrderQuery query, CancellationToken ct)
        => await db.Orders
               .Where(o => o.Id == query.OrderId)
               .Select(o => new OrderDto(o.Id, o.Status, o.TotalAmount))
               .FirstOrDefaultAsync(ct);
}

// виклик
var order = await mediator.Send(new GetOrderQuery(orderId));
```

### Notification (broadcast)

```csharp
// notification — INotification, може мати кілька handlers
public record OrderCreated(Guid OrderId) : INotification;

public class SendConfirmationEmail : INotificationHandler<OrderCreated>
{
    public async Task Handle(OrderCreated notification, CancellationToken ct)
        => await _emailService.SendConfirmationAsync(notification.OrderId, ct);
}

public class UpdateInventory : INotificationHandler<OrderCreated>
{
    public async Task Handle(OrderCreated notification, CancellationToken ct)
        => await _inventoryService.ReserveAsync(notification.OrderId, ct);
}

// всі handlers викликаються при Publish
await mediator.Publish(new OrderCreated(orderId));
```

### Pipeline Behaviors — middleware для MediatR

```csharp
// аналог middleware але для MediatR pipelines
public class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var failures = validators
            .Select(v => v.Validate(request))
            .SelectMany(r => r.Errors)
            .Where(e => e is not null)
            .ToList();

        if (failures.Any())
            throw new ValidationException(failures);

        return await next();
    }
}

// реєстрація
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
```

---

## 5. Read Model — денормалізація

Для Queries можна мати окремо оптимізовану проекцію даних:

```csharp
// Write-side: Order aggregate (нормалізований, з бізнес-логікою)
public class Order
{
    public Guid Id { get; private set; }
    public List<OrderItem> Items { get; private set; }
    public decimal TotalAmount => Items.Sum(i => i.Price * i.Quantity);
    // ... методи з інваріантами
}

// Read-side: OrderSummaryDto (денормалізований, готовий для UI)
public record OrderSummaryDto(
    Guid Id,
    string CustomerName,   // join з Customers
    string StatusLabel,    // перетворений статус
    decimal TotalAmount,
    int ItemCount,
    DateTime CreatedAt
);
```

---

## 6. Трейдофи

| Аспект | Плюс | Мінус |
|---|---|---|
| Складність | Чіткий поділ відповідальності | Більше класів і файлів |
| Продуктивність | Read/Write можна оптимізувати окремо | Overhead медіатора |
| Масштабування | Read/Write масштабуються незалежно | Eventual consistency при фізичному розділенні |
| Тестування | Handler-и тестуються ізольовано | Більше unit-тестів |

**Коли НЕ потрібен CQRS:**
- Простий CRUD без складної бізнес-логіки
- Reads і writes однакові за частотою і складністю
- Команда маленька, overhead неприйнятний

---

## 7. Типові питання на співбесіді

**Q: Яка різниця між CQS і CQRS?** CQS (Command Query Separation) — принцип на рівні методів: метод або повертає дані, або змінює стан. CQRS — архітектурний патерн на рівні системи: окремі моделі і потоки для читання і запису.

**Q: Чи обов'язково при CQRS мати різні бази даних?** Ні. CQRS — про логічне розділення handler-ів і моделей, не про фізичну інфраструктуру. Фізичне розділення (read replica, окремий read store) — окреме рішення, яке може поєднуватись з CQRS але не є його частиною.

**Q: Що таке Pipeline Behavior у MediatR?** Middleware для MediatR pipeline — виконується до і після handler для кожного request. Типові use cases: логування, валідація (FluentValidation), caching (для queries), transaction management (для commands).

**Q: Як CQRS пов'язаний з Event Sourcing?** Вони часто поєднуються але незалежні. CQRS — розділення reads/writes. Event Sourcing — спосіб зберігання стану через журнал подій. При їх поєднанні: Command → Events (write side), Events → Read Model проекція (read side).
