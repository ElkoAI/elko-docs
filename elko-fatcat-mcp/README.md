# elko-fatcat-mcp

A self-hosted "fat cat monitor" MCP server that tracks where powerful entities put their money. Exposes **15 free tools** across SEC filings, congressional disclosures, federal contracts, and lobbying — via MCP, REST API, and a web dashboard.

🐳 **[Docker Hub → hub.docker.com/r/jsoprych/elko-fatcat-mcp](https://hub.docker.com/r/jsoprych/elko-fatcat-mcp)**

---

## What it tracks

| Source | Coverage |
|--------|----------|
| **SEC Form 13F** | Hedge fund & institutional holdings (~4,700 filers with $100M+ AUM) |
| **SEC Form 4** | Corporate insider trades (executives, directors, >10% holders) |
| **SEC Schedule 13D/G** | 5%+ activist & passive beneficial ownership filings |
| **SEC Form 144** | Insider notice-of-proposed-sale (leading indicator for Form 4) |
| **STOCK Act** | Congressional member trades (House + Senate PTRs) |
| **Congress.gov** | Bills sponsored by each member |
| **FEC** | Top campaign donors by employer per cycle |
| **USAspending.gov** | Federal contract awards |
| **Senate LDA** | Lobbying disclosures — who buys influence on what |

---

## Quick Start

```bash
docker pull jsoprych/elko-fatcat-mcp:latest
docker run -d --name elko-fatcat \
  -p 8083:8083 \
  -v elko-fatcat-data:/data \
  jsoprych/elko-fatcat-mcp:latest
```

Open **http://localhost:8083** — the dashboard loads immediately. Background ingest begins on first start; expect ~10 min before the first 13F snapshot is queryable.

---

## Tools

| Category | Tool | Description |
|----------|------|-------------|
| **Smart Money** | `fatcat_13f_holdings` | Holdings snapshot for a specific fund by CIK or name |
| **Smart Money** | `fatcat_13f_changes` | Quarter-over-quarter position diff (new/added/trimmed/exited) |
| **Smart Money** | `fatcat_13f_ticker` | Which funds hold a given ticker + position sizes |
| **Smart Money** | `fatcat_13f_top_moves` | Biggest position changes across all filers |
| **Insider** | `fatcat_insider_trades` | Form 4 trades by company ticker or insider name |
| **Insider** | `fatcat_insider_intents` | Form 144 notice-of-proposed-sale — lead indicator for Form 4 |
| **Ownership** | `fatcat_activist_filings` | Schedule 13D/G filings (5%+ stakes, activist or passive) |
| **Political** | `fatcat_congress_trades` | STOCK Act trades filtered by member, ticker, or party |
| **Political** | `fatcat_member_profile` | Full "follow the money" profile: trades + legislation + donors |
| **Political** | `fatcat_member_legislation` | Recent bills sponsored by a Congress member |
| **Political** | `fatcat_member_donors` | Top campaign donors by employer for a Congress member |
| **Influence** | `fatcat_lobbying_filings` | Senate LDA filings filtered by registrant, client, year, issue |
| **Influence** | `fatcat_lobbying_top_spenders` | Top lobbying spenders by client or registrant for a year/period |
| **Ops** | `fatcat_ingest_status` | Warehouse status: record counts + last ingest dates |
| **Ops** | `fatcat_parse_misses` | Filings we couldn't extract (image-only PDFs, parse errors) |

---

## How it's different from market-mcp

`elko-market-mcp` queries upstream APIs live (Yahoo, Treasury, FRED, BLS) — fast tools, no warehouse.

`elko-fatcat-mcp` is the opposite: a **local SQLite data warehouse** of ~20 tables, populated by a background ingest scheduler that pulls from SEC bulk ZIPs, congressional disclosure PDFs, and several public APIs. Tools query the warehouse, not upstream — giving you sub-second cross-source joins like "show me funds that bought TSLA the same quarter Pelosi did."

---

## Documentation

- [Getting Started](getting-started.md)
- [MCP Setup](mcp-setup.md)
- [Dashboard Guide](dashboard.md)
- [Ingest Pipeline](ingest.md)
- [REST API](api.md)
- [Docker & Configuration](docker.md)


---

*Built by [Elko.AI](https://elko.ai) & [DARKFABRIK.AI](https://darkfabrik.ai) · [dev@elko.ai](mailto:dev@elko.ai)*
