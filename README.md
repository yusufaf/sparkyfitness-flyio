# SparkyFitness on Fly.io

Fly.io deployment config for [SparkyFitness](https://github.com/CodeWithCJ/SparkyFitness),
a self-hosted, privacy-first MyFitnessPal alternative.

It maps the project's 3-service `docker-compose.prod.yml` onto **three Fly apps** in one org,
connected over Fly's private network:

| Fly app | image | exposure |
|---|---|---|
| `sparkyfitness-db-af` | `postgres:18.3-alpine` | private (`.internal`), always-on |
| `sparkyfitness-server-af` | `codewithcj/sparkyfitness_server:latest` | private (`.flycast`), auto-stop |
| `sparkyfitness-frontend-af` | `codewithcj/sparkyfitness:latest` | **public** HTTPS, auto-stop |

Traffic: browser → **frontend** (nginx, public) → reverse-proxies `/api` etc. →
**server** (`.flycast:3010`) → **db** (`.internal:5432`). Only the frontend has a public IP.

## Prerequisites
- [`fly` CLI](https://fly.io/docs/flyctl/install/) installed and `fly auth login` done
- `openssl` (for generating secrets)

## Cost (rough, LA region)
db 256mb always-on ≈ $2.02/mo + 2×1GB volumes ≈ $0.30 + occasional server/frontend
machine time ≈ **~$3–5/mo** total. (shared-cpu-1x: 256mb $2.02, 512mb $3.32; volumes $0.15/GB/mo.)
New Fly accounts have **no free tier** — this is pay-as-you-go.

## Deploy

Generate secrets first and keep them handy (don't commit them):
```bash
DBPW=$(openssl rand -hex 16)
APPPW=$(openssl rand -hex 16)
APIKEY=$(openssl rand -hex 32)
AUTHSECRET=$(openssl rand -hex 32)
```

### 1. Database
```bash
fly apps create sparkyfitness-db-af
fly volumes create sparky_pgdata --app sparkyfitness-db-af --region lax --size 1
fly secrets set --app sparkyfitness-db-af \
  POSTGRES_DB=sparkyfitness_db POSTGRES_USER=sparky POSTGRES_PASSWORD="$DBPW"
fly deploy --config db/fly.toml
```

### 2. Backend server (private via flycast)
```bash
fly apps create sparkyfitness-server-af
# Allocate the private flycast address the frontend will dial:
fly ips allocate-v6 --private --app sparkyfitness-server-af
fly volumes create sparky_uploads --app sparkyfitness-server-af --region lax --size 1
fly secrets set --app sparkyfitness-server-af \
  SPARKY_FITNESS_DB_PASSWORD="$DBPW" \
  SPARKY_FITNESS_APP_DB_PASSWORD="$APPPW" \
  SPARKY_FITNESS_API_ENCRYPTION_KEY="$APIKEY" \
  BETTER_AUTH_SECRET="$AUTHSECRET"
fly deploy --config server/fly.toml
fly logs --app sparkyfitness-server-af   # watch: DB migrations run, "listening on 3010"
```

### 3. Frontend (public)
```bash
fly apps create sparkyfitness-frontend-af
fly deploy --config frontend/fly.toml
```

Open `https://sparkyfitness-frontend-af.fly.dev`.

> If you rename apps, update the hostnames in `server/fly.toml`
> (`SPARKY_FITNESS_FRONTEND_URL`, `SPARKY_FITNESS_DB_HOST`) and
> `frontend/fly.toml` (`SPARKY_FITNESS_SERVER_HOST`, `SPARKY_FITNESS_FRONTEND_URL`).

## First login
Signups are disabled by default. To create your account:
```bash
fly secrets set --app sparkyfitness-server-af SPARKY_FITNESS_DISABLE_SIGNUP=false
# (also set it to 'false' in server/fly.toml temporarily, or it reverts on next deploy)
# register at the site, then re-lock:
fly secrets set --app sparkyfitness-server-af SPARKY_FITNESS_DISABLE_SIGNUP=true
```
Optional admin: set `SPARKY_FITNESS_ADMIN_EMAIL` (secret) to your email to auto-grant admin.

## Verify
1. `fly status` on each app — db + server up, frontend may be stopped until first hit.
2. `fly logs --app sparkyfitness-server-af` — migrations applied, no DB connection errors.
3. Browser loads the UI; register; log a food entry; reload → it persists.
4. `fly ssh console --app sparkyfitness-db-af` → `psql -U sparky sparkyfitness_db -c '\dt'` shows tables.

## Ongoing
```bash
fly logs --app <app>            # tail logs
fly status --app <app>          # machine status
fly ssh console --app <app>     # shell in
fly deploy --config <dir>/fly.toml   # redeploy after image/config change
```
Images are pinned to `:latest`; `fly deploy` re-pulls. Pin to a digest if you want reproducibility.

## Notes / gotchas
- **flycast**: the backend has no public IP. The `fly ips allocate-v6 --private` step is required
  or nginx can't resolve `sparkyfitness-server-af.flycast`. The proxy auto-starts the stopped server
  on the first request (a few seconds of cold start).
- **`.internal` vs `.flycast`**: db uses `.internal` (postgres binds dual-stack, fine over IPv6);
  the Node server uses `.flycast` to avoid IPv4-only-bind issues over 6PN.
- **BETTER_AUTH_SECRET** must never change once users enable 2FA.
- Garmin sync + standalone MCP services are intentionally omitted; MCP is available in-process at
  `/mcp` on the server if you want it later.
