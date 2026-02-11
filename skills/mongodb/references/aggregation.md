# Aggregation Pipeline Reference

## Pipeline Ordering Rules

**Filter early, reshape late.**

1. `$match` first — only stage that uses indexes (when first after optimizer reordering)
2. `$match` + `$sort` at start = index-backed sort
3. `$sort` → `$limit` coalesces into top-N sort (O(N) memory)
4. `$project` last — server auto-projects needed fields internally; early `$project` rarely helps and often removes fields needed later
5. `$match` before `$unwind` always — avoid cardinality explosion

## Stage Quick Reference

| Stage | SQL Equivalent | Purpose |
|---|---|---|
| `$match` | `WHERE` / `HAVING` | Filter documents |
| `$group` | `GROUP BY` | Aggregate by key. Accumulators: `$sum`, `$avg`, `$min`, `$max`, `$push`, `$addToSet`, `$first`, `$last`, `$topN` |
| `$project` | `SELECT` | Include/exclude/compute fields |
| `$set` / `$addFields` | — | Add fields without dropping others |
| `$unset` | — | Remove fields |
| `$sort` | `ORDER BY` | Sort documents |
| `$limit` / `$skip` | `LIMIT` / `OFFSET` | Pagination |
| `$unwind` | — | Deconstruct array → one doc per element |
| `$lookup` | `LEFT JOIN` | Join collections |
| `$graphLookup` | Recursive CTE | Recursive/transitive joins |
| `$unionWith` | `UNION ALL` | Combine collections |
| `$facet` | — | Multiple sub-pipelines on same input. ⚠️ As first stage = COLLSCAN — put `$match` before it |
| `$bucket` / `$bucketAuto` | — | Group into ranges |
| `$setWindowFields` | Window functions | Ranking, running totals, moving averages |
| `$densify` | — | Fill gaps in time series / sequences |
| `$fill` | — | Fill null/missing values |
| `$merge` | `INSERT ... ON CONFLICT UPDATE` | Upsert into collection (materialized views). Can target source collection |
| `$out` | `CREATE TABLE AS SELECT` | Replace entire collection (destructive) |

## SQL → Aggregation Mapping

| SQL | Aggregation |
|---|---|
| `WHERE` | `$match` (before `$group`) |
| `GROUP BY` | `$group` |
| `HAVING` | `$match` (after `$group`) |
| `SELECT` | `$project` |
| `ORDER BY` | `$sort` |
| `LIMIT` | `$limit` |
| `OFFSET` / `SKIP` | `$skip` |
| `JOIN` / `LEFT JOIN` | `$lookup` |
| `UNION ALL` | `$unionWith` |
| `COUNT(*)` | `{ $group: { _id: null, n: { $sum: 1 } } }` |
| `SUM(col)` | `{ $group: { _id: ..., total: { $sum: "$col" } } }` |
| `DISTINCT` | `{ $group: { _id: "$field" } }` |
| `CASE WHEN` | `$switch` or `$cond` |
| Window functions | `$setWindowFields` |
| `CREATE TABLE AS SELECT` | `$out` |
| `INSERT ... ON CONFLICT UPDATE` | `$merge` |
| `EXISTS (subquery)` | `$lookup` + `$match: { result: { $ne: [] } }` |

## $lookup Patterns

### ✅ Equality match (preferred — SBE-eligible in 6.0+)
```js
{ $lookup: { from: "B", localField: "bId", foreignField: "_id", as: "b" } }
```
**Always index `foreignField`** on the foreign collection.

### ✅ Concise correlated with filter (5.0+)
```js
{ $lookup: {
    from: "B", localField: "bId", foreignField: "_id",
    pipeline: [{ $project: { name: 1 } }],
    as: "b"
} }
```

### ⚠️ Full pipeline $lookup (use carefully)
```js
{ $lookup: {
    from: "B", let: { x: "$field" },
    pipeline: [{ $match: { $expr: { $eq: ["$y", "$$x"] } } }],
    as: "b"
} }
```
Not SBE-eligible. Runs sub-pipeline per input doc. Ensure sub-pipeline starts with indexed `$match`.

### ❌ Anti-patterns
- Unindexed `foreignField` — full collection scan per input doc
- Returning huge arrays without immediate `$unwind` + `$match`
- Multiple `$lookup` stages when **embedding** would eliminate the join
- `$lookup` inside `$facet` — 100MB limit, no disk spill
- `$out`/`$merge` inside `$lookup` pipeline — not allowed

## Performance Rules

### Memory Limits

| Constraint | Limit | Fix |
|---|---|---|
| Per-stage memory | 100 MB | `{ allowDiskUse: true }` |
| `$facet` sub-pipeline | 100 MB | `allowDiskUse` does **NOT** help. Use `$limit` inside facets or separate aggregations |
| Output document | 16 MB (BSON) | Use `$merge`/`$out` or restructure |

### Optimizer Auto-Reordering

| Before | After | Why |
|---|---|---|
| `$project` → `$match` | `$match` (independent filters) → `$project` → `$match` (dependent) | Use index, reduce docs early |
| `$sort` → `$match` | `$match` → `$sort` | Fewer docs to sort |
| `$project`/`$unset` → `$skip` | `$skip` → `$project`/`$unset` | Skip before reshaping |
| `$redact` → `$match` | `$match` (duplicate) → `$redact` → `$match` | Enable index use |

### Optimizer Coalescence

| Pattern | Becomes | Effect |
|---|---|---|
| `$sort` → `$limit` | `$sort` (with limit N) | Top-N sort, O(N) memory |
| `$limit` → `$limit` | `$limit` (min of both) | |
| `$skip` → `$skip` | `$skip` (sum of both) | |
| `$match` → `$match` | `$match` (`$and` combined) | |
| `$lookup` → `$unwind` → `$match` | `$lookup` (with unwinding + matching) | Avoids large intermediate arrays |
| `$sort` → `$skip` → `$limit` | `$sort` (limit = skip+limit) → `$skip` → `$limit` | Top-(skip+limit) sort |

### Index-Eligible Stages
- `$match` — only when first (after optimizer reordering)
- `$sort` — only when not preceded by `$project`/`$unwind`/`$group`
- `$group` — can use index for `$first`/`$last` with matching sort order
- `$geoNear` — always uses geospatial index

## $merge for Materialized Views

```js
// Incremental refresh — only process new data
db.orders.aggregate([
  { $match: { date: { $gte: lastRefreshDate } } },
  { $group: { _id: { month: { $month: "$date" } }, total: { $sum: "$amount" } } },
  { $merge: {
      into: "monthlySales", on: "_id",
      whenMatched: [{ $set: { total: { $add: ["$$new.total", "$total"] } } }],
      whenNotMatched: "insert"
  } }
])
```

`$merge` can write back to the **same** collection being aggregated.

## Expression Operators

### Commonly missed
| Operator | Use |
|---|---|
| `$ifNull` | `{ $ifNull: ["$field", defaultValue] }` — coalesce nulls |
| `$mergeObjects` | Combine objects: `{ $mergeObjects: ["$defaults", "$overrides"] }` |
| `$first` / `$last` (array) | `{ $first: "$arr" }` — cleaner than `$arrayElemAt` for [0] |
| `$filter` | Filter array inline without `$unwind` |
| `$map` | Transform array elements inline |
| `$reduce` | Fold over array: `{ $reduce: { input: "$arr", initialValue: 0, in: { $add: ["$$value", "$$this"] } } }` |
| `$zip` | Combine parallel arrays |
| `$getField` | Access fields with dots/special chars |
| `$setField` | Set fields with special chars |
| `$dateTrunc` | Truncate to unit: `{ $dateTrunc: { date: "$d", unit: "month" } }` |
| `$dateAdd` / `$dateSubtract` | Date arithmetic |
| `$convert` | Safe type conversion with `onError` fallback |
| `$accumulator` | Custom JS accumulator in `$group` (escape hatch) |
| `$function` | Arbitrary JS expression (escape hatch) |

### Commonly misused
| Pattern | Problem | Fix |
|---|---|---|
| `{ $eq: [field, null] }` | Matches both `null` and missing | Use `$type` check if distinction matters |
| String concat with `+` | Not valid in agg expressions | Use `$concat` |
| `$toObjectId` on invalid input | Throws | Use `$convert` with `onError` |
| `$project` + `$addFields` | Redundant reshaping | Use `$set` to add, `$project` only at end |
| `$group` + `$push: "$$ROOT"` | Accumulates entire docs in memory | Use `$limit` upstream or `$topN` |

## Common Mistakes

### 1. `$unwind` before `$match`
```js
// ❌ Processes N×M documents
[{ $unwind: "$items" }, { $match: { "items.status": "shipped" } }]

// ✅ Double-$match: pre-filter docs, then filter unwound elements
[
  { $match: { "items.status": "shipped" } },
  { $unwind: "$items" },
  { $match: { "items.status": "shipped" } }
]

// ✅ Even better — avoid $unwind entirely with $filter
[{ $project: { items: { $filter: { input: "$items", cond: { $eq: ["$$this.status", "shipped"] } } } } }]
```

### 2. Missing `preserveNullAndEmptyArrays` on `$unwind`
```js
// ❌ Silently drops docs with empty/null/missing arrays
{ $unwind: "$tags" }
// ✅
{ $unwind: { path: "$tags", preserveNullAndEmptyArrays: true } }
```

### 3. `$group` + `$push: "$$ROOT"` — accumulates entire docs in memory. Use `$topN` or `$limit` upstream.

### 4. Early `$project` removing fields needed later — server auto-optimizes; only `$project` at the end.

### 5. Client-side aggregation instead of server-side — let the database do the work.

### 6. Missing indexes on `$match` fields and `$lookup` foreign fields.

### 7. `$facet` output exceeding 16MB — use `$limit` inside sub-pipelines.
