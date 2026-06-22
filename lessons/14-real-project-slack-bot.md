# Lesson 14 — Real Project 1: API Monitor & Telegram Alert Bot

## Goal
Build a complete, production-grade monitoring workflow. Every 5 minutes, it
checks a public API for anomalies, formats a rich Telegram message, and routes
alerts based on severity. Applies: scheduling, HTTP Request, Code node,
IF/Switch, error handling, and sub-workflows.

---

## Why Telegram

- Free bot API — no workspace, no subscription
- Setup takes 2 minutes via BotFather
- n8n has a built-in Telegram node
- Supports rich formatting (bold, italic, code blocks) via Markdown
- Works on any device — phone, desktop, web

---

## What We're Building

```
[Schedule: every 5 min]
         │
         ▼
[HTTP Request: CoinGecko API]
         │
         ▼
[Set: extract price + change]
         │
         ▼
[Code: compute severity]
         │
         ▼
[Switch: route by severity]
   │            │          │
   ▼ critical   ▼ warning  ▼ ok
[Telegram]   [Telegram]  [no-op]
         │
         ▼
[HTTP Request: log to Google Sheets]
```

We'll use the **CoinGecko API** (free, no auth) to monitor Bitcoin price.
Alert if the 24h change crosses ±5% (warning) or ±10% (critical).

---

## Step 1 — Create a Telegram Bot (2 minutes)

### 1a — Create the bot via BotFather

1. Open Telegram → search **@BotFather** → start a chat
2. Send: `/newbot`
3. Give it a name: `BTC Monitor`
4. Give it a username: `btc_monitor_yourname_bot` (must end in `bot`)
5. BotFather replies with your **Bot Token**: `7123456789:AAF...`
6. Save this token — you'll add it to n8n credentials

### 1b — Get your Chat ID

1. Search for **@userinfobot** in Telegram → start a chat → send any message
2. It replies with your **Chat ID** (a number like `123456789`)
3. Save this — it's where alerts will be sent

Alternatively, send a message to your new bot first, then:
```
GET https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates
```
Find `message.chat.id` in the response.

### 1c — Add Telegram credential in n8n

1. Settings → Credentials → Add → search "Telegram"
2. Name: `Telegram - BTC Monitor Bot`
3. Access Token: paste your Bot Token
4. Save

---

## Step 2 — Create the Workflow

1. New workflow → name: **"BTC Price Monitor"**
2. Add a **Schedule Trigger**
3. Cron: `*/5 * * * *` (every 5 minutes)

---

## Step 3 — Fetch Bitcoin Price

Add an **HTTP Request** node:

```
Method: GET
URL: https://api.coingecko.com/api/v3/simple/price

Query Parameters:
  ids:                 bitcoin
  vs_currencies:       usd
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

**On this node — set Retry On Fail: ON (3 tries, 5000ms wait)**
CoinGecko's free tier occasionally rate limits. Retry handles it silently.

---

## Step 4 — Extract Clean Fields

Add an **Edit Fields (Set)** node:

```
price:      {{ $json.bitcoin.usd }}
change_24h: {{ $json.bitcoin.usd_24h_change }}
timestamp:  {{ $now.toISO() }}
```

Keep Only Set Fields: ON

---

## Step 5 — Compute Severity

Add a **Code** node (Run Once Per Item):

```javascript
const price = $json.price;
const change = $json.change_24h;
const absChange = Math.abs(change);
const direction = change > 0 ? '📈' : '📉';

let severity;
let alertMessage;

if (absChange >= 10) {
  severity = 'critical';
  alertMessage = `🚨 *CRITICAL: Bitcoin ${change > 0 ? 'surged' : 'crashed'} ${absChange.toFixed(2)}% in 24h!*`;
} else if (absChange >= 5) {
  severity = 'warning';
  alertMessage = `⚠️ *WARNING: Bitcoin ${change > 0 ? 'up' : 'down'} ${absChange.toFixed(2)}% in 24h*`;
} else {
  severity = 'ok';
  alertMessage = `✅ Bitcoin price stable (${change.toFixed(2)}% in 24h)`;
}

const priceFormatted = `$${price.toLocaleString('en-US', { maximumFractionDigits: 2 })}`;

return {
  price,
  change_24h: change,
  absChange,
  severity,
  direction,
  alertMessage,
  priceFormatted,
  timestamp: $json.timestamp
};
```

---

## Step 6 — Route by Severity

Add a **Switch** node:

```
Mode: Rules

Rule 1: $json.severity  equals  critical  → Output 1
Rule 2: $json.severity  equals  warning   → Output 2
Rule 3: $json.severity  equals  ok        → Output 3
```

---

## Step 7 — Telegram Alerts

### Critical branch (Output 1):

Add a **Telegram** node:
```
Credential:  Telegram - BTC Monitor Bot
Operation:   Send Message
Chat ID:     YOUR_CHAT_ID
Text:
{{ $json.alertMessage }}

💰 *Price:* {{ $json.priceFormatted }}
📊 *24h Change:* {{ $json.change_24h.toFixed(2) }}%
🕐 *Time (IST):* {{ $now.toFormat('dd MMM HH:mm') }}

Parse Mode: Markdown
```

### Warning branch (Output 2):

Add another **Telegram** node with the same config — the message formatting
already handles the severity emoji via `$json.alertMessage`.

### OK branch (Output 3):

Add a **No Operation** node — no alert for stable prices.

---

## Step 8 — Log Every Check

Reconnect all 3 branches with a **Merge** node (Append mode), then log to
**Google Sheets** (free, no credit card) or a simple HTTP Request to any
logging endpoint.

### Option A — Google Sheets (recommended)

1. Create a Google Sheet with columns:
   `Timestamp | Price | Change 24h | Severity | Message`
2. Add Google OAuth2 credential in n8n
3. Add **Google Sheets** node → Append Row:
```
Spreadsheet:  (select yours)
Sheet:        Sheet1
Column:       A = {{ $json.timestamp }}
Column:       B = {{ $json.priceFormatted }}
Column:       C = {{ $json.change_24h }}
Column:       D = {{ $json.severity }}
Column:       E = {{ $json.alertMessage }}
```

### Option B — Log via Telegram to a separate channel/group

Create a second Telegram group for logs, get its Chat ID, send all records there:
```
Text: [LOG] {{ $json.severity }} | {{ $json.priceFormatted }} | {{ $json.change_24h.toFixed(2) }}% | {{ $now.toFormat('HH:mm') }}
```

---

## Step 9 — Error Handler Workflow

Create a new workflow: **"Error Handler"**

```
[Error Trigger]
      │
      ▼
[Telegram: Send Message]
Chat ID: YOUR_CHAT_ID
Text:
❌ *Workflow Failed*

*Workflow:* {{ $json.workflow.name }}
*Error:* {{ $json.execution.error.message }}
*Node:* {{ $json.execution.error.node.name }}
*Execution ID:* {{ $json.execution.id }}
*Time:* {{ $now.toFormat('dd MMM HH:mm') }}

Parse Mode: Markdown
```

Then in the BTC Price Monitor workflow:
- Workflow Settings → Error Workflow → select "Error Handler"

---

## Step 10 — Activate and Test

1. Save both workflows
2. Click **Execute Workflow** on BTC Price Monitor
3. Check your Telegram — you should receive a message within seconds
4. Check your Google Sheets log — a row should appear
5. Toggle **Active: ON** — it now runs every 5 minutes automatically

---

## Making It Production-Grade

### Prevent alert spam — cooldown pattern

```javascript
// In the Code node, after computing severity:
const lastAlertTime = $vars.LAST_ALERT_TIME;
const lastAlertSeverity = $vars.LAST_ALERT_SEVERITY;

if (lastAlertTime && lastAlertSeverity === severity && severity !== 'ok') {
  const minutesSince = $now.diff(DateTime.fromISO(lastAlertTime), 'minutes').minutes;
  if (minutesSince < 30) {
    return { ...(above return object), suppress: true };
  }
}
return { ...(above return object), suppress: false };
```

Add an IF node after the Switch: `suppress === true` → skip Telegram → log only.

---

### Monitor multiple coins simultaneously

Replace the single HTTP Request with a data-driven approach:

```javascript
// Code node before HTTP Request — returns one item per coin
return [
  { json: { coin: 'Bitcoin',  id: 'bitcoin',  threshold_warn: 5,  threshold_crit: 10 } },
  { json: { coin: 'Ethereum', id: 'ethereum', threshold_warn: 7,  threshold_crit: 15 } },
  { json: { coin: 'Solana',   id: 'solana',   threshold_warn: 10, threshold_crit: 20 } },
];
```

HTTP Request runs once per item (3 API calls). Use `$json.id` in the URL:
```
URL: https://api.coingecko.com/api/v3/simple/price?ids={{ $json.id }}&vs_currencies=usd&include_24hr_change=true
```

Use `$json.threshold_warn` and `$json.threshold_crit` in the severity Code node
instead of hardcoded values.

---

## What You've Applied

| Concept | Where |
|---------|-------|
| Schedule Trigger | Every 5 minutes |
| HTTP Request + Retry | CoinGecko API |
| Edit Fields (Set) | Extract clean fields |
| Code node | Multi-step severity logic |
| Switch node | Route by severity |
| Merge node | Reconnect branches for logging |
| Telegram node | Alert messages |
| Google Sheets node | Audit log |
| Error Workflow | Catch total failures |
| Expressions | Formatting throughout |

---

## Check Your Understanding — Q&A

### Q1. The CoinGecko API returns a 429. Your workflow fails. How should you handle this at the node level?

**Answer:** On the HTTP Request node: enable **Retry On Fail** (3 tries, wait 30,000ms between retries — 429 means rate limited, not a transient blip, so wait longer). Also check for a `Retry-After` header using Full Response mode — use that value as the wait time if present. For a price monitoring workflow that fires every 5 minutes, a single 429 is usually fine to retry 2–3 times; if it keeps failing, the Error Workflow fires and Telegram alerts you.

---

### Q2. You want to add a cooldown so you don't send the same severity alert more than once every 30 minutes. How do you implement this in n8n?

**Answer:** After computing severity, check `$vars.LAST_ALERT_TIME` and `$vars.LAST_ALERT_SEVERITY` in the Code node. If the same severity was alerted less than 30 minutes ago, set `suppress: true`. Add an IF node: `suppress === true` → skip Telegram → log only. Update the vars after sending via the n8n Variables API or a Set Variables node.

---

### Q3. Your Google Sheets log is filling up with "ok" status records every 5 minutes. How do you reduce noise without losing alert history?

**Answer:** Add an IF node before the Sheets node: only log when `severity !== 'ok'`. This keeps the log clean — only warnings and criticals are recorded. The execution history in n8n still has every run for debugging. Signal-only logs are far more useful than noise-filled ones.

---

### Q4. How would you extend this to monitor 5 coins simultaneously?

**Answer:** Add a Code node before the HTTP Request returning one item per coin (with its id and thresholds). The HTTP Request runs once per item (5 calls). The severity Code node uses `$json.threshold_warn` and `$json.threshold_crit` from the item instead of hardcoded values. Include `$json.coin` in the Telegram message. Adding a 6th coin = adding one object to the Code node array — no workflow restructuring needed.

---

## Next Lesson

**[Lesson 15 →](15-real-project-lead-enrichment.md)** — Lead enrichment pipeline:
receive new contacts via a webhook, enrich with a free API, store results,
and send a Telegram notification — all with idempotency and error handling.
