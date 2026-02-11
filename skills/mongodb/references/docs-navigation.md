# MongoDB Documentation Navigation

Quick-reference for AI agents to find MongoDB docs efficiently.

## URL Pattern Table

All URLs prefix: `https://www.mongodb.com/docs`

| Section | Pattern | Example |
|---------|---------|---------|
| Server Manual | `/manual/{topic}/` | `/manual/indexes/` |
| Manual Reference | `/manual/reference/{type}/` | `/manual/reference/command/` |
| Agg Stages | `/manual/reference/operator/aggregation/{stage}/` | `.../aggregation/lookup/` |
| Query Operators | `/manual/reference/operator/query/{op}/` | `.../query/eq/` |
| Atlas | `/atlas/{feature}/` | `/atlas/atlas-search/` |
| mongosh | `/mongodb-shell/` | — |
| Database Tools | `/database-tools/` | — |
| Atlas CLI | `/atlas/cli/current/` | — |
| Pinned Version | `/manual/v{MAJOR}.{MINOR}/...` | `/manual/v8.0/indexes/` |

Default `/manual/` = latest stable. Pin only for version-specific debugging.

## Driver Docs

| Driver | URL |
|--------|-----|
| PyMongo | `/languages/python/pymongo-driver/current/` |
| Node.js | `/drivers/node/current/` |
| Java Sync | `/drivers/java/sync/current/` |
| Go | `/drivers/go/current/` |
| C#/.NET | `/drivers/csharp/current/` |
| Rust | `/drivers/rust/current/` |
| Kotlin | `/drivers/kotlin/coroutine/current/` |
| Motor (async Python) | `/drivers/motor/` |
| C++ | `/languages/cpp/cpp-driver/current/` |
| PHP | `/drivers/php-drivers/` |
| Ruby | `/drivers/ruby/current/` |
| Swift | `/drivers/swift/current/` |
| Scala | `/drivers/scala/current/` |
| Mongoose (ODM) | External: [mongoosejs.com/docs](https://mongoosejs.com/docs) |
| Drivers Hub | `/drivers/` |

**Note**: PyMongo + C++ use `/languages/` path; others use `/drivers/`. Mongoose is community-maintained, not MongoDB official docs.

## Key Landing Pages

### Server Manual
| Topic | Path |
|-------|------|
| CRUD | `/manual/crud/` |
| Indexes | `/manual/indexes/` |
| Aggregation | `/manual/aggregation/` |
| Data Modeling | `/manual/data-modeling/` |
| Schema Patterns | `/manual/data-modeling/design-patterns/` |
| Transactions | `/manual/core/transactions/` |
| Replication | `/manual/replication/` |
| Sharding | `/manual/sharding/` |
| Change Streams | `/manual/changeStreams/` |
| Time Series | `/manual/core/timeseries-collections/` |
| Security | `/manual/security/` |
| Connection Strings | `/manual/reference/connection-string/` |
| Limits | `/manual/reference/limits/` |
| Error Codes | `/manual/reference/error-codes/` |
| Shell Methods | `/manual/reference/method/` |

### Atlas
| Topic | Path |
|-------|------|
| Atlas Search | `/atlas/atlas-search/` |
| Vector Search | `/atlas/atlas-vector-search/vector-search-overview/` |
| $vectorSearch | `/atlas/atlas-vector-search/vector-search-stage/` |
| AI Integrations | `/atlas/ai-integrations/` |
| Triggers | `/atlas/atlas-ui/triggers/` |
| Stream Processing | `/atlas/atlas-stream-processing/` |
| Atlas Admin API | `/atlas/reference/api-resources-spec/v2/` |

## Search Strategy

1. **Construct URL directly** — use patterns above. Fastest for known topics. Agg stages are the most predictable: `aggregation/{stage}/`.
2. **`brave-search "site:mongodb.com/docs {query}"`** — for discovery, error messages, niche topics, or when URL isn't guessable.
3. **`firecrawl scrape {url}`** — to extract page content. MongoDB docs are JS-rendered; `trafilatura` fails on them.
4. **Development hub** (`/docs/development/`) — best single-page index for orientation across all dev topics.

> **Note:** Tool recommendations above (firecrawl, trafilatura, brave-search) are suggestions. Agents should use whatever web fetching tools they have available — any HTTP client, browser automation, or MCP-based fetch tool will work.

## Gotchas

- **Search & Vector Search docs live under `/atlas/`** even for self-managed deployments.
- **`llms.txt`** exists at `/docs/llms.txt` (~4.5MB, 23K lines) — too large/noisy; always prefer search.
- **Atlas CLI versions**: `/atlas/cli/current/` not `/atlas/cli/latest/`.
- Driver `current` = latest stable. Pinned driver versions: `/drivers/{lang}/v{X}.{Y}/` (rarely needed).
