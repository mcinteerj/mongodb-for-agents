# MongoDB MCP Server Setup

The [official MongoDB MCP server](https://github.com/mongodb-js/mongodb-mcp-server) gives your agent direct access to MongoDB and Atlas.

## What It Provides

- CRUD operations (insert, find, update, delete, aggregate)
- Schema inspection and validation
- Index management (create, list, drop)
- Atlas admin (clusters, database users, network access)
- Explain plans and query profiling
- Log access and monitoring

## What It Does NOT Provide

Design guidance, best practices, search/vector search configuration — that's what the skills in this repo are for. The MCP server is the _hands_; the skills are the _brain_.

## Setup

Full configuration docs: [mongodb-mcp-server README](https://github.com/mongodb-js/mongodb-mcp-server#readme)

### Claude Code

```bash
claude mcp add mongodb-mcp-server -- npx -y mongodb-mcp-server
```

### Cursor

Add to `.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "mongodb": {
      "command": "npx",
      "args": ["-y", "mongodb-mcp-server"]
    }
  }
}
```

### VS Code Copilot

Add to `.vscode/mcp.json`:

```json
{
  "servers": {
    "mongodb": {
      "command": "npx",
      "args": ["-y", "mongodb-mcp-server"]
    }
  }
}
```

### Pi

Add to your pi MCP config (see [pi MCP docs](https://github.com/mariozechner/pi-coding-agent#mcp)):

```json
{
  "mongodb": {
    "command": "npx",
    "args": ["-y", "mongodb-mcp-server"]
  }
}
```

## Connection

Pass a connection string via environment variable or let the agent use the `connect` tool at runtime:

```json
{
  "command": "npx",
  "args": ["-y", "mongodb-mcp-server"],
  "env": {
    "MDB_MCP_CONNECTION_STRING": "mongodb+srv://..."
  }
}
```

For Atlas admin tools, set `MDB_MCP_API_CLIENT_ID` and `MDB_MCP_API_CLIENT_SECRET`.
