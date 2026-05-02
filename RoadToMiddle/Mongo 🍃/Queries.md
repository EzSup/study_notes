---
tags: [database, mongodb, nosql, queries]
aliases: [MongoDB Queries, MQL, MongoDB find]
---

> Запити в MongoDB будуються через query filter document — об'єкт який передається першим аргументом у `find`, `updateOne`, `deleteMany` тощо.

## Базовий синтаксис

```js
// всі документи
db.collection.find({})

// рівність (implicit AND між полями)
db.movies.find({ rated: "G", runtime: 90 })

// оператор порівняння
db.movies.find({ runtime: { $lt: 90 } })

// OR
db.movies.find({ $or: [{ rated: "G" }, { runtime: { $lt: 90 } }] })

// AND + OR разом
db.movies.find({
  rated: "G",
  $or: [{ runtime: { $lt: 90 } }, { title: /^T/ }]
})
```

## Оператори порівняння

| Оператор | Значення | SQL аналог |
|---|---|---|
| `$eq` | рівно | `=` |
| `$ne` | не рівно | `!=` |
| `$gt` | більше | `>` |
| `$gte` | більше або рівно | `>=` |
| `$lt` | менше | `<` |
| `$lte` | менше або рівно | `<=` |
| `$in` | є в списку | `IN (...)` |
| `$nin` | нема в списку | `NOT IN (...)` |

## Логічні оператори

```js
// AND — implicit (просто кілька полів)
{ city: "Kyiv", age: { $gt: 18 } }

// AND — explicit (потрібен коли два оператори на одне поле)
{ $and: [{ age: { $gte: 18 } }, { age: { $lte: 30 } }] }

// OR
{ $or: [{ city: "Kyiv" }, { city: "Lviv" }] }

// NOR — жодна з умов
{ $nor: [{ city: "Kyiv" }, { age: { $lt: 18 } }] }

// NOT — інвертує оператор
{ age: { $not: { $gt: 30 } } }
```

⚠️ `$in` замість `$or` коли перевіряєш одне поле на кілька значень — коротше і швидше.

## Текстовий пошук (regex)

```js
// містить "iv" — аналог LIKE '%iv%'
db.users.find({ city: { $regex: "iv" } })

// починається з "Ky" — аналог LIKE 'Ky%'
db.users.find({ city: { $regex: "^Ky" } })

// закінчується на "iv"
db.users.find({ city: { $regex: "iv$" } })

// case-insensitive — аналог ILIKE
db.users.find({ city: { $regex: "kyiv", $options: "i" } })

// короткий синтаксис (тільки в mongosh)
db.users.find({ city: /^Ky/i })
```

⚠️ Regex без `^` = collscan (повний перебір). З `^` може використати індекс.

---

## Вкладені документи (Nested Documents)

Для доступу до полів вкладеного документа використовується **dot notation** в лапках.

```js
// рівність по вкладеному полю
db.inventory.find({ "size.uom": "in" })

// оператор на вкладеному полі
db.inventory.find({ "size.h": { $lt: 15 } })

// кілька умов на вкладених полях
db.inventory.find({ "size.h": { $lt: 15 }, "size.uom": "in", status: "D" })
```

### ⚠️ Точний збіг вкладеного документа — порядок полів важливий

```js
// ✅ знайде — порядок збігається
db.inventory.find({ size: { h: 14, w: 21, uom: "cm" } })

// ❌ не знайде — порядок інший
db.inventory.find({ size: { w: 21, h: 14, uom: "cm" } })
```

Краще завжди використовувати dot notation замість точного збігу документа.

---

## Масиви (Arrays)

### Базові запити

```js
// містить елемент
db.users.find({ tags: "admin" })

// точний збіг масиву (порядок важливий!)
db.inventory.find({ tags: ["red", "blank"] })

// містить всі елементи (порядок не важливий)
db.inventory.find({ tags: { $all: ["red", "blank"] } })

// містить хоча б один з переліку
db.users.find({ tags: { $in: ["admin", "editor"] } })

// за розміром масиву
db.users.find({ tags: { $size: 2 } })

// за індексом елемента (0-based)
db.inventory.find({ "dim_cm.1": { $gt: 25 } })
```

### Умови на елементах масиву

```js
// хоча б один елемент > 25 (умова застосовується до будь-якого елемента)
db.inventory.find({ dim_cm: { $gt: 25 } })

// УВАГА: без $elemMatch умови можуть виконуватись на різних елементах
// один елемент > 15, інший < 20 — не обов'язково один і той самий
db.inventory.find({ dim_cm: { $gt: 15, $lt: 20 } })

// $elemMatch — обидві умови на ОДНОМУ елементі
db.inventory.find({ dim_cm: { $elemMatch: { $gt: 22, $lt: 30 } } })
```

### Масив об'єктів

```js
// dot notation — поле всередині будь-якого елемента масиву
db.inventory.find({ "instock.qty": { $lte: 20 } })

// по конкретному індексу
db.inventory.find({ "instock.0.qty": { $lte: 20 } })

// точний збіг елемента (порядок полів важливий!)
db.inventory.find({ instock: { warehouse: "A", qty: 5 } })

// $elemMatch — обидві умови на ОДНОМУ елементі масиву об'єктів
db.inventory.find({ instock: { $elemMatch: { qty: 5, warehouse: "A" } } })

// без $elemMatch — умови можуть виконуватись на РІЗНИХ елементах
// qty: 5 — в одному елементі, warehouse: "A" — в іншому
db.inventory.find({ "instock.qty": 5, "instock.warehouse": "A" })
```

---

## Cursor та результати

`find()` повертає **cursor** — не масив документів. Cursor — це вказівник на результати, документи підтягуються поступово.

```js
// ітерація в mongosh
db.users.find().forEach(doc => print(doc.name))

// limit і sort
db.users.find().sort({ age: -1 }).limit(10)

// skip (для пагінації)
db.users.find().skip(20).limit(10)

// findOne — повертає один документ (не cursor)
db.users.findOne({ name: "Alice" })
```

---

## SQL → MongoDB шпаргалка

| SQL | MongoDB |
|---|---|
| `SELECT * FROM users` | `db.users.find({})` |
| `WHERE city = 'Kyiv'` | `{ city: "Kyiv" }` |
| `WHERE age > 18` | `{ age: { $gt: 18 } }` |
| `WHERE city IN ('Kyiv', 'Lviv')` | `{ city: { $in: ["Kyiv", "Lviv"] } }` |
| `WHERE name LIKE 'A%'` | `{ name: { $regex: "^A" } }` |
| `WHERE a = 1 AND b = 2` | `{ a: 1, b: 2 }` |
| `WHERE a = 1 OR b = 2` | `{ $or: [{ a: 1 }, { b: 2 }] }` |

## Посилання

- [[MongoDB]]
- [[MongoDB — Documents]]
- [[MongoDB — Aggregation Pipeline]]
- [[MongoDB — Індекси]]
- [[C# MongoDB Driver]]