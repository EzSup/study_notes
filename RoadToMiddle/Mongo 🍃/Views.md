---
tags: [database, mongodb, nosql, views]
aliases: [MongoDB Views, Materialized Views]
---

> View — збережений aggregation pipeline який виглядає як колекція. Є два принципово різних типи.

## Порівняння типів

|                      | Standard View                         | On-Demand Materialized View     |
| -------------------- | ------------------------------------- | ------------------------------- |
| Зберігає дані        | ні                                    | так (окрема колекція)           |
| Обчислення           | кожен запит заново                    | один раз, при запуску `$merge`  |
| Актуальність даних   | завжди свіжі                          | залежить від частоти оновлення  |
| Індекси              | не можна створити                     | можна, як на звичайній колекції |
| Запис                | read-only                             | звичайна колекція, можна писати |
| Коли використовувати | прості фільтри, завжди актуальні дані | важкий pipeline, часті reads    |

---

## Standard View

Збережений pipeline без фізичного зберігання даних.
При кожному зверненні — виконується pipeline заново на оригінальній колекції.

### Створення

```js
db.createView(
  "adultUsers",      // назва view
  "users",           // source колекція
  [
    { $match: { age: { $gte: 18 } } },
    { $project: { name: 1, age: 1, _id: 0 } }
  ]
)
```

### Використання — як звичайна колекція

```js
db.adultUsers.find({ age: { $lt: 30 } })
db.adultUsers.aggregate([...])
```

### Join через view ($lookup)

```js
// view яка джойнить orders з users
db.createView("ordersWithUsers", "orders", [
  { $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user"
  }},
  { $unwind: "$user" }
])
```

### Обмеження

- Read-only — `insert`, `update`, `delete` заборонені
- Не можна створити індекс на view
- Не зберігає дані — повільно на великих колекціях з важким pipeline
- `$text` (full-text search) не підтримується
- Не можна використати `$natural` sort

### Видалення / оновлення

```js
// view — це колекція, дропається так само
db.adultUsers.drop()

// оновити pipeline можна тільки через collMod
db.runCommand({
  collMod: "adultUsers",
  viewOn: "users",
  pipeline: [
    { $match: { age: { $gte: 21 } } }  // змінили умову
  ]
})
```

---

## On-Demand Materialized View

Фізично існуюча колекція яка заповнюється через `$merge` або `$out`.
Оновлюється вручну або по крону — не автоматично.

### Створення і оновлення через `$merge`

```js
db.orders.aggregate([
  { $group: {
      _id: "$userId",
      totalSpent: { $sum: "$amount" },
      orderCount: { $sum: 1 },
      lastOrder: { $max: "$createdAt" }
  }},
  { $merge: {
      into: "userStats",        // target колекція
      on: "_id",                // поле для match
      whenMatched: "replace",   // замінити якщо існує
      whenNotMatched: "insert"  // вставити якщо нема
  }}
])
```

### `$merge` vs `$out`

| | `$merge` | `$out` |
|---|---|---|
| Якщо колекція існує | merge або replace по полю | повністю перезаписує |
| Зберігає незачеплені документи | так | ні |
| Гнучкість | висока | низька |
| Коли використовувати | інкрементальне оновлення | повна перегенерація |

```js
// $out — простіше але знищує всі попередні дані
db.orders.aggregate([
  { $group: { _id: "$userId", total: { $sum: "$amount" } } },
  { $out: "userStats" }
])
```

### Читання — як звичайна колекція

```js
// швидко — просто читання колекції, без обчислень
db.userStats.find({ totalSpent: { $gt: 1000 } })

// можна створити індекс
db.userStats.createIndex({ totalSpent: -1 })
```

### Стратегії оновлення

```js
// варіант 1: повна перегенерація (просто, але дорого)
// запускаєш весь pipeline заново по крону

// варіант 2: інкрементальне оновлення (тільки нові дані)
const since = new Date(Date.now() - 24 * 60 * 60 * 1000) // останні 24г
db.orders.aggregate([
  { $match: { createdAt: { $gte: since } } },  // тільки нові
  { $group: { _id: "$userId", total: { $sum: "$amount" } } },
  { $merge: { into: "userStats", on: "_id", whenMatched: "merge" } }
])
```

---

## Коли що використовувати

```
Дані мають бути завжди свіжі + pipeline простий
→ Standard View

Дані можуть бути трохи застарілі + pipeline важкий + потрібні індекси
→ Materialized View

Потрібен JOIN двох колекцій для читання
→ Standard View з $lookup

Потрібна агрегована статистика (суми, підрахунки)
→ Materialized View
```

## Посилання

- [[MongoDB]]
- [[MongoDB — Aggregation Pipeline]]
- [[MongoDB — Індекси]]