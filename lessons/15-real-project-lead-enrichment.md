# Lesson 15 — Real Project 2: Lead Enrichment Pipeline

## Goal
Build a production-grade lead enrichment pipeline. A webhook receives new
signups, enriches them with company data via a free API, deduplicates by
email, stores results in Google Sheets, and sends a Telegram notification.
Applies: webhooks, idempotency, HTTP Request, error handling, sub-workflows.

---

## What We're Building

```
[Webhook: POST /new-signup]
          │
          ▼
[Code: validate + extract fields]
          │
          ▼
[HTTP Request: check if email already processed]  ← idempotency
          │
          ▼
[IF: duplicate?]
  True: [Respond: 200 "Already processed"]
  False:
          │
          ▼
[HTTP Request: enrich via Hunter.io / Clearbit free tier]
          │
          ▼
[Set: combine signup + enriched data]
          │
          ▼
[Google Sheets: append row]
          │
          ▼
[Telegram: notify team]
          │
          ▼
[Respond to Webhook: 200 OK]
```

---

## APIs Used (All Free)

| Service | Purpose | Free tier |
|---------|---------|-----------|
| [Hunter.io](https://hunter.io) | Email validation + company lookup | 25 req/month |
| [Abstraction API](https://www.abstractapi.com/api/email-validation-verification-api) | Email validation | 100 req/month |
| Google Sheets | Store enriched leads | Free |
| Telegram | Team notification | Free |

We'll use **Hunter.io** (email verification + company data). Sign up for a
free API key at hunter.io.

---

## Step 1 — Set Up the Webhook

1. New workflow → name: **"Lead Enrichment Pipeline"**
2. Add a **Webhook** trigger node
3. Method: `POST`
4. Path: `new-signup`
5. Response Mode: **Using 'Respond to Webhook' Node**

Your webhook URL will be:
```
Test:       http://localhost:5678/webhook-test/new-signup
Production: http://localhost:5678/webhook/new-signup
```

---

## Step 2 — Validate and Extract Fields

Add a **Code** node (Run Once Per Item):

```javascript
const body = $json.body;

// Basic validation
if (!body.email || !body.email.includes('@')) {
  throw new Error(`Invalid email: ${body.email}`);
}

// Extract and normalise
return {
  email: body.email.toLowerCase().trim(),
  firstName: body.firstName || body.first_name || '',
  lastName: body.lastName || body.last_name || '',
  source: body.source || 'unknown',
  submittedAt: $now.toISO(),
  eventId: body.eventId || $json.headers['x-webhook-id'] || $now.toMillis().toString()
};
```

---

## Step 3 — Idempotency Check

Use Google Sheets as a simple processed-events store. Before enriching,
check if this email was already processed.

Add an **HTTP Request** node to query Google Sheets:

```
Method: GET
URL: https://sheets.googleapis.com/v4/spreadsheets/YOUR_SHEET_ID/values/Leads!A:A
Authentication: Google OAuth2 API
```

Then a **Code** node (All Items mode) to check if the email exists:

```javascript
const incoming = $('Validate Fields').item.json.email;
const existingEmails = $json.values?.flat() || [];

return [{
  json: {
    ...$('Validate Fields').item.json,
    isDuplicate: existingEmails.includes(incoming)
  }
}];
```

Add an **IF** node: `isDuplicate === true`
- **True branch**: Respond to Webhook → 200 "Already processed" → stop
- **False branch**: continue to enrichment

---

## Step 4 — Enrich the Lead

Add an **HTTP Request** node to call Hunter.io:

```
Method: GET
URL: https://api.hunter.io/v2/email-finder

Query Parameters:
  domain:   {{ $json.email.split('@')[1] }}
  api_key:  YOUR_HUNTER_API_KEY

Continue On Fail: ON  ← enrichment failure shouldn't block lead capture
```

Expected response:
```json
{
  "data": {
    "first_name": "Alice",
    "last_name": "Smith",
    "position": "CTO",
    "company": "Acme Corp",
    "linkedin_url": "https://linkedin.com/in/alice",
    "confidence": 87
  }
}
```

---

## Step 5 — Combine Signup + Enriched Data

Add an **Edit Fields (Set)** node, Keep Only Set Fields: ON:

```
email:        {{ $json.email }}
firstName:    {{ $json.data?.first_name || $('Validate Fields').item.json.firstName || 'Unknown' }}
lastName:     {{ $json.data?.last_name  || $('Validate Fields').item.json.lastName  || '' }}
company:      {{ $json.data?.company    || 'Unknown' }}
jobTitle:     {{ $json.data?.position   || 'Unknown' }}
confidence:   {{ $json.data?.confidence ?? 0 }}
source:       {{ $('Validate Fields').item.json.source }}
enriched:     {{ !!$json.data }}
submittedAt:  {{ $('Validate Fields').item.json.submittedAt }}
```

---

## Step 6 — Store in Google Sheets

Add a **Google Sheets** node → Append Row:

```
Operation:    Append
Spreadsheet:  (select your spreadsheet)
Sheet:        Leads
Columns:
  A = email
  B = firstName
  C = lastName
  D = company
  E = jobTitle
  F = source
  G = enriched
  H = confidence
  I = submittedAt
```

---

## Step 7 — Telegram Notification

Add a **Telegram** node:

```
Chat ID: YOUR_CHAT_ID
Text:
🆕 *New Lead*

👤 {{ $json.firstName }} {{ $json.lastName }}
📧 {{ $json.email }}
🏢 {{ $json.company }} — {{ $json.jobTitle }}
📌 Source: {{ $json.source }}
{{ $json.enriched ? '✅ Enriched' : '⚠️ Not enriched' }} (confidence: {{ $json.confidence }}%)

Parse Mode: Markdown
```

---

## Step 8 — Respond to Webhook

Add a **Respond to Webhook** node at the end:

```
Response Code: 200
Response Body: { "status": "ok", "email": "{{ $json.email }}" }
```

---

## Step 9 — Error Handler

Create a separate **"Error Handler"** workflow (or reuse the one from Lesson 14):

```
[Error Trigger]
      │
      ▼
[Telegram: send alert]
❌ *Lead Pipeline Failed*
Workflow: {{ $json.workflow.name }}
Error: {{ $json.execution.error.message }}
Execution: {{ $json.execution.id }}
```

Set it as the Error Workflow in workflow settings.

---

## Step 10 — Test It

With the workflow open, click **"Listen for test event"** on the Webhook node,
then send a test POST from your terminal:

```bash
curl -X POST http://localhost:5678/webhook-test/new-signup \
  -H "Content-Type: application/json" \
  -d '{
    "email": "alice@acme.com",
    "firstName": "Alice",
    "lastName": "Smith",
    "source": "website"
  }'
```

Watch the nodes execute in the editor. Check:
1. Telegram — alert received?
2. Google Sheets — row added?
3. Send the same request again — does idempotency block it?

---

## Production Improvements

### Rate limiting Hunter.io (25 req/month free)

Cache enrichment results in Google Sheets — before calling Hunter.io,
check if you already enriched this domain:

```javascript
// Code node: check if domain was already enriched
const domain = $json.email.split('@')[1];
const cached = existingRows.find(r => r.domain === domain);
if (cached) return { ...$json, ...cached, fromCache: true };
return { ...$json, fromCache: false };
```

### Batch enrichment for existing contacts

```
[Schedule: daily at 9am]
      │
      ▼
[Google Sheets: get rows where enriched = false]
      │
      ▼
[SplitInBatches: 5 at a time]
      │
      ▼
[HTTP Request: Hunter.io]
      │
      ▼
[Wait: 2 seconds]   ← avoid rate limits
      │
      ▼
[Google Sheets: update row]
      │
      ▼ (loop back to SplitInBatches)
```

---

## What You've Applied

| Concept | Where |
|---------|-------|
| Webhook trigger | Receive new signups |
| Respond to Webhook | Acknowledge caller |
| Idempotency | Check before processing |
| HTTP Request (enrichment) | Hunter.io API |
| Continue On Fail | Enrichment can fail gracefully |
| Edit Fields (Set) | Combine + clean data |
| Google Sheets node | Store results |
| Telegram node | Team notification |
| Error Workflow | Catch total failures |
| Code node | Validation + dedup logic |

---

## Check Your Understanding — Q&A

### Q1. Why do you check for duplicates BEFORE enriching rather than after?

**Answer:** Enrichment costs API quota — Hunter.io's free tier gives only 25 requests/month. If you enrich first and then detect a duplicate, you've permanently burned the API call for data you'll immediately discard. The same principle applies to any irreversible or quota-limited operation: always gate on idempotency before you commit resources. The correct order is always validate → deduplicate → enrich → store. This also applies to side effects like sending emails or writing to a CRM — once sent, you can't unsend. Dedup-first prevents double-notifications and double-records, not just wasted API calls. The general rule: order operations from cheapest/most-reversible to most-expensive/least-reversible.

---

### Q2. The Hunter.io API returns a 404 for an unknown domain. With Continue On Fail OFF, what happens? With it ON?

**Answer:** With Continue On Fail **OFF** — the workflow halts at the HTTP Request node. The execution is marked failed, the lead is never written to Sheets, and the caller's webhook receives no response (their request eventually times out). Any team notification is lost. With it **ON** — the node passes forward a synthetic error item: `{ error: "Not Found", statusCode: 404 }`. The downstream Set node handles the missing `data` field gracefully using `?.` and `??` defaults — `company` becomes `"Unknown"`, `enriched` becomes `false`, `confidence` becomes `0`. The lead is stored and the Telegram notification fires, making clear that enrichment failed without stopping the pipeline. The core insight: enrichment is a *nice-to-have* that should never block lead capture. Continue On Fail is the correct tool whenever a node is optional in the success path.

---

### Q3. Your webhook receives 50 simultaneous signups from a campaign. How does n8n handle this and what could go wrong?

**Answer:** n8n processes each webhook invocation as a separate execution running concurrently (in main mode, up to the available Node.js thread capacity). The problem is the idempotency check: it is a **read-modify-write** operation with no locking. Two executions for `alice@acme.com` can both read the Sheets column, both find "not found", both proceed to enrich, both write a row — resulting in a duplicate record despite the dedup logic. This is a classic TOCTOU (time-of-check/time-of-use) race. The fix is to replace the Sheets-based dedup with a **Postgres `INSERT ... ON CONFLICT DO NOTHING`** on a column with a unique index on `email`. The database enforces uniqueness atomically — the second insert is silently rejected at the DB level, so only one execution proceeds. Google Sheets has no atomic operations, making it unsafe as a dedup store under concurrent load. At truly high volumes, also enable n8n queue mode so executions are serialized through Redis workers rather than running all at once.

---

### Q4. How do you test the idempotency logic without sending real webhooks?

**Answer:** Three approaches in increasing rigour: (1) **Pin data on the Webhook node** — after capturing one real test event, click the pin icon. The workflow now uses that captured payload on every manual run without needing a live HTTP call. Modify the pinned email to one that already exists in your Sheets to simulate a duplicate. (2) **Hardcode in the Code node** — temporarily override `isDuplicate: true` directly in the validation Code node to force the duplicate branch regardless of Sheets content. This tests the IF routing and Respond-to-Webhook path in isolation. (3) **Manual Trigger variant** — create a copy of the workflow with a Manual Trigger and hardcoded test items instead of a real Webhook. This version is safe to run any time, never needs an HTTP client, and can be committed as a self-contained test fixture. Avoid relying only on live `curl` calls — they depend on network, real credentials, and Sheets state, making tests fragile and hard to repeat.

---

### Q5. The Google Sheets append works in testing but fails in production with a `429 Too Many Requests` error during a campaign spike. What is the root cause and how do you fix it?

**Answer:** The Google Sheets API enforces per-project write quotas (default: 60 writes/minute per project, 300 requests/minute per user). During a campaign spike with 50 simultaneous signups, each execution hits the Sheets API at the same time, collectively exceeding the quota. The `429` causes those executions to fail and leads to be dropped. The fix has two parts: (1) **Short term** — add a `Wait` node (1-2 seconds) before the Sheets append to spread writes over time, and enable **Continue On Fail** on the Sheets node with a retry Code node that backs off and retries up to 3 times. (2) **Proper fix** — buffer incoming leads into a Postgres table first (a single fast insert, no quota), then process them in batches via a scheduled workflow that appends to Sheets at a controlled rate (e.g. `SplitInBatches` of 10 with a 1-second wait between batches). The webhook returns `200 OK` immediately after the DB insert — the caller is never blocked by downstream quota issues. This is the **ingest/process separation** pattern: accept fast, process slowly, never let a downstream rate limit become a user-facing failure.

---

## Next Lesson

**[Lesson 16 →](16-real-project-ai-agent.md)** — Build an AI agent with memory:
a Telegram bot powered by Claude/GPT-4 that remembers conversation history,
answers questions about your data, and calls n8n workflows as tools.
