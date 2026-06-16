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

## Оновлення (Update)

### updateOne / updateMany

Перший аргумент — фільтр (як у `find`), другий — оператор оновлення.

```js
// оновити одне поле
db.users.updateOne(
  { _id: userId },
  { $set: { city: "Lviv" } }
)

// оновити кілька документів
db.users.updateMany(
  { status: "inactive" },
  { $set: { archived: true }, $unset: { sessionToken: "" } }
)
```

### Оператори оновлення

**Поля:**

| Оператор | Що робить |
|---|---|
| `$set` | Встановити значення поля |
| `$unset` | Видалити поле |
| `$inc` | Збільшити числове поле на N (від'ємне — зменшити) |
| `$mul` | Помножити числове поле на N |
| `$rename` | Перейменувати поле |
| `$min` | Оновити якщо нове значення менше поточного |
| `$max` | Оновити якщо нове значення більше поточного |
| `$currentDate` | Встановити поточну дату |

```js
db.products.updateOne(
  { _id: id },
  {
    $inc: { stock: -1, soldCount: 1 },
    $set: { updatedAt: new Date() },
    $min: { lowestPrice: newPrice }  // оновить тільки якщо newPrice < lowestPrice
  }
)
```

**Масиви:**

| Оператор | Що робить |
|---|---|
| `$push` | Додати елемент в кінець масиву |
| `$pull` | Видалити всі елементи що збігаються з умовою |
| `$addToSet` | Додати елемент тільки якщо його ще немає (унікальність) |
| `$pop` | Видалити перший (`-1`) або останній (`1`) елемент |
| `$each` | Модифікатор — додати кілька елементів через `$push` або `$addToSet` |

```js
// додати один тег
db.posts.updateOne({ _id: id }, { $push: { tags: "mongodb" } })

// додати кілька одразу
db.posts.updateOne({ _id: id }, { $push: { tags: { $each: ["db", "nosql"] } } })

// видалити тег
db.posts.updateOne({ _id: id }, { $pull: { tags: "outdated" } })

// додати тільки якщо немає
db.posts.updateOne({ _id: id }, { $addToSet: { tags: "mongodb" } })
```

**Позиційні оператори:**

```js
// оновити перший елемент масиву що відповідає фільтру
db.orders.updateOne(
  { _id: id, "items.productId": "p1" },
  { $set: { "items.$.qty": 5 } }   // $ — позиція знайденого елемента
)

// оновити всі елементи масиву
db.orders.updateOne({ _id: id }, { $set: { "items.$[].price": 0 } })

// оновити елементи що відповідають умові (arrayFilters)
db.orders.updateOne(
  { _id: id },
  { $set: { "items.$[el].discounted": true } },
  { arrayFilters: [{ "el.price": { $gt: 100 } }] }
)
```

### Upsert — вставити якщо не знайдено

```js
db.users.updateOne(
  { email: "alice@example.com" },
  { $set: { name: "Alice", city: "Kyiv" }, $setOnInsert: { createdAt: new Date() } },
  { upsert: true }
)
// якщо документ знайдено — оновить name і city
// якщо не знайдено — вставить новий документ + createdAt
```

`$setOnInsert` — виконується тільки при вставці нового документа, не при оновленні.

### replaceOne — повна заміна документа

```js
// замінює весь документ (крім _id)
db.users.replaceOne(
  { _id: userId },
  { name: "Alice", city: "Lviv", updatedAt: new Date() }
)
// ⚠️ поля яких немає в новому документі — зникають
```

### findOneAndUpdate — знайти, оновити, повернути

```js
// повертає документ ДО оновлення (дефолт)
db.inventory.findOneAndUpdate(
  { _id: id },
  { $inc: { stock: -1 } }
)

// повертає документ ПІСЛЯ оновлення
db.inventory.findOneAndUpdate(
  { _id: id },
  { $inc: { stock: -1 } },
  { returnDocument: "after" }
)
```

Корисно для атомарних операцій "прочитати і змінити" — наприклад, списати залишок товару і одразу отримати нове значення.

---

## Видалення (Delete)

```js
// видалити один документ
db.users.deleteOne({ _id: userId })

// видалити всі що відповідають фільтру
db.users.deleteMany({ status: "inactive" })

// видалити всі документи в колекції (структура залишається)
db.users.deleteMany({})

// findOneAndDelete — видалити і повернути документ
const deleted = db.users.findOneAndDelete({ _id: userId })
```

---

## Підрахунок документів

```js
// точна кількість за фільтром (читає колекцію)
db.users.countDocuments({ status: "active" })

// приблизна кількість всіх документів (читає метадані — дуже швидко)
db.users.estimatedDocumentCount()
```

> ⚠️ `estimatedDocumentCount()` не підтримує фільтр — тільки для загальної кількості. Для фільтрованого підрахунку — тільки `countDocuments()`.

---

## bulkWrite — пакетні операції

Відправляє кілька різних write операцій одним запитом до сервера:

```js
db.users.bulkWrite([
  { insertOne: { document: { name: "Bob", city: "Odesa" } } },
  { updateOne: { filter: { name: "Alice" }, update: { $set: { city: "Kyiv" } } } },
  { deleteOne: { filter: { name: "old-user" } } },
  { replaceOne: { filter: { _id: id }, replacement: { name: "New", status: "active" } } }
])
```

За замовчуванням операції виконуються **по порядку** (ordered: true) — зупиняється при першій помилці. Для паралельного виконання:

```js
db.users.bulkWrite([...], { ordered: false })
// усі операції виконуються, помилки збираються і повертаються разом
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
| `UPDATE ... SET city = 'Lviv' WHERE ...` | `updateOne({ ... }, { $set: { city: "Lviv" } })` |
| `UPDATE ... SET count = count + 1` | `updateOne({ ... }, { $inc: { count: 1 } })` |
| `DELETE FROM users WHERE ...` | `deleteOne({ ... })` |
| `SELECT COUNT(*) WHERE ...` | `countDocuments({ ... })` |

## Посилання

- [[MongoDB]]
- [[MongoDb - Documents]]
- [[MongoDB - Aggregation Pipeline]]
- [[MongoDb - Indexes]]
- [[CSharp MongoDB Driver]]