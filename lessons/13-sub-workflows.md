# Lesson 13 — Sub-Workflows & Modular Design

## Goal
Learn how to build modular n8n systems — reusable workflow components that
any workflow can call. Apply software engineering's single-responsibility
principle to automation architecture.

---

## Why Sub-Workflows Matter

Without modularity, automation systems grow into spaghetti:
- Every workflow reimplements the same Slack notification logic
- When the Slack format changes, you update 12 workflows
- Debugging means reading the same 40-node workflow every time

With sub-workflows:
```
Parent Workflow A ──▶ [Execute: send-slack-alert]
Parent Workflow B ──▶ [Execute: send-slack-alert]
Parent Workflow C ──▶ [Execute: send-slack-alert]

Change format in send-slack-alert → all three updated instantly.
```

Sub-workflows are functions in your automation architecture.

---

## How Execute Workflow Works

The `Execute Workflow` node calls another workflow and returns its output.

```
Parent workflow:
   [Execute Workflow: "enrich-lead"]
   Pass items: [{email: "alice@example.com"}]
                │
                ▼
   Sub-workflow "enrich-lead":
   Receives: [{email: "alice@example.com"}]
   Processes, returns: [{email: "...", company: "Acme", ...}]
                │
                ▼
   Parent continues with enriched data
```

The sub-workflow starts with a `When Called By Another Workflow` trigger (not
a webhook or schedule). It receives items from the parent and returns items.

---

## Setting Up a Sub-Workflow

### Sub-workflow structure:
```
[When Called By Another Workflow]  ← trigger (not manual/webhook/schedule)
           │
           ▼
   [Your processing nodes]
           │
           ▼
   [Set: format output cleanly]    ← output only what the parent needs
```

The `When Called By Another Workflow` trigger:
- Receives items passed by the parent
- Outputs those items into the sub-workflow
- Has options for how to handle the passed data

### Calling the sub-workflow:
In the parent, add an `Execute Workflow` node:
```
Mode: Run Once (all items as batch) → most efficient
   or Run Once Per Item (one call per item → many executions)

Workflow: select by name

Source of data: Database (recommended for production)
   Determines how the sub-workflow's definition is loaded.
```

---

## Real Utility Sub-Workflows to Build

### 1. send-slack-alert

Centralizes all Slack alerts. Any workflow can call this.

```
Inputs expected:
  {
    channel: "#alerts",
    title: "Something happened",
    message: "Details here",
    severity: "error" | "warning" | "info",
    workflowName: "the caller"
  }

Sub-workflow:
[When Called By Another Workflow]
        │
        ▼
[Set: format message]
icon = $json.severity === 'error' ? '🔴' : $json.severity === 'warning' ? '🟡' : '🟢'
text = `${icon} *${$json.title}*\n${$json.message}\n_From: ${$json.workflowName}_`
        │
        ▼
[Slack: Post Message]
Channel: {{ $json.channel }}
Text: {{ $json.text }}
```

Any parent workflow calls it with:
```
Execute Workflow: "send-slack-alert"
Pass: { channel: "#deploys", title: "Deploy started", message: "...", severity: "info", workflowName: $workflow.name }
```

### 2. enrich-contact

Takes an email, returns company data from Clearbit.

```
Input: { email: "alice@company.com" }
Output: { email, firstName, lastName, company, jobTitle, country }

Sub-workflow:
[When Called By Another Workflow]
        │
        ▼
[HTTP Request: Clearbit /v2/people/find?email={{ $json.email }}]
(Continue On Fail: ON — Clearbit won't find everyone)
        │
        ▼
[IF: $json.error exists]
  True: return { email: $json.email, enriched: false }
  False: 
[Set: pick fields]
email = $json.email
firstName = $json.name.givenName
company = $json.employment.name
jobTitle = $json.employment.title
enriched = true
```

### 3. log-to-airtable

Centralized audit log. Any workflow logs events here.

```
Input: { event, workflowName, executionId, metadata }

Sub-workflow:
[When Called By Another Workflow]
        │
        ▼
[Airtable: Create Record]
Table: "Audit Log"
Fields:
  Event: {{ $json.event }}
  Workflow: {{ $json.workflowName }}
  Execution ID: {{ $json.executionId }}
  Metadata: {{ JSON.stringify($json.metadata) }}
  Timestamp: {{ $now.toISO() }}
```

---

## Execute Workflow Modes

### Run Once (all items as a single batch)

```
Parent has 50 items → sub-workflow runs ONCE, receives all 50 items
Sub-workflow processes them, returns output → parent continues
```

Best for: batch operations, aggregation in the sub-workflow.

### Run Once Per Item

```
Parent has 50 items → sub-workflow runs 50 TIMES (once per item)
```

Best for: item-level processing where each item needs independent sub-workflow logic.
Warning: creates 50 execution records — can flood your execution history.

---

## Passing Data Between Workflows

### Parent → Sub-workflow (input)

The `Execute Workflow` node passes its input items to the sub-workflow.
The sub-workflow's `When Called By Another Workflow` trigger outputs them.

### Sub-workflow → Parent (output)

Whatever the sub-workflow's last node outputs becomes the parent's input
after the `Execute Workflow` node.

**Best practice:** End sub-workflows with a clean `Set` node that outputs
only the fields the parent needs. Don't return massive response objects.

---

## Designing Your Automation Architecture

### Single Responsibility Principle for Workflows

Each workflow should do one thing:

| Workflow Name | Responsibility |
|---------------|---------------|
| `stripe-new-customer` | Triggered on Stripe event |
| `enrich-lead` | Takes email, returns enriched data |
| `add-to-hubspot` | Takes contact data, creates/updates CRM |
| `send-slack-alert` | Sends formatted Slack message |
| `log-event` | Writes audit record to database |

A `stripe-new-customer` workflow orchestrates:
```
[Stripe Trigger]
      │
      ▼
[Execute: enrich-lead]
      │
      ▼
[Execute: add-to-hubspot]
      │
      ▼
[Execute: send-slack-alert]
      │
      ▼
[Execute: log-event]
```

Each sub-workflow is independently testable and reusable.

---

## Error Handling in Sub-Workflows

If a sub-workflow throws an error, the `Execute Workflow` node in the parent
also throws. The parent's error handling applies.

To handle sub-workflow errors gracefully:

**Option 1:** Enable "Continue On Fail" on the `Execute Workflow` node.
```
If sub-workflow fails → parent gets { error: "..." } item → handle it
```

**Option 2:** Sub-workflow catches its own errors internally and returns
a structured error item instead of throwing.
```
Sub-workflow ends with: { success: false, error: "message" }
Parent checks: IF $json.success === false → handle error
```

---

## Summary

- Sub-workflows = functions in your automation system
- `When Called By Another Workflow` trigger starts a sub-workflow
- `Execute Workflow` node calls a sub-workflow, returns its output
- Build utility sub-workflows: send-slack-alert, enrich-contact, log-event
- Single responsibility: each workflow does one thing
- End sub-workflows with clean Set nodes — output only what parents need

---

## Check Your Understanding — Q&A

### Q1. What trigger node do you use to start a sub-workflow, and why can't you use a Webhook or Schedule trigger?

**Answer:** `When Called By Another Workflow` trigger. Webhooks and Schedule triggers start workflows from external events or time — they can't receive items from a parent workflow or return results back to it. The `When Called By Another Workflow` trigger is the only one designed to receive items from an `Execute Workflow` node and return output that the parent can use. It's the function signature in n8n's architecture.

---

### Q2. You call a sub-workflow in "Run Once Per Item" mode with 50 items. How many sub-workflow executions are created? What is the performance implication?

**Answer:** 50 executions — one per item. Each execution is a separate record in the database, a separate process spawn, and logged independently. For 50 items this is usually fine. For 5,000 items it floods your execution history and creates significant overhead. For large batches, use "Run Once (all items as batch)" mode — the sub-workflow receives all items at once and runs once. Use Per Item mode only when each item truly needs an independent execution context (e.g. different error handling, different credentials per item).

---

### Q3. How do you pass data back from a sub-workflow to the parent workflow?

**Answer:** Whatever the sub-workflow's last node outputs becomes the `Execute Workflow` node's output in the parent. The sub-workflow's final items flow directly back. Best practice: end every sub-workflow with a Set node (Keep Only Set Fields: ON) that outputs exactly the fields the parent needs — don't return the entire accumulated item with all intermediate fields. This creates a clean interface contract between parent and sub-workflow, exactly like a function's return type.

---

### Q4. A sub-workflow throws an unhandled error. What happens in the parent workflow?

**Answer:** The `Execute Workflow` node in the parent also throws — the error propagates up. The parent's error handling applies: if Retry On Fail is set on the Execute Workflow node, it retries the sub-workflow call. If Continue On Fail is on, the parent gets an error item and can handle it downstream. If neither is set, the parent workflow fails entirely and the parent's Error Workflow fires. Design sub-workflows to catch their own errors internally and return `{ success: false, error: "..." }` — this gives the parent structured error info instead of an exception.

---

### Q5. You have the same "send Slack alert" logic copied in 8 different workflows. What is the correct refactor and what are the exact steps?

**Answer:**
1. Create a new workflow named `send-slack-alert`
2. Add `When Called By Another Workflow` as the trigger
3. Move the Slack logic into it — expect `{ channel, title, message, severity }` as input
4. End with a Set node returning `{ sent: true, timestamp: $now.toISO() }`
5. In each of the 8 workflows, replace the Slack node with `Execute Workflow → send-slack-alert`
6. Pass the required fields in the Execute Workflow node's input mapping

Now all 8 workflows use one implementation. Change the Slack format once → all 8 updated. This is the DRY principle applied to automation architecture.

---

## Next Lesson

**[Lesson 14 →](14-real-project-slack-bot.md)** — Build a complete production
Slack alert bot: monitors an API endpoint every 5 minutes, detects anomalies,
sends formatted alerts with context, and uses all the patterns from lessons 1–13.
