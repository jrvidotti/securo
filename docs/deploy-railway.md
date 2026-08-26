# Deploying Securo on Railway

Three services: Postgres, Redis, and one app container that serves the frontend, the API and the
background worker.

[Railway](https://railway.com) does not run `docker-compose.yml` files, and it allows **only one
volume per service** — which is exactly why this guide uses the
[all-in-one container](deploy-all-in-one.md) rather than the six-service Compose topology. Split
across services, the worker cannot see the volume the API writes to, so knowledge-base ingestion
fails outright. One container, one volume, and the problem stops existing.

Throughout, `${{service.VAR}}` is Railway's own reference syntax, resolved at deploy time. Using
references instead of pasted literals keeps things working when a password or a domain changes.

## 1. Databases

1. **New Project → Empty project**.
2. **+ New → Template → "Postgres with pgvector"**. This guide assumes the service is named
   **`pgvector`**; if you name it something else, adjust the `${{pgvector.*}}` references below.
   The stock Railway Postgres ships no extensions, and the `046_agents_foundation` migration
   requires pgvector — the app will fail to start against it.
3. **+ New → Database → Redis**, named `Redis`.

Redis is not optional even if you never touch the AI features: login rate limiting, 2FA tokens,
passkey challenges, OIDC state and the bank OAuth callback all depend on it.

### The connection string

Securo needs the `postgresql+asyncpg://` scheme, but Railway's own `DATABASE_URL` uses plain
`postgresql://`. Define the right form once **on the `pgvector` service itself** (Variables tab),
where the `POSTGRES_*` and `RAILWAY_*` values are in scope without a prefix:

```
DATABASE_URL_PRIVATE="postgresql+asyncpg://${{POSTGRES_USER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:5432/${{POSTGRES_DB}}"
```

This goes over the internal network: no egress cost, no public exposure.

<details>
<summary>Optional: a public form, for connecting from your laptop</summary>

```
DATABASE_URL="postgresql+asyncpg://${{POSTGRES_USER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_TCP_PROXY_DOMAIN}}:${{RAILWAY_TCP_PROXY_PORT}}/${{POSTGRES_DB}}"
```

It goes through Railway's public TCP proxy, so it needs a public endpoint enabled and it bills
egress. Useful for a database client or a one-off script; the app does not need it.
</details>

## 2. The app service

**+ New → GitHub Repo**, pointing at this repository.

- **Leave Root Directory at the repository root.** The image contains both the frontend build and
  the backend, so the build context needs both trees.
- **Set `RAILWAY_DOCKERFILE_PATH` before the first deploy.** It is in the variables below, and it
  is the one that stops the build rather than the app: Railway looks for `./Dockerfile`, does not
  find one, and falls back to its Railpack autodetector — see §4.
- **Custom Start Command: leave it empty.** The image's entrypoint runs the migrations, then
  uvicorn and a Celery worker with beat embedded. A start command here would fight it.
- **Volume:** one, mounted at **`/app/data`**. Attachments, the agents knowledge store, the
  embedding-model cache and the Celery beat schedule all live under it.
- **Networking → Generate Domain.** The entrypoint binds to the `PORT` Railway injects, falling
  back to 8000, so let Railway detect the port. If you pin it by hand, pin it to the same value.
- **Settings → Deploy:** set **Healthcheck Path** to `/api/health` and **Restart Policy** to
  *On Failure*. Neither is required, but the healthcheck is what stops Railway routing traffic to
  a container whose migrations are still running, and *On Failure* matches how the entrypoint
  signals a dead process. Give the healthcheck a generous timeout — the first boot of a fresh
  database runs every migration.

### Variables

```
RAILWAY_DOCKERFILE_PATH="backend/Dockerfile.aio"
DATABASE_URL="${{pgvector.DATABASE_URL_PRIVATE}}"
REDIS_URL="${{Redis.REDIS_URL}}"
FRONTEND_URL="https://${{RAILWAY_PUBLIC_DOMAIN}}"
UVICORN_HOST="::"
SECRET_KEY="<openssl rand -base64 32>"
STORAGE_LOCAL_PATH="/app/data/attachments"
CELERY_CONCURRENCY="1"
```

Generate `SECRET_KEY` yourself — Railway does not run shell commands in variables.

Four of these are worth explaining:

- **`RAILWAY_DOCKERFILE_PATH`** — Railway builds `./Dockerfile` by default and there is none at
  the repository root. The path is relative to that root.

  A `railway.json` build config would express this without a manual step, and it is tempting.
  Don't: Railway deprecated Config as Code, services that never used it cannot opt in after
  2026-08-28, and the files stop working on 2026-12-01. Its replacement, Infrastructure as Code
  (`.railway/railway.ts`), defines a whole project in TypeScript and documents no equivalent of
  `dockerfilePath`. The variable is the durable answer.
- **`UVICORN_HOST="::"`** — Railway's private network is IPv6-only. `::` binds both stacks, so it
  is also correct for the public domain.
- **`FRONTEND_URL`** is not just a CORS setting: it builds the bank OAuth callback URL
  (`app/providers/base.py`) and the OIDC redirect URI (`app/api/oidc_auth.py`). Referencing
  `RAILWAY_PUBLIC_DOMAIN` with no service prefix gives this service's own domain.
- **`STORAGE_LOCAL_PATH`** is set explicitly because its default is relative. The agents knowledge
  path already defaults to `/app/data/agent_knowledge` and needs no variable.

`CELERY_CONCURRENCY="1"` is a suggestion. The periodic tasks are hourly and run one after
another, so a second prefork child mostly costs memory. Drop the variable for the default of 2.

### Integrations

Add only the ones you use.

```
PLUGGY_CLIENT_ID="..."
PLUGGY_CLIENT_SECRET="..."

OIDC_ENABLED="true"
OIDC_PROVIDER_NAME="OpenID"
OIDC_DISCOVERY_URL="https://id.example.com/.well-known/openid-configuration"
OIDC_CLIENT_ID="..."
OIDC_CLIENT_SECRET="..."
OIDC_SCOPES="openid email profile"
OIDC_AUTO_REGISTER="true"
OIDC_REQUIRE_VERIFIED_EMAIL="true"
```

For Enable Banking there is no file to mount on Railway, so use `ENABLE_BANKING_PRIVATE_KEY` with
the PEM as text (`\n`-escaped) instead of `ENABLE_BANKING_PRIVATE_KEY_FILE`.

Whatever the provider, its redirect URI must point at this service's domain — there is only one.

## 3. Verification

Bring up `pgvector` and `Redis` first, then the app. Its entrypoint runs the migrations on start;
a `pgvector is not installed` failure there means the wrong Postgres template.

```
https://<domain>/            → the login screen
https://<domain>/api/health  → {"status":"healthy"}
```

**Point uptime monitoring at `/api/health`, not `/`.** Frontend routes are served only to requests
that accept `text/html`, so `curl https://<domain>/transactions` returns 404 by design while a
browser gets the app.

Check that both processes came up:

```
railway logs | grep -E "\[aio\]|celery@"
```

You want the `[aio]` lines from the entrypoint and a `celery@… ready` line. A container that
restarts in a loop is the entrypoint working as designed — it exits as soon as either process
dies, so read the last error before the restart rather than the restart itself.

## 4. Troubleshooting

**The build fails with `No start command detected`, mentioning Railpack.**

```
using build driver railpack-v0.38.0
⚠ Script start.sh not found
✖ No start command detected. Specify a start command
```

Railway never saw the Dockerfile and fell back to autodetection, which cannot make sense of a
repo that is a Python backend and a Vite frontend at once. `RAILWAY_DOCKERFILE_PATH` is unset, or
set to a path that is not relative to the repository root:

```
RAILWAY_DOCKERFILE_PATH="backend/Dockerfile.aio"
```

The build log is the quickest way to confirm it: Railway passes every variable the service has to
the builder as `--env NAME` arguments, so the failing log lists exactly what was set.

**The domain returns 502 although the build succeeded.** Check `UVICORN_HOST` is `::`. Bound to
`0.0.0.0`, the app is unreachable over Railway's IPv6-only network.

**Migrations fail with `pgvector is not installed`.** Wrong Postgres template — see §1.

**The deployment restarts in a loop.** The entrypoint exits as soon as either process dies, so
read the last error *before* the restart, not the restart itself. An unreachable database shows
up here as a failed `alembic upgrade head`.

## 5. Optional: AI agents and the MCP server

`backend/mcp_server/` exposes ~30 Securo tools over JSON-RPC 2.0 on `POST /mcp` — read ones
(`list_transactions`, `get_net_worth`, `get_dashboard_snapshot`, `search_all`, `aggregate`…) and
write ones that only ever create a proposal for you to approve in the UI
(`propose_create_transaction`, `propose_categorize`…).

In this topology it runs inside the app process, so there is no service to create:

```
AGENTS_ENABLED="true"
AGENTS_MCP_INPROCESS="true"
AGENTS_MCP_JWT_SECRET="<openssl rand -base64 32>"
AGENTS_BUILTIN_MCP_URL="http://127.0.0.1:8000/mcp"
AGENTS_EXTERNAL_MCP_URL="https://${{RAILWAY_PUBLIC_DOMAIN}}/mcp"
```

- **`AGENTS_ENABLED`** is the master switch: the whole `/api/agents` router is mounted only when
  it is on, so without it the token endpoint 404s and the Connections page has nothing to talk to.
- **`AGENTS_MCP_INPROCESS`** is deliberately separate. Enabling agents should not silently widen
  what the API exposes, so serving `/mcp` from the same process is its own decision.
- **`AGENTS_BUILTIN_MCP_URL`** is a loopback call the app makes to itself. If you pinned a
  different port, match it here.
- **`AGENTS_EXTERNAL_MCP_URL`** is only a display value — the URL the token panel puts in the
  snippets it hands you. Left empty, the frontend derives `<protocol>//<hostname>:8765/mcp` from
  the browser location, which is right for Docker Compose and wrong here, where nothing listens on
  8765 at the edge. The copied config then points nowhere while the server is perfectly healthy.

### Authentication

There is no session and no cookie. Every call carries `Authorization: Bearer <JWT>`, verified in
`mcp_server/auth.py` against `AGENTS_MCP_JWT_SECRET` (HS256, `iss=securo-backend`,
`aud=securo-mcp`); the token's `sub` and `ws_id` claims are what scope the data. Two things mint
one: the agent runtime, per call, with a 600 s TTL; and **Agents → Connections → External MCP
access** in the UI, with a 90-day TTL (`AGENTS_MCP_EXTERNAL_TTL_DAYS`) for Claude Desktop/Code,
Cursor, n8n or the OpenAI Responses API.

A token is bound to the workspace that was active when it was minted. For a second workspace,
switch in the UI and mint again.

### Verification

```
https://<domain>/health   → {"status":"ok","tools":29}
```

Then mint a token in the UI (it requires write access to the workspace, because the tool set can
create proposals) and call the server:

```bash
curl -X POST https://<domain>/mcp \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

The panel also prints a ready-made config per client. For Claude Desktop, Claude Code and Cursor:

```json
{ "mcpServers": { "securo": {
  "url": "https://<domain>/mcp",
  "headers": { "Authorization": "Bearer <token>" } } } }
```

Knowledge-base ingestion and the `native` embedding provider both work here, because the worker
and the API share one container and one volume — the fastembed model (~100 MB) downloads once to
`/app/data/embedding_models` and persists.
