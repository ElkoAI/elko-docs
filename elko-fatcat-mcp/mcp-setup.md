# MCP Setup — elko-fatcat-mcp

Connect elko-fatcat-mcp to any MCP-compatible AI client.

---

## Claude Code (CLI)

```bash
# Generate config from running server
curl -s localhost:8083/mcp.json > .mcp.json

# Start Claude — picks up .mcp.json automatically
claude
```

Then ask:
> *"Use the elko-fatcat tools to show me which hedge funds increased their Apple stake last quarter and any congressional trades in AAPL since."*

---

## Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "elko-fatcat": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "-v", "elko-fatcat-data:/data",
        "jsoprych/elko-fatcat-mcp:latest",
        "mcp"
      ]
    }
  }
}
```

Or connect to a running HTTP server:
```json
{
  "mcpServers": {
    "elko-fatcat": {
      "url": "http://localhost:8083/mcp"
    }
  }
}
```

---

## Cursor

In Cursor settings → MCP → Add server:
```json
{
  "elko-fatcat": {
    "url": "http://localhost:8083/mcp"
  }
}
```

---

## Python (requests)

```python
import requests

BASE = "http://localhost:8083/mcp"

def call_tool(name, **args):
    r = requests.post(BASE, json={
        "jsonrpc": "2.0",
        "id": 1,
        "method": "tools/call",
        "params": {"name": name, "arguments": args}
    })
    return r.json()["result"]["content"][0]["text"]

# Examples
print(call_tool("fatcat_13f_ticker", ticker="NVDA", quarter="2025Q4"))
print(call_tool("fatcat_congress_trades", ticker="NVDA", months=12))
print(call_tool("fatcat_lobbying_top_spenders", group_by="client", year=2026, limit=10))
```

---

## Python (langchain-mcp-adapters)

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent

client = MultiServerMCPClient({
    "elko-fatcat": {
        "url": "http://localhost:8083/mcp",
        "transport": "streamable_http"
    }
})

tools = await client.get_tools()
agent = create_react_agent(ChatOpenAI(model="gpt-4o"), tools)
result = await agent.ainvoke({"messages": [{"role": "user", "content": "Show me Nancy Pelosi's recent trades and any lobbying activity around her sponsored bills."}]})
```

---

## curl (JSON-RPC 2.0)

```bash
# List all available tools
curl -s -X POST http://localhost:8083/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'

# Call a tool
curl -s -X POST http://localhost:8083/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"fatcat_13f_top_moves","arguments":{"action":"new","limit":10}}}'
```

---

## MCP Config File

The server self-describes its tools at `/mcp.json`:

```bash
curl -s localhost:8083/mcp.json
```

```json
{
  "mcpServers": {
    "elko-fatcat": {
      "url": "http://localhost:8083/mcp",
      "transport": "streamable-http"
    }
  }
}
```

---

## Pairing with elko-market-mcp

fatcat's smart-money + political data + market's live quotes + financials makes a powerful agent combination:

```json
{
  "mcpServers": {
    "elko-market": { "url": "http://localhost:8082/mcp" },
    "elko-fatcat": { "url": "http://localhost:8083/mcp" }
  }
}
```

Then ask: *"Show me NVDA's current price and PE, plus which hedge funds added to it last quarter, plus any congressional trades."*


---

*Built by [Elko.AI](https://elko.ai) & [DARKFABRIK.AI](https://darkfabrik.ai) · [dev@elko.ai](mailto:dev@elko.ai)*
