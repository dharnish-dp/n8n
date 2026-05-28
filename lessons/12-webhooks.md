# Lesson 12 — Webhooks: Receiving Events from the World

## Goal
Master webhooks in n8n. Build a production-grade webhook receiver that
validates signatures, handles idempotency, responds correctly, and works
with GitHub, Stripe, and Typeform.

---

## What is a Webhook (Engineer's Definition)

A webhook is an HTTP endpoint that your n8n instance exposes. External services
POST data to this URL when events happen. n8n receives the request and starts
a workflow.

```
External Service        n8n Webhook
(GitHub, Stripe, etc.)
        │
        │  POST https://your-n8n.com/webhook/<id>
        │  Body: { "event": "payment.succeeded", "amount": 9900, ... }
        │
        ▼
[Webhook Node]
        │
        ▼
[Your workflow processes the event]
```

The webhook node is always the **first node** — it's the trigger.

---

## Test URL vs Production URL

n8n gives you two webhook URLs for every Webhook trigger node:

```
Test URL:       http://localhost:5678/webhook-test/<workflow-id>
Production URL: http://localhost:5678/webhook/<workflow-id>
```

| | Test URL | Production URL |
|--|----------|----------------|
| When active | Only when you click "Listen for Test Event" in the editor | Always, when workflow is Active |
| Purpose | Development / testing | Live production traffic |
| Execution mode | Shows data in editor | Saves to execution history |

**Development workflow:**
1. Use Test URL + "Listen" button to capture a real event
2. Pin that event data in the Webhook node
3. Build the rest of the workflow using pinned data
4. When done, activate the workflow → use Production URL

---

## Webhook Node Configuration

```
HTTP Method:  POST (most common), GET, PUT, PATCH, DELETE, HEAD
Path:         (custom suffix, e.g. "github-events" → /webhook/github-events)
Authentication: None | Header Auth | Basic Auth | JWT
Response:     Immediately | When Last Node Finishes | Using 'Respond to Webhook' Node
```

### Response modes explained:

**Immediately:** Sends `200 OK` as soon as the webhook receives the request,
before your workflow runs. The caller gets a fast response. Use when the caller
doesn't care about the result.

**When Last Node Finishes:** Waits until the entire workflow completes, then
sends the last node's output as the response. The caller may timeout if your
workflow is slow. Use for request/response patterns.

**Using 'Respond to Webhook' Node:** You control exactly what to send back and
when. Add a `Respond to Webhook` node anywhere in the workflow. Best for:
- Sending a quick acknowledgment, then processing asynchronously
- Custom response bodies
- Different responses based on conditions

---

## Accessing Webhook Data

The Webhook node outputs one item containing:
```json
{
  "headers": {
    "content-type": "application/json",
    "x-github-event": "push",
    "x-hub-signature-256": "sha256=abc123..."
  },
  "params": {},
  "query": { "utm_source": "email" },
  "body": {
    "ref": "refs/heads/main",
    "pusher": { "name": "alice" },
    "commits": [...]
  }
}
```

Access in expressions:
```javascript
$json.body.ref                // → "refs/heads/main"
$json.headers['x-github-event']  // → "push"
$json.query.utm_source        // → "email"
```

---

## Validating Webhook Signatures

External services sign webhook payloads so you can verify they came from
the real service (not an attacker spoofing events).

### GitHub Webhook Signature Validation

GitHub sends: `X-Hub-Signature-256: sha256=<hmac>`

Validate in a Code node:
```javascript
const crypto = require('crypto');

const secret = $vars.GITHUB_WEBHOOK_SECRET;  // your webhook secret
const signature = $json.headers['x-hub-signature-256'];
const payload = JSON.stringify($json.body);

const expectedSig = 'sha256=' + crypto
  .createHmac('sha256', secret)
  .update(payload)
  .digest('hex');

if (signature !== expectedSig) {
  throw new Error('Invalid webhook signature — possible attack');
}

// Signature valid, proceed
return { verified: true, event: $json.headers['x-github-event'] };
```

### Stripe Webhook Signature Validation

Stripe sends: `Stripe-Signature: t=timestamp,v1=signature`

```javascript
const crypto = require('crypto');

const secret = $vars.STRIPE_WEBHOOK_SECRET;
const sigHeader = $json.headers['stripe-signature'];
const payload = JSON.stringify($json.body);

// Parse timestamp and signature
const parts = sigHeader.split(',');
const timestamp = parts.find(p => p.startsWith('t=')).split('=')[1];
const sig = parts.find(p => p.startsWith('v1=')).split('=')[1];

// Verify: HMAC-SHA256(timestamp + "." + payload)
const expectedSig = crypto
  .createHmac('sha256', secret)
  .update(`${timestamp}.${payload}`)
  .digest('hex');

if (sig !== expectedSig) {
  throw new Error('Invalid Stripe signature');
}

// Check timestamp is recent (within 5 minutes)
const age = Math.floor(Date.now() / 1000) - parseInt(timestamp);
if (age > 300) {
  throw new Error('Webhook timestamp too old — possible replay attack');
}

return { verified: true, type: $json.body.type };
```

---

## Idempotency — Don't Process the Same Event Twice

External services may retry failed webhooks. You'll receive the same event
multiple times. Your workflow must be idempotent.

**The pattern:**

Every event has a unique ID. Before processing:
1. Check if you've seen this ID before
2. If yes → return early (already processed)
3. If no → process, then save the ID

```
[Webhook]
    │
    ▼
[Code: extract event ID]
id = $json.body.id or $json.headers['x-webhook-id']
    │
    ▼
[Postgres: SELECT * FROM processed_webhooks WHERE event_id = $json.eventId]
    │
    ▼
[IF: results.length > 0 (already processed)]
  True: [Respond: 200 "Already processed"]
  False: [Continue processing]
         │
         ▼
         [... your workflow ...]
         │
         ▼
         [Postgres: INSERT INTO processed_webhooks (event_id, processed_at)]
         │
         ▼
         [Respond: 200 "Processed"]
```

---

## Real-World Example: GitHub PR Notification

Build a workflow that posts to Slack whenever a PR is merged.

### Step 1 — Configure GitHub webhook

In GitHub repo → Settings → Webhooks → Add webhook:
```
Payload URL: https://your-n8n.com/webhook/<id>
Content type: application/json
Secret: (generate a random string, save as $vars.GITHUB_WEBHOOK_SECRET)
Events: Pull requests
```

### Step 2 — Build the n8n workflow

```
[Webhook: POST /webhook/github-pr]
        │
        ▼
[Code: validate signature + filter event]
const event = $json.headers['x-github-event'];
const action = $json.body.action;

// Only process "closed" events where merged = true
if (event !== 'pull_request' || action !== 'closed' || !$json.body.pull_request.merged) {
  return { skip: true };
}

return {
  prTitle: $json.body.pull_request.title,
  prUrl: $json.body.pull_request.html_url,
  author: $json.body.pull_request.user.login,
  repo: $json.body.repository.full_name,
  mergedAt: $json.body.pull_request.merged_at
};
        │
        ▼
[IF: $json.skip === true]
  True: [Respond to Webhook: 200 "Skipped"]
  False: [Continue]
        │
        ▼
[Slack: Send Message to #deploys]
Message:
"✅ PR merged in {{ $json.repo }}
*{{ $json.prTitle }}*
By: {{ $json.author }}
{{ $json.prUrl }}"
        │
        ▼
[Respond to Webhook: 200 "OK"]
```

---

## Testing Webhooks Locally with ngrok

Your local n8n is at `localhost:5678`. External services can't reach it.
Use ngrok to create a public tunnel.

```bash
# Install ngrok
brew install ngrok

# Expose your local n8n
ngrok http 5678

# Output:
# Forwarding  https://abc123.ngrok.io -> http://localhost:5678
```

Use `https://abc123.ngrok.io/webhook/<id>` in GitHub/Stripe/Typeform.

**Important:** ngrok URL changes each time you restart ngrok (free tier).
For stable URLs during development, use ngrok's fixed domain (paid) or
configure your n8n with `WEBHOOK_URL=https://your-ngrok-url.io`.

---

## Summary

- Webhook node exposes an HTTP endpoint; external services POST events to it
- Test URL (dev) vs Production URL (active workflows)
- Always validate webhook signatures in production — don't trust unsigned webhooks
- Implement idempotency to handle duplicate deliveries
- Use ngrok to expose local n8n to external services during development
- "Respond to Webhook" node gives you full control over the HTTP response

---

## Next Lesson

**[Lesson 13 →](13-sub-workflows.md)** — Sub-workflows and modular design:
building reusable automation components, calling them from multiple parent
workflows, and applying single-responsibility to your automation architecture.
