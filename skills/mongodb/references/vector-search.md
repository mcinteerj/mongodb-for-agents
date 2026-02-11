# Atlas Vector Search Reference

> Requires Atlas **v6.0.11+** or **v7.0.2+**. Supports up to **8192 dimensions**.

## Vector Index Definition

```javascript
db.collection.createSearchIndex(
  "vector_index",      // index name
  "vectorSearch",      // index type
  {
    fields: [
      {
        type: "vector",
        path: "embedding",       // field containing vectors
        numDimensions: 1536,     // must match embedding model output exactly (max 8192)
        similarity: "cosine"     // "cosine" | "euclidean" | "dotProduct"
      },
      { type: "filter", path: "category" },  // pre-filter fields
      { type: "filter", path: "year" }
    ]
  }
)
```

**Similarity metrics:**
- `cosine` — normalized embeddings (OpenAI, Cohere, most models). Default choice.
- `dotProduct` — pre-normalized unit vectors. Fastest. Equivalent to cosine for unit vectors.
- `euclidean` — absolute distance matters (spatial data, image features).

**Filter field types:** boolean, date, objectId, numeric, string, UUID.

## `$vectorSearch` Stage

**Must be the first stage.** Cannot appear in `$lookup` sub-pipeline or `$facet`.

```javascript
db.collection.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: [0.1, 0.2, ...],   // must match numDimensions
      numCandidates: 200,              // ANN candidates. Omit for ENN.
      limit: 10,                       // max results
      filter: {                        // optional pre-filter (fields must be indexed)
        category: "tech",
        year: { $gte: 2020 }
      },
      exact: false                     // true = exhaustive (ENN). Default false.
    }
  },
  {
    $project: {
      title: 1, text: 1,
      score: { $meta: "vectorSearchScore" }  // 0–1, higher = more similar
    }
  }
])
```

**Filter operators:** `$eq`, `$ne`, `$gt`, `$lt`, `$gte`, `$lte`, `$in`, `$nin`, `$exists`, `$not`, `$nor`, `$and`, `$or`.

## HNSW Tuning

Atlas manages HNSW internals (`efConstruction`, `M`) — **not user-configurable** via standard index JSON API.

> Note: Atlas UI may expose HNSW parameters on some tiers/versions. Check the JSON editor for your cluster.

The primary user-facing knob is **`numCandidates`** at query time:
- Controls recall vs latency tradeoff.
- **Set ≥20× `limit`**. E.g., `limit: 5` → `numCandidates: 100+`.
- Too low → poor recall. Too high → slower queries, diminishing returns.

## Pre-Filter vs Post-Filter

### Pre-filter (preferred) — `filter` in `$vectorSearch`
Narrows candidates **before** ANN search. Fields must be declared as `type: "filter"` in index.

### Post-filter — `$match` after `$vectorSearch`
Filters **after** ANN search. Can use any field. **Risk:** ANN returns `limit` docs, `$match` removes some → fewer results than expected.

**Rule:** Always pre-filter when possible. If you must post-filter, over-fetch with a higher `limit` then `$limit` again after `$match`.

## Hybrid Search (Vector + Full-Text)

**`$search` and `$vectorSearch` cannot be in the same pipeline.** Both require first-stage position and use different index types.

### Reciprocal Rank Fusion (RRF) Pattern

Run two queries, merge client-side:

```javascript
const vectorResults = await coll.aggregate([
  { $vectorSearch: { index: "vec_idx", path: "embedding",
    queryVector: qVec, numCandidates: 100, limit: 20 } },
  { $project: { _id: 1, title: 1, score: { $meta: "vectorSearchScore" } } }
]).toArray();

const textResults = await coll.aggregate([
  { $search: { index: "text_idx", text: { query: "search terms", path: "title" } } },
  { $limit: 20 },
  { $project: { _id: 1, title: 1, score: { $meta: "searchScore" } } }
]).toArray();

function rrfFuse(resultSets, k = 60) {
  const scores = {};
  for (const results of resultSets) {
    results.forEach((doc, rank) => {
      const id = doc._id.toString();
      if (!scores[id]) scores[id] = { doc, score: 0 };
      scores[id].score += 1 / (k + rank + 1);
    });
  }
  return Object.values(scores).sort((a, b) => b.score - a.score);
}

const merged = rrfFuse([vectorResults, textResults]);
```

### Alternative: `vectorSearch` Operator inside `$search` Stage
MongoDB also supports a `vectorSearch` **operator** within a `$search` compound query. This is **not** the same as the `$vectorSearch` aggregation stage — it is a search operator that runs inside a `$search` stage using an Atlas Search index (not a vectorSearch index). This enables single-pipeline hybrid search — combining text, fuzzy, phrase, and vector queries in one `$search` compound without client-side RRF. This does not contradict the "`$search` + `$vectorSearch` can't combine" rule, because no `$vectorSearch` aggregation stage is involved. Check Atlas Search docs for syntax and index requirements.

## Quantization

Store vectors in compressed formats to reduce index size and cost:

| Format | BSON Subtype | Storage vs float32 | Accuracy |
|---|---|---|---|
| `float32` | BinData vector | Baseline | Full |
| `int8` (scalar) | BinData vector | ~25% of float32 | Near-full; good default tradeoff |
| `int1` (binary) | BinData vector | ~3% of float32 | Lower; use for large-scale coarse retrieval |

**How:** Convert/generate embeddings to target format before insertion. The vectorSearch index operates on whatever is stored — no separate quantization config in index JSON.

**When to use:** Large collections where storage/memory cost matters. Start with `int8` for cost savings with minimal accuracy loss. Some providers output int8 natively (e.g., Cohere embed v3).

## RAG Pipeline Pattern

```
User query → Embed query → $vectorSearch (pre-filtered) → Top-K docs → LLM prompt → Response
```

```javascript
async function ragQuery(userQuery) {
  const queryEmbedding = await embedModel.embed(userQuery);
  const context = await db.collection("docs").aggregate([
    { $vectorSearch: {
        index: "rag_vec_idx", path: "embedding",
        queryVector: queryEmbedding, numCandidates: 200, limit: 5,
        filter: { status: "published" }
    }},
    { $project: { text: 1, title: 1, source: 1,
        score: { $meta: "vectorSearchScore" } } }
  ]).toArray();

  const ctxText = context.map(d => `[${d.title}]: ${d.text}`).join("\n\n");
  return await llm.complete(
    `Answer based on context:\n\n${ctxText}\n\nQuestion: ${userQuery}`
  );
}
```

**Chunking:** 500–1000 tokens per chunk, 50–100 token overlap. Each chunk = one document with its own embedding + `parentDocId` + `chunkIndex` for traceability.

**Hybrid RAG:** Combine vector + full-text via RRF (§Hybrid Search) for better retrieval — full-text catches exact keyword matches that embeddings miss.

## Embedding Storage Conventions

```javascript
{
  _id: ObjectId("..."),
  text: "Original content for RAG context",
  title: "Document Title",
  category: "tech",                     // filterable metadata
  year: 2024,
  source: "https://example.com/doc",    // provenance for RAG traceability
  embedding: [0.1, -0.02, ..., 0.3],   // float array or BSON vector BinData
  chunkIndex: 0,
  parentDocId: ObjectId("...")
}
```

- Field name: `embedding` (singular, top-level). Be consistent across collections.
- Co-locate text + metadata + embedding in same document.
- For storage savings: BSON vector subtype (`float32`, `int8`, `int1`) — see §Quantization.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Wrong `numDimensions` | Must exactly match embedding model output (max 8192) |
| Wrong similarity metric | Use `cosine` for OpenAI/Cohere/most. Check model docs. |
| `numCandidates` too low | Set ≥20× `limit` |
| Not pre-filtering | Add `filter` type fields to index; use `filter` in `$vectorSearch` |
| Filter field not in index | Add `{ type: "filter", path: "field" }` to index definition |
| `$search` + `$vectorSearch` same pipeline | Impossible. Use RRF or `vectorSearch`-in-`$search` operator. |
| `$vectorSearch` not first stage | Must always be first stage |
| Post-filter shrinks results below limit | Switch to pre-filter or over-fetch then re-limit |
| Query/doc embedding model mismatch | Same model for both. Always. |
| Not projecting score | Add `score: { $meta: "vectorSearchScore" }` for debugging |

*Sources: mongodb.com/docs/atlas/atlas-vector-search/ — Feb 2026*
