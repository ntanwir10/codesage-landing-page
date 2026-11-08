---
title: Other git hosts
description: Add GitLab, Bitbucket and self-hosted repositories, public or private with a token, from the dashboard, the REST API or the add_repository tool.
---

CodeSage can index a repository from GitLab, Bitbucket or a self-hosted git server, as well as GitHub. Public repositories need nothing; private ones need a token for the host in `GIT_HOST_CREDENTIALS` (see [Private repositories](#private-repositories)). The dashboard's Home form, `POST /api/v1/repositories/` and the `add_repository` MCP tool all take the same URLs and run the same checks. This page covers the URLs they accept, how the backend reads the repository, the links in search results and every refusal message.

In the dashboard, paste the URL (for example `https://gitlab.com/group/project`) into the Home page form. Typing `owner/repo` without a host still means GitHub. From your AI assistant, ask something like:

> Add https://gitlab.com/group/project to CodeSage.

The assistant calls `add_repository`, which returns `repository_id`, `name`, `status` and `Indexing started; check list_repositories for progress.` Indexing runs in the background. Ask it to check `list_repositories`, and search once the status is `completed`. Then try *"Search it for where requests are retried."*

## Accepted URL forms

The URL must be an `https` URL on a public host. The form, the REST API and the tool also accept `http://` (rewritten to `https://`) and `git@host:path` (rewritten to `https://host/path`).

| Host | URL | Result |
| --- | --- | --- |
| GitLab | `https://gitlab.com/group/project` | Name `gitlab.com/group/project`, default branch |
| GitLab, nested groups | `https://gitlab.com/group/subgroup/project` | Name `gitlab.com/group/subgroup/project` |
| GitLab, branch | `https://gitlab.com/group/project/-/tree/dev` | Branch `dev` |
| GitLab, branch with slash | `https://gitlab.com/group/project/-/tree/feature/login` | Branch `feature/login` |
| Bitbucket | `https://bitbucket.org/team/repo` | Name `bitbucket.org/team/repo` |
| Bitbucket, branch | `https://bitbucket.org/team/repo/src/main` | Branch `main` |
| Self-hosted | `https://git.example.com/team/repo` | Name `git.example.com/team/repo` |
| Self-hosted, nested path | `https://git.example.com/a/b/c` | Name `git.example.com/a/b/c` |
| GitHub | `https://github.com/owner/repo/tree/dev` | Name `owner/repo`, branch `dev` |
| Any host, `.git` suffix | `https://gitlab.com/group/project.git` | `.git` and a trailing slash are removed |
| Any host, SSH form | `git@gitlab.com:group/project.git` | Rewritten to `https://gitlab.com/group/project` |

Rules behind the table:

- **Branch markers.** On GitHub the branch follows `/tree/`. On other hosts it follows `/-/tree/`. On `bitbucket.org` it follows `/src/`. Everything after the marker is joined into the branch name, so a URL that ends in a file path becomes a very odd branch. Pass a plain repository URL plus the `branch` argument when in doubt.
- **Other hosts.** Without one of those markers, extra path segments are read as nested groups. A forge with a different URL style (Gitea's `/src/branch/main`, for example) isn't understood; the segments become part of the repository path and the clone later fails.
- **Argument wins.** If you pass `branch` it overrides the branch in the URL. An empty `branch` means the default branch, even with a `/tree/` URL.
- **Segment characters.** Each path segment may contain only letters, digits, `_`, `.` and `-`, and may not be `.` or `..`.
- **`www.`** is removed only from `www.github.com`.
- **Length.** The URL is at most 500 characters and the derived name at most 255.
- **One row per URL.** The stored URL has no branch, so you can't index two branches of the same repository. A second add, even with another branch or another letter case, is refused as a duplicate.

## Names

A GitHub repository is named `owner/repo`. Every other repository is named `<host>/<path>`, such as `gitlab.com/group/project`, so names from different hosts can't collide. Pass that name, its numeric ID, or the same name in any letter case as the `repository` argument of the other tools.

## Private repositories

Git can't ask for credentials here, so the backend reads them from `GIT_HOST_CREDENTIALS` in `backend/.env`: comma-separated `host=user:token` pairs, one per host. Restart the backend after changing it. github.com keeps `GITHUB_TOKEN` and can't be listed.

```bash
GIT_HOST_CREDENTIALS=gitlab.com=oauth2:<token>,bitbucket.org=x-token-auth:<token>
```

A read-only token is enough:

| Host | Entry | Token |
| --- | --- | --- |
| GitLab (gitlab.com or self-managed) | `gitlab.com=oauth2:<token>` | A personal, project or group access token with the `read_repository` scope |
| Bitbucket Cloud | `bitbucket.org=x-token-auth:<token>` | A repository, project or workspace access token with repository read access |
| Gitea and Forgejo (such as Codeberg) | `codeberg.org=<user>:<token>` | An access token with repository read access |

This was checked end to end with a private gitlab.com repository and a personal access token with the `read_api` scope. The repository indexed and could be searched and read. The token appeared in no log, in none of the process command lines sampled during the clone, and nowhere in the clone. Without the token, and with a wrong one, CodeSage gave the two refusals shown below. The table recommends `read_repository`, the scope GitLab documents for reading repositories over git. The test used `read_api` instead, which GitLab documents only for its API, though the clone worked with it.

- **Checked at startup.** A malformed entry, a host listed twice, or `github.com` stops the backend from starting. The error names the entry's position, never its value. The token may contain `=` and `:` but not a comma.
- **Sent to that host only.** git gets an `Authorization: Basic` header scoped to `https://<host>/`, through `GIT_CONFIG_*` environment variables rather than its command line, which any user's `ps` can read. Only the same user and root can read a process's environment, and the clone's `.git/config` never holds it. The check, the clone and the freshness check all send it.
- **No redirects.** With a token, git follows no redirects, so the header can't follow one either. git is given `https://<host>/<path>.git`, because gitlab.com redirects the path without `.git`. A repository that moved must be added at its current address.
- **One token per host.** It is used for every repository on that host. The dashboard and `add_repository` can't pass a token of their own. A bad token fails that host's public repositories too.
- **Needs git 2.31 or later**, for the environment variables. The backend images have it.

Without a token, a repository that needs a login is treated as missing:

```text
gitlab.com/group/project was not found or is private; only public repositories are supported on gitlab.com.
```

With one, the same refusal blames the token instead:

```text
gitlab.com rejected the token in GIT_HOST_CREDENTIALS or it lacks access to group/project.
```

A repository that answers `401`, `403` or `404` to git, or prints "authentication failed" or "terminal prompts disabled", is reported one of these two ways. A wrong or expired token shows up as git's "could not read Username", because git gets a `401` and can't prompt.

## Rejected URLs

A URL the backend rejects comes back as an error that starts with `url:` and says why. Nothing is created.

| Rejected | Message after `url: ` |
| --- | --- |
| `ssh://...`, `ftp://...`, `git://...` | `Invalid repository URL (only https is supported): <url>` |
| No host | `Invalid repository URL (missing host): <url>` |
| `https://user:pass@host/...` or `https://user@host/...` | `Invalid repository URL (credentials are not allowed): <url>` |
| Any port, even `:443` | `Invalid repository URL (ports are not supported): <url>` |
| `?query` or `#fragment` | `Invalid repository URL (query or fragment not allowed): <url>` |
| `localhost` or `*.localhost` | `Invalid repository URL (localhost is not allowed): <url>` |
| `127.0.0.1`, `10.0.0.5`, `192.168.1.5`, `[::1]` (any non-public IP literal) | `Invalid repository URL (private or loopback address): <url>` |
| A `/-/` not followed by `tree`; a GitHub URL with extra segments other than `/tree/<branch>`; a Bitbucket URL with extra segments other than `/src/<branch>` | `Invalid repository URL (unsupported path): <url>` |
| Fewer than two path segments | `Invalid repository URL (missing owner or repository): <url>` |
| A segment with other characters, or `.` and `..` | `Invalid repository URL (bad owner or repository name): <url>` |
| A branch that breaks git's branch-name rules (`git check-ref-format`: no spaces, `..`, `~`, `^`, `:`, `?`, `*`, `[`, `\` and so on) or starts with `-` | `Invalid repository URL (bad branch): <url>` |
| A branch argument with the same faults | `branch: Invalid branch name` |
| An empty or whitespace-only URL | `Invalid repository URL: empty` |
| Over 500 characters | `url: String should have at most 500 characters` |

The IP rule means a repository on a private network (an internal GitLab at `10.x.x.x`) can't be added by IP. A DNS name is not checked by the URL rules, but it is resolved before git runs: a name that resolves to a private, loopback or other non-public address (such as `10.0.0.1.nip.io`), or that doesn't resolve, is refused. See [Refusal messages](#refusal-messages).

## How the backend reads the repository

github.com goes through the GitHub REST API. Any other host is read with the `git` command, so the backend machine needs `git` (the prod Docker image includes it).

1. **Check.** Before anything is created, the backend resolves the host name and refuses it unless every address is public. It then runs `git ls-remote --heads --symref -- <url>` (with `refs/heads/<branch>` appended when a branch is set), pinned to the checked address. It times out after **10 seconds**.
2. **Create.** If the check passes, a repository row is created with status `pending` and a background job starts.
3. **Clone.** The job resolves and checks the host again, then runs `git clone --depth 1 --single-branch --no-tags` (with `--branch <b>` when set) into a temporary `codesage-clone-*` folder. It times out after **120 seconds**. `git rev-parse HEAD` records the commit that was cloned, and source links point at it. The folder is always removed afterwards.
4. **Select.** The same rules as for GitHub apply: skip `node_modules`, `dist`, `build`, `vendor`, `third_party`, `.git`, `__pycache__`, `.next` and `coverage`; skip minified files, source maps and lockfiles; keep known extensions only; skip files of 1 MB or more; keep the first 1000 files or 100 MB, taking source files first, then tests, then Markdown, then files that yield no entities, each group by path. Anything left out makes the index partial.
5. **Parse, store, index.** The same pipeline as for GitHub. See [Indexing pipeline](/docs/concepts/indexing-pipeline/).

### The git sandbox

Git runs without a shell (`create_subprocess_exec`), with no stdin, and is killed on timeout. Every call carries:

- `-c credential.helper=`, so your keychain is never consulted;
- `-c protocol.allow=never -c protocol.https.allow=always`, so only https works whatever your git config says;
- a clean environment: `PATH`, `HOME` set to the null device, `GIT_TERMINAL_PROMPT=0`, `GIT_CONFIG_GLOBAL` set to the null device, `GIT_CONFIG_NOSYSTEM=1` and `LC_ALL=C`. Only `SSL_CERT_FILE`, `SSL_CERT_DIR`, `GIT_SSL_CAINFO`, `HTTPS_PROXY`, `https_proxy` and `NO_PROXY` are passed through from the backend's own environment.

The `ls-remote` and `clone` calls also carry `-c http.curloptResolve=<host>:443:<ip>`, with the address the backend just checked (IPv4 when there is one). git never resolves the name itself, so a DNS answer that changes between the check and the request (DNS rebinding) can't send it elsewhere.

For a host in `GIT_HOST_CREDENTIALS`, the environment also carries `GIT_CONFIG_COUNT`, `GIT_CONFIG_KEY_n` and `GIT_CONFIG_VALUE_n` with the host's `http.https://<host>/.extraHeader` and `http.followRedirects=false`. `git rev-parse` in the local clone gets neither.

Without a token, redirects are still followed, because gitlab.com answers `/group/repo` with a `301` to `/group/repo.git`. git follows that redirect without the address check, so a hostile public host could point it at an internal address. If that matters, route the backend's traffic through an egress proxy that refuses private addresses. A host with a token gets no redirects.

Symlinks: a clone keeps symlinks as real links. The tree walk doesn't follow or list them, so they are never indexed. (Older notes that mention `core.symlinks=false` are out of date.)

For a host with a private certificate authority, set `SSL_CERT_FILE` (or `GIT_SSL_CAINFO`) in the backend's environment.

## Source links in search results

Search hits carry a link to the exact lines. The format depends on the host and uses the indexed commit as `<ref>`:

| Host | Link |
| --- | --- |
| github.com | `https://github.com/<owner>/<repo>/blob/<ref>/<file>#L<start>-L<end>` |
| `gitlab.com`, or any host whose name starts with `gitlab.` | `https://<host>/<path>/-/blob/<ref>/<file>#L<start>-<end>` |
| `bitbucket.org` | `https://bitbucket.org/<path>/src/<ref>/<file>#lines-<start>:<end>` |
| Any other host | No link |

The link points at the commit that was indexed, so its line numbers stay right after the branch moves on. A repository indexed before commits were recorded links to its branch (or `HEAD` when none was set) until you reprocess it. On the repository page the source link reads **View source** for hosts other than github.com.

## Refusal messages

For a non-GitHub URL the check refuses with one of these, and nothing is created. The dashboard shows it under the repository field, the REST API returns it as `detail`, and `add_repository` returns it as a tool error. (github.com URLs use GitHub's check and the messages in [the dashboard guide](/docs/guides/web-dashboard/#when-the-server-refuses-a-repository).)

| Message | Cause |
| --- | --- |
| `<host>/<path> was not found or is private; only public repositories are supported on <host>.` | The repository doesn't exist, or it needs a login and the host has no token in `GIT_HOST_CREDENTIALS` |
| `<host> rejected the token in GIT_HOST_CREDENTIALS or it lacks access to <path>.` | The host has a token, but it is wrong or expired, can't read the repository, or the repository doesn't exist |
| `<host> redirected <path>; with a token CodeSage doesn't follow redirects, so use its current address.` | The repository moved; add it at its new URL |
| `Can't index <host>/<path>: <host> resolves to a private or loopback address.` | The host name resolves to a non-public address (private, loopback, link-local and so on). Status `422` |
| `Can't index <host>/<path>: <host> could not be resolved.` | The host name doesn't resolve. Status `422` |
| `Branch '<b>' does not exist in <host>/<path>.` | The branch is missing |
| `<host>/<path> is empty; there is nothing to index.` | The repository has no branches (checked only when no branch was given) |
| `<host> did not respond in time; try again.` | The 10 s check timed out |
| `Could not reach <host>; try again.` | The host failed in another way, or `git` couldn't be started |
| `<host>/<path> is already added as repository <id>.` | The URL already exists (case-insensitive). This is the `add_repository` tool's text. The REST API and the dashboard say `<host>/<path> has already been added.` with status `409` |
| `Search indexing is not configured on the server (Algolia credentials are missing).` | `ALGOLIA_APP_ID` or `ALGOLIA_ADMIN_API_KEY` is unset |
| `url: ...` | The URL was rejected, see the table above |

The check makes one attempt, with no retries.

A repository that passes the check can still fail while it runs. The reason is stored in `error_message` and shows in `list_repositories`, the dashboard card and the repository page:

| `error_message` | Cause |
| --- | --- |
| `<host> did not respond in time while cloning <host>/<path>` | The clone passed 120 s |
| `Can't clone <host>/<path>: <host> resolves to a private or loopback address` | The host name now resolves to a non-public address, or (`...could not be resolved`) no longer resolves |
| `Could not run git to clone <host>/<path>` | `git` isn't installed or couldn't start |
| `Branch '<b>' not found in <host>/<path>` | The branch disappeared between the check and the clone |
| `<host>/<path> was not found or is private; only public repositories are supported on <host>` | The repository became private or was removed |
| `<host> rejected the token in GIT_HOST_CREDENTIALS or it lacks access to <path>` | The host's token was revoked, expired or lost access, or the repository was removed |
| `Could not clone <host>/<path> from <host>` | Any other clone failure |
| `No supported code files found in repository` | Nothing in the clone matched the file rules |

Fix the cause and reprocess from the repository page.

## Limits

- **Private repositories need a server token,** one per host in `GIT_HOST_CREDENTIALS`, shared by every repository on that host.
- **Redirects.** Without a token, git follows a host's first HTTP redirect without re-checking where it points, so a hostile public host could send it to an internal address. Route the backend through an egress proxy that refuses private addresses if that matters. With a token, no redirect is followed.
- **Tool limits.** `add_repository` needs the API key like every tool call and counts toward the 30 per minute limit.
- **Clone size.** A shallow clone of a huge repository must finish in 120 seconds and fit the 1000-file and 100 MB selection.

See also [MCP tools](/docs/reference/mcp-tools/), [MCP clients](/docs/guides/claude-desktop/) and [Troubleshooting](/docs/troubleshooting/).
