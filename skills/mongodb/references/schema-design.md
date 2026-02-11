# Schema Design Reference

## Embed vs Reference Decision

```
1. Read together >80% of the time?          → Embed
2. Array unbounded or could exceed ~100?     → Reference
3. Child needs independent queries?          → Reference
4. Embedded data updated frequently?         → Reference
5. Embedding pushes doc past ~1MB?           → Reference (or Subset)
6. Need atomic single-doc writes?            → Embed
```

**Hybrid (Extended Reference):** Reference the collection BUT duplicate rarely-changing fields (name, address) into parent. Avoids `$lookup` for common reads.

## Relationship Patterns

### One-to-One → Embed
```json
{ "_id": "user1", "name": "Jake", "profile": { "bio": "...", "avatar": "..." } }
```

### One-to-Few → Embed array in parent
```json
{ "_id": "post1", "title": "...", "tags": ["mongo", "schema"] }
```

### One-to-Millions → Reference from child side
```json
// logs collection — NEVER embed unbounded arrays in parent
{ "_id": "log1", "host_id": "server42", "message": "...", "ts": ISODate() }
```

### Many-to-Many → Array of refs or junction collection
Small cardinality: array of IDs on one/both sides. Large cardinality: junction collection `{ student_id, course_id, enrolled_date }`.

### User Permissions (Many-to-Many — Good vs Bad)

```json
// ❌ BAD: Full role objects embedded — duplicated permissions, update = touch every user
{ "user": "Jake", "roles": [
    { "role_id": "r1", "name": "admin", "permissions": [/* 200 perms */] }
  ]
}

// ✅ GOOD: Extended reference for display, canonical roles separate
// users: { "_id": "u1", "roles": [{ "role_id": "r1", "name": "admin" }] }
// roles: { "_id": "r1", "name": "admin", "permissions": ["read", "write", "delete", ...] }
// Permission check: lookup role by ID (cached). Display: no lookup needed. Role update: single doc.
```

## Schema Patterns

| Scenario | Pattern |
|----------|---------|
| Different doc shapes, one collection | **Polymorphic** — add `type` discriminator |
| Large docs, only subset accessed | **Subset** — split hot/cold into two collections |
| High-volume events / time series | **Bucket** — group into fixed-size docs (or Time Series collection) |
| Few docs way larger than rest | **Outlier** — `has_extras` flag + overflow collection |
| Expensive repeated calculations | **Computed** — pre-store derived values |
| Frequent `$lookup` on same fields | **Extended Reference** — duplicate stable fields |
| Category trees, org charts | **Tree** — materialized paths + ancestors array |
| Many sparse filterable fields | **Attribute** — `[{k, v}]` array + single compound index |
| Schema evolving over time | **Versioning** — `schema_version` field + lazy migration |
| Old data rarely accessed | **Archive** — move cold docs to archive collection on schedule |

## Good vs Bad: Concrete Examples

### E-Commerce Order

```json
// ❌ BAD: 4 lookups to render one page
// orders → customers (ref) → items (ref) → products (ref) → addresses (ref)

// ✅ GOOD: Single read, snapshot at time of order
{
  "_id": "ord789",
  "customer": { "customer_id": "c1", "name": "Jake" },
  "shipping_address": { "street": "123 Main", "city": "Auckland" },
  "items": [
    { "product_id": "p1", "name": "Widget", "price": 9.99, "quantity": 2 }
  ],
  "total": 44.97, "status": "shipped",
  "created_at": ISODate("2025-01-15")
}
```

### Blog Comments

```json
// ❌ BAD: Unbounded array, 16MB risk
{ "_id": "post1", "comments": [ /* 50,000 entries */ ] }

// ✅ GOOD: Subset + reference
// posts: { "_id": "post1", "recent_comments": [/* latest 5 */], "comment_count": 50000 }
// comments: { "_id": "c1", "post_id": "post1", "text": "...", "ts": ISODate() }
```

### IoT Sensor Data

```json
// ❌ BAD: One doc per reading = 86,400 docs/sensor/day
{ "sensor": "temp1", "value": 22.1, "ts": ISODate("2025-01-15T00:00:01") }

// ✅ GOOD: Bucket = 24 docs/sensor/day + pre-computed stats
{
  "_id": "temp1_2025-01-15_00",
  "sensor": "temp1", "hour": 0, "count": 60,
  "readings": [{ "ts": ISODate("..."), "v": 22.1 }],
  "avg": 22.05, "min": 21.8, "max": 22.3
}
```

## Document Size

- **Hard limit**: 16MB — this is a hard cap, not a target. For optimal cache and network performance, target <100KB per document (see performance.md). Target <1MB for most docs.
- **Working set**: Frequently-accessed docs should fit in RAM. Large docs waste cache.
- **Too large** → Subset pattern. **Too many tiny** → Bucket pattern. **Unbounded arrays** → Reference + Outlier.
- **Projections help** but don't fix underlying I/O cost of reading large docs from disk.
- **Field names stored in every doc** — short names save space at scale (don't sacrifice readability for marginal gains).
- **Binary data >16MB** → GridFS.

## Schema Versioning

Always include `schema_version` from day one. Old + new docs coexist.

```json
{ "schema_version": 1, "name": "Jake", "phone": "555-1234" }
{ "schema_version": 2, "name": "Jake", "contacts": { "phone": "555-1234", "email": "j@e.com" } }
```

- **Lazy migration**: update to latest version on next write
- **Batch migration**: background job converts old docs
- Track progress: `db.coll.countDocuments({ schema_version: { $lt: CURRENT } })`
- Migrations must be **backward compatible** — new code reads old + new

## $jsonSchema Validation

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email", "created_at", "schema_version"],
      properties: {
        name: { bsonType: "string" },
        email: { bsonType: "string", pattern: "^.+@.+\\..+$" },
        age: { bsonType: "int", minimum: 0, maximum: 150 },
        created_at: { bsonType: "date" },
        schema_version: { bsonType: "int" },
        status: { enum: ["active", "inactive", "suspended"] },
        tags: { bsonType: "array", maxItems: 50, items: { bsonType: "string" } }
      }
    }
  },
  validationLevel: "moderate",  // won't reject existing invalid docs on update
  validationAction: "error"     // "warn" during migration rollout
})
// NOTE: schema_version in `required` means existing docs without it will fail
// validation on update (with "moderate") or always (with "strict"). Add field to
// existing docs first, or use validationLevel: "moderate" during transition.
```

**Polymorphic validation** — use `oneOf` with discriminator to validate different shapes in one collection:

```javascript
db.createCollection("events", {
  validator: { $jsonSchema: {
    bsonType: "object", required: ["type", "timestamp"],
    oneOf: [
      { properties: { type: { enum: ["click"] }, element_id: { bsonType: "string" } },
        required: ["element_id"] },
      { properties: { type: { enum: ["purchase"] }, amount: { bsonType: "decimal" }, currency: { bsonType: "string" } },
        required: ["amount", "currency"] }
    ]
  }}
})
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Over-normalization | `$lookup` everywhere, slow reads | Embed related data |
| Unbounded arrays | 16MB risk, perf degradation | Reference from child side |
| Mega-documents | Wastes RAM, slow updates | Subset pattern |
| Inconsistent shapes | Unpredictable queries | `$jsonSchema` validation |
| ObjectId stored as string | Wastes space, breaks queries | Use proper ObjectId type |
| No indexes on query fields | Collection scans | Index every query pattern |
| `$lookup` in hot paths | Slow reads | Extended Reference pattern |
| Generic key-value for everything | Loses type safety | Use typed fields; Attribute pattern only for truly variable specs |
| Ignoring write patterns | Read-optimized schema suffers on updates | Design for both read AND write access patterns |
