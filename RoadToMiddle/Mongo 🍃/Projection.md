---
tags:
  - database
  - mongodb
  - nosql
  - queries
  - projection
aliases:
  - MongoDB Projection
  - MongoDB SELECT fields
---

> Projection — другий аргумент у `find()`, який контролює які поля повернути. Аналог `SELECT field1, field2` в SQL. За замовчуванням повертаються всі поля.

```js
db.collection.find(
  { /* filter */ },
  { /* projection */ }  // ← другий аргумент
)
```

---

## Основні правила

| Дія | Синтаксис | Примітка |
|---|---|---|
| Включити поле | `{ field: 1 }` | |
| Виключити поле | `{ field: 0 }` | |
| Виключити `_id` | `{ _id: 0 }` | Єдиний виняток для змішування |
| Включити і виключити | ❌ заборонено | Крім `_id` |

⚠️ **Не можна** змішувати inclusion і exclusion в одній проекції — крім `_id`, який можна виключити разом з inclusion.

---

## Приклади

### Включити конкретні поля (+ `_id` автоматично)

```js
// поверне: _id, item, status
db.inventory.find({ status: "A" }, { item: 1, status: 1 })
// SQL: SELECT _id, item, status FROM inventory WHERE status = "A"
```

### Виключити `_id`

```js
// поверне: item, status (без _id)
db.inventory.find({ status: "A" }, { item: 1, status: 1, _id: 0 })
// SQL: SELECT item, status FROM inventory WHERE status = "A"
```

### Виключити конкретні поля (решта повертається)

```js
// поверне все КРІМ status і instock
db.inventory.find({ status: "A" }, { status: 0, instock: 0 })
```

---

## Вкладені документи

Dot notation для включення або виключення конкретного поля вкладеного документа:

```js
// включити тільки size.uom із вкладеного документа
db.inventory.find({ status: "A" }, { item: 1, status: 1, "size.uom": 1 })
// результат: { _id, item, status, size: { uom: "cm" } }
// size повертається як об'єкт, але тільки з полем uom

// виключити size.uom
db.inventory.find({ status: "A" }, { "size.uom": 0 })
```

---

## Масиви об'єктів

Dot notation також працює для полів всередині масиву об'єктів:

```js
// повернути тільки qty із кожного елемента масиву instock
db.inventory.find(
  { status: "A" },
  { item: 1, status: 1, "instock.qty": 1 }
)
// результат: instock: [{ qty: 5 }, { qty: 15 }] — без warehouse
```

---

## Оператори проекції для масивів

Три спеціальні оператори для вибору елементів масиву:

### `$slice` — повернути N елементів масиву

```js
// останній елемент
db.inventory.find({ status: "A" }, { item: 1, instock: { $slice: -1 } })

// перші 2 елементи
db.inventory.find({ status: "A" }, { item: 1, instock: { $slice: 2 } })

// з offset: пропустити 1, взяти 2
db.inventory.find({ status: "A" }, { item: 1, instock: { $slice: [1, 2] } })
```

### `$elemMatch` — повернути перший елемент що відповідає умові

```js
// повернути тільки перший елемент instock де qty > 10
db.inventory.find(
  { status: "A" },
  { item: 1, instock: { $elemMatch: { qty: { $gt: 10 } } } }
)
```

### `$` (позиційний оператор) — повернути перший елемент що відповідав фільтру

```js
// поверне тільки той елемент instock що відповідав умові фільтра
db.inventory.find(
  { "instock.qty": 5 },
  { "instock.$": 1 }
)
```

⚠️ **Не можна** вибрати елемент за індексом через projection:
```js
// ❌ НЕ ПРАЦЮЄ — не поверне тільки перший елемент масиву
db.inventory.find({}, { "instock.0": 1 })
```

---

## Projection з aggregation expressions

У projection можна використовувати aggregation expressions для обчислення нових полів або трансформації існуючих:

```js
db.inventory.find({}, {
  _id: 0,
  item: 1,
  // трансформувати значення існуючого поля
  status: {
    $switch: {
      branches: [
        { case: { $eq: ["$status", "A"] }, then: "Available" },
        { case: { $eq: ["$status", "D"] }, then: "Discontinued" }
      ],
      default: "Unknown"
    }
  },
  // обчислити нове поле
  area: {
    $concat: [
      { $toString: { $multiply: ["$size.h", "$size.w"] } },
      " ", "$size.uom"
    ]
  },
  // вставити константу
  reportNumber: { $literal: 1 }
})
```

---

## SQL → MongoDB шпаргалка

| SQL | MongoDB Projection |
|---|---|
| `SELECT *` | `{}` або без projection |
| `SELECT item, status` | `{ item: 1, status: 1, _id: 0 }` |
| `SELECT _id, item` | `{ item: 1 }` |
| Всі крім `status` | `{ status: 0 }` |

---

## Нотатка про `$project` в aggregation

`$project` в pipeline — аналог projection в `find()`, але потужніший. Важливо: `$project` краще ставити **в кінці** pipeline, а не на початку — MongoDB сама оптимізує що передавати між стадіями.

---

## Посилання

- [[MongoDB — Queries]]
- [[MongoDB — Aggregation Pipeline]]
- [[MongoDB — Documents]]