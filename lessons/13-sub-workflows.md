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

## Next Lesson

**[Lesson 14 →](14-real-project-slack-bot.md)** — Build a complete production
Slack alert bot: monitors an API endpoint every 5 minutes, detects anomalies,
sends formatted alerts with context, and uses all the patterns from lessons 1–13.
