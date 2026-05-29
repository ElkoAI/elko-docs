# Ingest Pipeline — elko-fatcat-mcp

elko-fatcat-mcp is warehouse-first: every tool queries a local SQLite database (`/data/elko-fatcat.db`). The background scheduler keeps that warehouse fresh by pulling from upstream sources on per-channel cadences. This page documents what runs when and how to backfill historical ranges on-demand.

---

## Channels

Each upstream is a self-contained channel under `internal/ingest/channels/<name>/` in the source repo. All eleven channels register themselves at startup; the scheduler iterates them and respects per-host rate limits.

| Channel | Source | Lag | Cadence | Notes |
|---------|--------|-----|---------|-------|
| `edgar13f` | SEC bulk ZIP | 45d after Q-end | Daily (skips if current Q loaded) | ~80MB ZIP, 3-4k filers, ~2-10 min |
| `form4` | EDGAR `form.idx` + per-filing XML | 2 business days | Daily current Q + on-demand bulk | 4-worker pool, ~10 req/sec EDGAR limit |
| `edgar13dg` | EDGAR `form.idx` (Schedule 13D / 13G / 13D-A / 13G-A) | ~10 days | Daily current Q | Modern (≥ 2024) filings parse fully; pre-2024 skipped cleanly |
| `form144` | EDGAR `form.idx` (Form 144 only) | Days–weeks before Form 4 | Daily current Q + on-demand bulk | ~50k/quarter; ~1.4 hr per quarter at rate ceiling |
| `congress_house` | disclosures-clerk.house.gov yearly ZIP | 45 days | Annual ZIP (current+prior year) | PDF parse via `pdftotext -raw`; `parse_quality` tagged |
| `senate` | efdsearch.senate.gov CSRF session | 45 days | Weekly (3-month rolling); hourly retry on 503 | Dual-source: efdsearch primary + jeremiak GitHub mirror fallback |
| `members` | unitedstates.io legislators-current.json | Real-time | Weekly | 535 members + party enrichment |
| `legislation` | api.congress.gov | ~1 day | Weekly | Requires `CONGRESS_API_KEY`; limit 20 bills/member |
| `donors` | api.open.fec.gov | Per FEC filing | Weekly | Uses `FEC_API_KEY` or DEMO_KEY; top 20 employers/cycle |
| `usaspending` | usaspending.gov | Per award posting | Weekly | All-agency federal contract feed |
| `senatelda` | lda.senate.gov/api/v1 | Per filing | Weekly | ~28k filings/year, no auth, self-paced at 2 req/sec |

---

## Manual Ingest

Trigger any channel from inside the container:

```bash
docker exec elko-fatcat /elko-fatcat ingest 13f --db /data/elko-fatcat.db --year 2025 --quarter 4
docker exec elko-fatcat /elko-fatcat ingest form4 --db /data/elko-fatcat.db --since 2025-10-01
docker exec elko-fatcat /elko-fatcat ingest congress --db /data/elko-fatcat.db
docker exec elko-fatcat /elko-fatcat ingest senate --db /data/elko-fatcat.db --months 3
docker exec elko-fatcat /elko-fatcat ingest members --db /data/elko-fatcat.db
docker exec elko-fatcat /elko-fatcat ingest legislation --db /data/elko-fatcat.db
docker exec elko-fatcat /elko-fatcat ingest donors --db /data/elko-fatcat.db
docker exec elko-fatcat /elko-fatcat ingest usaspending --db /data/elko-fatcat.db
docker exec elko-fatcat /elko-fatcat ingest senatelda --db /data/elko-fatcat.db --year 2026

# Run every channel
docker exec elko-fatcat /elko-fatcat ingest all --db /data/elko-fatcat.db
```

Run with `--audit` to emit per-row HMAC-chained `audit_log_entries` (requires `ELKO_AUDIT_HMAC_SECRET`; ~20-30% slower).

---

## Backfill API

For historical bulk loads, POST a JSON spec to `/v1/ingest/backfill`. Each source runs as its own goroutine — independent tables, parallel by default.

```bash
curl -X POST http://localhost:8083/v1/ingest/backfill \
  -H "Content-Type: application/json" \
  -d '{
    "edgar13f":    { "year_from": 2020, "year_to": 2025 },
    "form4":       { "year_from": 2024, "year_to": 2025, "bulk": true },
    "form144":     { "year_from": 2024, "year_to": 2025 },
    "edgar13dg":   { "year_from": 2024, "year_to": 2025 },
    "congress_house": { "year_from": 2020, "year_to": 2025 }
  }'
```

Watch progress live:
```bash
curl -s http://localhost:8083/v1/ingest/status | jq
```

Returns a per-source snapshot with `started_at`, `current_phase`, `rows_processed`, `errors`, and `eta_seconds`.

---

## Source Health

`./elko-fatcat upstream-check` probes every upstream API with a short timeout and reports OK / DEGRADED / DOWN. Useful in CI or as a status-page input:

```bash
docker exec elko-fatcat /elko-fatcat upstream-check --db /data/elko-fatcat.db --json
```

The local-DB equivalent `./elko-fatcat check` runs canary queries against every MCP tool and classifies as OK / WARN / FAIL / SKIP:

```bash
docker exec elko-fatcat /elko-fatcat check --db /data/elko-fatcat.db
```

WARN means the handler works but the table is empty — run the relevant ingest.

---

## Integrity Trail (Optional)

Set `ELKO_AUDIT_HMAC_SECRET` to enable per-row HMAC-chained `audit_log_entries`. Each entry's HMAC chains off the previous entry's HMAC — any modification, insertion, or deletion breaks the chain. Verify with:

```bash
docker exec elko-fatcat /elko-fatcat audit-log verify --db /data/elko-fatcat.db
```

Every cached network fetch records `source_artifacts` (SHA-256 + bytes + URL + timestamp) regardless of audit setting — provenance for every byte that lands in the warehouse.


---

*Built by [Elko.AI](https://elko.ai) & [DARKFABRIK.AI](https://darkfabrik.ai) · [dev@elko.ai](mailto:dev@elko.ai)*
