---
tags: [architecture, event-sourcing, events, patterns, backend]
aliases: [Event Sourcing, EventStore, Projections, Snapshots]
---

> Event Sourcing — замість зберігання **поточного стану** (UPDATE рядка в БД) зберігаємо **послідовність подій** що до цього стану привели. Стан — результат відтворення (replay) всіх подій.

---

## 1. Традиційний підхід vs Event Sourcing

**CRUD — зберігаємо стан:**

```sql
-- після кожної зміни перезаписуємо рядок
UPDATE orders SET status = 'shipped', updated_at = NOW() WHERE id = 42;
-- попередній стан втрачено
```

**Event Sourcing — зберігаємо події:**

```
events таблиця:
│ orderId │ type             │ data                           │ timestamp           │
│ 42      │ OrderCreated     │ { customerId: 1, items: [...] }│ 2024-01-01 10:00:00 │
│ 42      │ PaymentConfirmed │ { amount: 150.00 }             │ 2024-01-01 10:05:00 │
│ 42      │ OrderShipped     │ { trackingNumber: "UA123" }    │ 2024-01-02 09:00:00 │
```

Поточний стан = replay всіх подій для `orderId = 42`.

---

## 2. Aggregate та Event Stream

**Aggregate** — кластер пов'язаних об'єктів з єдиним коренем (Aggregate Root), який захищає інваріанти.

**Event Stream** — послідовність подій для конкретного aggregate.

```csharp
// визначення подій
public record OrderCreated(Guid OrderId, Guid CustomerId, List<OrderItem> Items);
public record PaymentConfirmed(Guid OrderId, decimal Amount);
public record OrderShipped(Guid OrderId, string TrackingNumber);
public record OrderCancelled(Guid OrderId, string Reason);

// aggregate відновлює стан через Apply
public class Order
{
    public Guid Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public decimal TotalAmount { get; private set; }
    private readonly List<object> _uncommittedEvents = new();

    // відновлення з подій (replay)
    public static Order Rehydrate(IEnumerable<object> events)
    {
        var order = new Order();
        foreach (var e in events)
            order.Apply(e);
        return order;
    }

    // бізнес-операція: валідація → генерація події
    public void Ship(string trackingNumber)
    {
        if (Status != OrderStatus.Paid)
            throw new InvalidOperationException("Only paid orders can be shipped");

        var @event = new OrderShipped(Id, trackingNumber);
        Apply(@event);
        _uncommittedEvents.Add(@event);
    }

    public void Cancel(string reason)
    {
        if (Status == OrderStatus.Shipped)
            throw new InvalidOperationException("Cannot cancel shipped order");

        var @event = new OrderCancelled(Id, reason);
        Apply(@event);
        _uncommittedEvents.Add(@event);
    }

    // Apply — чиста функція, тільки мутує стан, без побічних ефектів
    private void Apply(object @event) => Apply((dynamic)@event);

    private void Apply(OrderCreated e)
    {
        Id = e.OrderId;
        Status = OrderStatus.Pending;
        TotalAmount = e.Items.Sum(i => i.Price * i.Quantity);
    }

    private void Apply(PaymentConfirmed e) => Status = OrderStatus.Paid;
    private void Apply(OrderShipped e) => Status = OrderStatus.Shipped;
    private void Apply(OrderCancelled e) => Status = OrderStatus.Cancelled;

    public IReadOnlyList<object> UncommittedEvents => _uncommittedEvents;
    public void ClearUncommittedEvents() => _uncommittedEvents.Clear();
}
```

### Event Store — запис і читання

```csharp
public interface IEventStore
{
    Task AppendAsync(string streamId, IEnumerable<object> events, long expectedVersion);
    Task<IEnumerable<object>> LoadAsync(string streamId);
}

public class OrderRepository(IEventStore store)
{
    public async Task<Order> GetAsync(Guid orderId)
    {
        var events = await store.LoadAsync($"order-{orderId}");
        return Order.Rehydrate(events);
    }

    public async Task SaveAsync(Order order)
    {
        var events = order.UncommittedEvents;
        await store.AppendAsync($"order-{orderId}", events, expectedVersion: -1);
        order.ClearUncommittedEvents();
    }
}
```

---

## 3. Projections — Read Models

Events зберігаються для write-side. Для ефективного читання — будуємо **проекції** (read models).

```
Events Stream → [Projection] → Read Model (таблиця/колекція/кеш)
```

```csharp
// projection — обробляє події і будує read model
public class OrderSummaryProjection
{
    private readonly IReadDb _db;

    public async Task On(OrderCreated e)
        => await _db.OrderSummaries.InsertAsync(new OrderSummary
        {
            Id = e.OrderId,
            CustomerId = e.CustomerId,
            Status = "Pending",
            TotalAmount = e.Items.Sum(i => i.Price * i.Quantity),
            CreatedAt = DateTime.UtcNow
        });

    public async Task On(OrderShipped e)
        => await _db.OrderSummaries.UpdateAsync(e.OrderId,
               s => s.Status = "Shipped");

    public async Task On(OrderCancelled e)
        => await _db.OrderSummaries.UpdateAsync(e.OrderId,
               s => s.Status = "Cancelled");
}
```

**Ключова перевага:** якщо потрібна нова проекція — replay всіх подій з початку і будуй нову таблицю. Дані ніколи не втрачаються.

---

## 4. Snapshots — оптимізація Replay

Якщо aggregate накопичив тисячі подій — replay кожного разу стає повільним.

**Snapshot** — збережений стан aggregate на певний момент (кожні N подій):

```
Events:  e1 e2 e3 ... e999 e1000 | e1001 e1002 e1003
                              ↑
                         Snapshot @ version 1000

// При завантаженні:
1. Завантажити останній snapshot (стан @ v1000)
2. Replay тільки подій після v1000 (e1001, e1002, e1003)
```

```csharp
public async Task<Order> GetAsync(Guid orderId)
{
    var snapshot = await _snapshotStore.GetLatestAsync($"order-{orderId}");

    var fromVersion = snapshot?.Version ?? 0;
    var events = await _eventStore.LoadAsync($"order-{orderId}", fromVersion);

    var order = snapshot is not null
        ? Order.FromSnapshot(snapshot)
        : new Order();

    foreach (var e in events)
        order.Apply(e);

    return order;
}
```

---

## 5. Temporal Queries — запити в часі

Одна з найбільших переваг Event Sourcing — можна відтворити стан на **будь-який момент в минулому**:

```csharp
// який був стан замовлення вчора о 14:00?
var events = await store.LoadAsync("order-42", until: yesterday14h);
var pastState = Order.Rehydrate(events);
```

Це неможливо в CRUD де поточний стан перезаписується.

---

## 6. Eventual Consistency між Write і Read Side

```
Command → Aggregate → Events saved → [async] → Projection updated → Query reads
              ↑                          ↑
           immediate                 small delay
```

Read model **трохи відстає** від write side. Для більшості бізнес-задач це прийнятно. Але деякі сценарії потребують strongly consistent reads — тоді читати напряму з event stream або використовувати synchronous projection.

---

## 7. Event Store — реалізації

| Рішення | Опис |
|---|---|
| **EventStoreDB** | Спеціалізована БД для Event Sourcing, streams, subscriptions |
| **Marten** | PostgreSQL як event store + document DB для .NET |
| **PostgreSQL вручну** | Проста таблиця `events (stream_id, version, type, data, timestamp)` |
| **Azure Cosmos DB** | Change feed як event stream |

### Marten — найпростіший старт для .NET

```csharp
// реєстрація
builder.Services.AddMarten(opt =>
{
    opt.Connection(connectionString);
    opt.Projections.Add<OrderSummaryProjection>(ProjectionLifecycle.Async);
})
.UseLightweightSessions()
.AddAsyncDaemon(DaemonMode.HotCold); // фоновий процес для проекцій

// запис events
await using var session = store.LightweightSession();
session.Events.Append(orderId, new OrderCreated(...), new PaymentConfirmed(...));
await session.SaveChangesAsync();

// завантаження aggregate
var order = await session.Events.AggregateStreamAsync<Order>(orderId);
```

---

## 8. Трейдофи

| Аспект | Плюс | Мінус |
|---|---|---|
| Аудит | Повна історія змін безкоштовно | Складніший запит поточного стану |
| Відлагодження | Точне відтворення будь-якого стану | Крива навчання для команди |
| Temporal queries | Стан на будь-який момент | Replay може бути повільним (без snapshots) |
| Зміна схеми | Додаєш нові поля не видаляючи старі | Event versioning ускладнюється |
| Read performance | Проекції оптимізовані під конкретний запит | Eventual consistency між write і read |

**Коли Event Sourcing підходить:**
- Фінансові системи (повний аудит обов'язковий)
- Складні бізнес-процеси де важлива історія змін
- Системи де потрібен undo/redo
- Domain-driven design з багатими aggregate

**Коли Event Sourcing зайвий:**
- Простий CRUD без складної логіки
- Дані де не важлива історія
- Команда не знайома з патерном

---

## 9. Типові питання на співбесіді

**Q: Яка головна відмінність Event Sourcing від CRUD?** CRUD зберігає поточний стан (перезаписує рядок). Event Sourcing зберігає всі події що привели до стану. Стан — похідна від replay подій. Дані ніколи не видаляються.

**Q: Що таке проекція?** Проекція — read model побудована з event stream. Обробляє події і будує оптимізовану для читання структуру (таблиця, документ, кеш). Можна мати кілька проекцій для одного event stream під різні запити. При необхідності — rebuild проекції з нуля.

**Q: Навіщо потрібні Snapshots?** Якщо aggregate накопичив тисячі подій — replay кожного разу повільний. Snapshot — збережений стан на певну версію. При завантаженні: restore snapshot + replay тільки нових подій.

**Q: Як вирішується eventual consistency між write і read side?** Read model трохи відстає від write side (мілісекунди-секунди при async projection). Більшість UI це допускає. Для критичних сценаріїв: 1) синхронна проекція, 2) читання напряму з event stream, 3) повернути клієнту очікуваний стан після команди (оптимістичне UI).

**Q: Як версіонувати події при зміні схеми?** Варіанти: 1) Upcasting — при завантаженні старих подій трансформувати їх до нового формату. 2) Weak schema — додавати нові поля опціональними. 3) Нові event типи замість зміни старих. 4) Версіоновані типи (`OrderCreatedV2`).
