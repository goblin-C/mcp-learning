# Architecture

## Overview

The server is a plain Node.js HTTP server with no web framework. It uses the `@modelcontextprotocol/sdk` to handle the MCP protocol over the Streamable HTTP transport.

```
Client (Claude.ai / Cursor)
  │
  └── POST /mcp
        │
        ├── Auth check (x-api-key header)
        ├── Session lookup or new session created
        ├── StreamableHTTPServerTransport.handleRequest()
        └── McpServer → tools / resources / prompts
                          │
                          └── src/store.js (shared in-memory tasks[])
```

---

## Transport: Streamable HTTP

MCP supports two transports:

| Transport | Endpoint | Status |
|---|---|---|
| **Streamable HTTP** | `POST /mcp` | Current — used by Claude.ai and Cursor |
| SSE (deprecated) | `GET /sse` + `POST /messages` | Older clients only |

This project uses Streamable HTTP. All MCP traffic — initialization, tool calls, resource reads, and responses — goes through a single `POST /mcp` endpoint. The SDK handles JSON-RPC framing, session management, and SSE streaming internally.

---

## Session Management

Each client connection gets its own `McpServer` and `StreamableHTTPServerTransport` instance. Sessions are stored in a `Map` keyed by `sessionId`.

```
First request (no mcp-session-id header)
  → new transport created
  → sessionIdGenerator fires, stores session in Map
  → initialize handshake completes
  → client receives mcp-session-id in response headers

Subsequent requests (mcp-session-id header present)
  → session looked up in Map
  → existing transport reused
```

**Why a fresh instance per session:** `McpServer.connect()` can only be called once per instance. A single shared instance rejects reconnects with `Server already initialized`.

---

## The Three MCP Primitives

### Tools — `src/tools/tasks.js`

Actions Claude can trigger. Each tool has a name, a plain-English description (how Claude decides which tool to call), a Zod schema for input validation, and an async handler.

```
server.tool(name, description, schema, handler)
```

Tools registered:

| Tool | Description |
|---|---|
| `addTask` | Add a new task with title and optional priority |
| `listTasks` | List all tasks, optionally filtered by done status |
| `updateTask` | Update title, priority, or done status by ID |
| `deleteTask` | Delete a task by ID |

### Resources — `src/resources/tasks.js`

Read-only data endpoints addressed by URI. Claude fetches these when it needs to read data without triggering an action. Clients can also subscribe to resources for live updates.

```
server.resource(uri, description, handler)
```

Resources registered:

| URI | Description |
|---|---|
| `tasks://all` | All tasks as JSON |
| `tasks://pending` | Tasks where `done: false` |

### Prompts — `src/prompts/tasks.js`

Reusable message templates that inject live data at render time. The client triggers a prompt by name — the server builds the message with current task data and hands it to Claude.

```
server.prompt(name, description, args, handler)
```

Prompts registered:

| Prompt | Description |
|---|---|
| `daily-standup` | Standup summary from all current tasks |
| `prioritize` | Suggested order for pending tasks |

---

## Shared State — `src/store.js`

All tools and resources import from a single `store.js`. Node.js caches modules after the first `import`, so every file that imports `tasks` gets the same array reference — not a copy. A push in `addTask` is immediately visible in `listTasks` and the resources.

**Limitation:** state resets on every server restart. To persist tasks across restarts, replace the array in `store.js` with database calls (MongoDB, SQLite, Postgres). Nothing else in the codebase needs to change.

---

## Auth

A middleware check runs before every `/mcp` request:

```js
if (API_KEY && req.headers["x-api-key"] !== API_KEY) {
  res.writeHead(401).end();
}
```

If `MCP_API_KEY` is not set in the environment, the check is skipped — safe for local development.