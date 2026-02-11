# Performance Tuning Reference

## explain() Interpretation

Always use `explain("executionStats")` — queryPlanner alone lacks runtime numbers.

### Critical Fields

| Field | Meaning |
|-------|---------|
| `winningPlan.stage` | Root execution stage. `COLLSCAN` = red flag. `IXSCAN` = good. |
| `nReturned` | Documents returned |
| `totalKeysExamined` | Index entries scanned |
| `totalDocsExamined` | Documents fetched from storage |
| `executionTimeMillis` | Total time (plan selection + execution, excludes network) |
| `rejectedPlans` | Alternatives optimizer discarded |
| `executionStages` | Per-stage breakdown with `executionTimeMillisEstimate` |

### Key Stage Types

`COLLSCAN` — full scan (fix immediately) · `IXSCAN` — index scan · `FETCH` — doc retrieval · `SORT` — in-memory sort (100 MB limit, errors unless `allowDiskUse: true`) · `SHARD_MERGE` · `EXPRESS` (8.0+)

Use `explain("allPlansExecution")` to see partial trial-phase stats for rejected plans — useful for understanding why the optimizer chose the winning plan.

### Decision Ratios

| Ratio | Ideal | If Bad |
|-------|-------|--------|
| `totalKeysExamined / nReturned` | ~1.0 | >>1 → scanning excess index entries; tighten index |
| `totalDocsExamined / nReturned` | ~1.0 | >>1 → fetching then filtering; index missing fields |
| `COLLSCAN` + `totalDocsExamined > 0` | Never | Add index covering filter + sort fields |

### Decision Tree

1. **COLLSCAN?** → Add index on filter/sort fields.
2. **IXSCAN but high keysExamined/nReturned?** → Index covers filter but not selectively. Refine compound index (ESR rule).
3. **SORT stage present?** → Index doesn't provide sort order. Add sort fields to compound index after equality fields.
4. **High docsExamined/nReturned with IXSCAN?** → Index finds candidates but filter discards most. Add filter fields to index.
5. **Ratios ~1.0 but slow?** → Working set exceeds cache, or large documents. Add projection, check cache metrics.

---

## Common Bottlenecks

- **Unbounded queries**: missing `limit()` returns entire result set — always bound reads
- **Unanchored regex**: `{field: /pattern/}` can't use index; `{field: /^prefix/}` can
- **`$ne` / `$nin`**: generally can't use indexes efficiently, degrades to scan
- **Large `$in` arrays**: >1000 elements degrades query planning performance
- **Missing projections**: without projection, full document transferred + deserialized; always project needed fields only
- **In-memory sorts**: `SORT` stage without index-provided order; 100 MB limit, then errors unless `allowDiskUse: true`
- **Large document fetches**: documents >16 KB stress WiredTiger cache; working set exceeding cache → eviction storms

---

## Document Size Guidance

- Max BSON document: **16 MB**. Practical target: **<100 KB** for optimal cache/network performance.
- WiredTiger stores documents **uncompressed** in cache — a 10 KB on-disk doc may be 30–50 KB in cache (3–5× expansion).
- Use projections aggressively. 1000 queries × 50 KB docs = 50 MB transfer; with projection to needed fields: ~2 MB.
- Split large sub-documents into separate collections if accessed independently.
- Monitor: `db.collection.stats().avgObjSize` — track growth over time.
- GridFS for binary data >256 KB or any file >16 MB.

---

## Connection Pooling

**One client per application lifecycle.** Never open per-request.

### Driver Defaults

| Setting | PyMongo | Node.js | Java |
|---------|---------|---------|------|
| `maxPoolSize` | 100 | 100 | 100 |
| `minPoolSize` | 0 | 0 | 0 |
| `maxIdleTimeMS` | None | None | 0 |
| `waitQueueTimeoutMS` | blocks | None | 120000 |

### Sizing

- Default (100) sufficient for most apps. Scale to **1.5–2× peak concurrent ops** if pool exhaustion observed.
- Each connection ≈ 1 MB server RAM. Atlas M10+: max 1500. M0/M2/M5: 500.
- **mongos**: each opens its own pool to each shard — multiply accordingly.
- Monitor: `db.serverStatus().connections` → `current` vs `available`.

---

## Database Profiler

```javascript
// Level 0: off (default) | Level 1: slow ops | Level 2: all ops
db.setProfilingLevel(1, { slowms: 100 })  // default slowms: 100ms
db.setProfilingLevel(1, { slowms: 50, sampleRate: 0.5 }) // 50% sampling
```

**Level 2 degrades throughput significantly.** Use level 1 in production.

- **filter** (4.4+): profile only matching operations (e.g., specific collections/commands)
- **Alternative**: `$queryStats` aggregation stage — lower overhead than profiler for production monitoring

### Reading Profile Data

```javascript
db.system.profile.find({"planSummary": "COLLSCAN"}).sort({ts: -1})  // collection scans
db.system.profile.find({millis: {$gt: 500}}).sort({ts: -1})         // slow queries
```

Key fields: `op`, `ns`, `millis`, `planSummary`, `nreturned`, `docsExamined`, `keysExamined`, `command`.

`system.profile` is a 1 MB capped collection. Resize if needed:
```javascript
db.setProfilingLevel(0)
db.system.profile.drop()
db.createCollection("system.profile", {capped: true, size: 10485760})
db.setProfilingLevel(1, {slowms: 100})
```

---

## Read/Write Concern Performance Trade-offs

### Write Concerns

| Concern | Durability | Relative Latency | Use Case |
|---------|-----------|-------------------|----------|
| `w: 0` | None (fire-and-forget) | Lowest | Metrics, logs |
| `w: 1` (default) | Primary ack | Baseline | General purpose |
| `w: "majority"` | Majority replicated | +10–50ms | Critical writes |
| `w: "majority", j: true` | Journaled on majority | +20–100ms | Maximum durability |

`w: "majority"` throughput ≈ **50–70%** of `w: 1` under heavy write load. Set `wtimeout` (e.g., 5000ms) to prevent indefinite blocking.

### Read Concerns

| Concern | Consistency | Relative Latency | Notes |
|---------|------------|-------------------|-------|
| `"local"` (default) | May read uncommitted | Baseline | Can read rolled-back data |
| `"available"` | Same as local, no causal | Baseline | ⚠️ Sharded: may return orphaned docs during migrations |
| `"majority"` | Committed only | +5–50ms | Increases cache pressure (snapshot history) |
| `"linearizable"` | Linearizable (single-doc) | +50–200ms | Issues no-op write to confirm primary |
| `"snapshot"` | Snapshot isolation | Medium | Multi-doc transactions only |

### Read Preference

| Preference | Best For | Trade-off |
|-----------|----------|-----------|
| `primary` | Consistency | All reads hit primary |
| `secondary` / `secondaryPreferred` | Offload primary, analytics | Stale by replication lag (0–10ms typical, can spike) |
| `nearest` | Geo-distributed apps | Saves 50–200ms cross-DC; may be stale |

**Caveat**: secondary reads don't scale total capacity linearly — secondaries also apply oplog entries, consuming resources.

---

## Monitoring Quick Reference

### Connections & Operations
```javascript
db.serverStatus().connections    // current, available, totalCreated
db.serverStatus().opcounters     // insert, query, update, delete
db.serverStatus().globalLock     // currentQueue.readers/writers (>0 = contention)
```

### WiredTiger Cache
```javascript
var c = db.serverStatus().wiredTiger.cache
// "bytes currently in the cache" / "maximum bytes configured" → fill ratio
// "pages evicted by application threads" → MUST be 0; >0 = cache pressure
// "tracked dirty bytes in the cache" → high = checkpoint/eviction falling behind
```

Cache default: **50% of (RAM − 1 GB)**. App-thread eviction starts at 95% full — operations stall.

### WiredTiger Eviction Thresholds

| Trigger | Default | Effect |
|---------|---------|--------|
| `eviction_target` | **80%** full | Background eviction threads start (default 4 threads) |
| `eviction_trigger` | **95%** full | **App threads** join eviction — ops stall |
| `eviction_dirty_target` | **5%** dirty | Background eviction prioritizes dirty pages |
| `eviction_dirty_trigger` | **20%** dirty | **App threads** throttle on dirty writes |

---

## Bulk Operations

```javascript
db.col.bulkWrite([
  {insertOne: {document: {a: 1}}},
  {updateOne: {filter: {a: 2}, update: {$set: {b: 2}}}},
  {deleteOne: {filter: {a: 3}}}
], {ordered: false})
```

- **Batch size**: 1,000–10,000 ops per batch (sweet spot). Wire protocol limit: 16 MB per message; driver auto-splits.
- `ordered: false` — ~2–5× faster than ordered (parallelism + continues on error)
- `ordered: true` (default) — stops at first error, sequential execution
- Fewer network round-trips than individual ops
- **Import pattern**: drop non-essential indexes → bulk insert (`w: 1`, `ordered: false`) → recreate indexes
- `updateMany`/`deleteMany` hold intent locks — can block other ops on same collection

---

### Active Operations
```javascript
db.currentOp({"secs_running": {$gt: 10}})  // long-running ops
db.killOp(<opId>)                            // kill slow query
```

### Key Alerts

| Metric | Warning | Critical |
|--------|---------|----------|
| Cache fill ratio | >80% | >95% |
| App-thread evictions | >0 | sustained |
| `globalLock.currentQueue` | >10 | >50 |
| Connection utilization | >70% pool | >90% pool |
| Replication lag | >1s | >10s |
