# Lesson 07 — HTTP Request: The Universal API Connector

## Goal
Master the HTTP Request node completely. It's the most used node in n8n —
every API that doesn't have a dedicated integration uses this. By the end
you'll be able to integrate any REST API in minutes.

---

## Why the HTTP Request Node is Different

The HTTP Request node is not just "make a GET request." It handles:
- Every HTTP method
- Every auth type (API key, Bearer, OAuth2, Basic, Digest, AWS SigV4)
- Query parameters, headers, body (JSON, form data, raw, binary)
- Pagination (automatic multi-page fetching)
- File uploads and downloads
- SSL verification
- Full response (status + headers + body) vs just body
- Batching with delays (for rate-limited APIs)

Understanding this node deeply means you can integrate **any** REST API.

---

## Basic Configuration

```
Method:         GET | POST | PUT | PATCH | DELETE | HEAD | OPTIONS
URL:            https://api.example.com/v1/endpoint
Authentication: (covered below)
Send Headers:   toggle to add custom headers
Send Query:     toggle to add URL parameters
Send Body:      toggle for POST/PUT/PATCH

Response format: JSON (default auto-parse), String, Binary
```

---

## Authentication Methods

### No Auth
```
Use for: public APIs, test endpoints
No configuration needed
```

### API Key
```
Where to send:
  - Header: "X-API-Key: your-key" (most common)
  - Query parameter: ?api_key=your-key
  - Cookie: (rare)

Config:
  Name:  X-API-Key   (or whatever the API requires)
  Value: {{ $credentials.apiKey }}
```

### Bearer Token
```
Sends: "Authorization: Bearer <token>"
Use for: JWT tokens, personal access tokens

Config:
  Token: (your token value)
Note: Creates credential in n8n for reuse
```

### Basic Auth
```
Sends: "Authorization: Basic <base64(user:password)>"
Use for: older APIs, some services

Config: username + password in credential
```

### OAuth2
```
Most complex but most secure. Used by:
  Google APIs, GitHub, Salesforce, HubSpot, etc.

n8n handles the OAuth dance:
  - You configure client ID + secret
  - n8n redirects for authorization
  - n8n stores and refreshes tokens automatically

Config: set up credential first via Settings → Credentials
```

### Header Auth (Custom)
```
Use for: APIs with proprietary auth headers
  Example: some APIs need both "X-App-Id" and "X-App-Key"

Config:
  Name:  X-App-Id
  Value: your-app-id
  (Add second header via HTTP Request node headers)
```

---

## Request Body Types

### JSON Body (most common for REST APIs)

```json
{
  "name": "{{ $json.name }}",
  "email": "{{ $json.email }}",
  "score": {{ $json.score }}
}
```

Or use "Specify Body" → "Using JSON" and paste/build the JSON directly.
Expressions work inside.

### Form Data (application/x-www-form-urlencoded)
```
For APIs that expect HTML form-style data (older APIs, auth endpoints)
Add key-value pairs in the Form Parameters section
```

### Multipart Form Data (file uploads)
```
For uploading files. Add:
  - Text fields: add as parameters
  - File fields: reference binary data from previous node
    Field name: "file"
    Input Data Field Name: "data"  (the binary property name)
```

### Raw / Custom body
```
For XML, GraphQL, or non-standard formats:
  Content Type: application/xml
  Body: <xml>{{ $json.value }}</xml>
```

---

## Full Response Mode

By default, n8n gives you just the parsed response body.
Enable "Full Response" to get:

```json
{
  "statusCode": 200,
  "headers": {
    "content-type": "application/json",
    "x-rate-limit-remaining": "95"
  },
  "body": {
    "id": 123,
    "name": "Alice"
  }
}
```

Use full response when:
- You need to check the HTTP status code
- You need to read response headers (rate limit remaining, pagination cursors)
- You need to detect 4xx/5xx errors explicitly

---

## Error Handling

By default, the HTTP Request node **throws an error** on 4xx/5xx responses —
this stops the workflow.

To handle HTTP errors gracefully:

**Option 1:** Enable "Continue on Fail" in node settings
```
The node outputs an item with { statusCode, error } instead of throwing.
Check downstream: IF $json.statusCode >= 400 → handle error
```

**Option 2:** Use "Full Response" + IF node
```
HTTP Request (Full Response ON) → IF (statusCode < 300) → success branch
                                                         → error branch
```

**Option 3:** Set node to ignore 404s specifically
```
Sometimes 404 just means "doesn't exist yet" — not a real error.
Handle: IF ($json.statusCode === 404) → create it | else → update it
```

---

## Pagination — Fetching All Pages Automatically

Many APIs paginate results (e.g., return 100 items at a time).
n8n has built-in pagination support in the HTTP Request node.

Enable: "Add Option" → "Pagination"

### Cursor-based pagination (most modern APIs)
```
Type: Cursor-based pagination
Pagination Expression:
  {{ $response.body.next_cursor }}
  (the field in the response that contains the next page cursor)

Request Body Parameter:
  cursor: {{ $pageData.nextPageToken }}

Stop Condition:
  {{ !$response.body.next_cursor }}  (stop when no cursor)
```

### Offset-based pagination
```
Type: Offset-based pagination
Limit Query Parameter: limit
Offset Query Parameter: offset
Limit Value: 100

Stop when: response length < limit (last page)
```

### Page-based pagination
```
Type: Page-based pagination
Page Token Query Parameter: page
Root Property: data (where the items are in the response)
```

---

## Hands-On: Build a Real API Integration

We'll integrate the GitHub API to fetch all open issues from a repository.

### Step 1 — GitHub API basics
GitHub API docs: https://docs.github.com/en/rest

Base URL: `https://api.github.com`
Auth: Bearer token (create at github.com → Settings → Developer settings → PAT)

### Step 2 — Fetch issues

Create a credential:
1. Settings → Credentials → New → Header Auth
2. Name: "GitHub Token"
3. Header Name: `Authorization`
4. Header Value: `token YOUR_GITHUB_TOKEN`

Configure HTTP Request:
```
Method: GET
URL: https://api.github.com/repos/n8n-io/n8n/issues
Authentication: (use the credential you created)
Send Headers:
  Accept: application/vnd.github.v3+json
Query Parameters:
  state: open
  per_page: 30
```

### Step 3 — Handle multiple pages

The GitHub API paginates. Add pagination:
```
Pagination Type: Response Contains Next URL
Next URL Expression: {{ $response.headers['link'] }}
(GitHub uses the Link header for pagination)
```

Better approach — use a parameter to get a bigger page:
```
per_page: 100   (GitHub's max)
```

### Step 4 — Extract the fields you need

After the HTTP Request, add a **Split Out** node (to split the array into items),
then a **Set** node to pick only what you care about:

```
title:      {{ $json.title }}
number:     {{ $json.number }}
url:        {{ $json.html_url }}
author:     {{ $json.user.login }}
created_at: {{ $json.created_at }}
labels:     {{ $json.labels.map(l => l.name).join(', ') }}
```

---

## Rate Limiting — The Production Pattern

When calling APIs with rate limits, add delays between requests.

**For per-item calls (one API request per item):**
```
HTTP Request node → Settings → "Batch Size: 1" + "Batch Interval: 500"
```
This processes 1 item every 500ms = 2 requests per second.

**For loops (SplitInBatches pattern):**
```
SplitInBatches (10) → HTTP Request → Wait (1 second) → back to SplitInBatches
```

**Reading rate limit headers:**
```
Enable Full Response, then check:
  $json.headers['x-ratelimit-remaining']
  $json.headers['x-ratelimit-reset']

Use an IF node:
  IF remaining <= 5 → Wait until reset time → continue
```

---

## Downloading Files

When the API returns binary data (image, PDF, CSV):

```
HTTP Request:
  Response Format: Binary
  Output Binary Content Property: data  (stores it as $binary.data)
```

The file is now in `$binary.data` and can be:
- Uploaded to S3, Google Drive, Dropbox
- Attached to an email
- Processed by a PDF/image node
- Saved to disk via Write Binary File node

---

## Uploading Files

When you need to send a file to an API:

```
Previous node: Read Binary File (or Google Drive download, etc.)
                → outputs {binary: {data: {...}}}

HTTP Request:
  Method: POST
  Body Type: Multipart Form Data
  Add field:
    Type: File
    Field Name: file        (what the API expects)
    Input Binary Field: data (the binary property name)
```

---

## Common Patterns

### POST JSON with dynamic values
```
Method: POST
URL: https://api.crm.com/contacts
Body (JSON):
{
  "firstName": "{{ $json.first_name }}",
  "email": "{{ $json.email }}",
  "source": "n8n",
  "tags": {{ JSON.stringify($json.tags) }}
}
```

### Conditional request based on previous data
```
URL: {{ $json.type === 'user' ? 'https://api.com/users' : 'https://api.com/orgs' }}/{{ $json.id }}
```

### Pass auth token from a login response
```
Step 1: HTTP Request → POST /auth/login → response has { token: "..." }
Step 2: HTTP Request → set Header: Authorization = Bearer {{ $('Login').item.json.token }}
```

---

## Summary

- HTTP Request handles every HTTP method and auth type
- Use Full Response for status codes and headers
- Enable "Continue on Fail" to handle errors without stopping the workflow
- Built-in pagination for offset, cursor, and page-based APIs
- Rate limit with Batch Size/Interval or SplitInBatches + Wait
- Binary content (files) stored in `$binary.data`

---

## Next Lesson

**[Lesson 08 →](08-working-with-data.md)** — Set, Merge, Aggregate, Split Out,
and Filter in depth. Real patterns for the data manipulation tasks you face
in every production workflow.
