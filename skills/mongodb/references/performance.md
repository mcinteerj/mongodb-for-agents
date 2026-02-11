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
// Level 0: off | Level 1: slow ops | Level 2: all ops
db.setProfilingLevel(1, { slowms: 100 })
db.setProfilingLevel(1, { slowms: 50, sampleRate: 0.5 }) // 50% sampling
```

**Level 2 degrades throughput significantly.** Use level 1 in production.

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
| `"majority"` | Committed only | +5–50ms | Increases cache pressure (snapshot history) |
| `"linearizable"` | Linearizable (single-doc) | +50–200ms | Issues no-op write to confirm primary |
| `"snapshot"` | Snapshot isolation | Medium | Multi-doc transactions only |

### Read Preference

| Preference | Best For | Trade-off |
|-----------|----------|-----------|
| `primary` | Consistency | All reads hit primary |
| `secondary` / `secondaryPreferred` | Offload primary, analytics | Stale by replication lag (0–10ms typical, can spike) |
| `nearest` | Geo-distributed apps | Saves 50–200ms cross-DC; may be stale |

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
