# Deploying Securo on Railway

[Railway](https://railway.com) does not run `docker-compose.yml` files. Each service in
`docker-compose.prod.yml` becomes a separate Railway service, all built from this repo.
You end up with five services.

You can drag a Compose file onto the project canvas, but the import ignores `command`,
`depends_on`, `profiles`, YAML anchors (`x-backend-env`) and `${VAR:-default}` interpolation,
and Railway allows only one volume per service — with this Compose file it is faster to create
the services by hand.

## 1. Databases

1. **New Project → Empty project**.
2. **+ New → Template → "Postgres with pgvector"**, named `Postgres`. The stock Railway Postgres
   ships no extensions, and the `046_agents_foundation` migration requires pgvector — the backend
   will fail to start against it.
3. **+ New → Database → Redis**, named `Redis`.

## 2. Shared variables

Under **Project Settings → Shared Variables**:

```
DATABASE_URL=postgresql+asyncpg://${{Postgres.PGUSER}}:${{Postgres.PGPASSWORD}}@${{Postgres.RAILWAY_PRIVATE_DOMAIN}}:5432/${{Postgres.PGDATABASE}}
REDIS_URL=${{Redis.REDIS_URL}}
SECRET_KEY=<openssl rand -hex 32>
AGENTS_MCP_JWT_SECRET=<openssl rand -hex 32>
FRONTEND_URL=https://<your-frontend-domain>
STORAGE_LOCAL_PATH=/app/data/attachments
AGENTS_KNOWLEDGE_STORAGE_PATH=/app/data/agent_knowledge
```

`DATABASE_URL` must be rebuilt by hand: Railway's own `DATABASE_URL` uses the `postgresql://`
scheme, and the backend needs `postgresql+asyncpg://`.

Add only the integrations you use (`PLUGGY_*`, `OIDC_*`, `OPENEXCHANGERATES_APP_ID`, …). For
Enable Banking there is no file to mount, so use `ENABLE_BANKING_PRIVATE_KEY` with the PEM as
text (`\n`-escaped) instead of `ENABLE_BANKING_PRIVATE_KEY_FILE`.

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
- **Variables:** reference the shared variables above.
- **No public domain** — the frontend proxies to it over the private network.

## 4. `worker`

Same repo, **Root Directory `backend`**, same variables, no volume, no domain.

```
celery -A app.worker worker -B --loglevel=info --concurrency=2
```

`-B` runs beat inside the worker, so the `celery-worker` and `celery-beat` services from Compose
collapse into one Railway service.

## 5. `frontend`

- Same repo, **Root Directory:** `frontend`.
- **Variables:**
  ```
  BACKEND_URL=http://backend.railway.internal:8000
  FRONTEND_URL=https://<your-frontend-domain>
  ```
  The hostname must match the backend service's name. No resolver variable is needed: the image
  derives nginx's DNS resolver from the container's own `/etc/resolv.conf`
  (see `frontend/docker-entrypoint.d/19-resolver.envsh`).
- **Networking → Generate Domain**, then **set the target port to 8080**. This is the one setting
  that is easy to get wrong: `8080` is the container port. The `3000` in
  `docker-compose.prod.yml` is a *host* port that does not exist on Railway, and pointing the
  domain at it makes Railway's edge return **502 Bad Gateway on every path**, including `/` —
  even though the nginx logs look completely healthy.
- Copy the generated domain into the `FRONTEND_URL` shared variable; the backend uses it for CORS
  and OAuth/OIDC callbacks.

## 6. Order and verification

Bring up Postgres and Redis, then `backend` (its start command runs the migrations — a
`pgvector is not installed` failure here means the wrong Postgres template), then `worker` and
`frontend`.

- `https://<domain>/` → the login screen.
- `https://<domain>/api/docs` → FastAPI's docs, proving the nginx proxy reaches the backend.

A 502 on `/` is an edge/target-port problem, not an nginx one: that location is a plain
`try_files` with no upstream and cannot itself emit a 502. A 502 only on `/api/` points at
`BACKEND_URL` or at the backend not listening on `::`.

## Optional: the agents MCP server

Only if `AGENTS_ENABLED=true`. A sixth service from the same repo, Root Directory `backend`:

```
uvicorn mcp_server.main:app --host :: --port 8765
```

Give it its own public domain (target port 8765) for external clients, and set
`AGENTS_BUILTIN_MCP_URL=http://mcp-server.railway.internal:8765/mcp` in the shared variables.
