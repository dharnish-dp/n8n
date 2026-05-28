# n8n Cheat Sheet

## Expression Variables

```javascript
$json.fieldName                       // current item's field
$json.nested.field                    // nested access
$json.array[0]                        // array element
$json.field?.subfield                 // safe access (no crash if missing)
$json.field ?? 'default'              // default if null/undefined

$('Node Name').item.json.field        // from a specific named node
$('Node Name').first().json           // first item from a node
$('Node Name').last().json            // last item from a node
$('Node Name').all()                  // all items from a node (array)
$('Node Name').all().length           // count items from a node

$now                                  // current DateTime (Luxon)
$now.toISO()                          // ISO string
$now.toFormat('yyyy-MM-dd')           // formatted date
$now.minus({days: 7})                 // 7 days ago
$now.plus({hours: 24})                // 24 hours from now
$today                                // today at midnight

$workflow.id                          // workflow ID
$workflow.name                        // workflow name
$execution.id                         // current execution ID
$execution.mode                       // manual | trigger | webhook
$vars.MY_VAR                         // workflow variable
```

---

## Date Formats (Luxon)

| Code | Example |
|------|---------|
| `yyyy-MM-dd` | `2026-05-22` |
| `dd/MM/yyyy` | `22/05/2026` |
| `MM/dd/yyyy` | `05/22/2026` |
| `MMMM d, yyyy` | `May 22, 2026` |
| `EEE, MMM d` | `Fri, May 22` |
| `HH:mm:ss` | `14:30:00` |
| `h:mm a` | `2:30 PM` |
| `x` | Unix ms |

Parse dates: `DateTime.fromISO()` / `DateTime.fromFormat()` / `DateTime.fromMillis()`

---

## Common Cron Expressions

```
0 8 * * *          Every day at 8:00 AM
0 8 * * 1-5        Weekdays at 8:00 AM
0 */6 * * *        Every 6 hours
*/15 * * * *       Every 15 minutes
0 9 1 * *          First of every month at 9 AM
0 0 * * 0          Every Sunday at midnight
```

---

## HTTP Request Node — Quick Config

```
Method:          GET | POST | PUT | PATCH | DELETE
URL:             https://api.example.com/endpoint
Authentication:  Predefined Credential Type | Generic Credential | None
Body Type:       JSON | Form Data | Raw | Binary
Headers:         Add key-value pairs
Query Params:    Add key-value pairs
Response Type:   Autodetect (default) | JSON | String | Binary
```

---

## Node Keyboard Shortcuts

| Action | Shortcut |
|--------|----------|
| Save workflow | `Cmd+S` |
| Execute workflow | `Cmd+Enter` |
| Add node | `Tab` (on canvas) |
| Delete node | `Backspace` or `Delete` |
| Copy node | `Cmd+C` |
| Paste node | `Cmd+V` |
| Undo | `Cmd+Z` |
| Select all | `Cmd+A` |
| Zoom in/out | `+` / `-` |
| Fit workflow | `Cmd+Shift+H` |

---

## Key Node Reference

| Node | Purpose |
|------|---------|
| Manual Trigger | Dev testing |
| Schedule Trigger | Cron-based runs |
| Webhook | Receive HTTP events |
| HTTP Request | Call any API |
| Set | Create/rename fields |
| Code | JavaScript / Python |
| IF | Branch on condition |
| Switch | Multi-branch routing |
| Merge | Combine branches |
| Split Out | Array → multiple items |
| SplitInBatches | Process in chunks |
| Wait | Pause execution |
| Execute Workflow | Call sub-workflow |
| Sticky Note | Document your workflow |

---

## Code Node — Patterns

```javascript
// Process current item (Run Once Per Item mode)
const name = $json.name.toUpperCase();
return { name, processed: true };

// Process all items (Run Once for All Items mode)
const items = $input.all();
const total = items.reduce((sum, i) => sum + i.json.amount, 0);
return [{ json: { total } }];

// Return multiple items from one
const tags = $json.tags.split(',');
return tags.map(tag => ({ json: { tag: tag.trim() } }));

// Filter items
const items = $input.all();
return items.filter(i => i.json.score > 60);
```

---

## Webhook — Quick Setup

1. Add a **Webhook** trigger node
2. Set method: `POST` (or `GET` for simple callbacks)
3. Authentication: `None` (dev) or `Header Auth` (production)
4. Copy the **Test URL** for development
5. Set `Respond` to: `Using 'Respond to Webhook' node` for custom responses
6. Test URL changes to **Production URL** when workflow is activated

Test URL: `http://localhost:5678/webhook-test/<id>`
Production URL: `http://localhost:5678/webhook/<id>`

---

## Error Handling

```
Workflow Settings → Error Workflow → select a workflow
→ That workflow runs whenever this one fails
→ It receives: $json.execution.id, $json.execution.error, $json.workflow.name
```

Per-node retry:
```
Node → Settings → Retry On Fail → On (set max tries, wait between retries)
```

---

## Useful HTTP Status Codes to Handle

| Code | Meaning | Action |
|------|---------|--------|
| 200 | Success | Continue |
| 201 | Created | Continue |
| 400 | Bad Request | Check your request body |
| 401 | Unauthorized | Check credentials |
| 403 | Forbidden | Check permissions |
| 404 | Not Found | Resource doesn't exist |
| 429 | Rate Limited | Add delay + retry |
| 500 | Server Error | Retry with backoff |
| 503 | Service Unavailable | Retry later |
