# n8n Mastery — Complete Learning Guide

A complete, lesson-by-lesson n8n course built for **automation engineers with
strong Python/JavaScript backgrounds** who want to reach the top 1% of
workflow automation practitioners.
Lessons assume technical fluency — analogies go deep, not wide.

---

## Progress

| Lesson | Topic | Status |
|--------|-------|--------|
| [Lesson 01](lessons/01-the-problem-n8n-solves.md) | The Problem n8n Solves & Core Mental Model | ✅ Done |
| [Lesson 02](lessons/02-installation-and-architecture.md) | Installation & n8n Architecture | ✅ Done |
| [Lesson 03](lessons/03-your-first-workflow.md) | Your First Workflow | ✅ Done |
| [Lesson 04](lessons/04-the-data-model.md) | The Data Model — Items, JSON & Pinning | ✅ Done |
| [Lesson 05](lessons/05-expressions-and-dynamic-data.md) | Expressions & Dynamic Data | ✅ Done |
| [Lesson 06](lessons/06-core-node-types.md) | Core Node Types Deep Dive | ✅ Done |
| [Lesson 07](lessons/07-http-request-node.md) | HTTP Request — The Universal API Connector | ✅ Done |
| [Lesson 08](lessons/08-working-with-data.md) | Working With Data — Set, Merge, Split, Filter | ✅ Done |
| [Lesson 09](lessons/09-credentials-and-auth.md) | Credentials & Authentication Patterns | ✅ Done |
| [Lesson 10](lessons/10-code-node.md) | The Code Node — JavaScript & Python Power | 🔄 In Progress |
| [Lesson 11](lessons/11-error-handling.md) | Error Handling & Resilient Workflows | ⏳ Upcoming |
| [Lesson 12](lessons/12-webhooks.md) | Webhooks — Receiving Events from the World | ⏳ Upcoming |
| [Lesson 13](lessons/13-sub-workflows.md) | Sub-Workflows & Modular Design | ⏳ Upcoming |
| [Lesson 14](lessons/14-real-project-slack-bot.md) | Real Project 1: Slack Alert Bot | ⏳ Upcoming |
| [Lesson 15](lessons/15-real-project-lead-enrichment.md) | Real Project 2: Lead Enrichment Pipeline | ⏳ Upcoming |
| [Lesson 16](lessons/16-real-project-ai-agent.md) | Real Project 3: AI Agent with Memory | ⏳ Upcoming |
| [Lesson 17](lessons/17-self-hosting-and-production.md) | Self-Hosting, Scaling & Production | ⏳ Upcoming |
| [Lesson 18](lessons/18-advanced-patterns.md) | Advanced Patterns & Top 1% Techniques | ⏳ Upcoming |

---

## Quick Reference

- [Cheat Sheet](reference/cheatsheet.md) — expressions, shortcuts, common patterns
- [Glossary](reference/glossary.md) — every n8n term defined
- [Node Reference](reference/node-reference.md) — the most important nodes explained

---

## Real Projects

| Project | What It Builds | Lesson |
|---------|---------------|--------|
| [Slack Alert Bot](projects/slack-alert-bot/) | Monitor an API, alert on anomalies | 14 |
| [Lead Enrichment Pipeline](projects/lead-enrichment/) | Enrich contacts via Clearbit + save to Airtable | 15 |
| [AI Agent with Memory](projects/ai-agent/) | GPT-4 agent that remembers context | 16 |

---

## Who This Course Is For

Built for **Dharnish** — automation test engineer with strong Python and
JavaScript. Lessons skip "what is an API" basics and go straight to
engineering depth:

- How n8n works internally (not just how to click)
- When to use each node vs writing code
- Production patterns: error handling, retries, idempotency, observability
- Real integrations with real APIs — not toy examples

---

## How This Course Works

- **Lessons build on each other** — do them in order
- **Every lesson has hands-on exercises** — run them in your n8n instance
- **Technical depth** — since you know Python/JS, lessons go into internals,
  not just surface-level "drag this node here"
- **Real production workflows** used throughout, not toy examples
- **Quick checks after each lesson** — test your understanding before moving on

---

## Prerequisites

- Strong Python and JavaScript knowledge ✅
- Understanding of APIs, HTTP, JSON, REST ✅
- n8n installed locally (`npm install -g n8n`)
- Start command: `GENERIC_TIMEZONE=Asia/Kolkata N8N_SECURE_COOKIE=false n8n start`

---

## Philosophy

Most n8n tutorials teach you to drag nodes. This course teaches you to **think**
in workflows — to look at any automation problem and know exactly how to model
it, what nodes to reach for, and how to make it production-grade.

The difference between a casual n8n user and a top 1% practitioner:
- Casual: knows what nodes exist
- Top 1%: understands the **data model**, can debug any failure, writes clean
  modular workflows, handles errors gracefully, applies software engineering
  principles (single responsibility, DRY, observability) to automation
