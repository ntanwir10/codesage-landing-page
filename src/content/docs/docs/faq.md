---
title: FAQ
description: Short, accurate answers to the questions people ask most about running and using CodeSage.
---

## The basics

### Is there a hosted version?

No. CodeSage is self-hosted. You run the backend and dashboard yourself, with your own Algolia account. There is no hosted demo and no CodeSage service to sign up for. This documentation site is static pages only.

### What does it cost?

CodeSage itself has no fee in these sources. What you pay for is on your side: your Algolia plan (CodeSage writes one record per code entity and sends search requests to your index), your AI assistant's plan, and your own machine or server. Check Algolia's current plan limits before you index a very large repository.

### Do I need Claude Desktop?

No. The dashboard works on its own for adding, watching and searching repositories. Claude Desktop (or another MCP client) is how you get the eight tools. See [Introduction](/docs/introduction/).

### Do I need Docker?

No. `make setup` and `make start` run everything locally with Python and Node. Docker is for the dev and prod Compose stacks. See [Docker](/docs/guides/docker/).

### Does CodeSage run my code?

No. It reads files as text, parses them into entities and stores them. It never executes, builds or tests the code it indexes.

### Which operating systems work?

The sources describe macOS and Linux paths for everything. The installer also knows each client's Windows config path, but the launcher is a bash script that runs `.venv/bin/python`, and the `--basic-auth` flow needs the macOS Keychain. Treat Windows as untested.

## Requirements and credentials

### Does it work without Algolia?

Only in part. The backend starts, `/health` reports `degraded`, and the tools that read the database (`list_repositories`, `get_file`, `list_definitions`, `analyze_repository`, `find_patterns`) work on repositories that were indexed earlier. But you can't index anything new: adding or reprocessing a repository returns `503` with `Search indexing is not configured on the server (Algolia credentials are missing).`, and `search_code` returns `Search is not configured on the server (ALGOLIA_APP_ID / ALGOLIA_ADMIN_API_KEY are missing).` Empty values and `your-...-here` placeholders count as unset. See [Installation](/docs/installation/#get-algolia-credentials).

### Why does the backend need the Admin API key?

It creates the index settings, saves records, deletes records by filter, lists indexes for the health check and runs every search. A search-only key can do none of the writes. Keep the Admin key in `backend/.env` only.

### Does it work without a GitHub token?

Yes, for public repositories. Without `GITHUB_TOKEN`, GitHub's low unauthenticated rate limit applies, so a big repository can hit a rate-limit message, and the backend logs `GitHub token not found. Rate limits will be lower.` A token also lets you read private github.com repositories.

### Can I index private repositories?

Yes. On github.com the server's `GITHUB_TOKEN` must be able to read the repository. On GitLab, Bitbucket, Gitea, Forgejo and other hosts, put a read-only token for the host in `GIT_HOST_CREDENTIALS`, such as `gitlab.com=oauth2:<token>`. The [Other git hosts](/docs/guides/other-git-hosts/#private-repositories) guide has the format for each host. CodeSage has one shared API key and no per-user permissions, so anyone who can use your instance can search every repository the tokens reached.

### Can I use a search-only or restricted Algolia key?

The project sources only describe the Admin key. Nothing in them says a narrower key works.

## What it indexes

### Which languages are supported?

Entities are extracted for Python, JavaScript, TypeScript, Java, Go, Rust, C, C++, C#, PHP and Ruby (functions, classes, methods, imports), Markdown (one `doc_section` per `#` heading) and Terraform or HCL (one `terraform_block` per top-level block). CSS, SCSS, HTML, SQL, JSON, YAML and XML files are stored and readable but produce no entities. See [Languages](/docs/reference/languages/).

### Why are some of my files missing?

The file must have a known extension. Only `.c` and `.cpp` count for C and C++, so `.h`, `.hpp` and `.cc` files aren't indexed. Files in `node_modules`, `dist`, `build`, `vendor`, `third_party`, `.git`, `__pycache__`, `.next` and `coverage` are skipped, as are minified files, source maps, lockfiles, symlinks, empty or binary files, and files of 1 MB or more.

### How big a repository can I index?

At most 1000 files and 100 MB per repository, with each file under 1 MB. Source files are kept first, then tests, then Markdown, then files that yield no entities, each group in path order. A body stored in Algolia is capped at 2000 characters per entity. For a GitHub repository too large for one tree call, the backend lists directories through the contents API, up to 300 directories. Anything beyond the limits is left out and the repository is marked partially indexed.

### Why does my repository say "Partially indexed"?

The run finished and the repository is searchable, but something was left out. The `error_message` names each cause, joined by semicolons:

- `N of M files could not be stored` (a database batch failed);
- `N files could not be parsed` (stored without entities);
- `N files over the 1000-file or 100 MB limit were skipped`;
- `N definitions have no semantic search vectors (the embedding API failed)`;
- `the repository is too large to list in full, so some directories were not indexed`;
- `search still holds old records for some changed or removed files until they change again or a full reprocess` (an incremental run could not delete them);
- `search still holds the previous run's records until the next run` (the run could not delete them).

The dashboard shows an amber card on the repository page. See [Web dashboard](/docs/guides/web-dashboard/#status-badge-and-card-colors).

### Can I index two branches of the same repository?

No. The stored URL has no branch and must be unique, so a second add of the same repository, even with another branch, is refused as already added. Delete the repository and add it again with the other branch, or change the branch of the existing one (`PUT /api/v1/repositories/<id>` with a new `branch`, refused with `409` while a run is active). A completed or failed repository then re-indexes from the new branch by itself.

### How do I refresh an index after the repository changes?

Reprocess it, or set `REFRESH_INTERVAL_MINUTES` so the backend checks each completed repository's branch on a schedule and runs an incremental reindex when it moved. For GitHub, a [push webhook](/docs/reference/rest-api/) (`GITHUB_WEBHOOK_SECRET`) starts that reindex as soon as you push. Use **Reprocess** on the repository page, or:

```bash
curl -X POST "http://127.0.0.1:8001/api/v1/repositories/1/process?force_reprocess=true" \
  -H "X-API-Key: $CODESAGE_API_KEY"
```

A reprocess builds the new index next to the old one, which stays in place until the new run completes. Only then are the old records deleted. Meanwhile a search across all repositories still finds the old records, but a search or tool call that names the repository is refused as still indexing, and the dashboard's search box is disabled. If the run fails, the old index stays and the repository stays `completed`.

## Privacy and data

### Does CodeSage send my code anywhere?

Your code touches these places, and the backend code contacts nothing else:

- **GitHub.** The backend downloads files from `api.github.com` and `raw.githubusercontent.com`, with your token if set. Other hosts are cloned with `git`.
- **Algolia.** One record per entity is sent to your index. A record holds `objectID`, `repository_id`, `run_id`, `file_path`, `entity_type`, `entity_name`, `description`, `parameters`, `return_type`, `line_start`, `line_end`, `language`, `signature`, `content`, `rank_weight`, `is_test`, `parameter_count` and `lines_of_code`. `content` is the signature plus at most 2000 characters of the entity body, and each record is trimmed to 9000 bytes. Whole files are not sent to Algolia.
- **Your embeddings provider, only if `EMBEDDING_API_KEY` is set.** For each definition the backend sends its kind, name, path and lines to `EMBEDDING_API_URL` (OpenAI by default), and each semantic query too.
- **Your database.** It holds the full text of every indexed file: by default the SQLite file `backend/codesage.db`, or the Postgres database in `DATABASE_URL`.
- **Your AI assistant.** What a tool returns, such as snippets or numbered file lines, becomes part of your conversation with the assistant.

The backend makes outbound calls only to GitHub, Algolia, git remotes and, with a key set, your embeddings provider. It has no analytics or telemetry calls.

### Where is my data stored?

Full file contents and entities are in the database: `backend/codesage.db` by default, or Postgres when `DATABASE_URL` points there. Search records are in your Algolia index (`ALGOLIA_INDEX_NAME`, default `codesage_code_entities`). Your keys are in `backend/.env` and `frontend/.env`, both mode 600.

### Is the API key in the dashboard a secret?

Not in the local setup. Vite inlines `VITE_CODESAGE_API_KEY` into the JavaScript, so anyone who can load the dashboard can read it. That's fine on localhost for one user. The prod Docker stack keeps the key in nginx behind a login instead. See [Security](/docs/concepts/security/).

## Using it with an MCP client

### Which MCP clients work?

The bridge is a standard stdio MCP server, so any client that can launch a command with environment variables can use it. Claude Desktop, Cursor, VS Code and Windsurf have installers (`make install-claude`, `install-cursor`, `install-vscode`, `install-windsurf`); only Claude Desktop has been tested end to end. For any other client, point it at `mcp_launcher.sh` and set `MCP_SERVER_URL` and `CODESAGE_API_KEY`. You can test the bridge with `npx @modelcontextprotocol/inspector ./mcp_launcher.sh`. See [MCP clients](/docs/guides/claude-desktop/#check-the-bridge-with-the-mcp-inspector).

### Why does my assistant say a repository is "still indexing" or "failed to index"?

Every tool except `list_repositories` and `add_repository` refuses a repository that isn't `completed`. A `pending` or `processing` one says `is still indexing (n/m files)`. A failed one gives the reason and says to reprocess it from the dashboard.

### Why does search return results that don't contain all my words?

If no result matches every word, Algolia drops the words that miss, ignores stop words and plurals, and returns the best partial matches. That is what lets a plain question work as a query.

### Why does `find_patterns` stop early?

A regex search gives each file 1 second and the whole call 10 seconds, and skips files over 1,000,000 characters. A call that runs out of time returns `truncated: true` with the skipped files listed. Literal search has no time limit.

### I hit a rate limit. What are the limits?

Per client address: 30 per minute for MCP tool calls and resource reads, and 100 per minute for repository create, process, freshness and search suggestions. Dashboard searches count toward the 30. The error is `429` with a `Retry-After` header.

## Running it

### Can I run it on a server?

Yes, with the prod Docker stack: HTTPS on port 443, basic auth in nginx and the API key held by nginx. The backend refuses to start on a non-loopback `MCP_SERVER_HOST` without `CODESAGE_API_KEY`. Add any extra hostnames to `ALLOWED_HOSTS` or they get `403`. The stack was last run by hand; CI builds the images but never starts them. See [Production deployment](/docs/guides/production/).

### Can the API run as more than one process?

Yes, with Redis and external workers. Rate-limit counters live in each API process's memory, and so do job state and the two-runs-at-once limit in the default inline mode. So `API_WORKERS` above 1 needs `INDEX_WORKERS=external` (jobs go to the `jobs` table, which `python -m app.worker` processes take them from) and `RATE_LIMIT_STORAGE_URL=redis://…`. The backend refuses to start without them. The prod Compose `scale` profile adds Redis and the workers. See [More than one API process](/docs/guides/production/#more-than-one-api-process).

### What happens if the backend restarts during indexing?

At startup every repository left `pending` or `processing` is started again as a full run, so you don't need to do anything. Its previous index stays in place until that run completes. With external workers, a job whose worker stops sending heartbeats for 5 minutes is queued again for another worker.

### Why do I get `403 Host not allowed`?

With an API key set, the backend answers only to loopback hosts, the hosts in `CORS_ORIGINS` and the hosts in `ALLOWED_HOSTS`. Browser writes and WebSocket handshakes from another origin get `Cross-origin request refused`. This blocks DNS rebinding. Add your hostname (no port) to `ALLOWED_HOSTS`, or your origin to `CORS_ORIGINS`.

### Why does `make health` print "Backend not responding" when the backend is up?

`make health` pipes the answer through `jq`. Without `jq` installed, it prints that message anyway. Install `jq`, or call `curl http://127.0.0.1:8001/health` yourself.

### How do I rotate the API key?

Delete the `CODESAGE_API_KEY` line in `backend/.env`, run `make setup`, run `make install-claude` (or your client's target), then restart the backend, Vite and the client. See [Installation](/docs/installation/#rotate-the-key).

### How do I delete everything?

1. Delete each repository in the dashboard. That removes its Algolia records and database rows.
2. Run `make db-reset` to delete `backend/codesage.db` and its `-journal`, `-wal` and `-shm` files.
3. Delete the Algolia index in your Algolia dashboard, if you no longer need it. `make db-reset` and `make clean` don't touch Algolia.
4. Remove the `codesage` entry from your MCP client's config (the installer left `.bak-*` backups next to it that you can delete too).
5. On macOS, if you used `--basic-auth`, remove the Keychain item for service `codesage-mcp`.
6. Optionally delete `.venv`, `frontend/node_modules` and the cloned folder. For Docker, `make clean` deletes the dev stack's containers, network and volume.

### Can several people share one instance?

Technically yes, but there is one shared API key and no per-user permissions. Everyone with access can search, add and delete every repository. Use the prod stack's login and keep it to people you trust.

## Related pages

- [Troubleshooting](/docs/troubleshooting/)
- [Limitations](/docs/limitations/)
- [Configuration reference](/docs/reference/configuration/)
