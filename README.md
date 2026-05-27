# langfuse-selfhost

One-command self-host for [Langfuse v3](https://langfuse.com) — an open-source LLM
observability platform. Replaces paid LangSmith / LangFuse Cloud with a stack
you control end-to-end.

## What you get

A `docker compose` stack with everything Langfuse needs:

- `langfuse-web` — the Next.js UI + ingestion API
- `langfuse-worker` — background queue processor
- `postgres` — application database
- `clickhouse` — trace storage
- `redis` — queues + cache
- `minio` — S3-compatible blob storage for events & media

All services bind to `127.0.0.1` only; pair this with your reverse proxy of
choice (Tailscale Serve, Caddy, Nginx, Cloudflare Tunnel) to expose it.

## Quick start

```bash
git clone https://github.com/aryanranderiya/langfuse-selfhost
cd langfuse-selfhost
./setup.sh https://langfuse.example.com
```

`setup.sh` will:

1. Generate strong secrets for every component
2. Write them to `.env` (chmod 600, gitignored)
3. Pre-create an admin user, organization, project, and Langfuse API keys
4. Pull images and bring the stack up

After ~30 seconds the API will be up:

```bash
curl -fsS http://127.0.0.1:3000/api/public/health
# {"status":"OK","version":"3.x.x"}
```

The script prints the admin password and Langfuse API keys — save them
somewhere safe.

## Reverse proxy examples

### Tailscale Serve (private)

Expose the UI inside your tailnet over HTTPS with no public ingress:

```bash
sudo tailscale serve --bg --https=8443 http://127.0.0.1:3000
# https://<your-host>.<tailnet>.ts.net:8443
```

### Caddy (public HTTPS)

```caddy
langfuse.example.com {
    reverse_proxy 127.0.0.1:3000
}
```

### Cloudflare Tunnel

```yaml
ingress:
  - hostname: langfuse.example.com
    service: http://127.0.0.1:3000
```

## Pointing an SDK at this instance

```python
from langfuse import Langfuse
from langfuse.langchain import CallbackHandler

lf = Langfuse(
    public_key="pk-lf-...",
    secret_key="sk-lf-...",
    host="https://langfuse.example.com",
    environment="production",  # tag traces; filter in the UI
)
handler = CallbackHandler()  # use in LangChain / LangGraph callbacks
```

Per-environment segregation in a single project is built in — pass
`environment="development"` from your dev process, `"staging"` from
staging, `"production"` from prod. The Langfuse UI exposes an
environment filter on every dashboard and the traces list. Promoting to
fully separate Langfuse projects per environment later is also a drop-in
change (create the projects in the UI, swap API keys per env).

## Verifying it works

### 1. Health check

```bash
curl -fsS https://your-langfuse-url/api/public/health
# {"status":"OK","version":"3.x.x"}
```

### 2. Confirm the bootstrap created the org / project / API keys

```bash
curl -fsS \
  -u "pk-lf-...:sk-lf-..." \
  "https://your-langfuse-url/api/public/projects"
# {"data":[{"id":"default","name":"default","organization":{...}}]}
```

### 3. Send a trace end-to-end

This exercises the full path — auth → ingestion → ClickHouse → trace
API — without writing any application code:

```bash
uv run --with langfuse --with langchain-openai python - <<'PY'
import os
from langfuse import Langfuse
from langfuse.langchain import CallbackHandler
from langchain_openai import ChatOpenAI

os.environ["LANGFUSE_HOST"] = "https://your-langfuse-url"
os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-lf-..."
os.environ["LANGFUSE_SECRET_KEY"] = "sk-lf-..."

lf = Langfuse(environment="development")
ChatOpenAI(model="gpt-4o-mini", callbacks=[CallbackHandler()], max_tokens=20)\
    .invoke("In exactly 5 words, say hi")
lf.flush()
print("trace flushed — check the Langfuse UI")
PY
```

Within a few seconds the trace shows up in the UI:

- Open `https://your-langfuse-url`, log in, pick your project
- **Traces** in the sidebar — you'll see a `ChatOpenAI` row
- Click it to see prompt, response, tokens, and timing
- The **Environment** filter on the traces list confirms the
  `development` tag was applied

### 4. List recent traces via API

```bash
curl -fsS \
  -u "pk-lf-...:sk-lf-..." \
  "https://your-langfuse-url/api/public/traces?limit=3"
```

### 5. If a trace never lands

```bash
# stack healthy?
docker compose ps

# web container logs (most useful when something breaks)
docker compose logs --tail 50 langfuse-web

# worker logs (ingestion pipeline)
docker compose logs --tail 50 langfuse-worker
```

A trace landing in `/api/public/traces` but not appearing in the UI
usually means the UI is filtered to a different environment — clear
the env filter on the traces list.

## Common operations

```bash
# Stop the stack (keeps data)
docker compose down

# Stop and wipe (DANGEROUS)
docker compose down -v

# View logs
docker compose logs -f langfuse-web
docker compose logs -f langfuse-worker

# Restart after editing .env
docker compose up -d
```

## Hardware

Recommended minimum:

- 4 GB RAM (peak ~3 GB across all services at light load)
- 2 vCPU
- 20 GB free disk (ClickHouse + MinIO grow with trace volume)

## Backups

State lives in four Docker volumes:

- `postgres_data` — accounts, projects, configuration
- `clickhouse_data` — traces and observations
- `clickhouse_logs` — server logs
- `minio_data` — large event blobs and media

A simple snapshot script:

```bash
docker compose stop
docker run --rm -v langfuse-selfhost_postgres_data:/data -v $PWD/backups:/out alpine \
    tar czf /out/postgres-$(date +%F).tgz -C /data .
docker run --rm -v langfuse-selfhost_clickhouse_data:/data -v $PWD/backups:/out alpine \
    tar czf /out/clickhouse-$(date +%F).tgz -C /data .
docker compose up -d
```

## Upgrades

Both `langfuse-web` and `langfuse-worker` use the floating `:3` tag. Pull
and restart to upgrade:

```bash
docker compose pull
docker compose up -d
```

Migrations run automatically on container startup.

## Security notes

- Every host port is bound to `127.0.0.1`. Don't change this unless you
  understand the implications.
- `.env` contains every secret — keep `chmod 600` and never commit it.
- `AUTH_DISABLE_SIGNUP=true` by default — only the bootstrapped admin can
  log in until you invite users from the UI.
- `TELEMETRY_ENABLED=false` by default — Langfuse will not phone home.

## License

MIT.
