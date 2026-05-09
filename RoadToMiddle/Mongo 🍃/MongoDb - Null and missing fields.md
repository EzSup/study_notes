---
tags: [database, mongodb, nosql, queries, csharp]
aliases: [MongoDB null, MongoDB missing fields, MongoDB $exists]
---
> В MongoDB `null` і "поле відсутнє" — різні речі, але деякі оператори трактують їх однаково. Важливо розуміти різницю.

## Три сценарії

Маємо два документи:
```js
{ _id: 1, item: null }  // поле є, значення null
{ _id: 2 }              // поля item взагалі нема
```

---

## Equality filter `{ field: null }`

Знаходить **обидва** — і `null`, і відсутнє поле:

```js
// mongosh
db.inventory.find({ item: null })
// поверне: { _id: 1 } і { _id: 2 }
```

```csharp
// C#
var filter = Builders.Filter.Eq("item", BsonNull.Value);
var result = collection.Find(filter).ToList();
// поверне обидва документи
```

⚠️ Це часта несподіванка — якщо хочеш тільки null, потрібен `$type`.

---

## Non-equality filter `{ field: { $ne: null } }`

Знаходить документи де поле **існує і не null**:

```js
// mongosh
db.inventory.find({ item: { $ne: null } })
// поверне тільки документи де item є і не null
```

```csharp
// C#
var filter = Builders.Filter.Ne("item", BsonNull.Value);
var result = collection.Find(filter).ToList();
```

---

## Type check `{ field: { $type: 10 } }`

Знаходить **тільки null** (BSON Type 10) — поле без значення не матчить:

```js
// mongosh
db.inventory.find({ item: { $type: 10 } })
// поверне тільки { _id: 1, item: null }
// { _id: 2 } — НЕ поверне, бо поле відсутнє
```

```csharp
// C#
var filter = Builders.Filter.Type("item", BsonType.Null);
var result = collection.Find(filter).ToList();
// поверне тільки документ де item: null
```

---

## Existence check `{ field: { $exists: false } }`

Знаходить документи де поле **взагалі відсутнє**:

```js
// mongosh
db.inventory.find({ item: { $exists: false } })
// поверне тільки { _id: 2 } — де item нема взагалі
```

```csharp
// C#
var filter = Builders.Filter.Exists("item", false);
var result = collection.Find(filter).ToList();
// поверне тільки { _id: 2 }
```

Навпаки — перевірити що поле **існує** (будь-яке значення, включно з null):
```js
db.inventory.find({ item: { $exists: true } })
```

```csharp
var filter = Builders.Filter.Exists("item", true);
```

---

## Шпаргалка

| Що шукаємо | mongosh | Результат |
|---|---|---|
| null або відсутнє | `{ item: null }` | обидва |
| тільки null | `{ item: { $type: 10 } }` | тільки null |
| тільки відсутнє | `{ item: { $exists: false } }` | тільки відсутні |
| існує і не null | `{ item: { $ne: null } }` | тільки з реальним значенням |

---

## Нотатка для aggregation pipeline

`$exists` **не працює** в aggregation expressions. Замість нього — `$type` з перевіркою на `"missing"`:

```js
db.inventory.aggregate([
  { $match: {
      $expr: { $ne: [{ $type: "$item" }, "missing"] }
  }}
])
```

---

## Посилання

- [[MongoDb - Queries]]
- [[MongoDb - Documents]]
- [[CSharp MongoDB Driver]]