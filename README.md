# MongoDB for Agents

Ready-made context files that help AI coding agents work effectively with MongoDB. Drop them into your project and your agent gets schema design patterns, indexing strategy, query optimization, and more.

## Quick Setup

### Claude Code

Copy `CLAUDE.md` to your repo root, or symlink it:

```bash
cp CLAUDE.md /path/to/your/repo/
```

### Cursor

Copy `.cursorrules` to your repo root, or copy `skills/mongodb/` into `.cursor/rules/`:

```bash
cp .cursorrules /path/to/your/repo/
# or
cp -r skills/mongodb/ /path/to/your/repo/.cursor/rules/mongodb/
```

### Codex / OpenAI

Copy `AGENTS.md` to your repo root:

```bash
cp AGENTS.md /path/to/your/repo/
```

### Gemini

Copy `AGENTS.md` to your repo root and configure `context.fileName` in your Gemini settings to include it.

### Windsurf

Copy `.windsurfrules` to your repo root:

```bash
cp .windsurfrules /path/to/your/repo/
```

### Pi

Symlink the skill into your Pi skills directory:

```bash
ln -s /path/to/mongodb-for-agents/skills/mongodb ~/.pi/agent/skills/mongodb
```

### Any Agent

Point your agent at the markdown files in `skills/mongodb/`. The content is plain markdown — any agent that reads files can use it.

## What's Included

**Rules files** — `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, `.windsurfrules` — top-level context with key conventions and common mistakes.

**Skill** — `skills/mongodb/SKILL.md` — strategic overview with progressive disclosure into 8 reference files:

| Reference | Description |
|---|---|
| `schema-design.md` | Embed vs reference decisions, document modeling patterns |
| `indexing.md` | ESR rule, compound indexes, index selection strategy |
| `aggregation.md` | Pipeline ordering, stage patterns, optimization |
| `performance.md` | `explain()` interpretation, profiling, tuning |
| `security.md` | Authentication, authorization, encryption, auditing |
| `atlas-search.md` | Full-text search setup, analyzers, scoring |
| `vector-search.md` | Atlas Vector Search indexes, queries, hybrid search |
| `docs-navigation.md` | URL patterns for finding MongoDB docs efficiently |

## MCP Server

For direct database interaction, use the official [MongoDB MCP Server](https://github.com/mongodb-js/mongodb-mcp-server).

See [`mcp/README.md`](mcp/README.md) for setup instructions.

## Contributing

Issues and PRs welcome. If you find a pattern that helps agents work better with MongoDB, send it in.

## License

[Apache 2.0](LICENSE)
