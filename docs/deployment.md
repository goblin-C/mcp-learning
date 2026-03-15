# Deployment — Render

This project is deployed as a **Web Service** on [Render](https://render.com). Render runs a persistent Node.js process (unlike serverless platforms), which is required for the MCP Streamable HTTP transport to work correctly.

---

## Serverless Platform Cavets

Serverless platforms spin up a fresh process per request and kill it after a short timeout. MCP sessions are stateful — the server must stay alive between the `initialize` request and subsequent tool calls. Render's web service keeps one persistent process running, which is what MCP needs.

---

## Prerequisites

- A [Render](https://render.com) account (free tier is sufficient)
- The project pushed to a GitHub repository

---

## Step 1 — Push to GitHub

Make sure your repo is up to date:

```bash
git add .
git commit -m "initial commit"
git push origin main
```

Ensure `.env` is in `.gitignore` — never push your API key to GitHub.

---

## Step 2 — Create a Web Service on Render

1. Go to [render.com](https://render.com) and sign in
2. Click **New → Web Service**
3. Connect your GitHub account and select the `mcp-learning` repository
4. Configure the service:

| Setting | Value |
|---|---|
| **Name** | `mcp-learning` (or any name) |
| **Region** | Any (pick closest to you) |
| **Branch** | `main` |
| **Runtime** | Node |
| **Build Command** | `npm install` |
| **Start Command** | `node server.js` |
| **Instance Type** | Free |

5. Click **Create Web Service**

---

## Step 3 — Set Environment Variables

In your Render service dashboard go to **Environment** and add:

| Key | Value |
|---|---|
| `MCP_API_KEY` | your-secret-key-here |

> Pick a strong random value for `MCP_API_KEY`. This is the key clients must pass in the `x-api-key` header to connect.

Render automatically sets `PORT` — your server reads it via `process.env.PORT`. Do not hardcode a port.

---

## Step 4 — Deploy

Render automatically deploys on every push to `main`. For a manual deploy:

1. Go to your service dashboard
2. Click **Manual Deploy → Deploy latest commit**

Watch the build logs — a successful deploy ends with:

```
==> Your service is live 🎉
==> Available at https://your-app.onrender.com
```

---

## Step 5 — Verify

```bash
curl https://your-app.onrender.com/health
# → ok
```

---

## Step 6 — Connect Clients

### Claude.ai

Go to **Settings → Integrations** and add:

```
https://your-app.onrender.com/mcp
```

If Claude.ai supports custom headers in your plan, add:

```
x-api-key: your-secret-key-here
```

### Cursor

Edit `mcp.json` (Cursor Settings → MCP):

```json
{
  "mcpServers": {
    "mcp-learning": {
      "url": "https://your-app.onrender.com/mcp",
      "headers": {
        "x-api-key": "your-secret-key-here"
      }
    }
  }
}
```

Restart Cursor after saving. You should see the server connect and list 4 tools, 2 prompts, and 2 resources in the MCP panel.

---

## Free Tier Behaviour

Render's free tier spins down a service after **15 minutes of inactivity**. The next request triggers a cold start that takes ~30 seconds.

**Fix — keep the service warm with UptimeRobot (free):**

1. Sign up at [uptimerobot.com](https://uptimerobot.com)
2. Click **Add New Monitor**
3. Type: **HTTP(s)**
4. URL: `https://your-app.onrender.com/health`
5. Monitoring interval: **5 minutes**

This keeps your server warm at no cost.

---

## Redeploying After Changes

Every `git push` to `main` triggers an automatic redeploy. Render runs `npm install` then `node server.js` on the new code.

If a deploy fails, check the build logs in the Render dashboard. Common causes:

| Error | Fix |
|---|---|
| `npm` with no arguments | Set build command to `npm install` not `npm` |
| Port binding error | Make sure you use `process.env.PORT`, not a hardcoded port |
| Module not found | Run `npm install` locally and commit `package-lock.json` |

---

## Common Issues

### `Server already initialized`
Happens when the same transport instance is reused across reconnects. Each client session must get a fresh `McpServer` and `StreamableHTTPServerTransport` instance. The session is stored in a `Map` keyed by `sessionId`, which is generated inside `sessionIdGenerator` — the only moment the SDK assigns it.

### `Bad Request: Server not initialized`
Happens when a tool call arrives before the `initialize` handshake completes. AI clients handle this automatically. When testing with curl, always send `initialize` first and pass the returned `mcp-session-id` in all subsequent requests.

### `Parse error: Invalid JSON`
The SDK reads the request body as a raw stream. Any middleware that consumes the stream before `handleRequest()` (e.g. `express.json()`) leaves it unreadable. This project uses a plain Node.js `http` server with no middleware to avoid this entirely.

### `Cannot POST /sse`
Cursor probes `POST /sse` first (Streamable HTTP protocol check) before falling back to SSE. This is expected if your server only exposes `/mcp`. Cursor falls back and connects via SSE automatically — or connects directly to `/mcp` if configured correctly in `mcp.json`.