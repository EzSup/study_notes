---
tags: [caching, redis, database, backend, distributed]
aliases: [Redis, StackExchange.Redis, IDistributedCache]
---

> Redis — in-memory сховище структур даних. Використовується як кеш, брокер повідомлень, сховище сесій, черга та розподілений замок. Все зберігається в RAM → доступ за мікросекунди.

---

## 1. Структури даних

Ключова перевага Redis над звичайним кешем — різні структури даних з власними командами.

### String

Базовий тип. Зберігає рядки, числа, бінарні дані (до 512 MB).

```
SET user:1:name "Alice"
GET user:1:name           → "Alice"
INCR user:1:loginCount    → 1  (атомарний інкремент)
INCRBY user:1:score 10    → 10
SETEX session:abc 3600 "data"  → встановити з TTL 3600 сек
```

### Hash

Об'єкт з полями — аналог словника. Ефективніше ніж окремий ключ на кожне поле.

```
HSET user:1 name "Alice" age 30 city "Kyiv"
HGET user:1 name          → "Alice"
HGETALL user:1            → { name: Alice, age: 30, city: Kyiv }
HMGET user:1 name city    → ["Alice", "Kyiv"]
HINCRBY user:1 age 1      → 31
HDEL user:1 city
```

### List

Двозв'язний список. Швидкий push/pop з обох кінців. Підходить для черг і стеків.

```
RPUSH queue:tasks "task1" "task2"   → push справа
LPUSH queue:tasks "urgent"          → push зліва
LPOP queue:tasks                    → забрати зліва (FIFO якщо RPUSH+LPOP)
RPOP queue:tasks                    → забрати справа (LIFO)
LLEN queue:tasks                    → довжина
LRANGE queue:tasks 0 -1             → всі елементи
BLPOP queue:tasks 30                → blocking pop (чекати 30 сек)
```

### Set

Невпорядкована колекція унікальних елементів.

```
SADD online:users "u1" "u2" "u3"
SISMEMBER online:users "u1"   → 1 (є)
SMEMBERS online:users          → { u1, u2, u3 }
SCARD online:users             → 3 (кількість)
SREM online:users "u2"
SUNION set1 set2               → об'єднання
SINTER set1 set2               → перетин
SDIFF set1 set2                → різниця
```

### Sorted Set (ZSet)

Унікальні елементи з числовим score — автоматично відсортовані. Ідеально для лідербордів, рейтингів, черг з пріоритетом.

```
ZADD leaderboard 1500 "Alice" 2200 "Bob" 900 "Carol"
ZRANK leaderboard "Alice"         → 1 (0-indexed, від меншого)
ZREVRANK leaderboard "Bob"        → 0 (від більшого)
ZSCORE leaderboard "Alice"        → 1500
ZRANGE leaderboard 0 -1 WITHSCORES → всі від меншого до більшого
ZREVRANGE leaderboard 0 2          → топ-3 від більшого
ZINCRBY leaderboard 100 "Alice"   → 1600
ZRANGEBYSCORE leaderboard 1000 2000 → елементи з score від 1000 до 2000
```

---

## 2. TTL та Expiration

```
SET key "value" EX 60        → встановити з TTL 60 сек
SET key "value" PX 5000      → TTL в мілісекундах
EXPIRE key 300               → встановити TTL на існуючий ключ
TTL key                      → скільки секунд залишилось (-1 = без TTL, -2 = не існує)
PERSIST key                  → видалити TTL (зробити постійним)
```

---

## 3. Патерни кешування

### Cache-Aside (найпоширеніший)

Додаток сам керує кешем. Читання: спочатку кеш → якщо miss → БД → записати в кеш.

```
Cache miss                     Cache hit
   ↓                              ↓
App → Redis (miss) → DB → Redis  App → Redis (hit) → App
                     ↑
               записати в кеш
```

```csharp
public async Task<User?> GetUserAsync(int id)
{
    var cached = await _cache.GetStringAsync($"user:{id}");
    if (cached is not null)
        return JsonSerializer.Deserialize<User>(cached);

    var user = await _db.Users.FindAsync(id);
    if (user is not null)
        await _cache.SetStringAsync($"user:{id}",
            JsonSerializer.Serialize(user),
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10) });

    return user;
}
```

### Write-Through

При кожному записі в БД — одразу оновлюється кеш. Кеш завжди актуальний, але кожен запис дорожчий.

```
App → DB + Cache (одночасно)
```

### Write-Behind (Write-Back)

Запис іде в кеш, а в БД — асинхронно пізніше. Швидкий запис, але ризик втрати даних.

### Invalidation (інвалідація)

```csharp
// при оновленні — видалити кеш
await _db.SaveChangesAsync();
await _cache.RemoveAsync($"user:{id}");
```

---

## 4. .NET інтеграція

### IDistributedCache (абстракція)

Вбудована абстракція ASP.NET Core. Підходить для простого кешування рядків/байтів.

```csharp
// реєстрація
builder.Services.AddStackExchangeRedisCache(opt =>
{
    opt.Configuration = "localhost:6379";
    opt.InstanceName = "myapp:";
});

// використання
public class MyService(IDistributedCache cache)
{
    public async Task<string?> GetAsync(string key)
        => await cache.GetStringAsync(key);

    public async Task SetAsync(string key, string value)
        => await cache.SetStringAsync(key, value,
               new DistributedCacheEntryOptions
               {
                   AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
               });
}
```

### StackExchange.Redis (прямий клієнт)

Дає доступ до всіх структур даних і команд Redis.

```csharp
// реєстрація
builder.Services.AddSingleton<IConnectionMultiplexer>(
    ConnectionMultiplexer.Connect("localhost:6379"));

// використання
public class LeaderboardService(IConnectionMultiplexer redis)
{
    private IDatabase Db => redis.GetDatabase();

    public async Task AddScoreAsync(string user, double score)
        => await Db.SortedSetAddAsync("leaderboard", user, score);

    public async Task<SortedSetEntry[]> GetTop10Async()
        => await Db.SortedSetRangeByRankWithScoresAsync("leaderboard", 0, 9, Order.Descending);

    public async Task<bool> SetIfNotExistsAsync(string key, string value, TimeSpan ttl)
        => await Db.StringSetAsync(key, value, ttl, When.NotExists); // SET ... NX
}
```

> `IConnectionMultiplexer` — singleton. `IDatabase` — легковагий об'єкт, не пул з'єднань, можна отримувати без DI.

---

## 5. Розподілений замок (Distributed Lock)

Проблема: в distributed системі кілька інстанцій можуть одночасно виконувати одну операцію.

```csharp
// атомарна операція: SET key value NX EX seconds
// NX = тільки якщо ключ НЕ існує
var lockKey = "lock:process-payment";
var lockValue = Guid.NewGuid().ToString(); // унікальний власник
var acquired = await db.StringSetAsync(lockKey, lockValue, TimeSpan.FromSeconds(30), When.NotExists);

if (!acquired)
    throw new Exception("Could not acquire lock");

try
{
    await ProcessPaymentAsync();
}
finally
{
    // видалити замок тільки якщо він наш (перевіряє значення)
    var script = @"
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end";
    await db.ScriptEvaluateAsync(script, new RedisKey[] { lockKey }, new RedisValue[] { lockValue });
}
```

> Бібліотека **RedLock.net** реалізує Redlock алгоритм для замків на кластері (N вузлів, кворум).

---

## 6. Pub/Sub

```csharp
var sub = redis.GetSubscriber();

// підписник
await sub.SubscribeAsync("notifications", (channel, message) =>
{
    Console.WriteLine($"Got: {message}");
});

// видавець
await sub.PublishAsync("notifications", "Hello!");
```

> Redis Pub/Sub — fire-and-forget, повідомлення не зберігаються. Якщо підписник offline — повідомлення втрачено. Для надійної черги — краще Redis Streams або RabbitMQ.

---

## 7. Persistence

| Режим | Що робить | Ризик втрати даних |
|---|---|---|
| **None** | Тільки RAM | Все при перезапуску |
| **RDB** (snapshot) | Знімок на диск кожні N сек або M операцій | Дані з останнього снімку |
| **AOF** (append-only file) | Записує кожну операцію | Мінімальний (налаштовується) |
| **RDB + AOF** | Обидва режими | Мінімальний |

```
# redis.conf
save 900 1      # RDB: якщо 1 зміна за 900 сек
appendonly yes  # увімкнути AOF
appendfsync everysec  # flush на диск кожну секунду
```

---

## 8. Конвенції іменування ключів

```
# формат: object-type:id:field
user:1:profile
user:1:sessions
order:42:items
session:abc123

# namespace для ізоляції між додатками
myapp:user:1
myapp:cache:products
```

- Розділювач `:` — стандарт (Redis розуміє і відображає як дерево у RedisInsight)
- Не роби ключі занадто довгими — вони зберігаються в пам'яті

---

## 9. Типові питання на співбесіді

**Q: Яка різниця між IDistributedCache і StackExchange.Redis?** `IDistributedCache` — абстракція ASP.NET Core, тільки byte[]/string, для простого кешування, не прив'язана до Redis. StackExchange.Redis — прямий клієнт, дає доступ до всіх структур даних (Hash, ZSet, Set тощо), pipeline, Lua scripts.

**Q: Яка різниця між RDB і AOF?** RDB — повний знімок бази кожні N секунд (компактний, швидке відновлення, але можна втратити дані між знімками). AOF — лог кожної операції (менша втрата даних, але більший розмір файлу і повільніше відновлення).

**Q: Як реалізувати distributed lock в Redis?** Команда `SET key value NX EX seconds` — атомарно встановлює ключ тільки якщо він не існує, з TTL. При звільненні замку — перевіряти що значення твоє через Lua script (атомарне порівняння + видалення).

**Q: Яка різниця між Set і Sorted Set?** Set — унікальні елементи без порядку. Sorted Set — унікальні елементи з числовим score, автоматично відсортовані. ZSet підтримує `ZRANGE`, `ZRANK`, `ZRANGEBYSCORE` — ідеально для лідербордів і черг з пріоритетом.

**Q: Чим Redis Pub/Sub відрізняється від RabbitMQ?** Redis Pub/Sub — fire-and-forget, повідомлення не персистуються, subscriber що offline — пропустить повідомлення. RabbitMQ — надійна доставка, черги зберігають повідомлення, ACK, retry, DLQ. Для критичних повідомлень — RabbitMQ.
