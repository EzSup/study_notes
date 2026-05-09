---
tags: [database, mongodb, nosql, indexes]
aliases: [MongoDB Indexes, MongoDB createIndex, MongoDB індекси]
---

> Індекс — окрема B-tree структура яка зберігає відсортовані значення поля з посиланням на документ. Без індексу — **collscan** (перебір всієї колекції). Колекція завжди має мінімум один індекс: автоматичний унікальний на `_id`.

---

## Create / Drop

```js
// створити індекс
db.users.createIndex({ city: 1 })          // asc
db.users.createIndex({ city: -1 })         // desc

// з опціями
db.users.createIndex(
  { email: 1 },
  {
    name: "idx_email_unique",   // кастомна назва
    unique: true,               // унікальний
    sparse: true,               // індексувати тільки документи де поле існує
    expireAfterSeconds: 3600    // TTL індекс (автовидалення документів)
  }
)

// переглянути всі індекси
db.users.getIndexes()

// дропнути за назвою
db.users.dropIndex("idx_email_unique")

// дропнути за специфікацією
db.users.dropIndex({ city: 1 })

// дропнути всі крім _id
db.users.dropIndexes()
```

### EXPLAIN — перевірити чи використовується індекс

```js
db.users.find({ city: "Kyiv" }).explain("executionStats")
// IXSCAN → індекс використано ✅
// COLLSCAN → повний перебір, індекс не допоміг ❌
```

---

## Типи індексів

### Single Field

Найпростіший — по одному полю. Напрямок (1/-1) не важливий для одиночних індексів — MongoDB читає B-tree в обох напрямках однаково.

```js
db.users.createIndex({ age: 1 })

db.users.find({ age: { $gt: 18 } })  // використає індекс
```

---

### Compound

Індекс по кількох полях. Порядок полів і напрямок **мають значення**.

```js
db.users.createIndex({ city: 1, age: -1 })

// цей запит використає індекс:
db.users.find({ city: "Kyiv" }).sort({ age: -1 })

// цей теж (prefix rule — перше поле покриває):
db.users.find({ city: "Kyiv" })

// а цей НЕ використає ефективно (немає першого поля):
db.users.find({ age: { $gt: 18 } })
```

**ESR Rule** — оптимальний порядок полів у compound індексі:
1. **E**quality — поля де фільтр рівності (`city: "Kyiv"`)
2. **S**ort — поля по яких сортуєш
3. **R**ange — поля де діапазон (`age: { $gt: 18 }`)

```js
// правильно: рівність → сортування → діапазон
db.users.createIndex({ city: 1, name: 1, age: 1 })
```

---

### Multikey

Автоматично створюється коли індексуєш поле-масив. MongoDB індексує кожен елемент масиву окремо.

```js
// документ: { tags: ["admin", "editor"] }
db.users.createIndex({ tags: 1 })
// MongoDB автоматично робить його multikey

db.users.find({ tags: "admin" })  // використає індекс
```

⚠️ Обмеження: в compound індексі не можна мати два multikey поля одночасно (два поля-масиви).

---

### Text

Для full-text пошуку по рядкових полях. Підтримує стемінг, стоп-слова, релевантність.

```js
// створити text індекс
db.articles.createIndex({ title: "text", body: "text" })

// або на всі рядкові поля
db.articles.createIndex({ "$**": "text" })

// пошук
db.articles.find({ $text: { $search: "mongodb aggregation" } })

// пошук з релевантністю
db.articles.find(
  { $text: { $search: "mongodb" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

⚠️ На колекцію може бути тільки **один** text індекс. Для серйозного пошуку краще Atlas Search.

---

### Hashed

Індексує хеш значення поля. Використовується для **sharding** — рівномірно розподіляє документи по шардах.

```js
db.users.createIndex({ userId: "hashed" })
```

- ✅ рівномірний розподіл по шардах
- ❌ не підтримує range queries (`$gt`, `$lt`)
- ❌ не підтримує sorting
- Тільки для equality: `{ userId: "abc123" }`

---

## Спеціальні опції

### Unique

```js
db.users.createIndex({ email: 1 }, { unique: true })
// insert з дублікатом кине помилку
```

### Sparse

Індексує тільки документи де поле **існує**. Документи без поля — не потрапляють в індекс.

```js
db.users.createIndex({ phone: 1 }, { sparse: true })
// документи без phone — не індексуються
// корисно для рідкісних опціональних полів
```

### TTL (Time To Live)

Автоматично видаляє документи через N секунд після значення поля Date.

```js
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 86400 }  // видалити через 24 години
)
// документ: { createdAt: new Date(), ... }
```

### Partial

Індексує тільки документи що відповідають умові фільтра. Менший розмір ніж повний індекс.

```js
// індексувати тільки активних користувачів
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { status: "active" } }
)
```

---

## Загальні правила

- Індекси прискорюють **читання**, але уповільнюють **запис** — при кожному insert/update/delete MongoDB оновлює всі індекси
- Не створюй індекс на кожне поле — тільки на поля в `find()` фільтрах і `sort()`
- Завжди перевіряй через `.explain("executionStats")` — IXSCAN vs COLLSCAN
- Compound індекс на `{ a, b }` покриває запити по `{ a }`, але не по `{ b }` окремо

---

## Посилання

- [[MongoDb - Queries]]
- [[MongoDb - Documents]]
- [[CSharp MongoDB Driver]]