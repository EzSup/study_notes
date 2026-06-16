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

// для e-commerce/каталогу — типова схема
db.orders.createIndex({
  description: "text",   // довгий опис товару
  title: "text",         // назва
  tags: "text"           // теги
})

// з вагами — опис важливіший за теги
db.orders.createIndex(
  { title: "text", description: "text", tags: "text" },
  { weights: { title: 10, description: 5, tags: 1 } }
)
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

### Wildcard

Індексує всі поля документа (або піддерева). Корисно коли структура документів непередбачувана або поліморфна.

```js
// індексувати всі поля на всіх рівнях
db.products.createIndex({ "$**": 1 })

// індексувати тільки піддерево attributes
db.products.createIndex({ "attributes.$**": 1 })
```

- ✅ гнучкість для динамічних схем
- ❌ більший розмір індексу ніж точковий compound
- ❌ не підтримує compound-запити (не можна замінити `{ a: 1, b: 1 }`)

---

## Covered Queries — запит повністю з індексу

**Covered query** — запит де MongoDB повертає результат **тільки з індексу**, без звернення до самих документів. Це максимально швидке читання.

Умови для covered query:
1. Всі поля у фільтрі є в індексі
2. Всі поля у проекції є в індексі
3. Проекція **виключає** `_id` (або `_id` є в індексі)

```js
db.users.createIndex({ city: 1, name: 1, age: 1 })

// covered query — всі поля запиту покриті індексом
db.users.find(
  { city: "Kyiv" },
  { _id: 0, name: 1, age: 1 }  // тільки поля з індексу, _id виключено
)
```

У `explain()` — шукай `"stage": "PROJECTION_COVERED"` і відсутність `FETCH` стадії:

```js
db.users.find({ city: "Kyiv" }, { _id: 0, name: 1 }).explain("executionStats")
// IXSCAN → PROJECTION_COVERED  ← немає FETCH = covered ✅
// IXSCAN → FETCH → PROJECTION  ← є FETCH = не covered ❌
```

---

## hint() — примусово обрати індекс

MongoDB сам обирає індекс через query planner. Якщо вибір неоптимальний — можна вказати вручну:

```js
// примусово використати конкретний індекс
db.users.find({ city: "Kyiv", age: { $gt: 18 } }).hint({ city: 1, age: 1 })

// примусово collscan (корисно для порівняння в explain)
db.users.find({ city: "Kyiv" }).hint({ $natural: 1 })
```

> Використовуй `hint()` тільки після діагностики через `explain()`. У більшості випадків query planner обирає правильно.

---

## Загальні правила

- Індекси прискорюють **читання**, але уповільнюють **запис** — при кожному insert/update/delete MongoDB оновлює всі індекси
- Не створюй індекс на кожне поле — тільки на поля в `find()` фільтрах і `sort()`
- Завжди перевіряй через `.explain("executionStats")` — IXSCAN vs COLLSCAN
- Compound індекс на `{ a, b }` покриває запити по `{ a }`, але не по `{ b }` окремо
- Covered query — найшвидший можливий запит: результат прямо з індексу без читання документів

---

## Посилання

- [[MongoDb - Queries]]
- [[MongoDb - Documents]]
- [[CSharp MongoDB Driver]]