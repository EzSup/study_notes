## 1. Навіщо Message Broker?

### Проблема синхронної комунікації

У мікросервісній архітектурі сервіси часто спілкуються через HTTP. Це створює **coupling по часу**: якщо сервіс B недоступний — сервіс A отримує помилку, навіть якщо B відновиться за секунду.

```
Service A  →  HTTP  →  Service B (впав) → ❌ Exception
```

### Рішення — Message Broker

RabbitMQ виступає посередником. Producer відправляє повідомлення в брокер і забуває про нього. Consumer отримає повідомлення коли буде готовий — навіть якщо він зараз недоступний.

```
Producer → [RabbitMQ] → Consumer
```

**Ключові переваги:**

- **Decoupling** — Producer не знає скільки консюмерів існує і чи вони живі
- **Fault tolerance** — повідомлення зберігаються у черзі під час недоступності консюмера
- **Async processing** — "fire and forget", латентність Producer не залежить від Consumer
- **Load balancing** — кілька інстанцій Consumer розподіляють навантаження

### AMQP протокол

RabbitMQ використовує **AMQP (Advanced Message Queuing Protocol)** — бінарний протокол поверх TCP.

На відміну від HTTP, AMQP є **stateful**: одне TCP-з'єднання (`IConnection`) мультиплексується через **channels** (`IModel` / `IChannel`). Рекомендована практика — один `IConnection` на процес, окремий `IChannel` на потік.

```
TCP Connection
  └── Channel 1  (thread 1)
  └── Channel 2  (thread 2)
  └── Channel 3  (thread 3)
```

---

## 2. Архітектура та ключові компоненти

### Повний шлях повідомлення

```
Producer
  → Connection / Channel
    → Exchange         ← routing logic
      → Binding        ← routing rule (key / pattern)
        → Queue        ← storage
          → Consumer
```

### Exchange

**Exchange** — маршрутизатор. Він приймає повідомлення від Producer і вирішує в яку чергу його відправити на основі **routing key** та **binding rules**.

> ⚠️ Exchange **не зберігає** повідомлення. Якщо немає жодного binding — повідомлення дропається (або повертається Producer якщо `mandatory=true`).

**Default Exchange** — неіменований `""`, автоматично прив'язаний до кожної черги по її імені. Тобто `routingKey = "my-queue"` напряму доставить у чергу `my-queue` без явного Exchange. Зручно для простих сценаріїв.

### Queue

**Queue** — буфер зберігання повідомлень. Повідомлення зберігаються у порядку FIFO і чекають поки Consumer їх забере.

Властивості черги при декларації:

- `durable` — черга виживе після рестарту RabbitMQ
- `exclusive` — доступна лише поточному з'єднанню, видаляється після його закриття
- `autoDelete` — видаляється коли від'єднається останній Consumer
- `arguments` — розширені налаштування (TTL, DLX, max-length тощо)

### Binding

**Binding** — логічний зв'язок між Exchange і Queue. Може містити **routing key** або **pattern** в залежності від типу Exchange.

```
Exchange ──(binding: routing key)──→ Queue
```

---

## 3. Типи Exchange

### Direct

Routing key повідомлення повинен **точно збігатися** з routing key binding.

```
Exchange(direct)
  ├── binding: "order.created"  →  Queue A
  └── binding: "order.updated"  →  Queue B
```

Використання: конкретні події → конкретний обробник.

### Fanout

Routing key **ігнорується**. Повідомлення відправляється у **всі** прив'язані черги.

```
Exchange(fanout)
  ├──────────────────→  Queue A (notification service)
  ├──────────────────→  Queue B (analytics service)
  └──────────────────→  Queue C (audit service)
```

Використання: broadcast — інвалідація кешу, push-сповіщення всім підписникам.

### Topic

Гнучка маршрутизація на основі **wildcard-патернів**. Routing key — рядок слів розділених крапкою.

**Wildcards:**

- `*` — рівно **одне** слово між крапками
- `#` — **нуль або більше** слів

```
Routing key: "order.eu.created"

  "order.*.*"       → ✅ match  (два слова після "order")
  "order.#"         → ✅ match  (будь-яка кількість слів після "order")
  "order.*.created" → ✅ match  (одне слово між "order" і "created")
  "*.eu.*"          → ✅ match  
  "order.created"   → ❌ no match (очікує тільки одне слово після "order")
  "#.created"       → ✅ match  (закінчується на "created")
```

Використання: динамічна обробка подій, логування по рівнях (`logs.error`, `logs.warn.#`).

### Headers

Routing на основі **message headers** замість routing key. Рідко використовується через складність.

---

## 4. Durability — три рівні стійкості

Щоб повідомлення пережило рестарт RabbitMQ, потрібні **всі три умови**:

|Рівень|Налаштування|Що відбувається без нього|
|---|---|---|
|Durable Exchange|`durable: true`|Exchange зникає після рестарту|
|Durable Queue|`durable: true`|Черга зникає після рестарту|
|Persistent Message|`deliveryMode: 2`|Повідомлення зберігається лише в RAM|

> ⚠️ Якщо хоча б одна умова не виконана — повідомлення може бути втрачено.

**В .NET (RabbitMQ.Client):**

```csharp
// Durable Exchange та Queue
channel.ExchangeDeclare("orders-exchange", ExchangeType.Direct, durable: true);
channel.QueueDeclare("orders", durable: true, exclusive: false, autoDelete: false);

// Persistent Message
var props = channel.CreateBasicProperties();
props.Persistent = true; // deliveryMode = 2

channel.BasicPublish(
    exchange: "orders-exchange",
    routingKey: "order.created",
    basicProperties: props,
    body: Encoding.UTF8.GetBytes(messageJson)
);
```

**RAM vs Disk:**

- За замовчуванням повідомлення зберігаються в RAM для швидкості
- При `deliveryMode: 2` RabbitMQ записує на диск перед підтвердженням
- Це повільніше, але безпечно при збоях

---

## 5. ACK / NACK — підтвердження обробки

### autoAck — небезпечний режим

При `autoAck = true` RabbitMQ видаляє повідомлення **одразу після доставки**, ще до того як Consumer обробив його. Якщо Consumer впаде під час обробки — повідомлення буде втрачено.

```
Queue → [deliver] → Consumer crashes → ❌ message lost
```

### Manual ACK — рекомендований підхід

Consumer підтверджує повідомлення лише після успішної обробки:

```csharp
var consumer = new EventingBasicConsumer(channel);
consumer.Received += (model, ea) =>
{
    try
    {
        var body = ea.Body.ToArray();
        ProcessMessage(body); // бізнес-логіка
        
        channel.BasicAck(ea.DeliveryTag, multiple: false); // ✅ успіх
    }
    catch (Exception ex)
    {
        // requeue: true  → повернути в чергу (спробувати ще раз)
        // requeue: false → відправити в DLQ
        channel.BasicNack(ea.DeliveryTag, multiple: false, requeue: false);
    }
};

channel.BasicConsume("orders", autoAck: false, consumer: consumer);
```

### BasicAck vs BasicNack vs BasicReject

|Метод|Параметр `multiple`|Опис|
|---|---|---|
|`BasicAck`|так|Підтвердити одне або всі попередні повідомлення|
|`BasicNack`|так|Відхилити одне або кілька (з можливістю requeue)|
|`BasicReject`|ні|Відхилити одне повідомлення|

**`multiple: true`** підтверджує всі повідомлення з `DeliveryTag` ≤ поточного. Корисно для batch-обробки.

### Publisher Confirms

З боку Producer — аналог ACK. RabbitMQ підтверджує що повідомлення збережено в черзі:

```csharp
channel.ConfirmSelect(); // увімкнути режим підтвердження

channel.BasicPublish(...);
channel.WaitForConfirmsOrDie(timeout: TimeSpan.FromSeconds(5));
```

---

## 6. prefetchCount і QoS

**Проблема без prefetch:** якщо на одну чергу підписано кілька Consumer-ів, RabbitMQ може відправити всі повідомлення першому доступному Consumer (якщо він швидко з'єднався), навантаживши тільки його.

**`BasicQos`** обмежує скільки повідомлень може бути **"in-flight"** (доставлено але ще не ACK) для одного Consumer:

```csharp
// Не більше 10 повідомлень одночасно на цей channel
channel.BasicQos(prefetchSize: 0, prefetchCount: 10, global: false);
```

- `prefetchCount: 1` — найбільш рівномірний розподіл (fair dispatch), але найповільніший
- `prefetchCount: 10-50` — компроміс між рівномірністю та throughput
- `global: false` — ліміт на кожен Consumer окремо
- `global: true` — ліміт на весь channel

> ⚠️ Завжди встановлюй `prefetchCount` при використанні manual ACK — без нього один Consumer може "захопити" всю чергу.

---

## 7. Dead Letter Queue (DLQ)

### Що таке Poison Message?

**Poison message** — повідомлення яке не вдається обробити (наприклад, некоректний формат, баг у логіці). Якщо повертати його в чергу (`requeue: true`) нескінченно — Consumer буде витрачати ресурси на безуспішні спроби.

### Архітектура DLQ

DLQ — звичайна черга, яка отримує "мертві" повідомлення через **Dead Letter Exchange (DLX)**:

```
Main Queue → [NACK requeue:false] → DLX → Dead Letter Queue
```

### Налаштування DLQ

```csharp
// 1. Оголосити DLX і DLQ
channel.ExchangeDeclare("dlx", ExchangeType.Direct, durable: true);
channel.QueueDeclare("orders.dead", durable: true, exclusive: false, autoDelete: false);
channel.QueueBind("orders.dead", "dlx", routingKey: "dead.orders");

// 2. Прив'язати DLX до основної черги через аргументи
var args = new Dictionary<string, object>
{
    { "x-dead-letter-exchange", "dlx" },
    { "x-dead-letter-routing-key", "dead.orders" },
    { "x-message-ttl", 60000 },   // опціонально: TTL в мс
    { "x-max-length", 10000 }      // опціонально: ліміт розміру черги
};

channel.QueueDeclare("orders", durable: true, exclusive: false, autoDelete: false, arguments: args);
```

### Коли повідомлення потрапляє в DLQ

|Причина|Умова|
|---|---|
|Відхилено Consumer-ом|`BasicNack` / `BasicReject` з `requeue: false`|
|Вийшов TTL|`x-message-ttl` на черзі або `expiration` у props|
|Черга переповнена|Досягнуто `x-max-length`|

### Обробка DLQ

DLQ дозволяє:

1. **Ізолювати** проблемні повідомлення — основна черга не блокується
2. **Інспектувати** — подивитись що саме пішло не так
3. **Повторно відправити** після виправлення багу

---

## 8. Quorum Queues vs Classic Queues

|Характеристика|Classic Queue|Quorum Queue|
|---|---|---|
|Реплікація|Mirrored (deprecated)|Raft consensus|
|Data safety|Слабша|Strong consistency|
|Performance|Вища для single node|Трохи нижча|
|Підтримка x-max-length|Так|Так|
|Підтримка non-durable|Так|Ні (завжди durable)|
|Рекомендація|Dev / single node|Production clusters|

**Quorum Queue** використовує алгоритм **Raft** для консенсусу між вузлами кластера. Якщо один вузол падає — черга залишається доступною поки є кворум (більшість вузлів).

```csharp
// Оголошення Quorum Queue
var args = new Dictionary<string, object>
{
    { "x-queue-type", "quorum" }
};

channel.QueueDeclare("orders", durable: true, exclusive: false, autoDelete: false, arguments: args);
```

> ✅ Для production кластера — завжди використовуй Quorum Queues.

---

## 9. Архітектурні патерни

### Competing Consumers (Load Balancing)

Кілька інстанцій одного сервісу слухають **одну чергу**. RabbitMQ розподіляє повідомлення між ними (Round-Robin з урахуванням prefetch).

```
Queue ──→ Consumer Instance 1
      ──→ Consumer Instance 2
      ──→ Consumer Instance 3
```

Використання: горизонтальне масштабування, розподіл важких задач.

### Pub/Sub

Fanout Exchange + окрема черга для кожного підписника.

```
Fanout Exchange
  ──→ Queue (Notification Service)
  ──→ Queue (Analytics Service)
  ──→ Queue (Email Service)
```

Кожен сервіс обробляє повідомлення незалежно.

### Request/Reply (RPC over RabbitMQ)

Producer відправляє повідомлення з тимчасовою чергою для відповіді:

```csharp
// Producer
var replyQueueName = channel.QueueDeclare().QueueName; // тимчасова черга

var props = channel.CreateBasicProperties();
props.ReplyTo = replyQueueName;
props.CorrelationId = Guid.NewGuid().ToString(); // для зіставлення відповіді

channel.BasicPublish("", "rpc-queue", props, body);

// Consumer відповідає в replyTo чергу
var replyProps = channel.CreateBasicProperties();
replyProps.CorrelationId = ea.BasicProperties.CorrelationId;
channel.BasicPublish("", ea.BasicProperties.ReplyTo, replyProps, responseBody);
```

Рідко використовується — порушує async природу брокера. Краще використовувати async callbacks.

### Retry Pattern

Автоматичні повторні спроби обробки з затримкою через DLQ + TTL:

```
Main Queue → [NACK] → DLX → Retry Queue (TTL: 5s) → [expire] → DLX → Main Queue
```

```csharp
// Retry queue з TTL і поверненням в основну чергу
var retryArgs = new Dictionary<string, object>
{
    { "x-dead-letter-exchange", "" },          // default exchange
    { "x-dead-letter-routing-key", "orders" }, // повернути в основну чергу
    { "x-message-ttl", 5000 }                  // затримка 5 секунд
};
channel.QueueDeclare("orders.retry", durable: true, exclusive: false, autoDelete: false, arguments: retryArgs);
```

---

## 10. Корисні налаштування та best practices

### Message TTL

```csharp
// TTL для всієї черги
var args = new Dictionary<string, object> { { "x-message-ttl", 30000 } };
channel.QueueDeclare("orders", arguments: args);

// TTL для конкретного повідомлення
var props = channel.CreateBasicProperties();
props.Expiration = "30000"; // мілісекунди, як рядок (!)
```

### Lazy Queues

Зберігають повідомлення на диск одразу (не в RAM). Корисно для черг з великим об'ємом:

```csharp
var args = new Dictionary<string, object> { { "x-queue-mode", "lazy" } };
```

### Пріоритети повідомлень

```csharp
var args = new Dictionary<string, object> { { "x-max-priority", 10 } };
channel.QueueDeclare("priority-orders", arguments: args);

var props = channel.CreateBasicProperties();
props.Priority = 5; // від 0 до x-max-priority
```

### Management UI

RabbitMQ має вбудований веб-інтерфейс на порту `15672` (логін/пароль: `guest/guest` для localhost):

```bash
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:3-management
```

---

## 11. Типові питання на співбесіді

**Q: У чому різниця між Exchange і Queue?** Exchange — маршрутизатор, не зберігає повідомлення. Queue — буфер зберігання. Exchange приймає повідомлення від Producer і на основі routing rules відправляє в одну або кілька Queue.

**Q: Що таке binding?** Binding — зв'язок між Exchange і Queue з правилом маршрутизації (routing key або pattern).

**Q: Яка різниця між autoAck і manual ACK?** `autoAck` видаляє повідомлення одразу після доставки. Manual ACK — Consumer підтверджує після успішної обробки. AutoAck ризикує втратою повідомлення якщо Consumer впаде під час обробки.

**Q: Що таке prefetchCount і навіщо він потрібен?** Ліміт кількості "in-flight" повідомлень (доставлених але ще не ACK) для Consumer. Без нього один Consumer може "захопити" всю чергу, порушуючи load balancing.

**Q: Як забезпечити що повідомлення не загубиться при рестарті RabbitMQ?** Три умови: durable Exchange + durable Queue + persistent Message (`deliveryMode: 2`).

**Q: Що таке Dead Letter Queue і навіщо вона потрібна?** Черга для повідомлень які не вдалося обробити (NACK з requeue:false, TTL, overflow). Ізолює проблемні повідомлення від основного потоку, дозволяє інспектувати та повторно відправляти після виправлення.

**Q: Яка різниця між Direct, Fanout і Topic Exchange?**

- Direct: точний match routing key → одна черга
- Fanout: broadcast у всі черги, routing key ігнорується
- Topic: wildcard-паттерни (`*` — одне слово, `#` — нуль або більше слів)

**Q: Чому Quorum Queues краще Classic в production?** Quorum Queues використовують Raft для реплікації між вузлами кластера, забезпечуючи strong consistency. Classic mirrored queues (deprecated) мали слабші гарантії при split-brain сценаріях.