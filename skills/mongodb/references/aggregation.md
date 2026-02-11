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
| `$facet` | — | Multiple sub-pipelines on same input |
| `$bucket` / `$bucketAuto` | — | Group into ranges |
| `$setWindowFields` | Window functions | Ranking, running totals, moving averages |
| `$densify` | — | Fill gaps in time series / sequences |
| `$fill` | — | Fill null/missing values |
| `$merge` | `INSERT ... ON CONFLICT UPDATE` | Upsert into collection (materialized views) |
| `$out` | `CREATE TABLE AS SELECT` | Replace entire collection (destructive) |

## SQL → Aggregation Mapping

| SQL | Aggregation |
|---|---|
| `WHERE` | `$match` (before `$group`) |
| `GROUP BY` | `$group` |
| `HAVING` | `$match` (after `$group`) |
| `SELECT` | `$project` |
| `ORDER BY` | `$sort` |
| `COUNT(*)` | `{ $group: { _id: null, n: { $sum: 1 } } }` |
| `SUM(col)` | `{ $group: { _id: ..., total: { $sum: "$col" } } }` |
| `DISTINCT` | `{ $group: { _id: "$field" } }` |
| `CASE WHEN` | `$switch` or `$cond` |
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

```js
db.collection.aggregate(pipeline, { allowDiskUse: true })
```

### Optimizer Auto-Reordering
- `$sort` → `$match` becomes `$match` → `$sort`
- `$project` → `$match` splits independent filters before `$project`
- `$lookup` → `$unwind` → `$match` coalesces into single `$lookup`
- `$sort` → `$limit` coalesces into top-N sort
- Adjacent `$match` stages combine with `$and`

### Index-Eligible Stages
- `$match` — only when first (after optimizer reordering)
- `$sort` — only when not preceded by `$project`/`$unwind`/`$group`
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

## Useful Expression Operators

| Operator | Use |
|---|---|
| `$ifNull` | Coalesce nulls: `{ $ifNull: ["$field", default] }` |
| `$filter` | Filter array inline without `$unwind` |
| `$map` | Transform array elements inline |
| `$mergeObjects` | Combine objects: `{ $mergeObjects: ["$defaults", "$overrides"] }` |
| `$first` / `$last` | Array element access (cleaner than `$arrayElemAt`) |
| `$dateTrunc` | Truncate to unit: `{ $dateTrunc: { date: "$d", unit: "month" } }` |
| `$getField` | Access fields with dots/special chars |
| `$convert` | Safe type conversion with `onError` fallback |

## Common Mistakes

### 1. `$unwind` before `$match`
```js
// ❌ Processes N×M documents
[{ $unwind: "$items" }, { $match: { "items.status": "shipped" } }]

// ✅ Filter first, or use $filter to avoid $unwind entirely
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
