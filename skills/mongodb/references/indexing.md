# MongoDB Indexing Reference

## ESR Rule: Equality → Sort → Range

Order compound index fields by: **E**quality (exact match) → **S**ort → **R**ange (inequalities).

```js
// Query
db.orders.find({ status: "shipped", total: { $gt: 100 } }).sort({ createdAt: 1 })

// ✅ CORRECT (ESR)
db.orders.createIndex({ status: 1, createdAt: 1, total: 1 })
//                       E            S              R

// ❌ WRONG — range before sort → SORT_IN_MEMORY
db.orders.createIndex({ status: 1, total: 1, createdAt: 1 })
//                       E           R            S

// ❌ WRONG — range first, equality buried → inefficient prefix
db.orders.createIndex({ total: 1, status: 1, createdAt: 1 })
```

**`$in` behavior**: <201 elements = equality (place early); ≥201 = range. `$ne`, `$nin`, `$regex` = always range.

**ERS exception**: When range predicate is highly selective (very few matching docs), placing Range before Sort can win despite in-memory sort on the tiny result set.

## Index Type Quick Reference

| Type | Syntax | Use When |
|------|--------|----------|
| Single field | `{ email: 1 }` | One-field exact/range/sort |
| Compound | `{ a: 1, b: -1, c: 1 }` | Multi-field filter + sort (ESR) |
| Multikey | `{ tags: 1 }` (auto) | Array field queries; max 1 array field per compound |
| Text | `{ body: "text" }` | Full-text search (1 per collection; prefer Atlas Search) |
| 2dsphere | `{ location: "2dsphere" }` | Geospatial `$near`, `$geoWithin` |
| Hashed | `{ _id: "hashed" }` | Hash-based sharding; no range queries |
| Wildcard | `{ "meta.$**": 1 }` | Polymorphic/unknown schemas; no compound-style multi-field |
| TTL | `{ lastAccess: 1 }, { expireAfterSeconds: 3600 }` | Auto-expire docs; single Date field only |
| Partial | `{ field: 1 }, { partialFilterExpression: {...} }` | Index subset of docs; preferred over sparse |
| Unique | `{ email: 1 }, { unique: true }` | Enforce uniqueness; combine with partial for optional fields |
| Hidden | `hideIndex("name")` | Test impact of drop before committing |

## Covered Queries

Index alone satisfies the query — no `FETCH` stage. Requirements:
1. All filter fields in the index
2. All projected fields in the index
3. `_id: 0` in projection (unless `_id` is in the index)
4. No field in the query is compared to `null` (null checks require doc fetch)

```js
db.inventory.createIndex({ type: 1, item: 1, qty: 1 })
db.inventory.find({ type: "food" }, { type: 1, item: 1, qty: 1, _id: 0 })
// → totalDocsExamined: 0, no FETCH stage
```

## Reading `explain()` Output

### Verbosity Modes
```js
.explain()                    // queryPlanner only
.explain("executionStats")    // + execution metrics (use this)
.explain("allPlansExecution") // + rejected plan stats
```

### Key Fields

| Field | Ideal Value | Meaning |
|-------|-------------|---------|
| `winningPlan.stage` | `IXSCAN` | Using index (not `COLLSCAN`) |
| `SORT` stage present | Absent | In-memory sort happening if present |
| `FETCH` after `IXSCAN` | Absent for covered | Doc lookup needed |
| `totalKeysExamined` | ≈ `nReturned` | Index entries scanned |
| `totalDocsExamined` | 0 (covered) or ≈ `nReturned` | Docs fetched from storage |
| `executionTimeMillis` | Low | Wall-clock time |

### Efficiency Ratios
- `totalKeysExamined / nReturned` → target ≈ 1.0
- `totalDocsExamined / nReturned` → 0 = covered; ≈ 1.0 = good; >> 1.0 = poor selectivity
- `SORT` stage without index sort → add sort field to index (ESR position)

## Index Management

```js
// List indexes
db.collection.getIndexes()

// Check usage (drop indexes with 0 ops)
db.collection.aggregate([{ $indexStats: {} }])

// Check sizes
db.collection.stats().indexSizes

// Index builds: since 4.2, exclusive lock only at start/end (not full duration).
// Monitor in-progress builds: db.currentOp({ "command.createIndexes": { $exists: true } })

// Safe drop: hide → monitor → drop or unhide
db.collection.hideIndex("index_name")
// ... monitor for regression ...
db.collection.dropIndex("index_name")     // or unhideIndex() to restore
```

## Common Mistakes

### 1. Wrong compound index field order
```js
// Query: tenantId (eq) + createdAt (range), sort by priority
db.tickets.find({ tenantId: "abc", createdAt: { $gte: d1 } }).sort({ priority: -1 })

// ❌ Range before sort
db.tickets.createIndex({ tenantId: 1, createdAt: 1, priority: -1 })
// ✅ ESR
db.tickets.createIndex({ tenantId: 1, priority: -1, createdAt: 1 })
```

### 2. Missing index on `$lookup` foreign field
```js
// Aggregation stage — foreignField MUST be indexed in the "from" collection
{ $lookup: {
    from: "orders",
    localField: "_id",
    foreignField: "customerId",   // ← index this field
    as: "customerOrders"
}}

// Without this index, every $lookup iteration does a COLLSCAN:
db.orders.createIndex({ customerId: 1 })
```

### 3. Redundant single-field index
`{ a: 1 }` is redundant when `{ a: 1, b: 1 }` exists. Drop it (unless it has unique/sparse properties the compound doesn't).

### 4. Sort direction mismatch
`{ a: 1, b: -1 }` ≠ `{ a: 1, b: 1 }` for sort. Index must match or be the exact inverse of query sort.

### 5. Forgetting `_id: 0` for covered queries
`_id` is returned by default. Without `_id: 0`, query fetches documents even if all other fields are indexed.

### 6. Indexing `$ne`/`$nin` fields
These are non-selective — they scan most of the index. Restructure the query or accept the scan.

### 7. Too many indexes
Each index costs RAM + write amplification. Max 64 per collection. Audit with `$indexStats` regularly.

### 8. Relying on index intersection over compound indexes
MongoDB can intersect multiple single-field indexes, but compound indexes almost always outperform intersection. Intersection can't provide covered queries or reliable sort support, and the optimizer may not choose it. Create compound indexes for known query patterns; treat intersection as a fallback for ad-hoc queries only.

### 9. Not running `explain()` after creating an index
Don't assume — verify the index is actually used and ratios are healthy.
