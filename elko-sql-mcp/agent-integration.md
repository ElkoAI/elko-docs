# Agent Integration — Hermes, OpenClaw, and others

elko-sql-mcp is designed to replace (or massively simplify) the storage, tools, and skills layer for MCP-compatible AI agents. Level 1 integration preserves all your existing data exactly as-is while making it queryable, auditable, and trigger-driven.

---

## What Level 1 integration gives you

| Before | After |
|--------|-------|
| Skills as Markdown files on disk | Skills in SQLite — queryable, searchable, versioned |
| MEMORY.md / SOUL.md / USER.md files | `hermes_memories` table with category/key/value |
| No audit trail | Every access logged to `_logs`, emitted to SSE |
| No metrics | `ELKO_METRIC` + `ELKO_COUNTER_INCREMENT` per skill use |
| Agent has to read files | Agent calls MCP tools: `get_hermes_skill`, `search_hermes_skills` |
| No triggers | Startup context injection, daily audit cron |

---

## Hermes-agent

### 1. Apply the sample schema

```bash
elko-sql sample hermes
```

This creates `hermes_skills` + `hermes_memories` tables and registers 9 MCP procedures including a startup trigger that auto-injects agent context.

### 2. Import existing skills

```bash
# Imports from ~/.hermes/skills/ automatically
elko-sql import hermes

# Preview without writing
elko-sql import hermes --dry-run

# Custom path
elko-sql import hermes --hermes-dir /path/to/hermes
```

After import, three procedures are immediately available as MCP tools:
- `get_hermes_skill(name)` — full Markdown body + metadata
- `list_hermes_skills(limit?)` — ordered by use count
- `search_hermes_skills(q, limit?)` — search name/description/tags

### 3. Configure Hermes to use elko

Add to `~/.hermes/config.yaml`:
```yaml
mcp_servers:
  elko-sql:
    transport: http
    url: http://localhost:8092/mcp
    api_key: ${ELKO_API_KEY}
```

### 4. Self-improvement loop

Hermes can register new skills directly via `register_hermes_skill`:
```
Agent: I need a skill for summarizing long documents.
→ calls register_hermes_skill(name="summarize_doc", body="...", tags=["text","ai"])
→ skill instantly appears in elko's _procedures as an MCP tool
→ ELKO_EMIT("hermes.skill.registered") fans out to dashboard + SSE
```

### Docker Compose

```bash
docker compose -f deploy/docker-compose.hermes.yml up -d
docker exec elko-sql elko-sql sample hermes
docker exec elko-sql elko-sql import hermes
```

---

## OpenClaw

OpenClaw stores skills in multiple directories with a precedence system:
`workspace > user > agent > hub > plugin`

### 1. Apply the sample schema

```bash
elko-sql sample openclaw
```

Creates `openclaw_skills` + `openclaw_config` tables with precedence-aware procedures.

### 2. Import existing skills

```bash
# Scans workspace/skills/, ~/.openclaw/skills/, ~/.agents/skills/ in order
elko-sql import openclaw

# Skills shadowed by higher-precedence copies are skipped
elko-sql import openclaw --workspace ./my-project --dry-run
```

### 3. Configure OpenClaw

Add to `~/.openclaw/openclaw.json`:
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

### Docker Compose

```bash
docker compose -f deploy/docker-compose.openclaw.yml up -d
docker exec elko-sql elko-sql sample openclaw
docker exec elko-sql elko-sql import openclaw \
  --workspace /data/workspace \
  --user-dir /data/.openclaw
```

---

## Other agents

Any MCP-compatible agent can use elko-sql-mcp with zero code changes — just add it to the MCP server config. For agents that aren't MCP-native, use the REST API directly or the TypeScript SDK.

### Pattern for any agent

```bash
# 1. Start elko
./elko-sql serve

# 2. Apply your schema
./elko-sql exec your_schema.sql

# 3. Register procedures as MCP tools
./elko-sql exec your_procedures.sql

# 4. Point your agent at http://localhost:8092/mcp
```

### TypeScript SDK (Node.js / Bun / Browser)

```ts
import { ElkoClient } from '@elko/sql-mcp-sdk';

const elko = new ElkoClient({
  baseURL: 'http://localhost:8092',
  apiKey: process.env.ELKO_API_KEY,
});

// Your agent uses elko as its memory backend
const skill = await elko.call('get_hermes_skill', { name: 'search_web' });
await elko.call('set_hermes_memory', {
  category: 'user', key: 'preferred_language', value: 'TypeScript'
});
```

---

## Shared volume pattern (Docker)

Both the agent and elko mount the **same named volume** at different paths.
The elko DB lives alongside the agent's native data files — one backup covers everything.

```yaml
services:
  elko-sql:
    volumes:
      - agent-data:/data           # elko sees DB as /data/agent-elko.db

  your-agent:
    volumes:
      - agent-data:/home/agent     # agent sees full data dir incl. elko DB

volumes:
  agent-data:
```
