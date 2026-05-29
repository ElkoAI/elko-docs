# OpenClaw + elko-sql-mcp — 5-minute quickstart

This guide gets you from zero to a running OpenClaw agent backed by elko-sql-mcp.
Copy-paste every command. No decisions required.

**What you need:**
- Docker and Docker Compose installed
- OpenClaw Docker image (`openclaw:latest` or your image name)

---

## Step 1 — Create your project folder

```bash
mkdir openclaw-elko && cd openclaw-elko
```

---

## Step 2 — Create the Docker Compose file

```bash
cat > docker-compose.yml << 'EOF'
services:
  elko-sql:
    image: ghcr.io/jsoprych/elko-sql-mcp:latest
    container_name: elko-sql
    restart: unless-stopped
    ports:
      - "8092:8092"
    volumes:
      - openclaw-data:/data
    environment:
      ELKO_DB: /data/openclaw-elko.db
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8092/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  openclaw:
    image: openclaw:latest
    container_name: openclaw
    restart: unless-stopped
    volumes:
      - openclaw-data:/home/openclaw/.openclaw
    environment:
      ELKO_MCP_URL: http://elko-sql:8092/mcp
    depends_on:
      elko-sql:
        condition: service_healthy

volumes:
  openclaw-data:
EOF
```

---

## Step 3 — Start elko

```bash
docker compose up elko-sql -d
```

Grab the auto-generated API key:

```bash
docker exec elko-sql elko-sql keygen
```

Save it:

```bash
echo "ELKO_API_KEY=esql_..." >> .env   # paste your actual key
```

---

## Step 4 — Apply the OpenClaw schema

```bash
docker exec elko-sql elko-sql sample openclaw
```

You should see:
```
✓ sample applied: openclaw     9 procedure(s) total in /data/openclaw-elko.db
```

---

## Step 5 — Import your existing skills (if you have them)

OpenClaw stores skills in multiple directories. elko imports them in precedence order —
workspace skills override user skills override agent skills.

```bash
# Default: scans workspace/skills/, ~/.openclaw/skills/, ~/.agents/skills/
docker exec elko-sql elko-sql import openclaw \
  --workspace /home/openclaw/.openclaw/workspace \
  --user-dir /home/openclaw/.openclaw \
  --agents-dir /home/openclaw/.agents

# Preview first (no writes)
docker exec elko-sql elko-sql import openclaw --dry-run
```

Expected output:
```
Scanning /home/openclaw/.openclaw/workspace/skills
  ✓  search_web                      Search the internet for current info
  ✓  summarize_document              Summarize long documents

Scanning /home/openclaw/.openclaw/skills
  ✓  remember_context                Store important context
  ↩  search_web                      shadowed by workspace

── openclaw import complete ──────────────────────────────
  imported : 3 skill(s) → openclaw_skills
  skipped  : 1 (shadowed by higher-precedence copy)
  MCP procedures registered:
    get_openclaw_skill
    list_openclaw_skills
    search_openclaw_skills
```

---

## Step 6 — Configure OpenClaw to use elko

Add to `~/.openclaw/openclaw.json` (or the container's config):

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

Or via `openclaw mcp set` if your version supports it:

```bash
docker exec openclaw openclaw mcp set elko-sql \
  --transport http \
  --url http://elko-sql:8092/mcp
```

---

## Step 7 — Start OpenClaw

```bash
docker compose up openclaw -d
```

---

## Step 8 — Verify it's working

```bash
# Health check
curl http://localhost:8092/health

# List available tools
curl -H "Authorization: Bearer $ELKO_API_KEY" \
     http://localhost:8092/v1/catalogue | jq '[.tools[].name]'

# Check workspace config
curl -X POST http://localhost:8092/v1/call/get_openclaw_config \
     -H "Authorization: Bearer $ELKO_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"key":"skills.precedence"}'

# Get full agent context
curl -X POST http://localhost:8092/v1/call/get_openclaw_context \
     -H "Authorization: Bearer $ELKO_API_KEY" \
     -d '{}'
```

Open the dashboard: **http://localhost:8092** → click **Observe** for live events.

---

## What OpenClaw can now do via MCP

| Tool | What it does |
|------|-------------|
| `get_openclaw_skill(name)` | Full skill body + precedence metadata |
| `list_openclaw_skills(limit?)` | All active skills, ordered by precedence then usage |
| `search_openclaw_skills(q)` | Search by name/description/tags |
| `use_openclaw_skill(name)` | Mark used + emit event + update counter |
| `register_openclaw_skill(name, body, source_dir?)` | Register new skill (workspace/user/agent/hub/plugin) |
| `get_openclaw_config(key)` | Read workspace configuration |
| `set_openclaw_config(key, value)` | Update workspace configuration |
| `get_openclaw_context()` | Context summary: skill counts by tier, top skills, workspace name |

---

## Understanding skill precedence

When you import skills from multiple directories, higher-precedence skills always win:

```
workspace (10)  →  your project-specific overrides
user      (50)  →  ~/.openclaw/skills/
agent     (70)  →  ~/.agents/skills/
hub       (80)  →  ClawHub installed skills
plugin    (90)  →  npm plugin skills
```

A skill in `workspace/skills/` will always shadow the same-named skill from `~/.openclaw/skills/`.
This mirrors OpenClaw's native precedence model exactly.

To register a workspace-level skill from the agent itself:
```
call register_openclaw_skill(
  name="my_skill",
  body="## Instructions\n...",
  source_dir="workspace"
)
```

---

## Keeping data persistent

All data (OpenClaw files + elko DB) lives in the `openclaw-data` volume:

```bash
docker run --rm -v openclaw-data:/data -v $(pwd):/backup alpine \
  tar czf /backup/openclaw-backup-$(date +%Y%m%d).tar.gz /data
```

---

## Troubleshooting

**Import finds no skills:**
```bash
# Check if skills directory exists inside the container
docker exec elko-sql ls /home/openclaw/.openclaw/skills/ 2>/dev/null || \
  echo "No skills directory found — volume may not be shared"

# Verify the volume is mounted in both containers
docker inspect elko-sql | jq '.[0].Mounts'
docker inspect openclaw | jq '.[0].Mounts'
```

**Skills not updating after changes:**
```bash
# Re-run import (it uses INSERT OR REPLACE — safe to run repeatedly)
docker exec elko-sql elko-sql import openclaw \
  --workspace /home/openclaw/.openclaw/workspace
```

**OpenClaw can't find elko tools:**
```bash
# Verify elko is reachable from OpenClaw container
docker exec openclaw wget -qO- http://elko-sql:8092/v1/catalogue
```

---

→ [Full agent integration docs](agent-integration.md) · [Observability guide](observability.md) · [Pro tier — debugger](pro.md)
