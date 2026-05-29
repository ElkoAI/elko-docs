# Dashboard Guide — elko-fatcat-mcp

Open **http://localhost:8083** after starting the container.

---

## Panes

| Pane | What it shows |
|------|---------------|
| **Tools** | All 15 MCP tools with auto-generated forms from each tool's JSON Schema. Pick a tool, fill in parameters, click **Run** — the markdown-table result renders inline with CSV / JSON / clipboard export. |
| **Ingest Status** | Per-source row counts + most recent ingest date. The same data `fatcat_ingest_status` returns. Refreshes every 30 seconds. |
| **Channels** | Per-channel metadata: upstream authority, rate limit, freshness SLA, schema version, parser version. Powered by the foundry `Channel.Source()` contract. |
| **Procedures** | Master/detail panel for user-defined SQL tools. Save / Run / Delete. Procedures auto-register as live MCP tools after save. |
| **Backfill** | Trigger historical ingest for House, 13F, Form 4, members, Senate via JSON spec. Live progress per source. |
| **Logs** | Tail of `call_log` (MCP tool calls) and `ingest_log` (background scheduler runs). |
| **Keys** | Manage `ELKO_KEY_SECRET`, `CONGRESS_API_KEY`, `FEC_API_KEY`, etc. without a container restart. |

---

## Theme

Toggle button (☀️ / 🌙 / 🔆) cycles **light → dark → high-contrast**. Setting persists in localStorage.

---

## Result Export

Every tool result with a markdown table gets a toolbar above it:
- 📋 **Copy** — clipboard as TSV (paste straight into Sheets/Excel)
- 💾 **CSV** — download as `<tool>-<timestamp>.csv`
- 📄 **JSON** — download as JSON array of row objects

---

## Quick Actions

The dashboard reads `ui` hints from each tool spec. Tools with `"auto_run": true` (like `fatcat_ingest_status` and `fatcat_parse_misses`) run on selection — no Run click required.


---

*Built by [Elko.AI](https://elko.ai) & [DARKFABRIK.AI](https://darkfabrik.ai) · [dev@elko.ai](mailto:dev@elko.ai)*
