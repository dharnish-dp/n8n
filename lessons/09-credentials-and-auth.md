# Lesson 09 — Credentials & Authentication Patterns

## Goal
Understand how n8n stores and uses credentials securely. Learn every
authentication type, when to use each, and best practices so your API
keys never end up in workflow fields or execution logs.

---

## Why Credentials Matter

The #1 security mistake in automation: putting API keys directly in node fields.

```
❌ Wrong — key visible in execution history, workflow JSON, screenshots:
URL: https://api.example.com/data?api_key=sk-1234abcd

✅ Right — key stored encrypted, referenced by name:
Authentication: API Key (My OpenAI Credential)
```

n8n's credential system:
- Stores secrets **AES-256 encrypted** in the database
- Never appears in execution logs or node output
- Reusable across multiple workflows
- Manageable from one place — rotate a key once, all workflows updated

---

## Where Credentials Live

**Settings → Credentials** (left sidebar or top menu)

This is your central secret store. Every credential you create here is
available to any node that needs authentication.

---

## Creating a Credential — Step by Step

1. Open **Settings → Credentials**
2. Click **"+ Add Credential"**
3. Search for the service (e.g. "OpenAI", "GitHub", "Slack")
4. Fill in the required fields (API key, token, etc.)
5. Click **"Save"** — n8n encrypts and stores it

To use it in a node:
- Open any node that needs auth
- Find the **Credential** field
- Select the credential you created from the dropdown

---

## Authentication Types

### Type 1: API Key

The most common type. A static secret string the API requires.

```
Where it's sent (depends on the API):
  - Header:  X-API-Key: your-key
  - Header:  Authorization: ApiKey your-key
  - Query:   ?api_key=your-key

Examples: OpenAI, Airtable, SendGrid, CoinGecko (paid)
```

**In n8n:** Create a "Header Auth" credential
```
Name:  X-API-Key          (or whatever the API requires)
Value: your-actual-key
```

Or use a dedicated credential type if the service has one (e.g. "OpenAI API" credential — n8n pre-fills the header name for you).

---

### Type 2: Bearer Token

A token sent in the `Authorization` header.

```
Header: Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

Used by: GitHub PAT, many modern REST APIs, JWT-based auth.

**In n8n:** Create a "Header Auth" credential
```
Name:  Authorization
Value: Bearer your-token-here
```

Or use the dedicated "GitHub API" credential which handles this automatically.

---

### Type 3: Basic Auth

Username + password encoded in the Authorization header.

```
Header: Authorization: Basic base64(username:password)
```

Used by: older APIs, some internal services, HTTP Basic protected endpoints.

**In n8n:** Create a "Basic Auth" credential
```
User:     your-username
Password: your-password
```

n8n handles the base64 encoding automatically.

---

### Type 4: OAuth2

The most secure, most complex. Used by Google, Slack, HubSpot, Salesforce,
GitHub (for user-level access), and most enterprise APIs.

**How OAuth2 works:**
```
1. You configure client ID + client secret (from the service's developer portal)
2. n8n redirects you to the service's login page
3. You approve access
4. Service sends n8n an access token + refresh token
5. n8n stores both encrypted
6. When the access token expires, n8n uses the refresh token to get a new one
   automatically — you never have to re-authenticate
```

**In n8n:** Most popular services have a dedicated OAuth2 credential type
(e.g. "Google OAuth2 API", "Slack OAuth2 API"). You just fill in:
- Client ID
- Client Secret
- Then click "Connect" and log in

**Where to get Client ID + Secret:**
- Google: console.cloud.google.com → APIs & Services → Credentials
- GitHub: github.com → Settings → Developer Settings → OAuth Apps
- Slack: api.slack.com → Your Apps → OAuth & Permissions

---

### Type 5: Custom Header Auth

For APIs with non-standard auth headers or multiple headers required.

```
Example: some APIs require both an App ID and an App Key:
  X-App-Id:  your-app-id
  X-App-Key: your-app-key
```

**In n8n:** Use "Header Auth" credential for one header, then add the
second header directly in the HTTP Request node's "Send Headers" section.

Or use a **Generic Credential Type** in the HTTP Request node which lets
you define multiple headers as key-value pairs.

---

## Credential Scoping — Who Can Use It

In n8n with user management enabled:

| Scope | Meaning |
|-------|---------|
| **Personal** | Only you can use this credential |
| **Shared** | You share it with specific other users |

For team environments, share credentials so multiple people can build
workflows using the same API keys — without anyone seeing the actual key.

---

## Best Practices

### Never put secrets in node fields
```
❌ URL: https://api.com/data?token=sk-1234
✅ Use a credential, reference it in the Auth dropdown
```

### Name credentials clearly
```
❌ "API Key 1"
✅ "OpenAI - Production"
✅ "GitHub - dharnish PAT (read-only)"
✅ "Slack - #alerts Bot Token"
```

Include: service name, environment (prod/dev), scope (read-only, admin).

### One credential per environment
```
"OpenAI - Development"   ← points to a test/limited key
"OpenAI - Production"    ← points to the real key
```

Switch between them by changing the credential reference in the node.

### Rotate keys without breaking workflows
1. Get new API key from the service
2. Open Settings → Credentials → find the credential
3. Update the value
4. Save — all workflows using this credential now use the new key automatically

No workflow editing needed.

### Back up your encryption key
```bash
cat ~/.n8n/config
# Copy the "encryptionKey" value and store it securely
```

If you lose this key, you **cannot decrypt your stored credentials**.
The workflows still exist but all API connections are broken.

---

## Hands-On: Create and Use 3 Credential Types

### Exercise 1 — GitHub API (Bearer Token)

1. Go to github.com → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)
2. Generate new token → select scope: `repo` (read)
3. Copy the token
4. In n8n: Settings → Credentials → Add → search "Header Auth"
5. Name: `GitHub - Personal Token`
6. Header Name: `Authorization`
7. Header Value: `token ghp_your_token_here`
8. Save

Test it:
```
HTTP Request node:
  URL: https://api.github.com/user
  Authentication: (select your GitHub credential)
  Header: Accept = application/vnd.github.v3+json
```
Should return your GitHub profile.

---

### Exercise 2 — OpenWeatherMap (API Key in Query)

1. Register at openweathermap.org → get free API key
2. In n8n: Settings → Credentials → Add → "OpenWeather API"
   (or "Query Auth" → Name: `appid`, Value: `your-key`)
3. Use in HTTP Request → URL: `https://api.openweathermap.org/data/2.5/weather`

---

### Exercise 3 — Airtable (API Key / Personal Access Token)

1. Go to airtable.com → Account → Developer Hub → Personal Access Tokens
2. Create token with scope: `data.records:read`, `data.records:write`
3. In n8n: Search for "Airtable Personal Access Token" credential
4. Paste the token

Now you can use the Airtable node in any workflow without putting the
token in any field.

---

## The Generic Credential Type (Advanced)

When a service has no dedicated n8n credential but needs a custom setup,
use the HTTP Request node's built-in "Generic Credential Type":

```
Authentication: Generic Credential Type
Generic Auth Type: HTTP Header Auth / API Key In Query / etc.
```

This handles auth inline without creating a named credential. Useful for
one-off integrations, but prefer named credentials for anything reused.

---

## Summary

- Credentials are AES-256 encrypted — never put secrets in node fields
- 5 auth types: API Key, Bearer Token, Basic Auth, OAuth2, Custom Header
- OAuth2: n8n handles token refresh automatically
- Name credentials clearly: `Service - Environment - Scope`
- Back up `~/.n8n/config` — losing the encryption key means losing all credentials
- Rotate keys in one place — all workflows update automatically

---

## Check Your Understanding — Q&A

### Q1. Why should you never put an API key directly in a URL or node field?

**Answer:** Because it appears in execution history, workflow JSON exports,
and logs — anywhere someone can read the workflow, they can read the key.
Credentials are encrypted and never appear in any output or log.

---

### Q2. What does n8n do when an OAuth2 access token expires?

**Answer:** n8n automatically uses the stored refresh token to get a new
access token from the service — without any action from you. This is why
OAuth2 credentials don't break overnight like manually pasted tokens do.

---

### Q3. You need to rotate an API key used in 10 different workflows. How many places do you update?

**Answer:** **One** — update the credential in Settings → Credentials.
All 10 workflows reference the credential by name and automatically use
the new key. This is the core advantage of the credential system.

---

## Next Lesson

**[Lesson 10 →](10-code-node.md)** — The Code node: JavaScript power inside
n8n, both execution modes, all built-in variables, and real production patterns
for when expressions aren't enough.
