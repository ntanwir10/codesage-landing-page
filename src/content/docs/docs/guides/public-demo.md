---
title: Public demo and MCP over HTTP
description: Try CodeSage read-only at codesage-demo.vercel.app, connect an MCP client to it over HTTP, and run a read-only server of your own.
---

## Try the demo

[codesage-demo.vercel.app](https://codesage-demo.vercel.app) is CodeSage in read-only mode, with three public repositories indexed: pmndrs/zustand, pallets/click and CodeSage itself.
- **What you can do.** In the dashboard, search the code, open files and list definitions.
- **What it refuses.** Nothing can be added, reprocessed or deleted.
- **Free plans.** It runs on Vercel's and Algolia's free plans. The data is a snapshot, and search pauses if the month's free Algolia quota runs out.

## Connect an MCP client over HTTP

The backend serves MCP itself at `/mcp`, over MCP's streamable HTTP transport. This is for clients that connect to a URL instead of starting the stdio bridge. It offers the same tools, resources and prompts as the bridge, with the same answers. The demo's endpoint needs no key:

```text
https://codesage-demo.vercel.app/mcp
```

These clients were checked against it:

| Client | Add it with | Check |
| --- | --- | --- |
| Claude Code | `claude mcp add --transport http codesage-demo https://codesage-demo.vercel.app/mcp` | `claude mcp get codesage-demo` shows Connected |
| Gemini CLI | `gemini mcp add --transport http codesage-demo https://codesage-demo.vercel.app/mcp` | `gemini mcp list` shows Connected |
| Qwen Code | `qwen mcp add --transport http codesage-demo https://codesage-demo.vercel.app/mcp` | `qwen mcp list` shows Connected |
| Codex | `codex mcp add codesage-demo --url https://codesage-demo.vercel.app/mcp` | Codex accepts it as a streamable HTTP server, but its `mcp list` doesn't test the connection. Back up `~/.codex/config.toml` first, since Codex rewrites it |

On a read-only server, `tools/list` has seven tools: `add_repository` is left out. Every tool is marked read-only (`readOnlyHint`), so a client can run them without asking each time.

### On your own server

Every CodeSage backend has `/mcp`:

- **Key.** With `CODESAGE_API_KEY` set, send it as `Authorization: Bearer <key>` or as `X-API-Key`. Without it, the answer is `401`.
- **Behind the prod nginx.** Sign in with basic auth (`Authorization: Basic ...`), and nginx adds the key.
- **Limit.** `RATE_LIMIT_AI` (30 per minute) per client address. Every request counts, `initialize` and `tools/list` included.
- **Host.** The `Host` header must be allowed, as for the rest of the API: loopback, a `CORS_ORIGINS` host or an `ALLOWED_HOSTS` entry.

```bash
curl -X POST https://your-host/mcp -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" -H "Authorization: Bearer $CODESAGE_API_KEY" \
  -d '{"jsonrpc": "2.0", "id": 1, "method": "tools/list"}'
```

## Run a read-only server

`READ_ONLY=true` serves searches and reads to anyone on an allowed host, with no key.
- **Writes.** Every write is refused with `403`: adding, reprocessing, editing, deleting and the GitHub webhook. `add_repository` is neither listed nor run.
- **Database.** It must already be at the latest migration. It is never migrated or written to, so a read-only SQLite file works.
- **Dashboard.** It swaps the add form for a notice and hides Reprocess and Delete.

The demo is built this way:
- `make demo-build` indexes the repositories into a SQLite file and their own Algolia index.
- `make demo-deploy` publishes it to Vercel. The deployment holds only a search key restricted to that index.

For the setup, see [deployment/vercel/README.md](https://github.com/ntanwir10/codesage-algolia-challenge/blob/master/deployment/vercel/README.md) in the repository.
