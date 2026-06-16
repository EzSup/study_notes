---
tags: [csharp, dotnet, linq, performance, advanced]
aliases: [LINQ Advanced, IEnumerable vs IQueryable, Deferred Execution]
---

> Неочевидні аспекти LINQ: пастки відкладеного виконання, різниця IEnumerable/IQueryable, методи що рідко використовуються але корисні.

---

## 1. Відкладене виконання (Deferred Execution)

LINQ-запит **не виконується** при визначенні — тільки при ітерації (`.ToList()`, `foreach`, `.First()` тощо).

```csharp
var query = users.Where(u => u.Age > 18); // ← нічого не відбувається
// ...
var result = query.ToList(); // ← тут виконується фільтрація
```

### Пастка: зміна колекції між визначенням і виконанням

```csharp
var numbers = new List<int> { 1, 2, 3 };
var query = numbers.Where(n => n > 1); // відкладено

numbers.Add(4); // додаємо після визначення query

var result = query.ToList(); // → [2, 3, 4] — враховує зміни!
```

### Пастка: закриття (closure) в циклі

```csharp
// ❌ класична помилка — всі query посилаються на одну змінну i
var queries = new List<IEnumerable<int>>();
for (int i = 0; i < 3; i++)
    queries.Add(numbers.Where(n => n == i)); // i — captured by reference

// при виконанні i вже = 3 (кінець циклу)
queries[0].ToList(); // → [] замість [0]

// ✅ зафіксувати значення в локальну змінну
for (int i = 0; i < 3; i++)
{
    int captured = i;
    queries.Add(numbers.Where(n => n == captured));
}
```

---

## 2. IEnumerable\<T\> vs IQueryable\<T\>

Найважливіша різниця в LINQ:

| | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| Де виконується | **У пам'яті** клієнта (LINQ to Objects) | **На сервері** (SQL, MongoDB) |
| Лямбда | `Func<T, bool>` (delegate) | `Expression<Func<T, bool>>` (expression tree) |
| Коли використовується | Колекції в пам'яті, після `.AsEnumerable()` | EF Core, MongoDB, будь-який LINQ provider |

```csharp
// IQueryable — фільтр йде в SQL (WHERE на сервері)
var query = dbContext.Users
    .Where(u => u.Age > 18)       // → SQL: WHERE Age > 18
    .OrderBy(u => u.Name)         // → SQL: ORDER BY Name
    .Take(10);                     // → SQL: TOP 10

// IEnumerable — всі дані витягуються в пам'ять, ПОТІМ фільтрація
var query = dbContext.Users
    .AsEnumerable()               // ← переключення на in-memory
    .Where(u => u.Age > 18)       // виконується в C#
    .OrderBy(u => u.Name)
    .Take(10);
// → SQL: SELECT * FROM Users (всі записи!) + фільтрація в пам'яті
```

### Коли AsEnumerable() потрібний (навмисно)

```csharp
// EF Core не вміє транслювати кастомний C# метод в SQL
var result = dbContext.Users
    .Where(u => u.Age > 18)        // → SQL фільтрація ✅
    .AsEnumerable()                 // завантажити в пам'ять
    .Where(u => MyCustomCheck(u))  // виконати кастомний C# метод
    .ToList();
```

---

## 3. Multiple Enumeration — подвійна ітерація

```csharp
// ❌ IEnumerable<T> від lazy джерела — запит виконується ДВІЧІ
IEnumerable<User> users = dbContext.Users.Where(u => u.IsActive);

var count = users.Count();   // запит 1
var list  = users.ToList();  // запит 2

// ✅ матеріалізувати один раз
var users = dbContext.Users.Where(u => u.IsActive).ToList();
var count = users.Count;     // на List — без запиту
```

Компілятор C# (`async/await` в .NET 7+) або Roslyn аналізатори можуть попередити про це. ReSharper/Rider показують "Possible multiple enumeration of IEnumerable".

---

## 4. SelectMany — flatten вкладених колекцій

```csharp
var orders = new List<Order>
{
    new Order { Items = ["book", "pen"] },
    new Order { Items = ["laptop"] }
};

// Select — список списків
var nested = orders.Select(o => o.Items);
// → [["book","pen"], ["laptop"]]

// SelectMany — плоский список
var flat = orders.SelectMany(o => o.Items);
// → ["book", "pen", "laptop"]

// SelectMany з трансформацією — (джерело, елемент)
var orderItems = orders.SelectMany(
    o => o.Items,
    (order, item) => new { order.Id, item });
// → [{ Id:1, item:"book" }, { Id:1, item:"pen" }, { Id:2, item:"laptop" }]
```

---

## 5. GroupBy — важлива деталь

`GroupBy` повертає `IEnumerable<IGrouping<TKey, TElement>>`. При LINQ to Objects — **відкладене виконання** і кожна група — окремий lazy iterator.

```csharp
var groups = users.GroupBy(u => u.City);

// ❌ пастка: якщо джерело одноразове (файл, API stream) — groups може бути вже порожнім
foreach (var group in groups) // перша ітерація — зчитує все
    Console.WriteLine(group.Key);
foreach (var group in groups) // друга ітерація — нічого немає!
    Process(group);

// ✅ якщо джерело одноразове — матеріалізувати
var groups = users.GroupBy(u => u.City).ToList();
```

```csharp
// GroupBy → Dictionary коли потрібний швидкий lookup
var byCity = users.GroupBy(u => u.City)
                  .ToDictionary(g => g.Key, g => g.ToList());
// або
var byCity = users.ToLookup(u => u.City); // ILookup — як Dictionary але допускає множинні значення
```

**`ToLookup` vs `GroupBy`:**

```csharp
// GroupBy — відкладений, ітерується один раз
// ToLookup — негайний, будує in-memory структуру, можна звертатись кілька разів
var lookup = users.ToLookup(u => u.City);
var kyivUsers = lookup["Kyiv"]; // O(1) lookup
var lvivUsers = lookup["Lviv"]; // O(1) lookup
```

---

## 6. Aggregate — fold/reduce

Найгнучкіший метод агрегації.

```csharp
// сума (еквівалент Sum)
var sum = numbers.Aggregate(0, (acc, n) => acc + n);

// конкатенація рядків
var csv = words.Aggregate((acc, w) => $"{acc},{w}");
// ["a","b","c"] → "a,b,c"

// побудова складного об'єкта
var stats = numbers.Aggregate(
    seed: new { Min = int.MaxValue, Max = int.MinValue, Sum = 0 },
    (acc, n) => new
    {
        Min = Math.Min(acc.Min, n),
        Max = Math.Max(acc.Max, n),
        Sum = acc.Sum + n
    }
);
```

---

## 7. Zip — паралельна ітерація

```csharp
var names = new[] { "Alice", "Bob", "Carol" };
var scores = new[] { 95, 87, 92 };

// zip двох колекцій в пари
var pairs = names.Zip(scores, (name, score) => new { name, score });
// → [{ name: Alice, score: 95 }, { name: Bob, score: 87 }, ...]

// .NET 6+: три колекції
var combined = names.Zip(scores, ranks);
// → (Alice, 95, 1), (Bob, 87, 2), ...
```

---

## 8. Expression Trees — чому IQueryable можна транслювати в SQL

```csharp
// Func<User, bool> — вже скомпільована функція, непрозора для LINQ provider
Func<User, bool> func = u => u.Age > 18;

// Expression<Func<User, bool>> — дерево виразу, можна інспектувати і транслювати
Expression<Func<User, bool>> expr = u => u.Age > 18;
// expr.Body → BinaryExpression { Left: MemberExpression(Age), Right: ConstantExpression(18) }
// EF Core читає це і генерує: WHERE Age > 18
```

```csharp
// будувати Expression динамічно (корисно для dynamic filters)
var parameter = Expression.Parameter(typeof(User), "u");
var property = Expression.Property(parameter, "Age");
var constant = Expression.Constant(18);
var body = Expression.GreaterThan(property, constant);
var lambda = Expression.Lambda<Func<User, bool>>(body, parameter);

dbContext.Users.Where(lambda).ToList(); // → WHERE Age > 18
```

---

## 9. Performance tips

```csharp
// ❌ Count() для перевірки "є елементи" — перебирає всю колекцію
if (users.Count() > 0) { }

// ✅ Any() — зупиняється на першому знайденому
if (users.Any()) { }

// ❌ Where().First() — не оптимально для EF Core (два оператори)
var user = users.Where(u => u.Id == id).First();

// ✅ First() з predicate — один оператор → кращий SQL
var user = users.First(u => u.Id == id);

// ❌ OrderBy().Where() — sort перед filter (дорого)
var result = users.OrderBy(u => u.Name).Where(u => u.Age > 18);

// ✅ Where().OrderBy() — filter спочатку
var result = users.Where(u => u.Age > 18).OrderBy(u => u.Name);

// ❌ ToList() в середині chain для IQueryable — передчасна матеріалізація
var result = dbContext.Users
    .Where(u => u.Age > 18)
    .ToList()               // ← вся фільтрація вже в SQL, але тут витягуємо в пам'ять
    .Select(u => u.Name)    // непотрібно — можна залишити в SQL
    .ToList();

// ✅
var result = dbContext.Users
    .Where(u => u.Age > 18)
    .Select(u => u.Name)    // все в SQL
    .ToList();
```

---

## 10. Типові питання на співбесіді

**Q: Яка різниця між IEnumerable і IQueryable?** `IEnumerable` — in-memory ітерація, лямбди компілюються в делегати, виконуються в C#. `IQueryable` — expression trees, провайдер (EF Core, MongoDB) транслює їх в SQL/MQL і виконує на сервері. `AsEnumerable()` переключає IQueryable на IEnumerable, після чого все решта виконується в пам'яті.

**Q: Що таке deferred execution і які є пастки?** LINQ-запит не виконується при визначенні, а при ітерації. Пастки: 1) closure в циклі захоплює змінну по reference, 2) зміна джерела між визначенням і виконанням впливає на результат, 3) multiple enumeration — кожна ітерація IEnumerable може заново виконати запит (N+1 до БД).

**Q: Яка різниця між GroupBy і ToLookup?** `GroupBy` — відкладений, повертає `IEnumerable<IGrouping<>>`, виконується при ітерації. `ToLookup` — негайний, будує in-memory `ILookup<>` (як Dictionary), підтримує множинні значення для одного ключа, O(1) доступ по ключу.

**Q: Чому `.Where(x => Custom(x))` не завжди працює в EF Core?** EF Core намагається транслювати Expression tree в SQL. Кастомний C# метод не знає як транслювати. Рішення: `AsEnumerable()` перед `Where()` — виконати фільтрацію в пам'яті, або використовувати EF Core функції що транслюються (`EF.Functions.Like`, `DbFunctions`).
