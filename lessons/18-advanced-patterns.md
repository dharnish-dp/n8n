# Lesson 18 — Advanced Patterns & Top 1% Techniques

## Goal
The patterns that separate experts from everyone else. These are not features —
they're architectural decisions, debugging skills, and design instincts that
make your n8n workflows maintainable, observable, and powerful.

---

## Pattern 1: The Workflow as a State Machine

Model complex multi-step processes as state machines persisted in a database.

```
States: created → enriching → enriched → crm_created → notified → done
                                                        → failed

Workflow:
1. Load next "enriching" item from DB
2. Call enrichment API
3. On success: update state to "enriched"
4. On failure: update state to "failed", increment retry_count
5. Schedule: run every minute, process up to 10 items
```

This gives you:
- Full audit trail of where every item is
- Graceful recovery from crashes (resume from last state)
- Easy monitoring (query DB for items stuck in a state)
- Retry logic with max attempts

---

## Pattern 2: The Webhook Fan-Out Pattern

One webhook receives events and dispatches them to multiple specialized
workflows based on event type.

```
[Webhook: /events]
        │
        ▼
[Code: extract event type]
event = $json.body.type
        │
        ▼
[Switch]
  payment.succeeded   → [Execute: process-payment]
  user.created        → [Execute: onboard-user]
  subscription.ended  → [Execute: offboard-user]
  refund.created      → [Execute: process-refund]
```

Benefits:
- Each event type has its own focused workflow
- Easy to add new event types without touching the router
- Each sub-workflow can have its own error handling
- Test each event type independently

---

## Pattern 3: Dynamic Workflow Configuration

Store workflow configuration in a database instead of hardcoding it.

```
DB table "workflow_config":
  workflow_name  │  key               │  value
  ─────────────────────────────────────────────
  price-monitor  │  threshold         │  5
  price-monitor  │  slack_channel     │  #alerts
  price-monitor  │  check_interval    │  5

Workflow start:
[Schedule Trigger]
        │
        ▼
[Postgres: SELECT * FROM workflow_config WHERE workflow_name = 'price-monitor']
        │
        ▼
[Code: build config object]
const config = {};
$input.all().forEach(i => { config[i.json.key] = i.json.value; });
return config;
        │
        ▼
[Use {{ $json.threshold }}, {{ $json.slack_channel }}, etc. throughout]
```

Now non-developers can change thresholds by updating the database row —
no workflow editing required.

---

## Pattern 4: Approval Workflows

Pause execution waiting for human decision.

```
[Trigger: expense submitted]
        │
        ▼
[Send Slack message with two buttons]
"Expense request from {{ $json.name }}: ${{ $json.amount }}"
[Approve] [Reject]
        │
        ▼
[Wait: Until Webhook]
(generates a unique resume URL)
        │
(Days later, manager clicks Approve in Slack)
        ▼
[Resume: receive approval]
        │
        ▼
[IF: approved]
  True: process payment
  False: notify rejection
```

The Wait node generates a unique webhook URL. When the manager clicks Approve,
Slack calls that URL, and n8n resumes the workflow from where it paused.

---

## Pattern 5: The Observer / Audit Pattern

Every important action emits a structured audit event.

```javascript
// In every workflow that modifies data:
const auditEvent = {
  event: 'contact.created',
  workflowName: $workflow.name,
  executionId: $execution.id,
  timestamp: $now.toISO(),
  actor: 'automation',
  resource: { type: 'contact', id: $json.contactId },
  data: { email: $json.email, source: 'typeform' }
};
```

Route all audit events to:
- A database table for querying
- A logging service (Datadog, Axiom)
- A Slack channel for visibility

This makes your automation system observable — you can answer "what happened
to this contact and when?" without reading logs.

---

## Pattern 6: Lazy Credential Rotation

API tokens expire. Handle it gracefully without failing workflows.

```
[HTTP Request: use cached token from $vars.API_TOKEN]
(Continue On Fail: ON)
        │
        ▼
[IF: statusCode === 401]
  True:
    [HTTP Request: POST /auth/refresh]
    [Set Variables: update $vars.API_TOKEN]
    [HTTP Request: retry original call]
  False: continue
```

Rotate only when you get a 401 — don't pre-emptively rotate tokens.

---

## Pattern 7: The Test Harness Workflow

Build a workflow that tests other workflows.

```
[Manual Trigger]
        │
        ▼
[Code: define test cases]
return [
  { json: { name: 'valid email', input: { email: 'alice@company.com' }, expected: { domain: 'company.com' } } },
  { json: { name: 'personal email', input: { email: 'bob@gmail.com' }, expected: { isPersonal: true } } },
  { json: { name: 'missing email', input: {}, expected: { error: true } } },
];
        │
        ▼
[Execute Workflow: "enrich-contact" — run once per item, pass $json.input]
        │
        ▼
[Code: compare actual vs expected]
const actual = $json;
const expected = $('Test Cases').item.json.expected;
const passed = Object.keys(expected).every(k => actual[k] === expected[k]);
return { ...actual, testName: $('Test Cases').item.json.name, passed };
        │
        ▼
[IF: any failed]
  True: Slack alert "Tests failing: {{ $json.testName }}"
  False: Slack "All tests passed ✅"
```

---

## Debugging at Expert Level

### Technique 1: Binary search debugging

When a long workflow fails, bisect:
1. Add a "throw error" node halfway through
2. If it passes the first half — the bug is in the second half
3. Move the throw to the 3/4 mark
4. Repeat until you isolate the failing node

### Technique 2: Data snapshot pinning

Pin the output of major checkpoints:
- After the trigger
- After each expensive API call
- After each complex transformation

Now you can iterate on any section without re-running the whole pipeline.

### Technique 3: Execution comparison

Run the workflow twice — once with good data, once with bad data.
Compare the executions side by side in the Executions tab.
The divergence point is where the bug is.

### Technique 4: The $execution.mode guard

```javascript
// Skip expensive operations in manual test runs
if ($execution.mode === 'manual') {
  console.log('Test mode: skipping database write');
  return { ...$json, skipped: true };
}
// Real execution continues
```

---

## Performance Optimization

### Only load what you need from databases

```sql
-- Bad: loads all fields
SELECT * FROM users WHERE active = true

-- Good: loads only what your workflow needs
SELECT id, email, name, created_at FROM users WHERE active = true
```

### Use SplitInBatches for large datasets

Never try to process 10,000 items at once. Always batch:
```
SplitInBatches (100) → process → Wait (100ms) → repeat
```

### Deduplicate before enriching

```
[Aggregate all items]
[Code: deduplicate by email]
→ only then call Clearbit
```

---

## n8n Variables vs Credentials vs Code

| What | How to store |
|------|-------------|
| API keys, tokens, passwords | Credentials (encrypted, reusable) |
| Config values (thresholds, URLs) | Variables ($vars) |
| Temporary computation state | Pass as item fields between nodes |
| Per-run state | Pass in item fields, don't use globals |

**Never pass secrets in item fields** — they appear in execution history.
Always use credentials or $vars for sensitive values.

---

## The Top 1% Checklist

**Design:**
- [ ] Every workflow has a single clear responsibility
- [ ] Reusable logic is in sub-workflows
- [ ] Configuration is externalized (DB or $vars), not hardcoded
- [ ] Workflows are named clearly: verb + noun (e.g., "sync-hubspot-contacts")

**Reliability:**
- [ ] HTTP Requests have Retry On Fail
- [ ] Error Workflow set for all important workflows
- [ ] Idempotency implemented for webhook processors
- [ ] Dead letter queue for items that must not be lost

**Observability:**
- [ ] Every execution has an audit log entry
- [ ] Errors include execution ID + workflow name + item context
- [ ] Alert Slack channel receives failure notifications
- [ ] Airtable/DB log shows what processed, when, with what outcome

**Maintainability:**
- [ ] Sticky Notes explain non-obvious sections
- [ ] Node names are descriptive, not "HTTP Request 1"
- [ ] Workflows can be run manually with test data (pinned data)
- [ ] Complex Code nodes have inline comments on the why

---

## Summary

You've reached the top 1%. The difference now is:
- You model automation problems as state machines and pipelines
- You build for observability by default, not as an afterthought
- You treat workflows as code: modular, testable, documented
- You know exactly which pattern to apply for any problem
- You debug systematically, not by trial and error

The best n8n practitioners aren't the ones who know the most nodes.
They're the ones who design automation systems that are readable,
maintainable, and reliable when things inevitably go wrong.
