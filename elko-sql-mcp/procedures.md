# Procedures — SQL, Lua, JavaScript

Procedures are stored in the `_procedures` table and instantly become MCP tools,
REST endpoints (`POST /v1/call/{name}`), and CLI commands (`elko-sql call {name}`).

---

## SQL procedures

Read-only queries. Must start with `SELECT` or `WITH`.

```sql
INSERT INTO _procedures (name, description, type, sql, parameters, mutation)
VALUES (
  'search_products',
  'Search products by name or category.',
  'sql',
  'SELECT id, name, price, stock
   FROM products
   WHERE (:q IS NULL OR name LIKE ''%''||:q||''%'')
     AND (:category IS NULL OR category = :category)
   ORDER BY name LIMIT :limit',
  '[
    {"name":"q","type":"string","required":false,"description":"Search term"},
    {"name":"category","type":"string","required":false},
    {"name":"limit","type":"integer","required":false,"default":20}
  ]',
  0
);
```

---

## Lua procedures

Full business logic with DB access, observability, and event emission.

```sql
INSERT INTO _procedures (name, description, type, lua, parameters, mutation)
VALUES (
  'place_order',
  'Create order and reserve stock.',
  'lua',
  'function place_order(args, db)
     -- Validate
     if not args.customer_id then return {error="customer_id required"} end

     -- Check stock
     local product = db.query_one(
       "SELECT id, name, stock FROM products WHERE id = ?", args.product_id)
     if not product then return {error="product not found"} end
     if product.stock < args.qty then
       return {error="insufficient stock", available=product.stock}
     end

     -- Write
     db.exec("INSERT INTO orders (customer_id, product_id, qty) VALUES (?,?,?)",
       args.customer_id, args.product_id, args.qty)
     db.exec("UPDATE products SET stock = stock - ? WHERE id = ?",
       args.qty, args.product_id)

     -- Observe
     local row = db.query_one("SELECT last_insert_rowid() AS id")
     ELKO_EMIT("order.placed", {id=row.id, product=product.name, qty=args.qty})
     ELKO_LOG_INFO("order placed", {id=row.id, customer=args.customer_id})
     ELKO_METRIC("orders.placed", 1, {product=product.name})

     return {ok=true, order_id=row.id}
   end',
  '[
    {"name":"customer_id","type":"integer","required":true},
    {"name":"product_id","type":"integer","required":true},
    {"name":"qty","type":"integer","required":true,"default":1}
  ]',
  1
);
```

### Lua DB bindings

```lua
db.query("SELECT * FROM table WHERE col = ?", value)        -- returns array of row objects
db.query_one("SELECT * FROM table WHERE id = ?", id)        -- returns one row or nil
db.exec("INSERT INTO table (col) VALUES (?)", value)        -- returns rows_affected
```

---

## JavaScript procedures

ES2015+ sandbox — same DB bindings and UDFs as Lua, more familiar syntax.

```sql
INSERT INTO _procedures (name, description, type, js, parameters, mutation)
VALUES (
  'analyze_skill',
  'Look up a skill and record usage metrics.',
  'js',
  'function analyze_skill(args, db) {
     const skill = db.queryOne(
       "SELECT * FROM hermes_skills WHERE name = ? AND active = 1",
       args.name
     );
     if (!skill) return { error: `skill not found: ${args.name}` };

     // Destructure frontmatter
     const fm = JSON.parse(skill.frontmatter || "{}");
     const tags = JSON.parse(skill.tags || "[]");

     ELKO_METRIC("skill.analyze", 1, { name: args.name });
     ELKO_LOG_INFO("skill analyzed", { name: args.name, version: fm.version });
     ELKO_EMIT("hermes.skill.analyzed", { name: args.name, tags });

     return { name: skill.name, description: skill.description, tags, body: skill.body };
   }',
  '[{"name":"name","type":"string","required":true}]',
  0
);
```

### JS DB bindings

```js
db.query("SELECT * FROM table WHERE col = ?", value)        // returns array of row objects
db.queryOne("SELECT * FROM table WHERE id = ?", id)         // returns one row or null
db.exec("INSERT INTO table (col) VALUES (?)", value)        // returns rows_affected
```

---

## Parameter types

```json
[
  {"name":"id",     "type":"integer",  "required":true},
  {"name":"name",   "type":"string",   "required":true,  "description":"User's full name"},
  {"name":"active", "type":"boolean",  "required":false, "default":true},
  {"name":"score",  "type":"number",   "required":false, "minimum":0, "maximum":100},
  {"name":"role",   "type":"string",   "required":false, "enum":["admin","user","guest"]},
  {"name":"tags",   "type":"array",    "required":false},
  {"name":"meta",   "type":"object",   "required":false},
  {"name":"limit",  "type":"integer",  "required":false, "default":20}
]
```

---

## REST API

```bash
# Call via REST
curl -X POST http://localhost:8092/v1/call/search_products \
  -H "Authorization: Bearer $ELKO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"q":"laptop","limit":5}'

# CRUD
GET    /v1/procedures           # list all
POST   /v1/procedures           # create/update
GET    /v1/procedures/{name}    # get one
DELETE /v1/procedures/{name}    # delete
POST   /v1/procedures/{name}/run  # run with body args
```

---

## Built-in samples

```bash
elko-sql sample list             # show available samples
elko-sql sample todo             # task management (7 procedures)
elko-sql sample catalog          # product catalog (8 procedures)
elko-sql sample kv               # key-value store (5 procedures)
elko-sql sample users            # auth (6 procedures)
elko-sql sample hermes           # Hermes-agent backbone (9 procedures)
elko-sql sample openclaw         # OpenClaw backbone (9 procedures)
```
