# Lesson 01 — The Problem n8n Solves & Core Mental Model

## Goal
Understand *why* n8n exists, what problem it solves, and build the mental model
that the top 1% of practitioners use before writing a single workflow.

---

## The Pain Point

You've been here before as an engineer:

```
Tool A produces data.
Tool B needs that data.
You write a script to move it.
The script breaks at 3am.
Nobody knows.
Data is lost or duplicated.
```

You're writing glue code. Not business logic — glue. Moving data from Slack to
a spreadsheet. Posting a Notion entry when someone fills a Typeform. Sending a
welcome email when a new Stripe customer is created.

This is **not** what engineers should be spending time on. But it's also not
trivial — it requires:

- Handling API auth (OAuth, API keys, tokens)
- Rate limiting and retries
- Parsing and transforming JSON
- Error alerting
- Scheduling and triggering
- Idempotency (don't process the same event twice)
- Logging and auditability

Writing this from scratch every time is expensive. That's the problem.

---

## The Traditional Solutions & Why They Fall Short

### 1. Custom Python/Node scripts

```python
# The classic: a cron job + requests
import requests, time

def sync_data():
    data = requests.get("https://api.source.com/items").json()
    for item in data:
        requests.post("https://api.dest.com/items", json=item)

while True:
    sync_data()
    time.sleep(3600)
```

Problems:
- No UI — you can't hand this to a non-engineer
- No retry logic (unless you build it)
- No error visibility — it fails silently
- No audit trail
- Hard to modify without breaking things
- Maintenance burden grows with each new integration

### 2. Zapier / Make (formerly Integromat)

These solve the visibility and no-code problems, but:

| Problem | Zapier/Make | n8n |
|---------|-------------|-----|
| Complex logic (loops, branches, transforms) | Limited | Full JavaScript |
| Self-hosting / data privacy | ❌ Always cloud | ✅ Can self-host |
| Cost at scale | Per-task pricing (expensive) | Free self-hosted |
| Custom code | Very limited | Full Code Node |
| Debugging | Black box | Full execution history |
| API flexibility | Connector-limited | Any HTTP API |

### 3. Apache Airflow / Prefect

Built for data pipelines and ETL — very powerful, but:
- Requires Python, DAG code, infrastructure
- Overkill for operational automations
- Not designed for event-driven workflows
- No built-in UI for business users to operate

---

## What n8n Is

n8n (pronounced "n-eight-n", standing for "nodemation") is an **open-source
workflow automation platform** that sits at the intersection:

```
                    ┌─────────────────────────────────┐
                    │          n8n lives here          │
                    │                                  │
  Engineer power ──▶│  Visual + Code + Infrastructure  │◀── Non-engineer accessible
                    │                                  │
                    │  Self-hostable + Cloud option    │
                    └─────────────────────────────────┘

         Zapier/Make ◀─────────────────────────────────▶ Airflow
         (no-code,                                        (full code,
          cloud-only)                                      infra-heavy)
```

It gives you:
- A **visual workflow editor** (nodes connected by edges)
- **400+ built-in integrations** (Slack, GitHub, Postgres, Stripe, OpenAI...)
- A **Code node** for arbitrary JavaScript or Python
- **Self-hosting** — your data never leaves your infra
- **Webhooks, cron triggers, manual triggers** — any execution model
- **Full execution history** — see exactly what ran, when, with what data
- **Error handling** — retry logic, error workflows, alerting

---

## The Core Mental Model — Workflows as Pipelines

This is the most important concept in n8n. Internalize it now.

```
TRIGGER ──▶ NODE ──▶ NODE ──▶ NODE ──▶ DESTINATION
```

Every n8n workflow is a **pipeline**:
1. Something **triggers** the workflow (a webhook, a schedule, a manual click)
2. Data flows through a series of **nodes**
3. Each node **receives** data, **transforms** it, and **outputs** new data
4. Data travels as a list of **items** (JSON objects)

The critical insight:

> **Data in n8n is always a list of items. Every node receives items and
> outputs items. Nothing else.**

This sounds simple. It is the source of 80% of confusion for beginners and the
key insight of experts. We will spend an entire lesson on the data model.

---

## What a Node Is

A node is a single unit of work. Every node does exactly one thing:

```
┌─────────────────────────────────┐
│            A NODE                │
│                                  │
│  Input items  ──▶  Logic  ──▶  Output items  │
│                                  │
│  Config (credentials, params)    │
└─────────────────────────────────┘
```

Types of nodes:
- **Trigger nodes** — start the workflow (Webhook, Cron, Email trigger)
- **Action nodes** — do something (HTTP Request, Slack, Postgres, Gmail)
- **Transform nodes** — reshape data (Set, Code, Function)
- **Flow control nodes** — branch, merge, loop (IF, Switch, Merge, SplitInBatches)

---

## A Real Example to Make It Concrete

**Business problem:** Every time someone fills our Typeform signup form, we
want to:
1. Look them up in Clearbit to get company info
2. Add them to our HubSpot CRM with enriched data
3. Post a Slack notification to the #new-signups channel

Without n8n: ~200 lines of Python + cron + error handling + logging.

With n8n:

```
[Typeform Trigger]
       │
       ▼
[HTTP Request → Clearbit API]   (enrich the lead)
       │
       ▼
[Set Node]                       (combine Typeform + Clearbit data)
       │
       ▼
[HubSpot → Create Contact]       (save to CRM)
       │
       ▼
[Slack → Send Message]           (notify the team)
```

5 nodes. Runs reliably. Has full execution history. Can be modified in the UI
without deploying code. The non-technical sales ops person can inspect it.

This is the power of n8n.

---

## n8n vs. Writing Python — When to Use What

This is a top 1% question. n8n is not always the answer.

| Use n8n when... | Write code when... |
|-----------------|-------------------|
| You're gluing existing services | You're building core product logic |
| Non-engineers need to inspect/modify it | It's purely developer-facing |
| Execution history & observability matter | You need version control & code review |
| You're integrating 3+ services | You have 1 simple API call |
| You want visual debugging | Performance is critical (milliseconds matter) |
| You want fast iteration | You need strict typing and tests |

n8n is not a replacement for your application code. It's a replacement for your
**operational glue code** — the code that connects services together.

---

## Where n8n Fits in Your Stack

```
┌──────────────────────────────────────────────────────────────┐
│                       Your Business                           │
│                                                               │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────────┐ │
│  │  Users  │   │ Product │   │  Data   │   │ Operations  │ │
│  └────┬────┘   └────┬────┘   └────┬────┘   └──────┬──────┘ │
│       │             │             │                │         │
│  ┌────▼─────────────▼─────────────▼────────────────▼──────┐ │
│  │                  n8n — Automation Layer                  │ │
│  │  (webhooks, schedules, API integrations, notifications)  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐   │
│  │  Slack   │ │ HubSpot  │ │ Postgres │ │   Stripe     │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

n8n is the automation layer — it orchestrates data flowing between all your
tools.

---

## Summary

- n8n solves the **operational glue code** problem
- It sits between no-code tools (Zapier) and full-code tools (Airflow)
- Core model: **Trigger → Nodes → Destination**, data flows as **items**
- Self-hostable, open-source, supports full JavaScript when needed
- Not a replacement for application code — a replacement for glue scripts

---

## Check Your Understanding — Q&A

### Q1. What is the one data type that flows between every node in n8n?

**Answer:** An **array of items** (JSON objects). Every trigger outputs items.
Every node receives items and outputs items. Understanding this is the
foundation of everything else in n8n.

---

### Q2. When would you choose n8n over a Python script?

**Answer:** When you need operational glue code that connects existing services
— especially when: (a) non-engineers need to inspect or run it, (b) you want
built-in execution history and error visibility, (c) you're integrating 3+
services. Use Python when you're building core application logic, need strict
typing/tests, or performance is critical.

---

### Q3. What is n8n's key advantage over Zapier?

**Answer:** Three advantages for engineers: (1) **Self-hosting** — data stays
in your infrastructure; (2) **Full Code Node** — run arbitrary JavaScript or
Python for any logic Zapier can't express; (3) **Cost** — self-hosted n8n is
free regardless of execution volume (Zapier charges per task).

---

### Q4. In one sentence — what is n8n?

**Answer:** n8n is an open-source workflow automation platform that lets you
visually connect APIs, services, and business tools using a trigger-node-output
pipeline model, with full support for custom JavaScript when needed.

---

## Next Lesson

**[Lesson 02 →](02-installation-and-architecture.md)** — Install n8n and understand
how it runs under the hood (the server, the queue, the database).
