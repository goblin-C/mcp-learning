# mcp-learning

A sample project for learning how to build a [Model Context Protocol (MCP)](https://modelcontextprotocol.io) server from scratch using Node.js.

This server exposes a task management system (add, list, update, delete tasks) to AI clients like Claude.ai and Cursor via the Streamable HTTP transport.

---

## What is MCP?

MCP (Model Context Protocol) is an open standard that lets AI clients (Claude, Cursor, etc.) connect to external servers to call tools, read resources, and use prompt templates — all in a structured, secure way.

This project demonstrates all three MCP primitives:

| Primitive | What it does | Example |
|---|---|---|
| **Tools** | Actions Claude can trigger | `addTask`, `deleteTask` |
| **Resources** | Read-only data Claude can fetch | `tasks://all`, `tasks://pending` |
| **Prompts** | Reusable templates with live data | `daily-standup`, `prioritize` |

---

## Project Structure

```
├── src/
│   ├── tools/
│   │   └── tasks.js        # Tool definitions (add, list, update, delete)
│   ├── resources/
│   │   └── tasks.js        # Resource definitions (all tasks, pending tasks)
│   ├── prompts/
│   │   └── tasks.js        # Prompt templates (standup, prioritize)
│   └── store.js            # Shared in-memory task state
├── server.js               # HTTP server + MCP transport
├── docs/                   # Full documentation
├── .env                    # Local environment variables (never commit)
├── package.json
└── README.md
```

---

## Prerequisites

- Node.js v18 or higher (uses native `crypto`, ESM)
- npm

---

## Installation

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

| Variable | Required | Description |
|---|---|---|
| `MCP_API_KEY` | Optional | API key for `x-api-key` header auth. If not set, auth is disabled (safe for local dev). |

---

## Running Locally

```bash
# Development (auto-restarts on file changes)
npm run dev

# Production
npm start
```

Server starts on `http://localhost:8000`

---

## Connecting to Claude.ai

Go to **claude.ai → Settings → Integrations** and add:

```
https://your-deployed-url.onrender.com/mcp
```

---

## Connecting to Cursor

Add to your Cursor `mcp.json` (Settings → MCP):

```json
{
  "mcpServers": {
    "mcp-learning": {
      "url": "https://your-deployed-url.onrender.com/mcp",
      "headers": {
        "x-api-key": "your-secret-key-here"
      }
    }
  }
}
```

---

## Documentation

| File | Description |
|---|---|
| [docs/architecture.md](docs/architecture.md) | How the project is structured and how MCP works |
| [docs/setup.md](docs/setup.md) | Local development setup in detail |
| [docs/deployment.md](docs/deployment.md) | Deploying to Render |
| [docs/api.md](docs/api.md) | Testing with curl, MCP protocol reference |

---
