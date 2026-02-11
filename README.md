# mongodb-for-agents
Helping AI agents work effectively with MongoDB.

## Agent Configuration

| Agent | Config |
|-------|--------|
| Claude (Code/Desktop) | Auto-reads `AGENTS.md` / `CLAUDE.md` |
| Cursor | Auto-reads `.cursorrules` |
| Windsurf | Auto-reads `.windsurfrules` |
| Gemini | Add to settings: `context.fileName: ['AGENTS.md']` |

All symlink to `AGENTS.md` — single source of truth.
