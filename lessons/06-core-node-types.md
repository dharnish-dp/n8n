# Lesson 06 — Core Node Types Deep Dive

## Goal
Master every important node category. After this lesson you'll know which node
to reach for in any situation — without having to browse the node library.

---

## The Node Categories

n8n has 400+ nodes, but they fall into a small number of categories.
Master the categories, not individual nodes.

```
┌─────────────────────────────────────────────────────────────┐
│                      n8n Node Categories                     │
│                                                              │
│  1. TRIGGER nodes     — start the workflow                  │
│  2. ACTION nodes      — do things (API calls, DB writes)    │
│  3. TRANSFORM nodes   — reshape data                        │
│  4. FLOW CONTROL      — route, branch, loop, merge          │
│  5. CORE / UTILITY    — Code, Wait, Webhook response        │
└─────────────────────────────────────────────────────────────┘
```

---

## Category 1: Trigger Nodes

A workflow can have **exactly one trigger** (the first node).
The trigger determines when the workflow runs.

### Manual Trigger
- **Use:** development only
- **Output:** one empty item `[{}]`
- **When to switch away:** always in production

### Schedule Trigger
- **Use:** time-based automation (reports, syncs, checks)
- **Config:** cron expression OR visual builder
- **Output:** one item with `{ "timestamp": "..." }`

```
Important: n8n evaluates crons every minute. If your n8n goes down for an hour,
it will NOT re-run missed schedules when it comes back up.
For mission-critical schedules, use an external scheduler and webhook n8n.
```

### Webhook
- **Use:** event-driven automation (receive data from anything)
- **Two URLs:** test (dev) vs production (active workflow)
- **Important detail:** The test URL only responds when you're actively
  listening (the workflow is open in editor with "Listening for test event").
  Production URL responds whenever the workflow is active.
- **Best practice:** Always return a response quickly. If your workflow is slow,
  use the Webhook + Respond-to-Webhook pattern to acknowledge immediately and
  process asynchronously.

### App Triggers (e.g., "On new Slack message", "On new GitHub issue")
- **Use:** integration-specific events without building the webhook yourself
- **Requirements:** credentials for that service
- **Under the hood:** most are polling (n8n checks the API every X minutes)
  or webhook-based (n8n registers a webhook with the service)

---

## Category 2: Action Nodes

These do things — API calls, database writes, file operations.

**Critical rule:** Action nodes execute **once per item**.

```
3 items flowing in → 3 Slack messages / 3 emails / 3 database rows
```

If you want to send ONE message summarizing all items, use an Aggregate node
first (collect all items into one) or use a Code node to format them all.

### HTTP Request — the universal node
- Can call any REST API
- Handles all auth types
- Returns JSON, text, or binary data
- See Lesson 07 for deep coverage

### Database nodes (Postgres, MySQL, MongoDB, Airtable, etc.)
- Select / Insert / Update / Delete
- Most support "upsert" (insert or update based on a key)
- Expression support in all fields — use `{{ $json.fieldName }}` in queries

### Communication nodes (Slack, Gmail, Twilio, etc.)
- Direct integrations with pre-built auth and message formatting
- Always check if the service has a native n8n node before using HTTP Request

---

## Category 3: Transform Nodes

These reshape data without external side effects.

### Set — the field picker

```
Use cases:
  Clean up: pick only the fields you need from a big response
  Rename: output.userId = $json.id
  Compute: output.tax = $json.price * 0.1
  Combine: output.fullName = $json.firstName + ' ' + $json.lastName
  Flatten: output.city = $json.address.city

Mode: "Keep Only Set Fields" ON = only your fields in output
      "Keep Only Set Fields" OFF = your fields PLUS all existing fields
```

### Split Out — array expander

```
Before: 1 item with { users: [{id:1}, {id:2}, {id:3}] }
After:  3 items: {id:1}, {id:2}, {id:3}

When to use: any time an API returns a list and you want to process each element
```

### Aggregate — item collector

```
Before: 10 items
After:  1 item with { data: [all 10 items] }

When to use: before a "send summary" action (send ONE email listing all items)
```

### Code — custom transform

See Lesson 10 for full coverage.

---

## Category 4: Flow Control Nodes

### IF — binary branching

```
Input items are evaluated against a condition.
True items → output 0 (left/top)
False items → output 1 (right/bottom)

Common pattern:
  IF (field exists?) → True branch: process it | False branch: skip/error
  IF (score >= 60?)  → True: pass | False: fail
  IF (status = "done"?) → True: notify | False: do nothing
```

### Switch — multi-branch routing

```
Use when you have more than 2 paths.

Example: Route support tickets by priority
  Rule 1: priority = "critical" → Slack channel #incidents
  Rule 2: priority = "high"     → Slack channel #support
  Rule 3: priority = "low"      → Email to support@
  Default:                      → Log to spreadsheet
```

### Merge — combine branches

The most misunderstood node. Four modes:

```
APPEND
  Input A: [1, 2, 3]
  Input B: [4, 5, 6]
  Output:  [1, 2, 3, 4, 5, 6]
  Use: when you just want all items together

COMBINE BY FIELD (like SQL JOIN)
  Input A: [{id:1, name:"Alice"}, {id:2, name:"Bob"}]
  Input B: [{id:1, score:95},    {id:2, score:80}]
  Field: "id"
  Output: [{id:1, name:"Alice", score:95}, {id:2, name:"Bob", score:80}]
  Use: enrich items with data from a parallel lookup

COMBINE BY POSITION
  Input A item 1 + Input B item 1 → merged item 1
  Input A item 2 + Input B item 2 → merged item 2
  Use: when you know items are aligned by position

CHOOSE BRANCH
  Wait for both inputs, then pass only one branch's items.
  Use: waiting for a parallel operation to complete before continuing
```

### SplitInBatches — loop control

```
Use case: process 500 users but API allows 10 per minute

Pattern:
  SplitInBatches (size: 10)
        │
        ▼
  [HTTP Request / API calls for these 10 items]
        │
        ▼
  Wait (6 seconds)
        │
        ▼ (connect back to SplitInBatches — it loops automatically)
  SplitInBatches ←── loop back here

When all batches are done, execution continues past the loop.

Check if we're done:
  {{ $node["SplitInBatches"].context.noItemsLeft }}   → true when finished
```

---

## Category 5: Core / Utility Nodes

### Code
Full JavaScript or Python in a node. Lesson 10.

### Wait
Pause execution. Use for:
- Rate-limiting delays (`Wait → 1 second → loop back`)
- Approval workflows (`Wait for webhook → someone calls URL to approve`)
- Polling (`Wait → check API → if not done, wait again`)

### Execute Workflow
Call a sub-workflow. Returns its output items. Lesson 13.

### Respond to Webhook
When a Webhook trigger starts the workflow, this node sends the HTTP response.
Without it, n8n responds immediately with 200 OK (before your workflow runs).
With it, you control what the caller receives.

### Sticky Note
Not a node — a documentation tool. Add text notes to your canvas to explain
what sections do. Use it. Workflows without documentation are unmaintainable.

---

## Choosing the Right Node — Decision Tree

```
Need to call an external API?
  └─ Has a dedicated n8n node? → Use that node (Slack, GitHub, etc.)
  └─ No dedicated node? → HTTP Request node

Need to transform data?
  └─ Simple field pick/rename/compute? → Set node
  └─ Expand an array into multiple items? → Split Out
  └─ Complex logic / loops? → Code node

Need to control flow?
  └─ Two branches? → IF node
  └─ Three+ branches? → Switch node
  └─ Process in chunks? → SplitInBatches node
  └─ Combine two streams? → Merge node

Need to start the workflow?
  └─ Time-based → Schedule Trigger
  └─ External HTTP event → Webhook
  └─ Service event (new Stripe charge, new GitHub PR) → App Trigger
  └─ Just testing → Manual Trigger
```

---

## Hands-On Exercise: Build a Data Processing Pipeline

Build this workflow:

```
[Schedule Trigger: every 5 minutes]
         │
         ▼
[HTTP Request: GET https://jsonplaceholder.typicode.com/todos]
         │
         ▼ (returns array of 200 todos in 1 item)
[Split Out: split field 'data' into items]
         │
         ▼ (200 items)
[IF: completed = true]
   │                 │
   ▼ (completed)     ▼ (not done)
[Set: status="done"] [Set: status="pending"]
   │                 │
   └────────┬────────┘
            ▼
[Merge: Append]
            │
            ▼
[Aggregate: all into one item]
            │
            ▼
[Code: count completed vs pending]
```

This exercises: HTTP Request, Split Out, IF, Set, Merge, Aggregate, Code.

---

## Summary

- 5 node categories: Trigger, Action, Transform, Flow Control, Utility
- Action nodes run once per item — always know your item count
- IF for 2 branches, Switch for 3+
- Merge has 4 modes — Append is most common, Combine by Field is most powerful
- SplitInBatches is the loop primitive — always pair it with a Wait for rate limiting
- Sticky Notes are not optional — document your workflows

---

## Check Your Understanding — Q&A

### Q1. You have 3 items flowing into a Gmail node. How many emails are sent? What if you want only 1 summary email?

**Answer:** 3 emails — action nodes execute once per item. For 1 summary email:
Aggregate node first (collapse 3 items into 1) → Code node (format all 3 into one message body) → Gmail node (sends 1 email). Without Aggregate, Gmail fires per item.

---

### Q2. What is the difference between an IF node and a Filter node?

**Answer:** IF routes items to two branches — True and False — so you can handle both cases. Filter silently drops non-matching items; there is no "false branch." Use Filter when non-matches should be discarded. Use IF when non-matches need to do something (log, alert, take alternate action).

---

### Q3. You have 5,000 items and an API that allows 10 requests per second. What pattern do you use and how do you configure it?

**Answer:** SplitInBatches (batch size 10) → HTTP Request → Wait (1 second) → loop back to SplitInBatches. This processes 10 items, waits 1 second, then processes the next 10 — giving exactly 10 req/s. Never pass all 5,000 items directly to the HTTP Request node — it fires all 5,000 calls immediately and gets rate-limited or banned.

---

### Q4. Explain all 4 Merge modes and give a real use case for each.

**Answer:**
- **Append** — stack items from both inputs. Use when reconnecting IF branches back into one stream.
- **Combine by Field** — SQL JOIN on a matching field. Use when you called two APIs (e.g. user profile + user score) and want one enriched item per user.
- **Combine by Position** — merge item 1 from A with item 1 from B. Use when items are positionally aligned and have no shared ID.
- **Choose Branch** — wait for both inputs, pass only one branch's items. Use when running a parallel lookup and you only want the primary data going forward.

---

### Q5. When should you use a Switch node vs chaining multiple IF nodes?

**Answer:** Switch node when routing on a single field's value to 3+ destinations — cleaner, single place to change routing logic. Chained IF nodes when each branch has a completely different condition (not the same field). Chained IFs also let you pass items through multiple conditions sequentially — Switch is parallel, exclusive routing.

---

### Q6. What is the purpose of a Sticky Note node and why is it not optional in production?

**Answer:** Sticky Notes add text documentation to the canvas — they don't process data. They're not optional because workflows without documentation are unmaintainable after 2 weeks. A future reader (or you at 3am during an incident) needs to know: what does this section do, what can fail here, why is this condition set this way. Treat them like code comments — explain the *why*, not the *what*.

---

## Next Lesson

**[Lesson 07 →](07-http-request-node.md)** — The HTTP Request node in exhaustive
detail: every authentication method, handling pagination, file uploads,
custom headers, error handling, and building a full API integration from scratch.
