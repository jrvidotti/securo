# Deploying Securo on Railway

[Railway](https://railway.com) does not run `docker-compose.yml` files. Each service in
`docker-compose.prod.yml` becomes a separate Railway service, all built from this repo.
You end up with five services — six if you also run the agents MCP server (§7).

You can drag a Compose file onto the project canvas, but the import ignores `command`,
`depends_on`, `profiles`, YAML anchors (`x-backend-env`) and `${VAR:-default}` interpolation,
and Railway allows only one volume per service — with this Compose file it is faster to create
the services by hand.

Throughout, `${{service.VAR}}` is Railway's own reference syntax, resolved at deploy time. Using
references instead of pasted literals is what keeps the services consistent when a password or a
domain changes.

## 1. Databases

1. **New Project → Empty project**.
2. **+ New → Template → "Postgres with pgvector"**. This guide assumes the service is named
   **`pgvector`**; if you name it something else, adjust the `${{pgvector.*}}` references below.
   The stock Railway Postgres ships no extensions, and the `046_agents_foundation` migration
   requires pgvector — the backend will fail to start against it.
3. **+ New → Database → Redis**, named `Redis`.

### Connection strings on the `pgvector` service

The backend needs the `postgresql+asyncpg://` scheme, but Railway's own `DATABASE_URL` uses plain
`postgresql://`. Rather than rewriting it in every consumer, define both forms once **on the
`pgvector` service itself** (Variables tab), where the `POSTGRES_*` and `RAILWAY_*` values are
in scope without a service prefix:

```
DATABASE_URL="postgresql+asyncpg://${{POSTGRES_USER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_TCP_PROXY_DOMAIN}}:${{RAILWAY_TCP_PROXY_PORT}}/railway"
DATABASE_URL_PRIVATE="postgresql+asyncpg://${{POSTGRES_USER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:5432/${{POSTGRES_DB}}"
```

- `DATABASE_URL_PRIVATE` goes over the internal network: no egress cost, no public exposure.
  Prefer it.
- `DATABASE_URL` goes through Railway's **public TCP proxy**. It requires the database to have a
  public endpoint enabled, and it bills egress. It is the fallback for anything that cannot reach
  the private network.
- **Both must point at the same database.** `railway` is the default database name on Railway's
  Postgres templates, so `${{POSTGRES_DB}}` normally resolves to exactly that — but if you renamed
  the database, the hardcoded `/railway` above becomes a different one, and the backend and the
  worker will silently read and write separate datasets.

## 2. Shared variables

Under **Project Settings → Shared Variables** — the values every service should agree on:

```
SECRET_KEY="(openssl rand -base64 32)"
AGENTS_MCP_JWT_SECRET="(openssl rand -base64 32)"
STORAGE_LOCAL_PATH="/app/data/attachments"
AGENTS_KNOWLEDGE_STORAGE_PATH="/app/data/agent_knowledge"
OIDC_ENABLED="true"
OIDC_PROVIDER_NAME="OpenID"
OIDC_DISCOVERY_URL="https://id.domain.com/.well-known/openid-configuration"
OIDC_CLIENT_ID="..."
OIDC_CLIENT_SECRET="..."
OIDC_SCOPES="openid email profile"
OIDC_AUTO_REGISTER="true"
OIDC_REQUIRE_VERIFIED_EMAIL="true"
PLUGGY_CLIENT_ID="..."
PLUGGY_CLIENT_SECRET="..."
```

Replace the `openssl` lines with real generated values — Railway does not run shell commands in
variables.

`DATABASE_URL` and `REDIS_URL` are deliberately **not** shared: the backend and the worker point
at different endpoints, so each service sets its own (see below).

The `OIDC_*` and `PLUGGY_*` blocks are examples — keep only the integrations you actually use.
For Enable Banking there is no file to mount on Railway, so use `ENABLE_BANKING_PRIVATE_KEY` with
the PEM as text (`\n`-escaped) instead of `ENABLE_BANKING_PRIVATE_KEY_FILE`.

Shared variables are not injected automatically; each service must reference the ones it needs,
which is what the `${{shared.*}}` lines in the next sections do.

## 3. `backend`

- **Source:** this GitHub repo. **Settings → Source → Root Directory:** `backend`.
- **Custom Start Command:**
  ```
  sh -c "alembic upgrade head && uvicorn app.main:app --host :: --port 8000"
  ```
  `--host ::` is required: Railway's private network is IPv6-only, so a service bound to
  `0.0.0.0` is unreachable from the other services.
- **Volume:** one, mounted at `/app/data`. Railway allows a single volume per service, and all
  three backend data paths (`attachments`, `agent_knowledge`, `embedding_models`) live under it.
- **No public domain** — the frontend proxies to it over the private network.

```
DATABASE_URL="${{pgvector.DATABASE_URL_PRIVATE}}"
REDIS_URL="${{Redis.REDIS_URL}}"
FRONTEND_URL="https://${{frontend.RAILWAY_PUBLIC_DOMAIN}}"
OIDC_AUTO_REGISTER="${{shared.OIDC_AUTO_REGISTER}}"
OIDC_CLIENT_ID="${{shared.OIDC_CLIENT_ID}}"
OIDC_CLIENT_SECRET="${{shared.OIDC_CLIENT_SECRET}}"
OIDC_DISCOVERY_URL="${{shared.OIDC_DISCOVERY_URL}}"
OIDC_ENABLED="${{shared.OIDC_ENABLED}}"
OIDC_PROVIDER_NAME="${{shared.OIDC_PROVIDER_NAME}}"
OIDC_REQUIRE_VERIFIED_EMAIL="${{shared.OIDC_REQUIRE_VERIFIED_EMAIL}}"
OIDC_SCOPES="${{shared.OIDC_SCOPES}}"
PLUGGY_CLIENT_ID="${{shared.PLUGGY_CLIENT_ID}}"
PLUGGY_CLIENT_SECRET="${{shared.PLUGGY_CLIENT_SECRET}}"
AGENTS_KNOWLEDGE_STORAGE_PATH="${{shared.AGENTS_KNOWLEDGE_STORAGE_PATH}}"
AGENTS_MCP_JWT_SECRET="${{shared.AGENTS_MCP_JWT_SECRET}}"
SECRET_KEY="${{shared.SECRET_KEY}}"
STORAGE_LOCAL_PATH="${{shared.STORAGE_LOCAL_PATH}}"
```

`FRONTEND_URL` references the frontend service's generated domain rather than a pasted string, so
it keeps working if that domain changes. The backend uses it for CORS (`app/main.py`), for the
bank OAuth callback (`app/providers/base.py`) and for the OIDC redirect
(`app/api/oidc_auth.py`).

## 4. `worker`

Same repo, **Root Directory `backend`**, no volume, no domain.

```
celery -A app.worker worker -B --loglevel=info --concurrency=2
```

`-B` runs beat inside the worker, so the `celery-worker` and `celery-beat` services from Compose
collapse into one Railway service.

```
AGENTS_KNOWLEDGE_STORAGE_PATH="${{shared.AGENTS_KNOWLEDGE_STORAGE_PATH}}"
AGENTS_MCP_JWT_SECRET="${{shared.AGENTS_MCP_JWT_SECRET}}"
DATABASE_URL="${{pgvector.DATABASE_URL}}"
OIDC_AUTO_REGISTER="${{shared.OIDC_AUTO_REGISTER}}"
OIDC_CLIENT_ID="${{shared.OIDC_CLIENT_ID}}"
OIDC_CLIENT_SECRET="${{shared.OIDC_CLIENT_SECRET}}"
OIDC_DISCOVERY_URL="${{shared.OIDC_DISCOVERY_URL}}"
OIDC_ENABLED="${{shared.OIDC_ENABLED}}"
OIDC_PROVIDER_NAME="${{shared.OIDC_PROVIDER_NAME}}"
OIDC_REQUIRE_VERIFIED_EMAIL="${{shared.OIDC_REQUIRE_VERIFIED_EMAIL}}"
OIDC_SCOPES="${{shared.OIDC_SCOPES}}"
PLUGGY_CLIENT_ID="${{shared.PLUGGY_CLIENT_ID}}"
PLUGGY_CLIENT_SECRET="${{shared.PLUGGY_CLIENT_SECRET}}"
REDIS_URL="${{Redis.REDIS_URL}}"
SECRET_KEY="${{shared.SECRET_KEY}}"
STORAGE_LOCAL_PATH="${{shared.STORAGE_LOCAL_PATH}}"
```

Three things differ from the backend, all of them deliberate:

- **`DATABASE_URL` is the public TCP proxy form**, not `DATABASE_URL_PRIVATE`. Switch it to the
  private one if the worker connects over the internal network in your project — it is faster and
  avoids egress charges. Keep the public form as the fallback.
- **No `FRONTEND_URL`.** The worker does not need it: the Celery tasks only reach
  `connection_service.sync_connection()`, and the code paths that derive a URL from
  `FRONTEND_URL` — CORS, `get_oauth_url()`, `get_reauth_url()`, the OIDC callback — are all
  request-driven and live in the API service. Leaving it unset means the settings default
  (`http://localhost:5173`) applies where nothing reads it.
- **No volume.** Railway volumes cannot be shared between services, so the worker cannot see
  `/app/data`. This is fine for the scheduled tasks as they stand; a future task that writes
  attachments would need rethinking.

`PLUGGY_*` is genuinely required here — that is the service that actually syncs the connections.
The `OIDC_*` block is authentication-only and unused by the worker; it is kept in sync purely so
the two services do not drift.

## 5. `frontend`

- Same repo, **Root Directory:** `frontend`.
- **Networking → Generate Domain**, then **set the target port to 8080**. This is the one setting
  that is easy to get wrong: `8080` is the container port. The `3000` in
  `docker-compose.prod.yml` is a *host* port that does not exist on Railway, and pointing the
  domain at it makes Railway's edge return **502 Bad Gateway on every path**, including `/` —
  even though the nginx logs look completely healthy.

```
BACKEND_URL="http://${{backend.RAILWAY_PRIVATE_DOMAIN}}:8000"
FRONTEND_URL="https://${{RAILWAY_PUBLIC_DOMAIN}}"
NGINX_RESOLVER="[fd12::10] ipv6=on"
```

`RAILWAY_PUBLIC_DOMAIN` without a service prefix is this service's own domain.

`NGINX_RESOLVER` is **optional** on an image built from this repo: the entrypoint derives the
resolver from the container's own `/etc/resolv.conf`, which on Railway already yields
`[fd12::10]` (see `frontend/docker-entrypoint.d/19-resolver.envsh`). Set it explicitly only to
override that detection, or when running an older published image that still hardcodes Docker's
`127.0.0.11` — without it, such an image returns 502 on every `/api/` request.

## 6. Order and verification

Bring up `pgvector` and `Redis`, then `backend` (its start command runs the migrations — a
`pgvector is not installed` failure here means the wrong Postgres template), then `worker` and
`frontend`.

- `https://<domain>/` → the login screen.
- `https://<domain>/api/docs` → FastAPI's docs, proving the nginx proxy reaches the backend.

A 502 on `/` is an edge/target-port problem, not an nginx one: that location is a plain
`try_files` with no upstream and cannot itself emit a 502. A 502 only on `/api/` points at
`BACKEND_URL` or at the backend not listening on `::`.

## 7. Optional: the agents MCP server

`backend/mcp_server/main.py` is a second FastAPI app in the same image: JSON-RPC 2.0 on
`POST /mcp`, plus `GET /health`. It exposes ~30 Securo tools — read ones (`list_transactions`,
`get_net_worth`, `get_dashboard_snapshot`, `search_all`, `aggregate`, `get_group_balances`…) and
write ones that only ever create a proposal for you to approve in the UI
(`propose_create_transaction`, `propose_categorize`, `propose_create_budget`…).

It has no session and no cookie. Every call carries `Authorization: Bearer <JWT>`, verified in
`mcp_server/auth.py` against `AGENTS_MCP_JWT_SECRET` (HS256, `iss=securo-backend`,
`aud=securo-mcp`); the token's `sub` and `ws_id` claims are what scope the data. Two things mint
that token, both in the backend: the agent runtime, per call, with a 600 s TTL; and
**Agents → Connections → External MCP access** in the UI, with a 90-day TTL
(`AGENTS_MCP_EXTERNAL_TTL_DAYS`) for Claude Desktop/Code, Cursor, n8n or the OpenAI Responses API.

### The service

Same repo, **Root Directory `backend`**, no volume:

```
uvicorn mcp_server.main:app --host :: --port 8765
```

**Networking → Generate Domain, target port 8765** — needed only for external clients; the
backend itself reaches this service over the private network. Same trap as the frontend: `8765`
is the *container* port, and Railway's edge still serves the domain on 443. External clients
therefore use `https://<mcp-domain>/mcp`, never `:8765`.

```
DATABASE_URL="${{pgvector.DATABASE_URL_PRIVATE}}"
REDIS_URL="${{Redis.REDIS_URL}}"
AGENTS_ENABLED="true"
AGENTS_MCP_JWT_SECRET="${{shared.AGENTS_MCP_JWT_SECRET}}"
AGENTS_KNOWLEDGE_STORAGE_PATH="${{shared.AGENTS_KNOWLEDGE_STORAGE_PATH}}"
SECRET_KEY="${{shared.SECRET_KEY}}"
```

The tools query the database directly, which is why `DATABASE_URL` is not optional here. A
secret that differs from the backend's is the single most common failure: it produces a 401 on
every call, with nothing else wrong.

### What the backend and the worker need

On **`backend`**:

```
AGENTS_ENABLED="true"
AGENTS_BUILTIN_MCP_URL="http://${{mcp-server.RAILWAY_PRIVATE_DOMAIN}}:8765/mcp"
AGENTS_EXTERNAL_MCP_URL="https://${{mcp-server.RAILWAY_PUBLIC_DOMAIN}}/mcp"
```

`AGENTS_ENABLED` is the master switch: `app/main.py` mounts the whole `/api/agents` router only
when it is on, so without it the token endpoint 404s and the Connections page has nothing to
talk to.

`AGENTS_EXTERNAL_MCP_URL` is the one that fails silently. It is only a display value — the URL
the token panel puts in the snippets it hands the user. Left empty, the frontend falls back to
deriving `<protocol>//<hostname>:8765/mcp` from the browser location
(`frontend/src/components/agents/mcp-external-panel.tsx`), which is right for Docker Compose and
wrong on Railway, where nothing listens on 8765 at the edge. The copied config then points
nowhere while the server itself is perfectly healthy. Note there is no way to serve MCP from the
frontend's domain instead: its nginx proxies `/api/` only, so the MCP service needs a domain of
its own.

On **`worker`**, only `AGENTS_ENABLED="true"`. Its one agents task is knowledge ingestion
(`app/agents/tasks/ingest.py`) — it never calls the MCP server, so `AGENTS_BUILTIN_MCP_URL` there
would be dead configuration.

### Verification

```
https://<mcp-domain>/health          → {"status":"ok","tools":29}
```

Then mint a token in the UI (**Agents → Connections → External MCP access**; it requires write
access to the workspace, because the tool set can create proposals) and call the server with it:

```bash
curl -X POST https://<mcp-domain>/mcp \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

The panel also prints a ready-made config per client. For Claude Desktop, Claude Code and Cursor
it is:

```json
{ "mcpServers": { "securo": {
  "url": "https://<mcp-domain>/mcp",
  "headers": { "Authorization": "Bearer <token>" } } } }
```

A token is bound to the workspace that was active when it was minted. For a second workspace,
switch in the UI and mint again.

### Two rough edges on Railway

- **Knowledge-base ingestion does not work as laid out here.** The upload endpoint writes the
  file to the backend's volume (`/app/data/agent_knowledge`) and dispatches
  `ingest_doc` to the worker, which reads it back by path — and the worker has no volume, so the
  read fails and the document stays in `failed`. Chat and every non-knowledge tool are
  unaffected. Fixing it means giving the ingestion a storage both services can see.
- **No volume on the MCP service** means `AGENTS_EMBEDDING_PROVIDER=native` re-downloads the
  fastembed model (~100 MB) into `/app/data/embedding_models` on every cold start. This only
  affects `search_knowledge_base`, which in turn only runs inside an agent conversation — an
  external token never reaches it, since the tool requires an `agent_id` claim. Mount a volume at
  `/app/data` here too, or switch to `openai`/`ollama`, if it bothers you.
