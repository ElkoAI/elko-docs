# elko-sql-mcp — SQL + Lua + JS as an MCP server

Port `8092` · 🐳 Docker Hub `ghcr.io/jsoprych/elko-sql-mcp` · CGO-free · Local-first

Turn any SQL schema into MCP tools your AI agent can call directly.
Write the procedures in SQL, Lua, or JavaScript. Set up triggers. Done.

```
Your schema.sql  +  Your procedures.sql  →  MCP tools  →  AI agent
```

---

## What makes it different

| Feature | elko-sql-mcp | Typical agent storage |
|---|---|---|
| Procedure types | SQL · Lua · JavaScript | — |
| Observability | `_logs` + `_metrics` + `_counters` + live SSE | None |
| Triggers | 7 types (interval, cron, event, webhook, watch, startup, shutdown) | None |
| Agent import | `elko-sql import hermes/openclaw` | Manual |
| Debugger | `ELKO_BREAKPOINT` → live web UI (Pro) | None |
| DB backend | SQLite (community) · PostgreSQL (Pro) | Varies |
| Transports | MCP stdio · HTTP streamable · REST · CLI | — |

---

## Quick links

**Agent quickstarts (5 minutes):**
- [⚡ Hermes quickstart](quickstart-hermes.md) — Docker Compose, import skills, done
- [⚡ OpenClaw quickstart](quickstart-openclaw.md) — Docker Compose, precedence-aware import, done

**Reference:**
- [Getting Started](getting-started.md)
- [Procedures — SQL, Lua, JavaScript](procedures.md)
- [Observability — logs, metrics, counters](observability.md)
- [Agent Integration — Hermes, OpenClaw](agent-integration.md)
- [Pro tier — debugger, PostgreSQL, HTTP UDFs](pro.md)

---

## 60-second start

```bash
docker run -d --name elko-sql -p 8092:8092 \
  -v elko-sql-data:/data \
  ghcr.io/jsoprych/elko-sql-mcp:latest

# Apply a sample schema
docker exec elko-sql elko-sql sample todo

# Connect Claude Desktop
# Add to ~/Library/Application Support/Claude/claude_desktop_config.json:
```

```json
{
  "mcpServers": {
    "my-db": { "transport": "http", "url": "http://localhost:8092/mcp" }
  }
}
```

Open the dashboard: `http://localhost:8092`
