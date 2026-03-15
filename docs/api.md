# API Reference

## Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check — returns `ok` |
| `POST` | `/mcp` | All MCP traffic (initialize, tool calls, resource reads) |

---

## Headers

Every `/mcp` request must include:

| Header | Value | Required |
|---|---|---|
| `Content-Type` | `application/json` | Yes |
| `Accept` | `application/json, text/event-stream` | Yes |
| `x-api-key` | your API key | Yes (if `MCP_API_KEY` is set) |
| `mcp-session-id` | session ID from initialize | Yes (all requests after initialize) |

---

## MCP Protocol Sequence

MCP requires an initialization handshake before any tool calls. Every new client must follow this sequence:

```
1. POST /mcp  { method: "initialize" }   → receive mcp-session-id
2. POST /mcp  { method: "tools/list" }   → verify tools are available
3. POST /mcp  { method: "tools/call" }   → call a tool
```

AI clients (Claude.ai, Cursor) handle this automatically. When testing manually with curl, you must do it yourself.

---

## curl Examples

Replace `BASE_URL` with `http://localhost:8000` for local or `https://your-app.onrender.com` for production. Replace `API_KEY` with your `MCP_API_KEY` value (omit the `x-api-key` header entirely if auth is disabled).

### Step 1 — Initialize

```bash
curl -X POST BASE_URL/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "x-api-key: API_KEY" \
  -D - \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": { "name": "test", "version": "1.0" }
    }
  }'
```

Copy the `mcp-session-id` value from the response headers. Use it in all subsequent requests.

---

### Step 2 — List Tools

```bash
curl -X POST BASE_URL/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "x-api-key: API_KEY" \
  -H "mcp-session-id: SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
```

---

### Add a Task

```bash
curl -X POST BASE_URL/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "x-api-key: API_KEY" \
  -H "mcp-session-id: SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "addTask",
      "arguments": { "title": "My task", "priority": "high" }
    }
  }'
```

### List All Tasks

```bash
curl -X POST BASE_URL/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "x-api-key: API_KEY" \
  -H "mcp-session-id: SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "id": 4,
    "method": "tools/call",
    "params": {
      "name": "listTasks",
      "arguments": {}
    }
  }'
```

### List Pending Tasks Only

```bash
  -d '{
    "jsonrpc": "2.0",
    "id": 5,
    "method": "tools/call",
    "params": {
      "name": "listTasks",
      "arguments": { "done": false }
    }
  }'
```

### Update a Task

```bash
  -d '{
    "jsonrpc": "2.0",
    "id": 6,
    "method": "tools/call",
    "params": {
      "name": "updateTask",
      "arguments": { "id": "TASK_ID", "done": true }
    }
  }'
```

### Delete a Task

```bash
  -d '{
    "jsonrpc": "2.0",
    "id": 7,
    "method": "tools/call",
    "params": {
      "name": "deleteTask",
      "arguments": { "id": "TASK_ID" }
    }
  }'
```

---

## Tool Schemas

### addTask

| Argument | Type | Required | Values |
|---|---|---|---|
| `title` | string | Yes | any string |
| `priority` | string | No | `low`, `medium`, `high` (default: `medium`) |

### listTasks

| Argument | Type | Required | Description |
|---|---|---|---|
| `done` | boolean | No | Filter by completion status |

### updateTask

| Argument | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Task ID |
| `title` | string | No | New title |
| `priority` | string | No | `low`, `medium`, `high` |
| `done` | boolean | No | Completion status |

### deleteTask

| Argument | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Task ID to delete |