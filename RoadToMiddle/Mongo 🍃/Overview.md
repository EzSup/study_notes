---
tags: [database, mongodb, nosql, backend]
aliases: [MongoDB, Mongo]
---
---

# MongoDB

> Document-орієнтована БД для гнучкого зберігання JSON-like даних без фіксованої схеми.

>юзає camelCase
## Ключові концепції

### Document
Основна одиниця даних — аналог JSON-об'єкту. Поля можуть містити вкладені документи та масиви.

```json
{
  "_id": ObjectId("..."),
  "name": "Alice",
  "address": { "city": "Springfield" },
  "hobbies": ["reading", "coding"]
}
```

### Collection
Аналог таблиці в SQL, але **без жорсткої схеми**. Документи в одній колекції можуть мати різну структуру (polymorphism).

## Архітектурні можливості

| Можливість | Суть |
|---|---|
| **ACID транзакції** | Multi-document, у т.ч. на sharded кластерах |
| **Replication** | Автоматичний failover, обирає новий primary |
| **Sharding** | Горизонтальне масштабування через shard key |

## Типи запитів

### OLTP (MQL)
Стандартні CRUD-операції через MongoDB Query Language.
Аналог `SELECT / INSERT / UPDATE / DELETE` в SQL, але працює з документами.
```js
db.users.find({ age: { $gt: 18 } })
```

### Aggregation pipeline
Послідовність стадій обробки даних — аналог SQL `GROUP BY` + `JOIN` + обчислень.
Кожна стадія трансформує потік документів: `$match → $group → $sort → $project`.
```js
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $group: { _id: "$userId", total: { $sum: "$amount" } } }
])
```

### Full-text search
Пошук по текстових полях із підтримкою стоп-слів, стемінгу, релевантності.
Підходить для пошукових рядків типу "знайди всі документи де є слово X".
```js
db.articles.find({ $text: { $search: "mongodb aggregation" } })
```

### Vector search
Пошук за семантичною схожістю через embeddings (числові вектори).
Використовується для AI-фіч: semantic search, RAG, рекомендації.
Потребує Atlas або окремого індексу.

### Geospatial queries
Запити на основі географічних координат.
Підтримує пошук "поряд з точкою", "в межах полігону", відстані між точками.
```js
db.places.find({
  location: { $near: { $geometry: { type: "Point", coordinates: [30.5, 50.4] } } }
})
```

### Time series
Спеціальні колекції оптимізовані для даних з часовою міткою (IoT, метрики, логи).
Автоматичне стиснення і сортування по часу під капотом.

## Переваги над SQL

- Немає потреби в ORM — структура документа = структура об'єкта в коді
- Embedded документи замінюють JOIN-и
- Гнучка схема → ітерації без downtime

## Посилання

- [[NoSQL vs SQL]]
- [[C# MongoDB Driver]]
- [[Aggregation Pipeline]]