# AGENTS.md — MongoDB for Agents

MongoDB is a document database that stores data as flexible JSON-like documents, enabling rich data models aligned to application objects.

## Official MCP Server

Use the MongoDB MCP server for direct database interaction:
- **Repo**: [github.com/mongodb-js/mongodb-mcp-server](https://github.com/mongodb-js/mongodb-mcp-server)

## Key Conventions

- Think in **documents, not tables**. Embed related data; don't default to joins.
- Design schemas for **access patterns first**. Ask "how will this be queried?" before modeling.
- Run `explain()` on **every non-trivial query**. Check `winningPlan`, `totalDocsExamined`, `executionTimeMillis`.
- Use `$lookup` sparingly — it's a red flag that the schema may need restructuring.
- Prefer **aggregation pipelines** over client-side data manipulation.
- Use **bulk operations** (`bulkWrite`, `insertMany`) for batch work.
- Always set **read/write concern** appropriate to the use case.

## Deep Methodology

See `skills/mongodb/SKILL.md` for full schema design patterns, indexing strategy, and query optimization workflow.

## Common Mistakes to Avoid

- **Over-normalization**: splitting data into many collections like a relational DB. Embed first, reference when necessary.
- **Unbounded arrays**: arrays that grow without limit cause document bloat and performance degradation. Cap or bucket them.
- **Missing indexes**: every query pattern needs a supporting index. No exceptions.
- **Ignoring explain() output**: don't assume queries are fast — prove it.
- **Schema-less thinking**: "schemaless" doesn't mean "no schema". Use validation (`$jsonSchema`) and consistent document shapes.
- **Large documents**: keep documents under 1MB (hard limit: 16MB). Restructure if approaching limits.
