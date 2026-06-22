# Lesson 17 — Self-Hosting and Production

## Goal
Deploy n8n as a reliable production service with persistent storage, secure external access, queue mode for scale, and a backup / monitoring strategy. This lesson turns local automation experiments into a hardened production deployment.

---

## Why Production-Grade n8n Matters
Local n8n is great for learning, but real automation needs:
- durable persistence beyond a browser session
- safe credential storage and secrets handling
- external webhook reachability with HTTPS
- retryable, concurrent executions for load
- backups and observability for operations

This lesson covers the deployment patterns you need to run n8n in a team or business environment.

---

## What We’re Building

```
[Docker Compose] ──▶ [n8n app] ──▶ [Postgres database]
                        │
                        ├──▶ [Redis queue + worker] (optional scale)
                        │
                        └──▶ [nginx / Caddy reverse proxy + HTTPS]
```

The result is a production deployment that can receive webhooks, store workflows and credentials, and scale safely.

---

## Step 1 — Choose Your Database

### SQLite: Good for learning, not production
- Embedded database stored in a local file
- Easy to start, no external service required
- Not recommended for multi-user or high-concurrency deployments

### PostgreSQL: Best practice for production
- Durable, transaction-safe storage
- Supports concurrent workflow executions
- Required for enterprise-grade deployments and queue mode
- Recommended when you want predictable behavior and backups

### MySQL / MariaDB: supported, but Postgres is preferred
If you already run MySQL, n8n supports it. Postgres is typically simpler for modern deployment patterns.

---

## Step 2 — Docker Compose Architecture

Create a `docker-compose.yml` for n8n + Postgres + Redis + proxy.

### Minimal production compose structure

- `n8n`: the workflow engine
- `db`: PostgreSQL for data persistence
- `redis`: queue backend for workers
- `worker`: optional background worker service
- `proxy`: nginx or Caddy for HTTPS / host routing

This gives a clean separation of responsibilities.

---

## Step 3 — Environment Variables

Configure n8n via env vars, not CLI flags. Store secrets securely.

Example env file:

```env
# n8n core
N8N_HOST=0.0.0.0
N8N_PORT=5678
N8N_PROTOCOL=https
WEBHOOK_TUNNEL_URL=https://automation.example.com

# database
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=db
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=supersecret

# credentials encryption
N8N_ENCRYPTION_KEY=replace-with-32-char-random

# queue mode
EXECUTIONS_PROCESS=main
EXECUTIONS_DATA_SAVE_ON_PROGRESS=true
EXECUTIONS_DATA_SAVE_ON_ERROR=true
EXECUTIONS_DATA_SNAPSHOT_ON_ERROR=true

# optional, for larger workloads
QUEUE_BULL_REDIS_HOST=redis
QUEUE_BULL_REDIS_PORT=6379

# trust proxy when behind reverse proxy
TRUST_PROXY=true

# email / telemetry
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=change-me
```

Important values:
- `N8N_ENCRYPTION_KEY`: must be 32 characters. Protect it.
- `WEBHOOK_TUNNEL_URL`: external URL clients use for webhooks.
- `DB_TYPE`: set to `postgresdb` in production.

---

## Step 4 — Reverse Proxy and HTTPS

A reverse proxy does three things:
1. terminates TLS
2. exposes a stable public hostname
3. forwards requests to n8n on localhost

### Option A: nginx

Use nginx if you want explicit control over routing.

Sample `nginx.conf`:

```nginx
server {
  listen 80;
  server_name automation.example.com;

  location / {
    proxy_pass http://n8n:5678;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;
  }
}
```

Use Certbot or Let’s Encrypt to add TLS. When TLS is managed by nginx, n8n should still run on HTTP internally and trust the proxy.

### Option B: Caddy

Caddy is easier for automatic HTTPS.

Sample `Caddyfile`:

```
automation.example.com {
  reverse_proxy n8n:5678
}
```

Caddy automatically obtains certificates and renews them.

---

## Step 5 — External Webhook Configuration

In production, n8n needs a public URL for webhooks. Set these env vars correctly:

```env
N8N_PROTOCOL=https
N8N_HOST=automation.example.com
N8N_PORT=443
WEBHOOK_TUNNEL_URL=https://automation.example.com
```

Also set:
- `WEBHOOK_URL` if you want a custom URL outside the default structure
- `N8N_PORT=5678` if you only expose TLS at the proxy layer, not n8n itself

If you use a proxy, enable `TRUST_PROXY=true` so n8n reads the original scheme.

---

## Step 6 — Queue Mode and Workers

For production, use queue mode if you expect parallel executions, long-running workflows, or many webhooks.

### How queue mode works
- `n8n` handles incoming requests and workflow scheduling
- `redis` stores execution jobs
- `worker` processes executions asynchronously

### Enable queue mode
In the main app service:

```env
EXECUTIONS_PROCESS=main
QUEUE_MODE=redis
```

In the worker service:

```env
EXECUTIONS_PROCESS=worker
```

Use `QUEUE_BULL_REDIS_HOST` and `QUEUE_BULL_REDIS_PORT` to connect workers to Redis.

### Why queue mode matters
- avoids blocking the main web server
- improves throughput under load
- makes long-running workflows safer
- enables horizontal scaling with multiple worker replicas

---

## Step 7 — Backup Strategy

A production system needs backups for:
- workflow definitions
- credentials
- execution history
- database state

### Database backups
For PostgreSQL, take regular dumps:

```sh
pg_dump -U n8n -h db n8n > n8n-backup-$(date +%F).sql
```

Schedule daily backups and retain at least 7 days. Store them off-host.

### Credentials backup
n8n stores credentials encrypted in the database. Backup the database and keep `N8N_ENCRYPTION_KEY` safe. Without the key, credentials cannot be decrypted.

### Workflow export
Periodically export important workflows as JSON from the n8n UI. This is a good secondary backup for business-critical automations.

---

## Step 8 — Monitoring and Health Checks

Make your deployment observable.

### Basic health check
Add a simple endpoint in your reverse proxy or container orchestration that verifies the n8n port is reachable.

### n8n metrics
n8n exposes basic runtime logs and process status. Use container logs to watch for:
- reboot/restart loops
- credential decryption errors
- webhook delivery failures
- queue worker crash loops

### External monitoring
Use uptime monitoring for the public URL and alert on:
- response time above expected SLA
- 5xx errors
- DNS or TLS failures

---

## Step 9 — Security Recommendations

- Run n8n behind HTTPS only
- Use strong authentication for the UI
  - `N8N_BASIC_AUTH_ACTIVE=true`
  - `N8N_BASIC_AUTH_USER` / `N8N_BASIC_AUTH_PASSWORD`
- Limit webhook access with firewall rules or IP allowlists if possible
- Keep secrets out of source control; use environment variables or a secret manager
- Rotate `N8N_ENCRYPTION_KEY` only with care — it breaks credential decryption if changed incorrectly

---

## Step 10 — Deployment Checklist

Use this checklist before declaring your deployment production-ready:

1. `DB_TYPE=postgresdb` configured
2. `N8N_ENCRYPTION_KEY` set and protected
3. `WEBHOOK_TUNNEL_URL` points to your public hostname
4. Reverse proxy is terminating TLS
5. `TRUST_PROXY=true` enabled
6. Queue mode enabled if workload is moderate or high
7. Backups are scheduled and tested
8. Monitoring and alerting are configured
9. Basic auth or another auth layer is active
10. Workflow export / recovery process documented

---

## Troubleshooting

- `Could not connect to database` → check host, port, username, password
- `Credentials cannot be decrypted` → verify `N8N_ENCRYPTION_KEY` is identical across restarts
- `Webhook registration failed` → confirm `WEBHOOK_TUNNEL_URL` and `N8N_PROTOCOL` are correct
- `Worker not processing jobs` → check Redis connection and `EXECUTIONS_PROCESS=worker`
- `504 or 502 from proxy` → verify proxy forwards to n8n and the app is healthy

---

## Next Lesson

See [README](../README.md) for full curriculum.
