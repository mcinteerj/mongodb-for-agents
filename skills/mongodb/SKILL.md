---
name: mongodb
description: "MANDATORY when working with MongoDB — covers schema design, data modeling, embedding vs referencing, document model, indexing strategy, compound indexes, ESR rule, partial indexes, aggregation pipelines, $lookup, $merge, Atlas Search, analyzers, $search, Atlas Vector Search, $vectorSearch, RAG patterns, performance tuning, explain plans, query optimization, connection pooling, security, RBAC, field-level encryption, connection strings, replica sets, sharding, transactions, change streams, time series collections, mongosh, Atlas CLI, PyMongo, Motor, Mongoose — YOU MUST read this skill before writing ANY MongoDB code, queries, schema definitions, index configurations, or Atlas deployments"
---

# MongoDB Agent Skill

Routing file for MongoDB expertise. Read the universal principles below, then load the relevant reference files for your task.

## Universal Principles

- **Document model thinking.** MongoDB is not a relational database. Stop normalizing. Design documents around how your application reads and writes data.
- **Design for access patterns first.** Identify your queries before you design your schema. The schema serves the queries, not the other way around.
- **Embed by default, reference when necessary.** Embed data that is read together. Reference data that is shared across documents, unbounded, or updated independently.
- **Every query should be backed by an index.** No collection scans in production. Period.
- **Compound indexes follow the ESR rule.** Equality → Sort → Range. Get this wrong and your index is useless.
- **Aggregation pipelines should filter early.** Push `$match` and `$project` to the front. Every document you eliminate early saves work in every subsequent stage.
- **Read explain plans before optimizing.** Don't guess. Use `explain("executionStats")` and read the actual execution plan.
- **Connection pooling is not optional.** One client per application lifecycle. Never open a connection per request.
- **Security is not a follow-up task.** Enable authentication, use SCRAM-SHA-256 or x.509, enforce least-privilege RBAC, encrypt in transit and at rest from day one.
- **Atlas features require Atlas.** Atlas Search, Vector Search, and serverless triggers are Atlas-only. Know what requires Atlas before architecting.

## Reference Routing Table

All reference files live in `references/` relative to this skill directory.

### `docs-navigation.md`
**YOU MUST read when:** navigating official MongoDB documentation, searching for specific driver APIs, or unsure where to find authoritative answers.
Contains: URL patterns for docs.mongodb.com, driver doc locations, manual section map, how to find version-specific content.

### `schema-design.md`
**YOU MUST read when:** designing a new collection, choosing between embedding and referencing, modeling one-to-many or many-to-many relationships, or restructuring existing documents.
Contains: Embedding vs referencing decision framework, document size limits, polymorphic pattern, subset pattern, bucket pattern, schema versioning strategies.

### `indexing.md`
**YOU MUST read when:** creating indexes, diagnosing slow queries, designing compound indexes, or working with unique/partial/TTL/text indexes.
Contains: ESR rule deep dive, index intersection, covered queries, index size estimation, partial index filters, wildcard indexes, hidden indexes for safe drops.

### `aggregation.md`
**YOU MUST read when:** writing `$lookup`, `$group`, `$unwind`, `$merge`, `$unionWith`, or any multi-stage pipeline, or converting SQL to aggregation.
Contains: Stage ordering for performance, pipeline optimization rules, `$facet` patterns, `$graphLookup` for tree structures, `$merge` for materialized views, memory limits and `allowDiskUse`.

### `performance.md`
**YOU MUST read when:** diagnosing slow queries, reading explain plans, tuning connection pools, profiling database operations, or capacity planning.
Contains: Explain plan interpretation, profiler configuration, connection pool sizing, WiredTiger cache tuning, read/write concern trade-offs, oplog sizing for replication lag.

### `atlas-search.md`
**YOU MUST read when:** implementing full-text search, configuring Atlas Search indexes, using `$search` or `$searchMeta`, or choosing analyzers.
Contains: Index definition syntax, analyzer selection guide, compound operator usage, faceting, scoring modifiers, autocomplete configuration, search vs find decision guide.

### `vector-search.md`
**YOU MUST read when:** implementing semantic search, RAG pipelines, storing embeddings, using `$vectorSearch`, building hybrid search, or configuring vector indexes.
Contains: Vector index definition with all parameters, `$vectorSearch` stage syntax, HNSW tuning (`numCandidates`), pre-filter vs post-filter patterns, hybrid search with RRF (why `$search` + `$vectorSearch` can't combine), RAG pipeline pattern, embedding storage conventions, common mistakes.

### `security.md`
**YOU MUST read when:** configuring authentication, setting up roles, managing connection strings with credentials, enabling encryption, or auditing access.
Contains: SCRAM-SHA-256 vs x.509, built-in roles vs custom roles, connection string auth options, TLS configuration, client-side field-level encryption (CSFLE), queryable encryption, audit log setup.

## Cross-Cutting Note

Schema design, indexing, and aggregation are deeply interdependent. A schema change affects which indexes are viable. Index design constrains aggregation performance. When working on any one of these, **read all three reference files** — `schema-design.md`, `indexing.md`, and `aggregation.md` — to avoid local optimizations that create global problems.
