# n8n Glossary

---

**Activation**
Switching a workflow from inactive (only runs manually) to active (runs
automatically on triggers). Toggle in the top-right of the editor.

---

**Binary data**
File content stored in an item's `binary` property. Used when nodes return
files (images, PDFs, CSVs). Referenced as `$binary.data` in expressions.

---

**Canvas**
The visual editing area where you build workflows by placing and connecting
nodes. Supports panning and zooming.

---

**Code Node**
A node that runs JavaScript or Python. Operates in two modes: "Run Once Per
Item" (processes each item individually) or "Run Once for All Items" (receives
the full array and returns a new array).

---

**Credential**
A stored authentication config (API key, OAuth token, username/password).
Credentials are stored encrypted in n8n's database and referenced by nodes
that need API access.

---

**Execution**
A single run of a workflow from trigger to completion. Each execution is
recorded with its status (success/error), timing, and per-node input/output
data.

---

**Expression**
Dynamic code embedded in node configuration using `{{ }}` syntax. Evaluated
as JavaScript. Examples: `{{ $json.name }}`, `{{ $now.toISO() }}`.

---

**IF Node**
A flow control node that evaluates a condition and routes items to one of two
outputs: True or False.

---

**Item**
The fundamental data unit in n8n. An item is an object with three properties:
`json` (the data), `binary` (file data), and `pairedItem` (lineage tracking).
All nodes receive and output arrays of items.

---

**Luxon**
The date library used by n8n. Accessed via `DateTime` in expressions or the
Code node. Powers `$now`, `$today`, and all date formatting/arithmetic.

---

**Main Mode**
n8n's single-process execution mode — one process handles the UI, API,
webhooks, and workflow execution. Suitable for dev and light production use.

---

**Manual Trigger**
A trigger node that starts the workflow only when you click "Execute Workflow"
in the UI. Used for development and testing.

---

**Node**
The fundamental building block of a workflow. A node receives items, performs
one action (API call, transformation, condition check), and outputs items.

---

**Pairing / pairedItem**
Internal tracking of which input item produced which output item. Used by
Merge nodes to combine corresponding items from two branches.

---

**Pinning**
Freezing the output of a node so downstream nodes use that exact data without
re-running the upstream node. Critical for development — prevents wasteful
API calls.

---

**Queue Mode**
n8n's multi-process execution mode. A main process handles the UI and webhook
intake; separate worker processes execute workflows. Workers communicate via
Redis. Used for production scale.

---

**Schedule Trigger**
A trigger node that fires workflows on a cron schedule. Uses standard 5-field
cron syntax: `minute hour day month weekday`.

---

**Split Out Node**
Converts a single item containing an array into multiple items — one per array
element. The inverse of Aggregate. Critical for processing lists returned by APIs.

---

**SplitInBatches Node**
Processes items in chunks (batches). Takes a large array of items and loops
through them N at a time. Useful for rate-limited APIs.

---

**Sub-workflow / Execute Workflow Node**
A workflow that is called by another workflow. Enables modular design — build
reusable automation components that larger workflows can invoke.

---

**Switch Node**
A multi-branch IF node. Routes items to different outputs based on a value.
Like a JavaScript `switch` statement.

---

**Trigger Node**
The first node in any workflow. Determines when and why the workflow runs
(manually, on a schedule, via webhook, on an event in an integrated service).

---

**Variable ($vars)**
A key-value store at the n8n instance level. Values are available in any
workflow via `$vars.KEY`. Set in Settings → Variables. Used for environment
config (API base URLs, environment names).

---

**Wait Node**
Pauses workflow execution for a specified duration or until a webhook arrives.
Used for polling loops, approval workflows, and retry patterns.

---

**Webhook**
A URL that, when called by an external service, triggers a workflow. n8n
provides two URLs: a test URL (for development, requires listening mode) and
a production URL (always active when workflow is active).

---

**Workflow**
A visual program in n8n: a directed graph of nodes connected by edges, starting
with a trigger and performing an automation. Stored as JSON in the database.

---

**Workflow JSON**
The underlying JSON representation of a workflow. Can be exported, imported,
version-controlled, and shared. Contains node definitions, connections,
settings, and pinned data.

---

**$execution.mode**
Tells you how the current execution was started:
- `"manual"` — clicked Execute Workflow in UI
- `"trigger"` — fired by an active trigger (cron, service event)
- `"webhook"` — incoming HTTP call to webhook URL
- `"cli"` — started from command line
