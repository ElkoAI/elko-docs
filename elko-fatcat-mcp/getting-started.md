# Getting Started — elko-fatcat-mcp

## 1. Run the Container

```bash
docker run -d --name elko-fatcat \
  -p 8083:8083 \
  -v elko-fatcat-data:/data \
  jsoprych/elko-fatcat-mcp:latest
```

Open **http://localhost:8083** — the dashboard is ready. The background ingest scheduler starts automatically.

---

## 2. Wait for First Ingest

Unlike live-API tools, fatcat queries a local warehouse populated by background ingest. First-run timing:

| Source | First snapshot available | Notes |
|--------|--------------------------|-------|
| 13F holdings | ~10 min | Most recent quarterly bulk ZIP (~80MB, 3-4k filers) |
| Form 4 insider trades | ~5 min | Current quarter's recent filings |
| Congress (House) | ~2 min | Current year ZIP |
| Congress (Senate) | ~5 min | Requires CSRF session establishment |
| Members | ~30 sec | legislators-current.json |
| Form 144 / 13D/G | ~10 min | Current quarter bulk index |

Track progress in the **Ingest Status** dashboard pane or via:
```bash
docker exec elko-fatcat /elko-fatcat call fatcat_ingest_status --db /data/elko-fatcat.db
```

---

## 3. Try a Tool

In the dashboard: select `fatcat_13f_holdings` from the sidebar, enter `Berkshire` for fund_name, click **Run**.

Or via curl:
```bash
curl -s -X POST http://localhost:8083/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"fatcat_13f_holdings","arguments":{"fund_name":"Berkshire","limit":20}}}'
```

---

## 4. Connect Your AI Assistant

**Generate MCP config:**
```bash
curl -s localhost:8083/mcp.json > .mcp.json
```

**Claude Code** — run from your project directory:
```bash
curl -s localhost:8083/mcp.json > .mcp.json
claude  # picks up .mcp.json automatically
```

**Claude Desktop** — add to `claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "elko-fatcat": {
      "command": "docker",
      "args": ["run","--rm","-i","-v","elko-fatcat-data:/data","jsoprych/elko-fatcat-mcp:latest","mcp"]
    }
  }
}
```

**Cursor** — same JSON in MCP settings.

→ [Full MCP setup guide](mcp-setup.md)

---

## 5. Ask Your AI Assistant

Once connected, you can ask in plain English:

> *"Which hedge funds bought the most NVIDIA stock last quarter?"*
> *"Show me Nancy Pelosi's recent stock trades and what bills she sponsored."*
> *"Were there any large insider sales at Tesla in the last 90 days?"*
> *"Who are the top lobbying clients of Akin Gump in 2026?"*
> *"List Schedule 13D filings (activist stakes) against retail companies this year."*
> *"Show me Form 144 notices that haven't yet been followed by Form 4 transactions — possible upcoming sales."*

The agent can join across sources — e.g. "which companies in Pelosi's portfolio also lobbied her committee on tax policy?"

---

## 6. Optional API Keys

Most tools work without keys. Two upgrades enable richer political data:

| Key | Tools / Feature | Required? | Where to get |
|-----|-----------------|-----------|-------------|
| `CONGRESS_API_KEY` | `fatcat_member_legislation`, full member-profile bills | **Yes** for legislation | [api.congress.gov](https://api.congress.gov/sign-up/) — free |
| `FEC_API_KEY` | `fatcat_member_donors`, higher rate limits | No (DEMO_KEY default) | [api.open.fec.gov](https://api.open.fec.gov/developers/) — free |
| `ELKO_AUDIT_HMAC_SECRET` | HMAC-chained `audit_log_entries` | No | Any random secret string |
| `POSTMARK_SERVER_TOKEN` | Weekly digest email | No | [postmarkapp.com](https://postmarkapp.com) account |

**Pass at container startup:**
```bash
docker run -d --name elko-fatcat \
  -p 8083:8083 \
  -v elko-fatcat-data:/data \
  -e CONGRESS_API_KEY=your_key \
  -e FEC_API_KEY=your_fec_key \
  jsoprych/elko-fatcat-mcp:latest
```

→ [Full Docker configuration](docker.md)


---

*Built by [Elko.AI](https://elko.ai) & [DARKFABRIK.AI](https://darkfabrik.ai) · [dev@elko.ai](mailto:dev@elko.ai)*
