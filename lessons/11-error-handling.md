# Lesson 11 — Error Handling & Resilient Workflows

## Goal
Build workflows that don't break silently. Learn every error handling mechanism
in n8n, design resilient retry patterns, and set up error monitoring that
actually alerts you when things fail.

---

## Why Error Handling Matters

A workflow without error handling is a liability:
- It fails silently at 3am
- You find out when a customer complains
- You have no idea which item failed or why
- Data may be partially processed — some rows in the DB, some not

Production workflows must:
1. **Detect** failures (which node, which item, what error)
2. **Retry** transient failures (network blips, rate limits)
3. **Route** unrecoverable failures somewhere useful (Slack alert, dead letter queue)
4. **Continue** processing remaining items when one item fails

---

## Error Level 1 — Node-Level: Continue On Fail

The simplest form: instead of stopping the workflow, a failed node outputs
an error item and the workflow continues.

**Enable:** Click the node → Settings → "Continue On Fail" → toggle ON

When a node fails with this on:
```json
Output item:
{
  "json": {
    "error": "Request failed with status code 404",
    "statusCode": 404
  }
}
```

Use immediately downstream:
```
HTTP Request (Continue On Fail)
        │
        ▼
IF → $json.error exists
   True: log error / skip / alert
   False: continue normal processing
```

**Best for:** Optional enrichment steps where failure = just skip that item.
Example: Clearbit enrichment fails for some emails → just skip those, don't
stop enriching the rest.

---

## Error Level 2 — Node-Level: Retry On Fail

For transient failures (network timeouts, 503 responses), automatic retry.

**Enable:** Node → Settings → "Retry On Fail" → toggle ON

Options:
```
Max Tries:           3 (try 3 times total before failing)
Wait Between Tries:  1000 ms (wait 1 second between each attempt)
```

**Best for:** HTTP requests, database connections — anything that can
temporarily fail due to external factors.

**Don't use for:** Validation errors (400 Bad Request) — retrying won't fix them.

---

## Error Level 3 — Workflow-Level: Error Workflows

When a workflow fails entirely (unhandled error), trigger another workflow.

**Enable:** Workflow Settings (gear icon) → Error Workflow → select workflow

Create an "Error Handler" workflow:

```
[Error Trigger]
      │
      ▼
[Slack → Send alert]
"❌ Workflow failed: {{ $json.workflow.name }}
Error: {{ $json.execution.error.message }}
Execution ID: {{ $json.execution.id }}
Time: {{ $now.toFormat('HH:mm') }}"
```

The Error Trigger receives:
```json
{
  "execution": {
    "id": "abc123",
    "url": "http://localhost:5678/execution/abc123",
    "retryOf": null,
    "error": {
      "message": "Something went wrong",
      "name": "NodeOperationError",
      "node": { "name": "HTTP Request", "type": "n8n-nodes-base.httpRequest" }
    },
    "lastNodeExecuted": "HTTP Request",
    "mode": "trigger"
  },
  "workflow": {
    "id": "1",
    "name": "Lead Enrichment"
  }
}
```

**This is your production monitoring.** Every important workflow should have
an Error Workflow pointing to a centralized alert handler.

---

## Error Level 4 — Item-Level: Try/Catch Pattern

For processing many items where some might fail, implement item-level
try/catch without stopping the whole batch.

### The Pattern

```
[Split Out: items]          ← 100 items
       │
       ▼
[HTTP Request]              ← might fail on some items
(Continue On Fail: ON)
       │
       ▼
[IF: $json.error exists?]
   │                   │
   ▼ True (failed)     ▼ False (success)
[Set: mark as error]  [Set: mark as success]
[Log to error sheet]  [Continue processing]
   │                   │
   └─────────┬─────────┘
             ▼
[Merge: Append both branches]
             │
             ▼
[Code: summarize results]
"Processed: 98 success, 2 errors"
```

This pattern ensures:
- Each item is attempted
- Failures are captured with context
- Successes continue normally
- A final summary shows what happened

---

## Building a Dead Letter Queue

For mission-critical workflows, failed items shouldn't just be logged —
they should be retried later.

**Pattern:**

```
[Process items]
      │
      ▼
[HTTP Request]
(Continue On Fail: ON)
      │
      ▼
[IF: error?]
   │             │
   ▼ True        ▼ False
[Airtable:      [Continue]
 log to
 "failed_queue"
 table with:
  - original item
  - error message
  - timestamp
  - retry_count: 0]
```

Create a separate "Retry Failed Items" workflow:
```
[Schedule: every hour]
      │
      ▼
[Airtable: get rows where retry_count < 3 AND status = "failed"]
      │
      ▼
[Process items again]
      │
      ▼
[IF: success?]
  True: update row to status="success"
  False: increment retry_count, if >= 3 → status="permanent_failure" → Slack alert
```

---

## Handling Specific HTTP Errors

### 429 Rate Limited — Exponential Backoff

```javascript
// Code node: compute wait time based on retry attempt
const retryAfter = $json.headers?.['retry-after'];
if (retryAfter) {
  return { waitSeconds: parseInt(retryAfter) };
}
// Exponential backoff: 2^attempt seconds
const attempt = $json.retryCount || 0;
return { waitSeconds: Math.pow(2, attempt) };
```

Then: `Wait node → {{ $json.waitSeconds }} seconds → retry`

### 401 Unauthorized — Token Refresh

```
[HTTP Request: get fresh token]
      │
      ▼
[Set: save token to $vars or pass forward]
      │
      ▼
[Original HTTP Request with new token]
```

### 404 Not Found — Create If Missing

```
[HTTP Request: GET /resource/{{ $json.id }}]
(Continue On Fail: ON)
      │
      ▼
[IF: $json.statusCode === 404]
  True: [HTTP Request: POST /resource (create)]
  False: [continue with existing]
```

---

## Workflow Timeout Handling

Long-running workflows can be killed by the n8n timeout.

Default execution timeout: 1 hour.

For workflows that might run longer:
1. In n8n Settings → Execution Timeout → set a higher value
2. Or split the workflow into smaller chunks using sub-workflows

For webhook-triggered workflows where the caller needs a quick response:
```
[Webhook] → [Respond to Webhook: "Accepted, processing..."] → [Rest of workflow]
```
Respond immediately, process in background. The caller gets 200 OK instantly.

---

## Logging Best Practices

### Always log the execution ID

Every execution has a unique ID. Include it in your error messages:
```
{{ $execution.id }}
```

This lets you find the exact execution in n8n's execution history.

### Log structured data, not strings

Instead of: `"Error: something went wrong for item 5"`
Do:
```json
{
  "executionId": "{{ $execution.id }}",
  "workflowName": "{{ $workflow.name }}",
  "itemId": "{{ $json.id }}",
  "errorMessage": "{{ $json.error }}",
  "timestamp": "{{ $now.toISO() }}",
  "retryCount": 0
}
```

Structured logs are searchable, alertable, and parseable.

---

## The Production Error Handling Checklist

For every workflow you put into production:

- [ ] Error Workflow assigned in workflow settings
- [ ] Critical HTTP Requests have Retry On Fail (3 tries, 1s wait)
- [ ] Batch processing uses Continue On Fail + error tracking
- [ ] Error alerting sends to Slack/email with execution ID + link
- [ ] 429 rate limits are handled with Wait + retry
- [ ] Dead letter pattern for any items that must not be lost
- [ ] Sticky Notes document what each section does and what can fail

---

## Summary

- 4 levels of error handling: Continue On Fail → Retry → Error Workflow → Item-level try/catch
- Error Workflow = your production monitoring — set one for every important workflow
- Item-level: Continue On Fail + IF node creates per-item try/catch
- Dead letter queue: log failed items, retry them in a separate scheduled workflow
- Always log the execution ID — it's your reference to the full execution history

---

## Next Lesson

**[Lesson 12 →](12-webhooks.md)** — Webhooks in depth: receiving events from
GitHub, Stripe, Typeform. Validating signatures, handling idempotency,
responding correctly, and testing webhooks locally with ngrok.
