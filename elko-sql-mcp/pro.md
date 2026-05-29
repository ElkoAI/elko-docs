# Pro tier — debugger, PostgreSQL, HTTP UDFs

Enable Pro features with `ELKO_TIER=pro`.

---

## Step-through debugger

`ELKO_BREAKPOINT()` pauses a procedure and waits for the debug UI to continue it.
Community tier: `ELKO_BREAKPOINT` is a registered no-op — zero overhead.

### Usage in Lua/JS

```lua
function add_todo(args, db)
  local row = db.query_one("SELECT * FROM users WHERE id = ?", args.user_id)
  ELKO_BREAKPOINT("user_loaded", {row=row, args=args})  -- pauses here
  -- ... rest of procedure
end
```

```js
function process_skill(args, db) {
  const skill = db.queryOne("SELECT * FROM skills WHERE name = ?", args.name);
  ELKO_BREAKPOINT("skill_loaded", { skill, args });  // pauses here
  return skill;
}
```

### Start the debug session

```bash
# Terminal 1 — run elko with Pro debugging
ELKO_TIER=pro ./elko-sql serve

# Terminal 2 — start the debugger UI (separate binary)
./elko-debug --connect http://localhost:8092 --api-key $ELKO_API_KEY
# Open http://localhost:9229
```

The debugger UI shows:
- **Sessions panel** — all paused breakpoints with procedure name and label
- **Source panel** — full procedure source with breakpoint location highlighted
- **Variables panel** — the `vars` object passed to ELKO_BREAKPOINT
- **Continue button** — resumes the paused procedure goroutine
- **Event stream** — live tail of all debug events

### WebSocket protocol

For custom tooling, connect to `ws://localhost:8092/ws/debug` and send:
```json
{"type": "continue", "session_id": "abc123"}
{"type": "ping"}
```

---

## PostgreSQL backend

```bash
ELKO_TIER=pro DATABASE_URL=postgres://user:pass@host/dbname ./elko-sql serve
```

The PostgreSQL driver implements the same interface as SQLite — all procedures,
triggers, and UDFs work unchanged. Named params (`:name`) are automatically
rewritten to `$1, $2, ...` Postgres syntax.

---

## HTTP UDFs

Available in Lua and JavaScript procedures when `ELKO_TIER=pro`.

```lua
-- GET request
local resp = ELKO_HTTP_GET("https://api.example.com/data", '{"Authorization":"Bearer token"}')
if resp.ok then
  local data = ELKO_JSON_DECODE(resp.body)
  -- use data...
end

-- POST request
local resp = ELKO_HTTP_POST(
  "https://api.example.com/submit",
  ELKO_JSON_ENCODE({key="value"}),
  '{"Content-Type":"application/json"}'
)
```

```js
// JavaScript
const resp = await ELKO_HTTP_GET("https://api.example.com/data");
if (resp.ok) {
  const data = JSON.parse(resp.body);
}

const resp = await ELKO_HTTP_POST(
  "https://api.example.com/submit",
  JSON.stringify({ key: "value" }),
  JSON.stringify({ "Content-Type": "application/json" })
);
```

All HTTP UDFs return:
```json
{"status": 200, "body": "...", "headers": {"Content-Type": "..."}, "ok": true}
```

Limits: 30-second timeout, 1MB response body.

Available: `ELKO_HTTP_GET`, `ELKO_HTTP_POST`, `ELKO_HTTP_PUT`, `ELKO_HTTP_DELETE`
