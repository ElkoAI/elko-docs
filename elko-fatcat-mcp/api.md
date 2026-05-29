# REST API — elko-fatcat-mcp

Every MCP tool is also a REST endpoint. The server exposes both MCP JSON-RPC 2.0 over HTTP and stdio plus a Chi-router REST surface.

---

## Public Routes (no auth)

| Endpoint | Returns |
|----------|---------|
| `GET /health` | `{"status":"ok"}` |
| `GET /mcp.json` | MCP client config for this server |
| `GET /openapi.json` | OpenAPI 3.0 spec generated from registered tools |
| `GET /v1/catalogue` | Every registered tool with name, description, schema, source, category |
| `GET /v1/key-status` | Easy-key + pro-key validity + remaining quota |

---

## Tool Invocation

```bash
# Generic tool call via REST
curl -s -X POST http://localhost:8083/v1/call/fatcat_13f_holdings \
  -H "Content-Type: application/json" \
  -d '{"fund_name":"Berkshire","limit":20}'

# Or via MCP JSON-RPC 2.0
curl -s -X POST http://localhost:8083/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"fatcat_13f_holdings","arguments":{"fund_name":"Berkshire","limit":20}}}'
```

Both return the same markdown-table response body.

---

## Protected Routes (auth required)

| Endpoint | Purpose |
|----------|---------|
| `POST /v1/call/<tool>` | Invoke any registered tool by name |
| `GET /v1/sources` | Per-channel upstream metadata (authority, rate limit, freshness SLA) |
| `GET /v1/logs` | Tail of `call_log` — MCP tool call history |
| `POST /v1/ingest/backfill` | Trigger historical backfill — see [Ingest Pipeline](ingest.md) |
| `GET /v1/ingest/status` | Live per-source ingest progress snapshot |
| `GET /v1/upstream-status` | Per-source reachability probe results |
| `GET /v1/procedures` | List saved user-defined SQL tools |
| `POST /v1/procedures` | Save a new procedure (auto-registers as live MCP tool) |
| `DELETE /v1/procedures/<name>` | Remove a procedure |

---

## Authentication

Two-tier key model, same as elko-market-mcp + elko-news-mcp:

- **Easy key** (`ELKO_EASY_KEY` env var): 91-day JWT with RPM/RPD limits, baked at container build time. Validated locally — no network calls.
- **Pro key** (`ELKO_KEY_SECRET` + optional `ELKO_REVOCATION_URL`): HMAC-SHA256 signed via elko-keyring.

Pass the key as a Bearer token:
```bash
curl -s -H "Authorization: Bearer YOUR_KEY" \
  -X POST http://localhost:8083/v1/call/fatcat_13f_holdings \
  -d '{"fund_name":"Berkshire"}'
```

---

## OpenAPI

`/openapi.json` is generated dynamically from registered tools' JSON Schemas. Drop it into Postman, Swagger UI, or any OpenAPI client to get typed bindings.

```bash
curl -s http://localhost:8083/openapi.json | jq '.paths | keys'
```

---

## Response Format

All tool responses are markdown tables wrapped in JSON-RPC content blocks:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      { "type": "text", "text": "## Berkshire Hathaway — 13F Holdings\n..." }
    ],
    "isError": false
  }
}
```

Error conditions return `isError: true` with the error message in the same content block — not JSON-RPC error objects. Matches market-mcp + news-mcp convention.


---

*Built by [Elko.AI](https://elko.ai) & [DARKFABRIK.AI](https://darkfabrik.ai) · [dev@elko.ai](mailto:dev@elko.ai)*
