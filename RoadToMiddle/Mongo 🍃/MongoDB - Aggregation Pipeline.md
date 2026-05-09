---
tags: [database, mongodb, nosql, aggregation, pipeline]
aliases: [MongoDB Aggregation, Aggregation Pipeline, MongoDB Pipeline Stages]
---
> A sequence of stages that process documents. Each stage receives documents from the previous one, transforms them, and passes results forward. Think of it as a Unix pipe: `cat data | grep | sort | awk`.

```js
db.collection.aggregate([
  { $stage1: { ... } },
  { $stage2: { ... } },
  { $stage3: { ... } }
])
```

---

## Sample Data

All examples below use this dataset:

```js
db.orders.insertMany([
  { _id: 1, userId: "u1", city: "Kyiv",   status: "paid",    amount: 120, items: ["book", "pen"],    date: new Date("2024-01-15") },
  { _id: 2, userId: "u1", city: "Kyiv",   status: "paid",    amount: 80,  items: ["pen"],            date: new Date("2024-02-10") },
  { _id: 3, userId: "u2", city: "Lviv",   status: "pending", amount: 200, items: ["book", "laptop"], date: new Date("2024-02-20") },
  { _id: 4, userId: "u3", city: "Kyiv",   status: "paid",    amount: 45,  items: ["pen", "notebook"],date: new Date("2024-03-05") },
  { _id: 5, userId: "u2", city: "Lviv",   status: "paid",    amount: 310, items: ["laptop"],         date: new Date("2024-03-18") },
  { _id: 6, userId: "u4", city: "Odesa",  status: "pending", amount: 60,  items: ["book"],           date: new Date("2024-04-01") }
])
```

---

## Core Stages

### $match — filter (WHERE)

Filters documents. Always put as early as possible — can use indexes.

```js
// paid orders only
db.orders.aggregate([
  { $match: { status: "paid" } }
])

// multiple conditions
db.orders.aggregate([
  { $match: { status: "paid", amount: { $gt: 100 } } }
])
```

⚠️ `$match` before `$group` → filters raw documents (uses index).
`$match` after `$group` → filters aggregated results (no index, like HAVING in SQL).

---

### $group — aggregate (GROUP BY)

Groups documents by a key and computes aggregate values.

```js
// total spent and order count per user
db.orders.aggregate([
  { $group: {
      _id: "$userId",              // group key — "$" prefix for field reference
      totalSpent: { $sum: "$amount" },
      orderCount: { $sum: 1 },
      avgOrder:   { $avg: "$amount" },
      maxOrder:   { $max: "$amount" },
      minOrder:   { $min: "$amount" }
  }}
])
// result: { _id: "u1", totalSpent: 200, orderCount: 2, avgOrder: 100, ... }

// group by multiple fields
db.orders.aggregate([
  { $group: {
      _id: { city: "$city", status: "$status" },
      count: { $sum: 1 }
  }}
])

// _id: null — aggregate entire collection
db.orders.aggregate([
  { $group: {
      _id: null,
      grandTotal: { $sum: "$amount" },
      totalOrders: { $sum: 1 }
  }}
])
```

**Accumulator operators:**

| Operator | Description |
|---|---|
| `$sum` | Sum of values, or `$sum: 1` to count |
| `$avg` | Average |
| `$min` / `$max` | Min / max value |
| `$first` / `$last` | First / last document in group |
| `$push` | Collect values into array |
| `$addToSet` | Collect unique values into array |
| `$count` | Count (MongoDB 5.0+) |

---

### $sort — sort (ORDER BY)

```js
db.orders.aggregate([
  { $group: { _id: "$userId", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } }   // -1 = desc, 1 = asc
])
```

⚠️ `$sort` before `$group` → can use index.
`$sort` after `$group` → sorts aggregated results, no index.

---

### $project — reshape (SELECT)

Include, exclude, rename, or compute fields.

```js
db.orders.aggregate([
  { $project: {
      _id: 0,
      userId: 1,
      amount: 1,
      // computed field
      amountWithTax: { $multiply: ["$amount", 1.2] },
      // rename field
      city: "$city",
      // conditional
      label: {
        $cond: { if: { $gte: ["$amount", 100] }, then: "big", else: "small" }
      }
  }}
])
```

---

### $unwind — deconstruct array

Turns each array element into a separate document.

```js
// { _id: 1, items: ["book", "pen"] }
// becomes:
// { _id: 1, items: "book" }
// { _id: 1, items: "pen" }

db.orders.aggregate([
  { $unwind: "$items" }
])

// count how many times each item was ordered
db.orders.aggregate([
  { $unwind: "$items" },
  { $group: { _id: "$items", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
```

By default, documents with missing or empty array are dropped. To keep them:
```js
{ $unwind: { path: "$items", preserveNullAndEmptyArrays: true } }
```

---

### $lookup — JOIN

Joins documents from another collection.

```js
// basic lookup
db.orders.aggregate([
  { $lookup: {
      from: "users",          // other collection
      localField: "userId",   // field in orders
      foreignField: "_id",    // field in users
      as: "user"              // output field (always array)
  }},
  { $unwind: "$user" }        // flatten array → object
])

// pipeline lookup — with additional filtering
db.orders.aggregate([
  { $lookup: {
      from: "products",
      let: { orderId: "$_id" },
      pipeline: [
        { $match: { $expr: { $eq: ["$orderId", "$$orderId"] } } },
        { $match: { inStock: true } }
      ],
      as: "products"
  }}
])
```

⚠️ Index `foreignField` for performance. `$lookup` is expensive on large collections.

---

### $limit and $skip — pagination

```js
// top 3 orders by amount
db.orders.aggregate([
  { $sort: { amount: -1 } },
  { $limit: 3 }
])

// pagination: page 2, 2 items per page
db.orders.aggregate([
  { $sort: { _id: 1 } },
  { $skip: 2 },
  { $limit: 2 }
])
```

---

### $count — count documents in pipeline

```js
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $count: "paidOrdersTotal" }
])
// result: { paidOrdersTotal: 4 }
```

---

### $addFields / $set — add or overwrite fields

`$set` is an alias for `$addFields` (MongoDB 4.2+). Adds new fields without removing existing ones.

```js
db.orders.aggregate([
  { $set: {
      amountWithTax: { $multiply: ["$amount", 1.2] },
      year: { $year: "$date" },
      isPaid: { $eq: ["$status", "paid"] }
  }}
])
```

vs `$project` — `$addFields` keeps all existing fields, `$project` only keeps what you specify.

---

### $replaceRoot / $replaceWith — promote nested document

Replaces the entire document with a sub-document.

```js
// { _id: 1, user: { name: "Alice", age: 25 }, amount: 100 }
db.orders.aggregate([
  { $replaceRoot: { newRoot: "$user" } }
])
// result: { name: "Alice", age: 25 }

// $replaceWith is shorthand (MongoDB 4.2+)
db.orders.aggregate([
  { $replaceWith: "$user" }
])

// merge original fields with sub-document
db.orders.aggregate([
  { $replaceRoot: {
      newRoot: { $mergeObjects: ["$$ROOT", "$user"] }
  }}
])
```

---

### $bucket — group into ranges

Group documents into manually defined buckets (ranges).

```js
// group orders by amount range
db.orders.aggregate([
  { $bucket: {
      groupBy: "$amount",
      boundaries: [0, 50, 100, 200, 500],  // bucket edges
      default: "other",                     // documents outside boundaries
      output: {
        count: { $sum: 1 },
        orders: { $push: "$_id" }
      }
  }}
])
// result:
// { _id: 0,   count: 1 }  → 0-50
// { _id: 50,  count: 1 }  → 50-100
// { _id: 100, count: 2 }  → 100-200
// { _id: 200, count: 1 }  → 200-500
```

**$bucketAuto** — automatically creates N equal buckets:
```js
db.orders.aggregate([
  { $bucketAuto: {
      groupBy: "$amount",
      buckets: 3    // MongoDB decides the boundaries
  }}
])
```

---

### $facet — multiple pipelines in parallel

Run several independent aggregations on the same input documents in one pass.

```js
db.orders.aggregate([
  { $facet: {
      // pipeline 1: summary by status
      byStatus: [
        { $group: { _id: "$status", count: { $sum: 1 } } }
      ],
      // pipeline 2: summary by city
      byCity: [
        { $group: { _id: "$city", total: { $sum: "$amount" } } }
      ],
      // pipeline 3: top 2 orders
      topOrders: [
        { $sort: { amount: -1 } },
        { $limit: 2 },
        { $project: { _id: 1, amount: 1 } }
      ]
  }}
])
// result: one document with three fields: byStatus, byCity, topOrders
```

Useful for dashboard queries — one request, multiple aggregations.

---

### $merge / $out — write results to collection

```js
// $out — replace entire collection with results
db.orders.aggregate([
  { $group: { _id: "$userId", total: { $sum: "$amount" } } },
  { $out: "userStats" }
])

// $merge — upsert into existing collection (MongoDB 4.2+)
db.orders.aggregate([
  { $group: { _id: "$userId", total: { $sum: "$amount" } } },
  { $merge: {
      into: "userStats",
      on: "_id",
      whenMatched: "replace",
      whenNotMatched: "insert"
  }}
])
```

Used for [[MongoDb - Views|Materialized Views]].

---

## Aggregation Expressions

Expressions compute values inside stages like `$project`, `$addFields`, `$group`.

### Arithmetic
```js
{ $add: ["$a", "$b"] }
{ $subtract: ["$a", "$b"] }
{ $multiply: ["$price", "$qty"] }
{ $divide: ["$total", "$count"] }
{ $mod: ["$value", 10] }
{ $abs: "$value" }
{ $round: ["$value", 2] }  // round to 2 decimal places
```

### String
```js
{ $concat: ["$first", " ", "$last"] }
{ $toUpper: "$name" }
{ $toLower: "$name" }
{ $substr: ["$name", 0, 3] }          // first 3 chars
{ $strLenCP: "$name" }                // string length
{ $split: ["$fullName", " "] }        // split to array
{ $trim: { input: "$name" } }
```

### Date
```js
{ $year: "$date" }
{ $month: "$date" }
{ $dayOfMonth: "$date" }
{ $dayOfWeek: "$date" }   // 1=Sun, 7=Sat
{ $hour: "$date" }
{ $dateToString: { format: "%Y-%m-%d", date: "$date" } }
```

### Conditional
```js
// $cond — ternary
{ $cond: { if: { $gte: ["$score", 90] }, then: "A", else: "B" } }
// shorthand
{ $cond: [{ $gte: ["$score", 90] }, "A", "B"] }

// $ifNull — fallback if null/missing
{ $ifNull: ["$optionalField", "default"] }

// $switch — multiple branches
{ $switch: {
    branches: [
      { case: { $gte: ["$score", 90] }, then: "A" },
      { case: { $gte: ["$score", 80] }, then: "B" },
      { case: { $gte: ["$score", 70] }, then: "C" }
    ],
    default: "F"
}}
```

### Array expressions
```js
{ $size: "$items" }                     // array length
{ $arrayElemAt: ["$items", 0] }         // element at index
{ $first: "$items" }                    // first element (4.4+)
{ $last: "$items" }                     // last element (4.4+)
{ $slice: ["$items", 2] }               // first 2 elements
{ $filter: {                            // filter array elements
    input: "$items",
    as: "item",
    cond: { $gte: ["$$item.price", 10] }
}}
{ $map: {                               // transform each element
    input: "$items",
    as: "item",
    in: { $multiply: ["$$item.price", 1.1] }
}}
{ $reduce: {                            // fold array to single value
    input: "$items",
    initialValue: 0,
    in: { $add: ["$$value", "$$this.price"] }
}}
```

---

## Performance Tips

```
1. $match early    → filters before heavy stages, uses indexes
2. $sort early     → can use indexes before $group
3. $project early  → reduces document size for subsequent stages
4. Index $lookup   → foreignField must be indexed
5. Avoid $unwind   → on large arrays without filtering first
```

### explain for pipelines
```js
db.orders.explain("executionStats").aggregate([
  { $match: { status: "paid" } },
  { $group: { _id: "$userId", total: { $sum: "$amount" } } }
])
```

---

## Full Example — Sales Dashboard

```js
db.orders.aggregate([
  // 1. only paid orders from 2024
  { $match: {
      status: "paid",
      date: { $gte: new Date("2024-01-01") }
  }},

  // 2. add computed fields
  { $addFields: {
      month: { $month: "$date" },
      amountWithTax: { $multiply: ["$amount", 1.2] }
  }},

  // 3. flatten items array
  { $unwind: "$items" },

  // 4. group by month and item
  { $group: {
      _id: { month: "$month", item: "$items" },
      revenue: { $sum: "$amountWithTax" },
      soldCount: { $sum: 1 }
  }},

  // 5. sort by revenue
  { $sort: { revenue: -1 } },

  // 6. reshape output
  { $project: {
      _id: 0,
      month: "$_id.month",
      item: "$_id.item",
      revenue: { $round: ["$revenue", 2] },
      soldCount: 1
  }}
])
```

---

## Stage Reference

| Stage | SQL equivalent | What it does |
|---|---|---|
| `$match` | WHERE / HAVING | Filter documents |
| `$group` | GROUP BY | Aggregate by key |
| `$sort` | ORDER BY | Sort documents |
| `$project` | SELECT | Shape output fields |
| `$unwind` | — | Deconstruct array to rows |
| `$lookup` | JOIN | Join another collection |
| `$limit` | LIMIT / TOP | Take first N |
| `$skip` | OFFSET | Skip first N |
| `$count` | COUNT(*) | Count documents |
| `$addFields` / `$set` | computed column | Add/overwrite fields |
| `$replaceRoot` | — | Promote sub-doc to root |
| `$bucket` | — | Group into ranges |
| `$bucketAuto` | — | Auto N equal ranges |
| `$facet` | — | Multiple pipelines in one pass |
| `$out` | INSERT INTO ... SELECT | Write results to collection |
| `$merge` | MERGE / UPSERT | Merge results into collection |

---

## Links

- [[MongoDb - Queries]]
- [[MongoDb - Views]]
- [[MongoDb - Indexes]]
- [[CSharp MongoDB Driver]]