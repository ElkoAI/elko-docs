# Observability — logs, metrics, counters, events

Every Lua/JS procedure has built-in access to the full observability stack.
All writes persist to the database **and** fan out live to:
- The SSE stream at `GET /v1/stream`
- The dashboard's **Observe** tab at `http://localhost:8092`
- Any subscriber (elko-debug, TypeScript SDK, custom tool)

---

## The triad

### 1. Structured logs → `_logs`

```lua
-- Lua
ELKO_LOG_INFO("order placed", {id=row.id, customer=args.customer_id})
ELKO_LOG_WARN("stock low", {product=name, remaining=stock})
ELKO_LOG_ERROR("payment failed", {order_id=args.id, reason=err})
ELKO_LOG("message", "debug", {extra="data"})  -- any level
```
```js
// JavaScript
ELKO_LOG_INFO("order placed", { id: row.id, customer: args.customer_id });
ELKO_LOG_WARN("stock low", { product: name, remaining: stock });
ELKO_LOG_ERROR("payment failed", { order_id: args.id });
```

SSE event: `log.entry` → `{"level":"info","message":"...","data":"..."}`

### 2. Metrics → `_metrics`

```lua
ELKO_METRIC("orders.placed", 1)
ELKO_METRIC("order.value", args.total, {currency="USD"})
ELKO_METRIC("skill.access.ms", elapsed_ms, {name=args.skill})
```

SSE event: `metric.recorded` → `{"name":"...","value":N,"tags":"..."}`

### 3. Counters → `_counters`

Stateful, atomic, windowed.

```lua
local n = ELKO_COUNTER_INCREMENT("api.calls")         -- returns new value
ELKO_COUNTER_DECREMENT("items.in_stock")
local v = ELKO_COUNTER_GET("api.calls")
```

SSE event: `counter.changed` → `{"key":"...","value":N,"delta":N}`

### 4. Events → `_events`

```lua
ELKO_EMIT("order.placed", {id=row.id, total=args.total})
```

SSE event: `order.placed` → your payload

---

## Rate limiting

```lua
function call_api(args, db)
  -- Allow 100 calls per 60 seconds per user
  if not ELKO_RATE_CHECK("api:" .. args.user_id, 100, 60) then
    return {error="rate limit exceeded — try again in a minute"}
  end
  ELKO_RATE_INCREMENT("api:" .. args.user_id, 60)
  -- ... do the API call
end
```

---

## Debug helpers

Non-blocking — available in community tier.

### ELKO_TRACE — passthrough tracing

Logs the value AND returns it unchanged. Wrap any expression without restructuring.

```lua
-- Lua — logs "after_query" and returns the row unchanged
local row = ELKO_TRACE("after_query", db.query_one("SELECT * FROM users WHERE id=?", id))

-- JavaScript
const row = ELKO_TRACE("after_query", db.queryOne("SELECT * FROM users WHERE id=?", id));
```

### ELKO_ELAPSED — duration tracking

```lua
local t = ELKO_NOW_MS()
-- ... do work ...
local ms = ELKO_ELAPSED(t, "import_skills")  -- logs elapsed_ms + emits timer.elapsed
```

### ELKO_INSPECT — serialize any value

```lua
return {debug = ELKO_INSPECT(complex_object)}  -- returns JSON string, no side effects
```

---

## Subscribing to the stream

**Browser (EventSource):**
```js
const es = new EventSource('http://localhost:8092/v1/stream?api_key=' + API_KEY);

es.addEventListener('log.entry', e => {
  const { level, message, data } = JSON.parse(e.data);
  console.log(`[${level}] ${message}`, data);
});

es.addEventListener('metric.recorded', e => {
  const { name, value } = JSON.parse(e.data);
  updateChart(name, value);
});
```

**TypeScript SDK:**
```ts
import { ElkoClient } from '@elko/sql-mcp-sdk';
const elko = new ElkoClient({ baseURL: 'http://localhost:8092', apiKey: API_KEY });

elko.subscribe('log.entry', entry => console.log(entry.message));
elko.subscribe('metric.recorded', m => updateDashboard(m.name, m.value));
```

**curl:**
```bash
curl -H "Authorization: Bearer $ELKO_API_KEY" \
     http://localhost:8092/v1/stream
```

---

## Query historical data

All observability data is in regular SQLite tables — query via procedures or directly.

```sql
-- Recent errors
SELECT ts, message, data FROM _logs WHERE level = 'error' ORDER BY created_at DESC LIMIT 20;

-- Metric aggregation
SELECT name, SUM(value), COUNT(*) FROM _metrics
WHERE created_at > strftime('%s','now') - 3600
GROUP BY name ORDER BY SUM(value) DESC;

-- Counter state
SELECT key, value FROM _counters ORDER BY value DESC;

-- Event trail
SELECT name, payload, created_at FROM _events ORDER BY created_at DESC LIMIT 50;
```
