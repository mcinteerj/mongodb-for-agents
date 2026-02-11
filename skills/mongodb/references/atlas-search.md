# Atlas Search Reference

## When to Use What

| Need | Use | Why |
|---|---|---|
| Relevance-ranked full-text, fuzzy, autocomplete, facets, synonyms | `$search` (Atlas Search) | Lucene engine, BM25 scoring, full analyzer chain |
| Simple keyword search (no Atlas) | `$text` + text index | Native MongoDB, basic stemming, `textScore` |
| Exact pattern matching, validation | `$regex` | Pattern match only, no scoring, prefix-indexed only |

**Rule**: If you have Atlas and need search UX → Atlas Search. Always.

## Index Definitions

### Static Mappings (preferred)

```json
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "title": { "type": "string", "analyzer": "lucene.standard" },
      "year": { "type": "number" },
      "genres": { "type": "token" },
      "plot": [
        { "type": "string", "analyzer": "lucene.english" },
        { "type": "autocomplete", "tokenization": "edgeGram", "minGrams": 3, "maxGrams": 15 }
      ]
    }
  }
}
```

### Dynamic Mappings (prototyping only)

```json
{ "mappings": { "dynamic": true } }
```

⚠️ Root-level dynamic indexes **every field** → bloated index. Scope to specific `document` fields in production.

### Multi Analyzer (same field, multiple analyzers)

```json
"title": {
  "type": "string", "analyzer": "lucene.standard",
  "multi": { "exact": { "type": "string", "analyzer": "lucene.keyword" } }
}
```

Query: `"path": "title"` (standard) or `"path": { "multi": "exact" }` (keyword).

### Nested Fields (no dot notation)

```json
"address": { "type": "document", "fields": { "city": { "type": "string" } } }
```

## Analyzer Selection

| Analyzer | Behavior | Use For |
|---|---|---|
| `lucene.standard` | Word boundaries + lowercase | Default general search |
| `lucene.english` | + stop words + stemming | English full-text ("running"→"run") |
| `lucene.keyword` | Entire string = 1 token | Exact match, IDs, enums, sorting |
| `lucene.whitespace` | Split on whitespace, case-sensitive | Code, technical tokens |
| `lucene.simple` | Split on non-letters + lowercase | Simple case-insensitive |

**Custom analyzer** = `charFilters` → `tokenizer` → `tokenFilters`. Define in index `analyzers` array, reference by name.

## Compound Operator

```javascript
db.collection.aggregate([{
  $search: {
    compound: {
      must:    [{ text: { query: "mongodb", path: "title" } }],
      mustNot: [{ text: { query: "deprecated", path: "status" } }],
      should:  [{ text: { query: "atlas", path: "title", score: { boost: { value: 2 } } } }],
      filter:  [{ range: { path: "year", gte: 2020 } }, { equals: { path: "published", value: true } }]
    }
  }
}])
```

| Clause | Must Match? | Affects Score? |
|---|---|---|
| `must` | ✅ | ✅ |
| `mustNot` | ❌ excludes | ❌ |
| `should` | Optional (unless no must/filter) | ✅ |
| `filter` | ✅ | ❌ |

**Key rule**: Equality/range/boolean → always `filter`, never `must`. Skips scoring = faster.

### OR logic inside filter

```javascript
filter: [{ compound: {
  should: [
    { equals: { path: "category", value: "A" } },
    { equals: { path: "category", value: "B" } }
  ],
  minimumShouldMatch: 1
}}]
```

## Scoring & Boosting

```javascript
// Static boost
{ text: { query: "premium", path: "tier", score: { boost: { value: 5 } } } }

// Boost by field value
{ text: { query: "term", path: "title", score: { boost: { path: "popularity", undefined: 1 } } } }

// Constant score
{ text: { query: "term", path: "field", score: { constant: { value: 10 } } } }

// Function score (relevance × field)
{ text: { query: "term", path: "title", score: {
  function: { multiply: [{ score: "relevance" }, { path: { value: "boost_field", undefined: 1 } }] }
}}}
```

Function expressions: `add`, `multiply`, `log1p`, `gauss`, `path`, `score`, `constant`.

## Autocomplete

### Index (dual-type recommended)

```json
"title": [
  { "type": "autocomplete", "tokenization": "edgeGram", "minGrams": 3, "maxGrams": 15, "foldDiacritics": true },
  { "type": "string" }
]
```

Tokenization: `edgeGram` (prefix, most common) | `nGram` (substring, heavier) | `rightEdgeGram` (suffix).

### Query

```javascript
db.collection.aggregate([
  { $search: { autocomplete: { query: "inter", path: "title", fuzzy: { maxEdits: 1 } } } },
  { $limit: 10 },
  { $project: { title: 1, score: { $meta: "searchScore" } } }
])
```

**Tip**: Use `compound` with `should` on both `autocomplete` and `string` typed fields to boost exact matches over partials.

## Facets (`$searchMeta`)

```javascript
db.movies.aggregate([{
  $searchMeta: {
    facet: {
      operator: { range: { path: "year", gte: 2000, lte: 2025 } },
      facets: {
        genresFacet: { type: "string", path: "genres", numBuckets: 10 },
        yearFacet:   { type: "number", path: "year", boundaries: [2000, 2010, 2020, 2025] }
      }
    }
  }
}])
```

String facets require `token` field type. Number facets require `number` type.

## Synonyms

Source collection (same DB):
- **Equivalent** (bidirectional): `{ "mappingType": "equivalent", "synonyms": ["car", "vehicle", "automobile"] }`
- **Explicit** (unidirectional): `{ "mappingType": "explicit", "input": ["beer"], "synonyms": ["beer", "brew", "pint"] }`

Index config: `"synonyms": [{ "name": "mySynonyms", "analyzer": "lucene.standard", "source": { "collection": "synonyms_col" } }]`

Query: `{ text: { query: "car", path: "desc", synonyms: "mySynonyms" } }` — works with `text` and `phrase` only. Changes to source collection picked up automatically (no reindex).

## Count (for pagination)

```javascript
db.collection.aggregate([{
  $searchMeta: {
    count: { type: "total" },  // or "lowerBound" for faster approximate
    text: { query: "term", path: "field" }
  }
}])
```

### `$$SEARCH_META` (metadata alongside results)

```javascript
db.collection.aggregate([
  { $search: { text: { query: "term", path: "field" } } },
  { $addFields: { meta: "$$SEARCH_META" } }
])
```

## Highlighting

Field must be `string` type with `indexOptions: "offsets"` (default).

```javascript
db.collection.aggregate([
  { $search: {
    text: { query: "variety bunch", path: "description" },
    highlight: { path: "description" }  // maxCharsToExamine, maxNumPassages available
  }},
  { $project: { description: 1, highlights: { $meta: "searchHighlights" } } }
])
```

Output: `texts` array with `{ value, type }` where type is `"hit"` or `"text"`. Can highlight fields not in query path.

## Atlas Search Index vs Regular Index

| | Atlas Search Index | Regular MongoDB Index |
|---|---|---|
| Engine | Lucene (`mongot` sidecar) | WiredTiger (same `mongod`) |
| Created via | `createSearchIndex()` / Atlas UI | `createIndex()` |
| Query syntax | `$search` / `$searchMeta` stage | `find()`, `$match`, etc. |
| Consistency | Eventually consistent (async to `mongot`) | Immediately consistent |
| Supports | Full-text, fuzzy, autocomplete, facets, synonyms, scoring | Exact match, range, sort, unique, TTL, geo |

**Completely separate systems.** One doesn't help the other. Need both if you use both query patterns.

## Common Mistakes

| Mistake | Fix |
|---|---|
| `lucene.keyword` for full-text search | Use `lucene.standard` or language analyzer |
| `lucene.standard` for IDs/enums | Use `lucene.keyword` or `token` type |
| Equality/range in `must` instead of `filter` | Move to `filter` — no scoring overhead |
| `$search` not first pipeline stage | Must be first. No exceptions |
| No `$limit` after `$search` | Always limit — returns all matches otherwise |
| Dynamic mappings at root in production | Use static or scope dynamic to `document` fields |
| Querying autocomplete field with `text` operator | Use `autocomplete` operator for autocomplete-typed fields |
| Expecting immediate consistency | Atlas Search is eventually consistent (async to `mongot`) |
| Confusing `$search` indexes with regular indexes | Completely separate systems. One doesn't help the other |
