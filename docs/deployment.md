# Deployment: Docker, VPS, Nginx, and CI/CD

A single VPS per environment, with Docker Compose. Goal: reproducible deployment with a single command, rollback within minutes, and a **tested** backup.

---

## 1. Environments

| Environment | Hosting | Data | SMS | Purpose |
|---|---|---|---|---|
| local | Developer machine, `docker-compose.dev.yml` | Seed dev | `fake` | Development |
| test | CI, `docker-compose.test.yml` | Temporary | `fake` | Tests |
| staging | Small VPS (2 vCPU / 4GB) | Seed dev + test data | **Real provider** with team numbers only (allowlist) | Acceptance, k6, and real SMS |
| production | VPS (4 vCPU / 8GB / 80GB SSD) | Real | Primary + fallback provider | — |

The `SMS_ALLOWLIST` variable in staging (comma-separated numbers): the provider rejects any number outside it.

## 2. Server Setup (one time)

- Ubuntu 24.04 LTS, with system-wide `UTC` time (the application handles `Asia/Damascus`).
- A `deploy` user with SSH keys only: `PasswordAuthentication no`, and `PermitRootLogin no`.
- UFW: allow only 22, 80, and 443. Postgres, Redis, and MinIO are **not exposed externally** (internal Docker network only).
- `fail2ban` for SSH, and `unattended-upgrades` for security updates.
- Docker Engine + Compose plugin, with `json-file` logging capped at `max-size: 50m`, and `max-file: 5`.
- Directories:
```
/opt/halak/
├── .env                    (600, root)
├── docker-compose.yml      (from deploy/)
├── nginx/                  (from deploy/nginx)
├── certs/                  (Let's Encrypt)
└── backups/                (temporary before off-server upload)
```

## 3. Docker Compose (production)

`deploy/docker-compose.yml` (abbreviated, and the full file is in the repository):
```yaml
name: halak
x-app: &app
  image: ${REGISTRY}/halak-backend:${APP_VERSION}
  env_file: .env
  restart: unless-stopped
  networks: [internal]
  depends_on:
    postgres: { condition: service_healthy }
    redis: { condition: service_healthy }

services:
  api:
    <<: *app
    command: ["node", "dist/main.js"]
    healthcheck: { test: ["CMD", "wget", "-qO-", "http://localhost:3000/api/v1/health/ready"], interval: 10s, retries: 5 }
  worker:
    <<: *app
    command: ["node", "dist/worker.js"]
  web:
    image: ${REGISTRY}/halak-web:${APP_VERSION}
    env_file: .env
    restart: unless-stopped
    networks: [internal]
  postgres:
    image: postgres:16-alpine
    volumes: [pgdata:/var/lib/postgresql/data]
    environment: { POSTGRES_DB: halak, POSTGRES_USER: halak_owner, POSTGRES_PASSWORD: ${POSTGRES_PASSWORD} }
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U halak_owner"], interval: 5s, retries: 10 }
    networks: [internal]
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes", "--maxmemory", "512mb",
              "--maxmemory-policy", "noeviction", "--requirepass", "${REDIS_PASSWORD}"]
    volumes: [redisdata:/data]
    healthcheck: { test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"], interval: 5s }
    networks: [internal]
  minio:
    image: minio/minio:RELEASE.<pinned>
    command: ["server", "/data"]
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
      MINIO_KMS_SECRET_KEY: ${MINIO_KMS_SECRET_KEY}
    volumes: [miniodata:/data]
    networks: [internal]
  nginx:
    image: nginx:1.27-alpine
    ports: ["80:80", "443:443"]
    environment: { APP_DOMAIN: ${APP_DOMAIN} }
    # The official nginx image applies envsubst to /etc/nginx/templates/*.template → conf.d
    volumes: ["./nginx/templates:/etc/nginx/templates:ro", "./nginx/static:/etc/nginx/static:ro", "./certs:/etc/letsencrypt:ro"]
    depends_on: [api, web]
    networks: [internal]

networks: { internal: {} }
volumes: { pgdata: {}, redisdata: {}, miniodata: {} }
```
Mandatory points:
- **`maxmemory-policy noeviction`** in Redis, because BullMQ loses jobs if Redis deletes keys due to memory pressure.
- Two Postgres users: `halak_owner` for migrations, and `halak_app` for the application with DML-only privileges, without `UPDATE`/`DELETE` on `audit_logs` (database §3.20). `DATABASE_URL` for the application, and `MIGRATION_DATABASE_URL` for the migrate command.
- Images use pinned versions, and never `latest`.
- Images run as a non-root user (`USER node`), with a multi-stage build and `npm ci --omit=dev` in the final stage.

<a id="nginx"></a>
## 4. Nginx

```nginx
# Redact secrets from the access log (security §8)
map $request_uri $log_uri {
    ~^/t/[^/?]+(?<rest>.*)$              "/t/[REDACTED]$rest";
    ~^(?<p>/api/v1/admin/events)\?.*$    "$p?[REDACTED]";
    default                               $request_uri;
}
log_format halak '$remote_addr - [$time_iso8601] "$request_method $log_uri $server_protocol" '
                 '$status $body_bytes_sent $request_time "$http_user_agent" rid=$request_id';
access_log /var/log/nginx/access.log halak;

limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
limit_req_zone $binary_remote_addr zone=otp:10m rate=10r/m;

server {
    listen 443 ssl;
    http2 on;
    server_name ${APP_DOMAIN};
    ssl_protocols TLSv1.2 TLSv1.3;
    client_max_body_size 10m;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Request-Id $request_id always;

    location /api/v1/auth/otp/ { limit_req zone=otp burst=5 nodelay; proxy_pass http://api:3000; include proxy.conf; }
    location /api/v1/admin/events {
        proxy_pass http://api:3000; include proxy.conf;
        proxy_buffering off; proxy_cache off; proxy_read_timeout 1h;
    }
    location /api/ { limit_req zone=api burst=30 nodelay; proxy_pass http://api:3000; include proxy.conf; }

    location /.well-known/assetlinks.json            { alias /etc/nginx/static/assetlinks.json; }
    location /.well-known/apple-app-site-association { alias /etc/nginx/static/aasa.json; default_type application/json; }

    location /t/ {
        add_header Referrer-Policy no-referrer always;
        add_header X-Robots-Tag "noindex, nofollow" always;
        proxy_pass http://web:3000; include proxy.conf;
    }
    location / { proxy_pass http://web:3000; include proxy.conf; }
}
server { listen 80; server_name ${APP_DOMAIN}; location /.well-known/acme-challenge/ { root /var/www/certbot; } location / { return 301 https://$host$request_uri; } }
```
`proxy.conf` forwards: `X-Forwarded-For $proxy_add_x_forwarded_for`, `X-Forwarded-Proto`, `X-Request-Id $request_id`, and `Host`. The Backend trusts only one proxy (`app.set('trust proxy', 1)`).

**The error log** may contain the full URI on errors. Therefore: `error_log ... warn;` with periodic review, and it is not shipped off the server.

**TLS:** certbot on the host with webroot, with `--deploy-hook "docker compose -f /opt/halak/docker-compose.yml exec nginx nginx -s reload"`.

## 5. CI/CD

### 5.1 Workflows (`.github/workflows/`)

| File | Trigger | Steps |
|---|---|---|
| `ci.yml` | Every PR and push | Path filters (`backend/**`, `web/**`, `mobile/**`, `docs/**`). Backend: lint, typecheck, unit, and `compose.test` + migrate + `check-constraints` + integration/e2e, coverage, and `openapi.json` match. Web: lint, typecheck, vitest, build, and Playwright (against a containerized backend). Mobile: `flutter analyze`, and `flutter test`. All: `gitleaks` |
| `deploy-staging.yml` | Push to `main` | Build images (tag = `sha`), push to the registry, SSH to staging, then `deploy.sh <sha>` |
| `deploy-prod.yml` | Tag `v*.*.*` + manual approval (GitHub Environment) | Deploy the same images tested in staging (**no rebuild**), then `deploy.sh <tag>` |
| `mobile-release.yml` | Tag `mobile-v*` | `flutter build appbundle` and `apk --split-per-abi`, signing from secrets, then upload the artifacts (Play internal track or direct distribution, see Q-05) |

### 5.2 `deploy/scripts/deploy.sh`
```bash
set -euo pipefail
VERSION="$1"; cd /opt/halak
set -a; source ./.env; set +a                                      # MIGRATION_DATABASE_URL, etc.
export APP_VERSION="$VERSION"
PREV=$(cat .current_version || echo "")

docker compose pull api worker web
./scripts/backup-db.sh pre-deploy-"$VERSION"                        # backup before the migration
docker compose run --rm -e DATABASE_URL="$MIGRATION_DATABASE_URL" api npx prisma migrate deploy
docker compose run --rm api node dist/cli.js db:check-constraints   # fails if constraints are missing
docker compose up -d --no-deps api worker web
./scripts/wait-healthy.sh api 60
./scripts/smoke.sh || { echo "smoke failed → rollback"; APP_VERSION="$PREV" docker compose up -d --no-deps api worker web; exit 1; }
echo "$VERSION" > .current_version
```
- **Migrations must be compatible with the previous version** (expand/contract, database §5), because rollback does not reverse the migration.
- `smoke.sh`: check `/health/ready`, `GET /api/v1/barbers`, `GET /` with status 200, and `GET /admin` with status 200.
- During deployment, the worker stops for a few seconds. This is acceptable, because the Sweeper catches up on the next cycle.

## 6. Backup and Restore (NFR-AVL)

| What | How | When | Retention |
|---|---|---|---|
| PostgreSQL | `pg_dump -Fc`, then `age` encryption, then upload to external storage (S3-compatible) | Daily at 02:30 Damascus time, and before every deployment | 30 daily + 12 monthly |
| MinIO | `mc mirror --overwrite` to an external bucket | Daily at 03:00 | 30 days |
| `.env` | An `age`-encrypted copy in a location separate from the other backups | On every change | — |
| Redis | No backup (its data is reconstructible) | — | — |

- The private key for decryption is **not on the server**, but with the owner and in a password vault.
- **Restore** (`deploy/scripts/restore.sh <date>`): stop `api` and `worker`, then `pg_restore --clean --if-exists`, then `prisma migrate deploy`, then `check-constraints`, then a reverse `mc mirror`, then start, then smoke.
- **A monthly restore test** on a temporary server, with the result and duration recorded (target RTO ≤ 4 hours).

## 7. Monitoring and Alerts

| What | Tool | Alert |
|---|---|---|
| Availability | Uptime Kuma on a **separate machine** checking `/api/v1/health/ready` and `/` every minute | 3 consecutive failures |
| Errors | GlitchTip or Sentry (A-05), with `beforeSend` redaction | New error, or > 10 in 5 minutes |
| Disk | A cron script | > 80% |
| Backups | The script sends a heartbeat to Uptime Kuma on success | Missing heartbeat for 26 hours |
| SMS | `sms_logs`: `failed` ratio over an hour | > 10%, or daily budget > 80% |
| The Sweeper | The last successful run is stored in the database and exposed at `/health/ready` | > 5 minutes |
| Pending requests | `awaiting_approval` bookings older than `admin_approval_reminder_minutes` | Repeated reminder in the admin dashboard (FR-BKM-09) |

## 8. Runbooks

- **Rollback:** `./deploy.sh <previous-tag>`. If the fault is in a migration: restore `pre-deploy-<version>` (loses whatever was written after deployment, and therefore forward-fixing is preferred).
- **Switching SMS provider:** change `SMS_PRIMARY` in `.env`, then `docker compose up -d worker`.
- **Redis loss:** `docker compose up -d redis`. The Sweeper is unaffected because it does not depend on Redis (D-20). Queues resume when the connection returns, and lost scheduled reminders are not restored (acceptable), and Idempotency keys and rate counters start from zero.
- **Rotating a secret:** see [security.md §10](security.md).
- **Incident of leaked tracking tokens:** `node dist/cli.js tracking:rotate-all --active-only`, then SMS with the new links for all active bookings.

<a id="availability"></a>
## 9. Service Availability (Q-05)

Before committing to any provider, **you must verify** its acceptance of an account, owner, or payment linked to Syria under its current terms and applicable sanctions regulations, because the situation has changed during 2025–2026 and may change again. We do not assume that what was previously forbidden or allowed is still so.

| Need | First option | Alternative if unavailable |
|---|---|---|
| VPS | Hetzner or Contabo | A regional or local provider, with the same Compose without change |
| Git and CI | GitHub + Actions | Self-hosted GitLab, or Gitea + Woodpecker on the same server |
| Registry | GHCR | A self-hosted registry (`registry:2`) behind Nginx with Basic auth |
| Error tracking | Sentry | Self-hosted GlitchTip |
| Android distribution | Google Play | A signed APK downloadable from the site with an update-check mechanism |
| iOS | App Store | Defer iOS, and the website covers the users |
| External backup | Backblaze B2 or S3 | A second VPS at a different provider with MinIO |

The architecture above does not depend on any managed cloud service, and therefore everything above works with the alternatives without any code changes.