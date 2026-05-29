# Getting Started — elko-sql-mcp

## Install

**Docker (recommended):**
```bash
docker run -d \
  --name elko-sql \
  -p 8092:8092 \
  -v elko-sql-data:/data \
  -e ELKO_API_KEY=your-key-here \
  ghcr.io/jsoprych/elko-sql-mcp:latest
```

**Binary:**
```bash
# Download from GitHub releases
./elko-sql serve --port 8092
```

On first `serve`, an API key is auto-generated and printed:
```
  ┌─ API key generated (first run) ─────────────────────���────┐
  │  ELKO_API_KEY=esql_a3f9b2c1d4e5...
  └───────────────────────────────────────────────────────────┘
```

Save it: `export ELKO_API_KEY=esql_...`

---

## Apply your first schema

```bash
# Option 1: Use a built-in sample
./elko-sql sample todo

# Option 2: Apply your own SQL
./elko-sql exec schema.sql
./elko-sql exec procedures.sql

# See what tools are registered
./elko-sql list
```

---

## Connect your agent

**Claude Desktop (`~/Library/Application Support/Claude/claude_desktop_config.json`):**
```json
{
  "mcpServers": {
    "my-database": {
      "transport": "http",
      "url": "http://localhost:8092/mcp"
    }
  }
}
```

**stdio (subprocess — no auth, no network):**
```json
{
  "mcpServers": {
    "my-database": {
      "command": "/path/to/elko-sql",
      "args": ["mcp"]
    }
  }
}
```

**Hermes-agent (`~/.hermes/config.yaml`):**
```yaml
mcp_servers:
  elko-sql:
    transport: http
    url: http://localhost:8092/mcp
    api_key: ${ELKO_API_KEY}
```

**OpenClaw (`~/.openclaw/openclaw.json`):**
```json
{
  "mcp": {
    "servers": {
      "elko-sql": {
        "transport": "http",
        "url": "http://localhost:8092/mcp"
      }
    }
  }
}
```

---

## Call a procedure from the CLI

```bash
./elko-sql call add_todo title="Buy milk" priority=high
./elko-sql call list_todos status=open
./elko-sql call find_product q=laptop limit=5
```

---

## Dashboard

Open `http://localhost:8092` — shows all registered procedures, lets you run them, and streams live events.

The **Observe** tab shows real-time logs, metrics, and counters as they're emitted by your procedures.

---

## Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `ELKO_DB` | auto | Explicit SQLite path |
| `ELKO_DATA_DIR` | `~/.local/share/elko-sql` | Data directory |
| `ELKO_API_KEY` | auto-generated | REST API bearer token |
| `ELKO_TIER` | `community` | `community` or `pro` |
| `DATABASE_URL` | — | PostgreSQL DSN (Pro only) |

---

## Next steps

- [Write SQL, Lua, and JS procedures →](procedures.md)
- [Set up triggers →](triggers.md)
- [Make your procedures observable →](observability.md)
- [Integrate with Hermes or OpenClaw →](agent-integration.md)
