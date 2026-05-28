# Node Reference — The Most Important Nodes

The 20 nodes that cover 95% of real-world n8n work.

---

## Trigger Nodes

### Manual Trigger
**When to use:** Development, one-off runs, testing
```
Output: [ { json: {} } ]   ← one empty item
```

### Schedule Trigger
**When to use:** Run on a schedule (cron)
```
Cron examples:
  0 8 * * *     = 8 AM daily
  */30 * * * *  = every 30 minutes
  0 9 1 * *     = 9 AM on the 1st of every month

Output: [ { json: { timestamp: "..." } } ]
```

### Webhook
**When to use:** Receive HTTP calls from external services (Stripe, GitHub, Typeform, etc.)
```
Method:   POST (most common), GET, PUT, etc.
Auth:     None | Header Auth | Basic Auth | JWT
Response: Immediately (200 OK) | Custom (via Respond to Webhook node)

Test URL:       http://localhost:5678/webhook-test/<id>  (manual trigger only)
Production URL: http://localhost:5678/webhook/<id>       (workflow must be active)

Output: [ { json: { body: {...}, headers: {...}, query: {...} } } ]
Access webhook body: $json.body.fieldName
Access query param:  $json.query.paramName
Access header:       $json.headers['x-custom-header']
```

---

## HTTP & API Nodes

### HTTP Request
**When to use:** Call any REST API — the most used node in n8n
```
Method:        GET | POST | PUT | PATCH | DELETE
URL:           https://api.example.com/endpoint
Auth:          None | API Key | Bearer Token | Basic | OAuth2
Body Type:     JSON | Form Data | Raw | Binary
Send Headers:  Add custom headers (X-API-Key, Content-Type, etc.)
Send Query:    Add URL parameters
Response:      Auto (JSON parsed automatically)

Key settings:
  - "Ignore SSL Issues": bypass self-signed cert errors
  - "Full Response": get status code, headers + body (not just body)
  - "Batching": add delay between requests for rate limits

Access response: $json.fieldName (n8n auto-parses JSON)
Access status code (Full Response on): $json.statusCode
```

---

## Data Transformation Nodes

### Set
**When to use:** Create clean output items with specific fields
```
Mode: Manual Mapping (click "Add field" for each)
or Expression Mode (write one expression per field)

Keep Only Set Fields: ✓ = output ONLY the fields you define (clean output)
                       ✗ = keep all existing fields + add your new ones

Use case examples:
- Pick only the fields you need from a large API response
- Rename fields (set output.userId = input $json.id)
- Add computed fields (set tax = $json.price * 0.1)
- Add constants (set env = "production")
```

### Code
**When to use:** Complex logic that expressions can't handle
```
Language:  JavaScript | Python
Mode:      Run Once Per Item   ← processes each item, return one object
           Run Once for All Items ← receive all items, return array

JavaScript (Per Item) — return an object:
  return { name: $json.name.toUpperCase(), score: $json.score * 2 };

JavaScript (All Items) — return an array of objects:
  const items = $input.all();
  return items.map(i => ({ json: { ...i.json, doubled: i.json.value * 2 } }));

Python (Per Item):
  return {"name": _json["name"].upper()}

Available in JS Code node:
  $input, $json, $now, $today, $workflow, $execution, $vars
  DateTime (Luxon), $('Node Name'), $env
```

### Split Out
**When to use:** Convert a single item with an array into multiple items
```
Field to Split Out: users   (the array field name)
Before: [ { json: { users: [{id:1}, {id:2}] } } ]  ← 1 item
After:  [ { json: {id:1} }, { json: {id:2} } ]      ← 2 items
```

### Aggregate
**When to use:** Collect multiple items back into one (inverse of Split Out)
```
Before: [ {name:"A"}, {name:"B"}, {name:"C"} ]    ← 3 items
After:  [ { json: { data: [{name:"A"}, ...] } } ]  ← 1 item with array
```

---

## Flow Control Nodes

### IF
**When to use:** Binary branching — True vs False
```
Condition types:
  String: equals, contains, startsWith, endsWith, regex
  Number: =, !=, >, <, >=, <=
  Boolean: is true, is false
  DateTime: before, after
  Array: contains, length equals

Two outputs: True (index 0) / False (index 1)
Multiple conditions: AND (all must pass) / OR (any must pass)
```

### Switch
**When to use:** Route items to different paths based on a value
```
Mode:  Rules (if/else-if chain)
       Expression (use a JS expression that returns an output index)

Example rules:
  Rule 1: $json.status equals "active" → Output 1
  Rule 2: $json.status equals "pending" → Output 2
  Rule 3: $json.status equals "cancelled" → Output 3
  Default → Output 4

Fallback: what to do if no rules match (ignore, or connect a default output)
```

### Merge
**When to use:** Combine data from two branches back into one stream
```
Modes:
  Append:          stack all items from both inputs together
  Combine by field: join items where a field value matches (like SQL JOIN)
  Combine by position: merge item 1 from A with item 1 from B, etc.
  Choose branch:   wait for both, then pass items from one specific input
  Multiplex:       cartesian product (every item from A × every item from B)

Most common: Append (when you just want all items together)
             Combine by field (when enriching — add fields from one to the other)
```

### SplitInBatches
**When to use:** Process large arrays in chunks (rate limiting, memory)
```
Batch Size:  10 (process 10 items at a time)
Options:     Reset → start a new loop from the beginning

Pattern:
  [SplitInBatches] → [HTTP Request] → [SplitInBatches]
                        ↑___________________________|
  The loop re-enters SplitInBatches until all batches are processed.

Access batch info:
  $node["SplitInBatches"].context.currentRunIndex   ← current batch number
  $node["SplitInBatches"].context.noItemsLeft        ← true when done
```

### Wait
**When to use:** Pause execution (polling, approval workflows, delays)
```
Modes:
  Specific time:  wait until a timestamp
  After X time:   wait for a duration (seconds, minutes, hours)
  Webhook:        pause until a specific URL is called (approval pattern)
  Form:           pause until a form is submitted

Use cases:
  - Delay between API calls (rate limiting)
  - Wait for human approval before proceeding
  - Poll status until a job completes
```

---

## Workflow Composition Nodes

### Execute Workflow
**When to use:** Call another workflow as a sub-routine
```
Workflow:     select by name
Mode:
  Run Once (all items as one batch) → passes all items to sub-workflow
  Run Once Per Item → calls sub-workflow once per item

Sub-workflow receives items, returns items.
Parent workflow receives the sub-workflow's output items.

Design pattern: Create "utility" workflows (send-slack, enrich-lead, log-error)
that any other workflow can call. Single responsibility principle for automation.
```

### Respond to Webhook
**When to use:** Send a custom HTTP response back to a webhook caller
```
Must be used in a workflow that starts with a Webhook trigger.
Without this node, n8n responds immediately with 200 OK.
With this node, response is sent when this node runs.

Response Code: 200, 201, 400, etc.
Response Type: JSON, Text, Binary
Body:          {{ JSON.stringify($json) }} or any expression
```

---

## Database & Storage Nodes

### Postgres / MySQL
```
Operations: Execute Query | Insert | Update | Delete | Select
Connection via: Credentials (host, port, database, user, password)

Execute Query mode (most flexible):
  SELECT * FROM users WHERE id = {{ $json.userId }}
  INSERT INTO logs (event, timestamp) VALUES ('{{ $json.event }}', NOW())

Column mapping mode:
  Map node output fields to database columns directly
```

---

## Utility Nodes

### Sticky Note
**When to use:** Document your workflow — describe what a section does
```
Not a node — doesn't process data.
Double-click to edit. Resize by dragging corners.
Color-code sections: yellow = warning, green = done, blue = info.
```

### No-op (No Operation)
**When to use:** Placeholder in a branch, explicit end point, or structural clarity
```
Passes items through unchanged.
Useful for: marking the "end" of a branch, temporary placeholder during build.
```
