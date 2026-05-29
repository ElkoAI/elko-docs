# Hermes + elko-sql-mcp — 5-minute quickstart

This guide gets you from zero to a running Hermes-agent backed by elko-sql-mcp.
Copy-paste every command. No decisions required.

**What you need:**
- Docker and Docker Compose installed
- Hermes-agent Docker image (`hermes-agent:latest` or your image name)

---

## Step 1 — Create your project folder

```bash
mkdir hermes-elko && cd hermes-elko
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
      - hermes-data:/data
    environment:
      ELKO_DB: /data/hermes-elko.db
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8092/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  hermes:
    image: hermes-agent:latest
    container_name: hermes
    restart: unless-stopped
    volumes:
      - hermes-data:/home/hermes/.hermes
    environment:
      ELKO_MCP_URL: http://elko-sql:8092/mcp
    depends_on:
      elko-sql:
        condition: service_healthy

volumes:
  hermes-data:
EOF
```

---

## Step 3 — Start elko

```bash
docker compose up elko-sql -d
```

Wait for it to be healthy (a few seconds), then grab the auto-generated API key:

```bash
docker exec elko-sql elko-sql keygen
```

Copy the key and save it:

```bash
echo "ELKO_API_KEY=esql_..." >> .env   # paste your actual key
```

---

## Step 4 — Apply the Hermes schema

```bash
docker exec elko-sql elko-sql sample hermes
```

You should see:
```
✓ sample applied: hermes      9 procedure(s) total in /data/hermes-elko.db
```

---

## Step 5 — Import your existing skills (if you have them)

```bash
# Import from ~/.hermes/skills/ (default)
docker exec elko-sql elko-sql import hermes \
  --hermes-dir /home/hermes/.hermes

# Or preview first
docker exec elko-sql elko-sql import hermes --dry-run
```

---

## Step 6 — Configure Hermes to use elko

Add to your Hermes config (`~/.hermes/config.yaml` or the container equivalent):

```yaml
mcp_servers:
  elko-sql:
    transport: http
    url: http://localhost:8092/mcp
    api_key: ${ELKO_API_KEY}
```

---

## Step 7 — Start Hermes

```bash
docker compose up hermes -d
```

---

## Step 8 — Verify it's working

```bash
# Check elko is healthy
curl http://localhost:8092/health

# See what tools Hermes now has available
curl -H "Authorization: Bearer $ELKO_API_KEY" \
     http://localhost:8092/v1/catalogue | jq '.count'

# Call a skill lookup directly
curl -X POST http://localhost:8092/v1/call/list_hermes_skills \
     -H "Authorization: Bearer $ELKO_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"limit":5}'
```

Open the live dashboard: **http://localhost:8092**  
Click **Observe** to see real-time logs and events as Hermes uses skills.

---

## What Hermes can now do via MCP

| Tool | What it does |
|------|-------------|
| `get_hermes_skill(name)` | Full skill body + metadata |
| `list_hermes_skills(limit?)` | All active skills, ordered by use |
| `search_hermes_skills(q)` | Search by name/description/tags |
| `use_hermes_skill(name)` | Mark used + emit observable event |
| `register_hermes_skill(name, body)` | Create a new skill (self-improvement) |
| `get_hermes_memory(category, key)` | Read a memory entry |
| `set_hermes_memory(category, key, value)` | Write a memory entry |
| `list_hermes_memories(category?)` | List all memories |
| `get_hermes_context()` | Full agent context (skill count, top skills, user name) |

---

## Keeping data persistent

All data lives in the `hermes-data` Docker volume. Both containers share it:
- Hermes native files (`~/.hermes/`) → `/home/hermes/.hermes` inside the Hermes container
- elko DB (`hermes-elko.db`) → `/data/` inside elko, same volume → `/home/hermes/.hermes/hermes-elko.db` from Hermes's perspective

**One backup command covers everything:**

```bash
docker run --rm -v hermes-data:/data -v $(pwd):/backup alpine \
  tar czf /backup/hermes-backup-$(date +%Y%m%d).tar.gz /data
```

---

## Troubleshooting

**elko won't start:**
```bash
docker compose logs elko-sql
```

**No skills after import:**
```bash
# Check what the import found
docker exec elko-sql elko-sql import hermes --dry-run
# Check the DB directly
docker exec elko-sql elko-sql call list_hermes_skills
```

**Hermes can't reach elko:**
```bash
# Verify they're on the same network
docker network inspect hermes-elko_default
# Test from inside Hermes container
docker exec hermes wget -qO- http://elko-sql:8092/health
```

---

→ [Full agent integration docs](agent-integration.md) · [Observability guide](observability.md) · [Pro tier — debugger](pro.md)
