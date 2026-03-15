# Local Setup

## Prerequisites

- Node.js v18 or higher
- npm

Check your version:

```bash
node --version   # must be v18+
npm --version
```

---

## Install

```bash
git clone https://github.com/goblin-C/mcp-learning.git
cd mcp-learning
npm install
```

---

## Environment Variables

Create a `.env` file in the project root:

```env
MCP_API_KEY=your-secret-key-here
```

Leave `MCP_API_KEY` empty or omit the `.env` file entirely to disable auth locally.

Make sure `.env` is in your `.gitignore`:

```
.env
```

---

## Run

```bash
# Auto-restarts on file changes (recommended for development)
npm run dev

# Standard start
npm start
```

The server starts on port `8000` by default:

```
MCP mcp-learning running on port 8000
```

---

## Verify It's Running

```bash
curl http://localhost:8000/health
# → ok
```

---

## Test with curl

See [docs/api.md](api.md) for the full curl reference including the MCP initialization sequence.

Quick test:

```bash
# Initialize
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -D - \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'
```

Copy the `mcp-session-id` from the response headers, then:

```bash
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: PASTE_SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
```

---

## Connect Cursor Locally

In Cursor settings → MCP → edit `mcp.json`:

```json
{
  "mcpServers": {
    "mcp-learning": {
      "url": "http://localhost:8000/mcp"
    }
  }
}
```

No `x-api-key` needed when `MCP_API_KEY` is not set locally.

---

## Project Scripts

| Script | Command | Description |
|---|---|---|
| `start` | `node server.js` | Production start |
| `dev` | `node --watch server.js` | Dev with auto-restart |