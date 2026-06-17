# Lesson 14 — Real Project 1: API Monitor & Slack Alert Bot

## Goal
Build a complete, production-grade monitoring workflow. Every 5 minutes, it
checks a public API for anomalies, formats a rich Slack message, and routes
alerts to different channels based on severity. Applies: scheduling, HTTP
Request, Code node, IF/Switch, error handling, and sub-workflows.

---

## What We're Building

```
[Schedule: every 5 min]
         │
         ▼
[HTTP Request: fetch current data]
         │
         ▼
[Code: detect anomalies + compute severity]
         │
         ▼
[Switch: route by severity]
   │          │          │
   ▼critical  ▼warning   ▼ok
[Slack:#inc] [Slack:#ops] [no-op]
         │
         ▼
[Airtable: log every check]
```

We'll use the **CoinGecko API** (free, no auth) to monitor Bitcoin price.
Alert if the price moves more than 5% in 5 minutes.

---

## Step 1 — Create the Workflow

1. New workflow → name: "BTC Price Monitor"
2. Add a Schedule Trigger
3. Set to: Every 5 minutes (`*/5 * * * *`)

---

## Step 2 — Fetch Bitcoin Price

Add an HTTP Request node after the Schedule Trigger.

```
Method: GET
URL: https://api.coingecko.com/api/v3/simple/price

Query Parameters:
  ids:             bitcoin
  vs_currencies:   usd
  include_24hr_change: true
  include_market_cap:  false
```

Expected response:
```json
{
  "bitcoin": {
    "usd": 67450.23,
    "usd_24h_change": -2.34
  }
}
```

---

## Step 3 — Extract and Compute

Add a Set node to extract clean fields:

```
price:       {{ $json.bitcoin.usd }}
change_24h:  {{ $json.bitcoin.usd_24h_change }}
timestamp:   {{ $now.toISO() }}
```

---

## Step 4 — Compute Severity with Code Node

This is the core logic. Add a Code node (Run Once Per Item):

```javascript
const price = $json.price;
const change = $json.change_24h;
const absChange = Math.abs(change);

// Determine severity
let severity;
let message;

if (absChange >= 10) {
  severity = 'critical';
  message = `🚨 Bitcoin ${change > 0 ? 'surged' : 'crashed'} ${absChange.toFixed(1)}% in 24h!`;
} else if (absChange >= 5) {
  severity = 'warning';
  message = `⚠️ Bitcoin ${change > 0 ? 'up' : 'down'} ${absChange.toFixed(1)}% in 24h.`;
} else {
  severity = 'ok';
  message = `✅ Bitcoin price stable.`;
}

return {
  price: price,
  change_24h: change,
  absChange: absChange,
  severity: severity,
  message: message,
  priceFormatted: `$${price.toLocaleString('en-US', {maximumFractionDigits: 2})}`,
  timestamp: $json.timestamp
};
```

---

## Step 5 — Route by Severity with Switch Node

Add a Switch node:

```
Mode: Rules

Rule 1: $json.severity equals "critical" → Output 1
Rule 2: $json.severity equals "warning"  → Output 2
Rule 3: $json.severity equals "ok"       → Output 3
```

---

## Step 6 — Slack Alerts per Severity

### Critical branch (Output 1):

Add a Slack node:
```
Channel: #incidents
Message:
{{ $json.message }}
*Current Price:* {{ $json.priceFormatted }}
*24h Change:* {{ $json.change_24h.toFixed(2) }}%
_Checked at {{ $now.toFormat('HH:mm') }}_
```

### Warning branch (Output 2):

Add a Slack node:
```
Channel: #monitoring
Message:
{{ $json.message }}
*Current Price:* {{ $json.priceFormatted }}
*24h Change:* {{ $json.change_24h.toFixed(2) }}%
```

### OK branch (Output 3):

Add a **No-op** node (or just leave it unconnected for now). No Slack message
for normal conditions.

---

## Step 7 — Log Every Check to Airtable

After all Slack branches, add a Merge node (Append mode) to bring all three
paths back together, then an Airtable node to log every check.

**Why log everything?** You want a history of all checks, not just alerts.
This lets you graph price history, audit when alerts fired, debug false positives.

```
Merge (Append mode) ← connects all 3 Switch outputs
        │
        ▼
Airtable: Create Record
Table: "BTC Checks"
Fields:
  Price:      {{ $json.priceFormatted }}
  Change 24h: {{ $json.change_24h }}
  Severity:   {{ $json.severity }}
  Message:    {{ $json.message }}
  Checked At: {{ $json.timestamp }}
```

---

## Step 8 — Add Error Handling

**On the HTTP Request node:**
- Retry On Fail: ON (3 tries, 2000ms wait)
- Continue On Fail: OFF (we want the Error Workflow to catch total failures)

**Set the Error Workflow:**
- Workflow Settings → Error Workflow → create a new workflow "Error Handler"

**Error Handler workflow:**
```
[Error Trigger]
      │
      ▼
[Slack: #incidents]
"❌ *Workflow Failed: {{ $json.workflow.name }}*
Error: {{ $json.execution.error.message }}
Execution: {{ $json.execution.id }}
Time: {{ $now.toFormat('HH:mm') }}"
```

---

## Step 9 — Activate and Test

1. Save the workflow
2. Click "Execute Workflow" to do a manual test run
3. Watch the nodes light up green
4. Check your Slack channels
5. Check your Airtable log
6. Toggle **Active** to ON — the workflow now runs every 5 minutes automatically

---

## Making It Production-Grade: Improvements

### Add price comparison vs previous check

```javascript
// Store last price in a global variable or Airtable
// Compare current price to previous price (not just 24h change)
const lastPrice = parseFloat($vars.BTC_LAST_PRICE || '0');
const priceDelta = lastPrice > 0 ? ((price - lastPrice) / lastPrice) * 100 : 0;
```

### Prevent alert spam (cooldown)

```javascript
// Don't re-alert if we already alerted in the last 30 minutes
const lastAlert = $vars.BTC_LAST_ALERT_TIME;
if (lastAlert) {
  const minutesSince = $now.diff(DateTime.fromISO(lastAlert), 'minutes').minutes;
  if (minutesSince < 30 && severity === 'warning') {
    return { ...result, severity: 'ok', suppressed: true };
  }
}
```

### Add multi-coin monitoring

Use a Code node to define the coins to monitor:
```javascript
return [
  { json: { coin: 'bitcoin', id: 'bitcoin', threshold: 5 } },
  { json: { coin: 'ethereum', id: 'ethereum', threshold: 7 } },
  { json: { coin: 'solana', id: 'solana', threshold: 10 } },
];
```

Then process each coin with the same HTTP Request node (one request per item).

---

## What You've Applied

| Concept | Where |
|---------|-------|
| Schedule Trigger | Every 5 minutes |
| HTTP Request | CoinGecko API fetch |
| Set node | Extract clean fields |
| Code node | Multi-step severity logic |
| Switch node | Route to different channels |
| Merge node | Reconnect branches for logging |
| External integration (Slack) | Alerts |
| External integration (Airtable) | Logging |
| Retry On Fail | HTTP Request |
| Error Workflow | Catch total failures |
| Expressions | Throughout (formatting, math) |

---

## Check Your Understanding — Q&A

### Q1. The CoinGecko API returns a 429. Your workflow fails. How should you handle this at the node level?

**Answer:** On the HTTP Request node: enable **Retry On Fail** (3 tries, wait 30,000ms between retries — 429 means rate limited, not a transient blip, so wait longer). Also check for a `Retry-After` header in the response using Full Response mode — use that value as the wait time if present. For a price monitoring workflow that fires every 5 minutes, a single 429 is usually fine to retry 2–3 times; if it keeps failing, the Error Workflow fires and alerts you.

---

### Q2. You want to add a cooldown so you don't send the same severity alert more than once every 30 minutes. How do you implement this in n8n?

**Answer:** After computing severity, check `$vars.LAST_ALERT_TIME` and `$vars.LAST_ALERT_SEVERITY`. In a Code node:
```javascript
const lastAlert = $vars.LAST_ALERT_TIME;
const lastSeverity = $vars.LAST_ALERT_SEVERITY;
if (lastAlert && lastSeverity === $json.severity) {
  const minutesSince = $now.diff(DateTime.fromISO(lastAlert), 'minutes').minutes;
  if (minutesSince < 30) return { ...$json, suppress: true };
}
return { ...$json, suppress: false };
```
Then IF `suppress === true` → skip Slack → log only. Update `$vars` after sending the alert using the n8n Variables API or a Set Variables node.

---

### Q3. Your Airtable log is filling up with "ok" status records every 5 minutes. How do you reduce noise without losing the alert history?

**Answer:** Two options: (1) Only log to Airtable when severity is `warning` or `critical` — add an IF node before Airtable and skip logging for `ok`. (2) Keep logging all but set `EXECUTIONS_DATA_PRUNE=true` and `EXECUTIONS_DATA_MAX_AGE=24` in n8n config so execution history auto-purges after 24h, while Airtable retains only the important events. Option 1 is cleaner — your audit log should contain signal, not noise.

---

### Q4. How would you extend this workflow to monitor 5 coins simultaneously instead of just Bitcoin?

**Answer:** Replace the Schedule Trigger → single HTTP Request with:
1. Schedule Trigger → Code node (return 5 items, one per coin with its threshold)
2. HTTP Request node (runs once per item — 5 API calls, one per coin)
3. Set node uses `$json.coin` to label each result
4. Code node computes severity per coin using its own threshold
5. Switch/Slack nodes use `$json.coin` in the message

The key insight: make the coin list a data-driven configuration (Code node or DB table), not 5 hardcoded workflow branches. Adding a 6th coin means adding one row to the config, not rebuilding the workflow.

---

## Next Lesson

**[Lesson 15 →](15-real-project-lead-enrichment.md)** — Lead enrichment pipeline:
receive new contacts from a Typeform webhook, enrich with Clearbit, create in
HubSpot, and notify the sales team — all with idempotency and error handling.
