# Lesson 02 — Installation & n8n Architecture

## Goal
Get n8n running locally and understand what's actually happening under the hood —
the server, the database, the queue, the execution model.

---

## How n8n Runs — The Architecture

Before installing, understand what you're running. n8n is a Node.js application
with the following components:

```
┌─────────────────────────────────────────────────────────────┐
│                        n8n Process                           │
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐  │
│  │   Web UI      │   │  REST API    │   │   Webhook      │  │
│  │  (Vue.js SPA) │   │  (Express)   │   │   Listener     │  │
│  └──────┬───────┘   └──────┬───────┘   └───────┬────────┘  │
│         │                  │                    │            │
│  ┌──────▼──────────────────▼────────────────────▼────────┐  │
│  │                   Workflow Engine                       │  │
│  │   (receives triggers, executes nodes, manages data)     │  │
│  └──────────────────────┬──────────────────────────────┘  │  │
│                         │                                   │
│  ┌──────────────────────▼──────────────────────────────┐  │
│  │                   Database (SQLite / Postgres)        │  │
│  │  workflows • credentials • executions • users        │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Key components:

| Component | What it does |
|-----------|-------------|
| **Web UI** | The visual editor you use in the browser |
| **REST API** | Internal API the UI calls; you can also call it directly |
| **Webhook Listener** | Listens on a dedicated port for incoming webhooks |
| **Workflow Engine** | Executes workflows: manages the queue, runs nodes |
| **Database** | Stores everything: workflows, credentials (encrypted), execution logs |

### Two execution modes:

**Main mode (default — what we'll use)**
```
Single process handles everything: webhook listening, UI, workflow execution.
Fine for development and light production use.
```

**Queue mode (production scale)**
```
Main process: handles UI, API, webhook intake
Worker processes: execute workflows (can scale horizontally)
Redis: message queue between main and workers
```

We start in main mode. Queue mode is Lesson 17.

---

## The Database

n8n defaults to **SQLite** — a local file. For production, use **PostgreSQL**.

What gets stored:
- `Workflow` — the JSON definition of each workflow
- `Credentials` — API keys / OAuth tokens (AES-256 encrypted)
- `Execution` — every run: start time, status, input/output data per node
- `User` — accounts if you enable user management

The execution data is the most valuable thing for debugging. n8n records
**exactly** what data each node received and output. This is your best friend.

---

## Installation — Three Ways

### Way 1: npx (fastest, good for trying it out)

```bash
npx n8n
```

Downloads and runs n8n without installing. Data stored in `~/.n8n/`.
Not recommended for serious use — slow startup, no persistence control.

### Way 2: Global npm install (recommended for local dev)

```bash
npm install -g n8n
n8n start
```

Installs n8n globally. Starts the server. UI at `http://localhost:5678`.
Data stored in `~/.n8n/` by default.

### Way 3: Docker (recommended for production / reproducible env)

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

- `-p 5678:5678` — expose the UI/API port
- `-v ~/.n8n:/home/node/.n8n` — persist data outside the container

---

## Hands-On: Install & Start n8n

We'll use the npm global install (cleanest for dev work).

### Step 1 — Install n8n

```bash
npm install -g n8n
```

This takes 1–2 minutes. n8n has many dependencies (400+ integrations).

Verify:
```bash
n8n --version
# Should output something like: 1.x.x
```

### Step 2 — Start n8n

```bash
n8n start
```

You'll see output like:
```
n8n ready on 0.0.0.0, port 5678
Version: 1.x.x
```

### Step 3 — Open the UI

Navigate to: **http://localhost:5678**

On first launch you'll see the setup wizard. Create an owner account:
- Email: use any email (stored locally)
- Password: choose anything

You're now in the n8n editor.

---

## Anatomy of the n8n UI

Let me orient you to the interface before writing any workflow:

```
┌───────────────────────────────────────────────────────────┐
│  n8n                    [Executions] [Templates] [...]     │
├──────────────────────┬────────────────────────────────────┤
│                      │                                     │
│   Workflow List      │         Canvas (Editor)             │
│   (left sidebar)     │                                     │
│                      │    ●──────●──────●──────●          │
│   My Workflows       │    (nodes connected by edges)       │
│   ├── Slack Bot      │                                     │
│   ├── Lead Sync      │                                     │
│   └── ...            │                                     │
│                      │                                     │
├──────────────────────┴────────────────────────────────────┤
│  [+] Add node   [Execute Workflow]   [Save]                │
└───────────────────────────────────────────────────────────┘
```

### Key areas:

| Area | Purpose |
|------|---------|
| **Left sidebar** | List of all your workflows |
| **Canvas** | Where you build workflows — drag, connect, configure nodes |
| **Top bar** | Executions history, Templates marketplace, Settings |
| **Bottom bar** | Add nodes, execute, save, zoom |

### The Executions tab (critical for debugging)

Every time a workflow runs, n8n records:
- Timestamp
- Status (success / error)
- Duration
- For each node: input data, output data, error message

This is how you debug. You will use this constantly.

---

## What Gets Stored in ~/.n8n

```bash
ls -la ~/.n8n/
```

```
~/.n8n/
├── config          # n8n settings (encryption key, DB path, etc.)
├── database.sqlite # Your workflows, credentials, executions
└── nodes/          # Any custom nodes you install
```

**The encryption key** in `~/.n8n/config` is critical. It encrypts all your
stored credentials (API keys, OAuth tokens). If you lose this file, you can't
decrypt your credentials. Back it up.

```bash
cat ~/.n8n/config
# Shows: { "encryptionKey": "...", "userManagement": {...}, ... }
```

---

## Environment Variables You Should Know

n8n is configured via environment variables. The most important ones:

```bash
# Database (default: SQLite)
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=localhost
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=secret

# Where to store data
N8N_USER_FOLDER=/path/to/n8n/data

# URL (important for webhooks to work correctly)
N8N_HOST=0.0.0.0
N8N_PORT=5678
WEBHOOK_URL=https://your-domain.com/

# Timezone (use your local timezone)
GENERIC_TIMEZONE=America/New_York   # US East
GENERIC_TIMEZONE=Asia/Kolkata       # India (IST, UTC+05:30)
GENERIC_TIMEZONE=Europe/London      # UK

# Execution data retention (default: save all)
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=72  # hours
```

For local dev, the defaults are fine. These matter for production (Lesson 17).

---

## The Execution Lifecycle — What Happens When You Run a Workflow

This is the mental model for debugging any issue:

```
1. TRIGGER fires
   (webhook arrives / cron fires / manual click)
          │
          ▼
2. n8n loads the workflow definition from DB
          │
          ▼
3. Engine starts with the trigger node
   → trigger node outputs: [{ "item": 1 }, { "item": 2 }, ...]
          │
          ▼
4. Each connected node receives the items from the previous node
   → processes them
   → outputs new items
          │
          ▼
5. This continues until the last node
          │
          ▼
6. Execution record saved to DB
   (status, duration, per-node input/output data)
```

**Key insight:** Step 4 is the entire n8n data model. Every node gets items in,
does something, puts items out. Nothing more.

---

## Hands-On Exercise: Explore the UI

1. Open **http://localhost:5678**
2. Click **"+ New workflow"**
3. You'll see an empty canvas with a **Manual Trigger** node already placed
4. Click the node — observe the right panel that appears (node configuration)
5. Click **"Execute workflow"** — the workflow runs (just the trigger, outputs empty)
6. Click **"Executions"** in the top bar — see the execution record

You haven't built anything yet, but you've run your first workflow and seen
where execution history lives. That's the foundation.

---

## Common Setup Issues & Fixes

### Issue 1 — Secure Cookie Error on localhost

**Error message:**
```
Your n8n server is configured to use a secure cookie, however you are either
visiting this via an insecure URL, or using Safari.
```

**Why it happens:** n8n's secure cookie requires HTTPS. Local dev runs on plain
HTTP, so the browser rejects the cookie. This also happens in Safari even on localhost.

**Fix — stop n8n, restart with the env variable:**
```bash
pkill -f n8n
GENERIC_TIMEZONE=Asia/Kolkata N8N_SECURE_COOKIE=false n8n start
```

Then open `http://localhost:5678` — the error will be gone.

This is safe for local development. In production (with HTTPS), the warning
disappears automatically and you don't need this flag.

---

### Issue 2 — Forgot Login Password

**Fix — reset the owner account without deleting your data:**
```bash
pkill -f n8n
n8n user-management:reset
N8N_SECURE_COOKIE=false n8n start
```

Open `http://localhost:5678` — the setup wizard appears again so you can set
a new email and password. **Your workflows, credentials, and execution history
are not deleted.** Only the login is reset.

---

### Issue 3 — Port 5678 Already in Use

```bash
# Find what's using the port
lsof -i :5678

# Kill it
pkill -f n8n

# Or start n8n on a different port
N8N_PORT=5679 N8N_SECURE_COOKIE=false n8n start
# Open: http://localhost:5679
```

---

### Issue 4 — n8n Command Not Found After Install

```bash
# Fix npm permissions first
sudo chown -R $(whoami) ~/.npm

# Reinstall
npm install -g n8n

# Verify
n8n --version
```

---

## Summary

- n8n is a Node.js app: Web UI + REST API + Webhook Listener + Workflow Engine + Database
- Default database: SQLite at `~/.n8n/database.sqlite`
- Credentials are AES-256 encrypted — back up `~/.n8n/config`
- Install: `npm install -g n8n` → `n8n start` → `http://localhost:5678`
- Every execution is recorded: timestamp, status, per-node input/output data
- The execution lifecycle: Trigger → Load workflow → Run nodes → Save record

---

## Check Your Understanding — Q&A

### Q1. Where does n8n store workflows and credentials?

**Answer:** In a database — SQLite by default (at `~/.n8n/database.sqlite`).
For production, switch to PostgreSQL via `DB_TYPE=postgresdb`. Credentials are
stored **encrypted** using an AES-256 key stored in `~/.n8n/config`.

---

### Q2. What is "execution data" and why does it matter?

**Answer:** Every time a workflow runs, n8n saves a complete record: timestamp,
success/failure status, duration, and for every node — the exact input data it
received and the exact output data it produced. This is your primary debugging
tool. When something fails, you open the execution, click the failed node, and
see exactly what data it received and what error it threw.

---

### Q3. What is the difference between n8n's main mode and queue mode?

**Answer:** Main mode runs everything in a single process (UI + webhook listener
+ workflow executor) — fine for development and light production. Queue mode
splits this: a main process handles the UI and webhook intake, while separate
worker processes execute workflows. Workers are scaled horizontally and
communicate via Redis. Use queue mode when you need concurrent execution of
many workflows.

---

## Next Lesson

**[Lesson 03 →](03-your-first-workflow.md)** — Build your first real workflow
from scratch: a scheduled HTTP request that transforms data and sends a Slack
message.
