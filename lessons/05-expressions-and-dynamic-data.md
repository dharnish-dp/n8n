# Lesson 05 — Expressions & Dynamic Data

## Goal
Master the n8n expression language. Learn every built-in variable, how to
write safe expressions that don't break on missing data, and when to reach
for a Code node instead.

---

## Expression Syntax — The Basics

Anything inside `{{ }}` is evaluated as JavaScript:

```
{{ $json.name }}                       → value of name field
{{ "Hello " + $json.name }}            → string concatenation
{{ $json.score > 60 ? "Pass" : "Fail" }}  → ternary expression
{{ new Date().toISOString() }}         → current timestamp
```

**Outside** `{{ }}` is plain text:
```
Hello {{ $json.name }}, your score is {{ $json.score }}.
→  Hello Alice, your score is 95.
```

You can mix text and expressions freely. But an entire field value can also
be a pure expression: `{{ $json.amount * 1.1 }}` with no surrounding text.

---

## The Built-In Variables — Complete Reference

### $json
The current item's data from the immediately previous node.

```javascript
{{ $json }}                          // entire object
{{ $json.name }}                     // top-level field
{{ $json.address.city }}             // nested field
{{ $json.tags[0] }}                  // array element
{{ $json.tags.join(', ') }}          // array to string
{{ Object.keys($json).length }}      // count fields
```

### $('Node Name')
Access data from any named node, not just the previous one.

```javascript
{{ $('HTTP Request').item.json.temperature }}   // current item
{{ $('HTTP Request').first().json.name }}       // first item
{{ $('HTTP Request').last().json.name }}        // last item
{{ $('HTTP Request').all() }}                   // all items as array
{{ $('HTTP Request').all().length }}            // count items
{{ $('HTTP Request').all()[2].json.name }}      // third item
```

**When to use this:** When you've branched, merged, or are in a sub-workflow
and need data that isn't directly from the previous node.

### $now
Current date/time as a Luxon DateTime object.

```javascript
{{ $now }}                                    // full DateTime object
{{ $now.toISO() }}                            // "2026-05-22T14:30:00.000Z"
{{ $now.toFormat('yyyy-MM-dd') }}             // "2026-05-22"
{{ $now.toFormat('MMMM d, yyyy') }}           // "May 22, 2026"
{{ $now.toMillis() }}                         // Unix milliseconds
{{ $now.minus({days: 7}).toISO() }}           // 7 days ago
{{ $now.plus({hours: 24}).toFormat('HH:mm') }} // 24 hours from now
{{ $now.weekday }}                            // 1=Mon, 7=Sun
{{ $now.hour }}                              // current hour (0-23)
```

n8n uses **Luxon** for dates. This is the most important date library to learn.

### $today
Current date at midnight (no time component).

```javascript
{{ $today.toFormat('yyyy-MM-dd') }}           // "2026-05-22"
{{ $today.plus({days: 30}) }}                 // 30 days from today
```

### $workflow
Information about the current workflow.

```javascript
{{ $workflow.id }}                            // workflow ID
{{ $workflow.name }}                          // workflow name
{{ $workflow.active }}                        // true if workflow is active
```

### $execution
Information about the current execution run.

```javascript
{{ $execution.id }}                           // unique execution ID
{{ $execution.mode }}                         // "manual" | "trigger" | "webhook"
{{ $execution.resumeUrl }}                    // for wait/resume workflows
```

### $vars
Access workflow-level variables (n8n Variables feature).

```javascript
{{ $vars.API_BASE_URL }}                      // a defined variable
{{ $vars.ENVIRONMENT }}                       // "production", "staging", etc.
```

### $env
Access environment variables from the n8n server.

```javascript
{{ $env.MY_CUSTOM_VAR }}
```

Must be explicitly allowed in n8n settings (`ALLOWED_EXTERNAL_ENV_VARS`).

### $input
The input to the current node (alternative to `$json` in some contexts).

```javascript
{{ $input.item.json.name }}                   // same as $json.name
{{ $input.all() }}                            // all input items
{{ $input.first().json }}                     // first input item
```

### $prevNode
Data about the node that fed the current node.

```javascript
{{ $prevNode.name }}                          // name of previous node
{{ $prevNode.outputIndex }}                   // which output branch (0 or 1)
{{ $prevNode.runIndex }}                      // which run iteration
```

---

## Date Handling — The Complete Luxon Reference

Dates are the #1 pain point in automation. Learn this section well.

### Parsing a date string from an API

```javascript
// Input: "2026-05-22T14:30:00Z" (ISO string)
{{ DateTime.fromISO($json.createdAt).toFormat('MMMM d, yyyy') }}
// Output: "May 22, 2026"

// Input: "05/22/2026" (US date format)
{{ DateTime.fromFormat($json.date, 'MM/dd/yyyy').toISO() }}

// Input: Unix timestamp (seconds)
{{ DateTime.fromSeconds($json.timestamp).toISO() }}

// Input: Unix timestamp (milliseconds)
{{ DateTime.fromMillis($json.timestamp).toISO() }}
```

### Date arithmetic

```javascript
// Add time
{{ DateTime.fromISO($json.startDate).plus({days: 30}).toISO() }}
{{ DateTime.fromISO($json.startDate).plus({hours: 2, minutes: 30}).toISO() }}

// Subtract time
{{ DateTime.fromISO($json.dueDate).minus({days: 7}).toISO() }}

// Difference between two dates
{{ DateTime.fromISO($json.endDate).diff(DateTime.fromISO($json.startDate), 'days').days }}
// Output: number of days between dates
```

### Comparing dates

```javascript
{{ DateTime.fromISO($json.expiresAt) > $now }}   // is it in the future?
{{ DateTime.fromISO($json.createdAt) < $today }}  // is it before today?
```

### Formatting reference

| Format string | Output |
|---------------|--------|
| `yyyy-MM-dd` | `2026-05-22` |
| `dd/MM/yyyy` | `22/05/2026` |
| `MMMM d, yyyy` | `May 22, 2026` |
| `EEE, MMM d` | `Fri, May 22` |
| `HH:mm` | `14:30` |
| `h:mm a` | `2:30 PM` |
| `yyyy-MM-dd HH:mm:ss` | `2026-05-22 14:30:00` |
| `x` | Unix milliseconds |

---

## Writing Safe Expressions — Avoiding Null Errors

The #1 cause of expression failures: accessing a field that doesn't exist.

```javascript
{{ $json.user.email }}
// FAILS if $json.user is null or undefined
// Error: Cannot read property 'email' of null
```

### Safe access patterns:

```javascript
// Optional chaining (JavaScript ?.)
{{ $json.user?.email }}
// Returns undefined instead of throwing

// Nullish coalescing — provide a default
{{ $json.user?.email ?? 'no-email@default.com' }}
// Returns default if null/undefined

// Check before using
{{ $json.user ? $json.user.email : 'unknown' }}

// Logical OR default
{{ $json.score || 0 }}
// Returns 0 if score is null/undefined/0/empty
// Note: || is falsy (0 counts as false), ?? is nullish (only null/undefined)
```

### Check if a field exists

```javascript
{{ $json.hasOwnProperty('email') }}           // true/false
{{ 'email' in $json }}                        // true/false
{{ $json.email !== undefined }}               // true/false
```

### Safe array access

```javascript
{{ $json.items?.[0]?.name }}                  // first item's name, safe
{{ $json.items?.length ?? 0 }}               // count, default 0
{{ ($json.items ?? []).filter(i => i.active) }} // filter with fallback empty array
```

---

## String Operations

```javascript
// Transform
{{ $json.name.toUpperCase() }}
{{ $json.name.toLowerCase() }}
{{ $json.name.trim() }}
{{ $json.name.replace('old', 'new') }}
{{ $json.description.substring(0, 100) }}     // first 100 chars
{{ $json.description.slice(0, 100) + '...' }} // truncate with ellipsis

// Check
{{ $json.email.includes('@') }}
{{ $json.status.startsWith('active') }}
{{ $json.url.endsWith('.pdf') }}

// Split / Join
{{ $json.tags.split(',') }}                   // string → array
{{ $json.tags.join(' | ') }}                  // array → string

// Template construction
{{ `Hello ${$json.name}, your order #${$json.orderId} is ready.` }}
```

---

## Number Operations

```javascript
{{ Math.round($json.temperature) }}           // round to integer
{{ Math.floor($json.price) }}                 // floor
{{ Math.ceil($json.price) }}                  // ceiling
{{ ($json.amount * 1.1).toFixed(2) }}         // 2 decimal places
{{ Math.max($json.a, $json.b) }}              // larger of two
{{ Math.min($json.a, $json.b) }}              // smaller of two
{{ Math.abs($json.change) }}                  // absolute value
{{ parseInt($json.count) }}                   // string → integer
{{ parseFloat($json.price) }}                 // string → float
{{ isNaN($json.value) }}                      // check if not a number
```

---

## Array Operations in Expressions

```javascript
// Map (transform each element)
{{ $json.users.map(u => u.name) }}
// [{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}] → ["Alice","Bob"]

// Filter
{{ $json.items.filter(i => i.price > 100) }}

// Find (first match)
{{ $json.items.find(i => i.id === 5) }}

// Check membership
{{ $json.roles.includes('admin') }}

// Count
{{ $json.items.filter(i => i.active).length }}

// Sum
{{ $json.items.reduce((sum, i) => sum + i.price, 0) }}

// Sort
{{ $json.items.sort((a, b) => b.score - a.score) }} // descending by score
```

---

## Practical Patterns — Real Use Cases

### Build a URL with dynamic parameters
```javascript
{{ `https://api.example.com/users/${$json.userId}/orders?status=${$json.status}` }}
```

### Format a Slack message
```javascript
{{ `*New order from ${$json.customer.name}*\nAmount: $${($json.total).toFixed(2)}\nItems: ${$json.items.length}` }}
```

### Compute a relative time string
```javascript
{{ $now.minus({hours: $json.age_hours}).toRelative() }}
// Output: "3 hours ago", "2 days ago", etc.
```

### Build a CSV row
```javascript
{{ [$json.id, $json.name, $json.email, $json.score].join(',') }}
```

### Check if a value is in a list
```javascript
{{ ['active', 'pending', 'trial'].includes($json.status) }}
```

### Extract a domain from an email
```javascript
{{ $json.email.split('@')[1] }}
```

### Slugify a title
```javascript
{{ $json.title.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-]/g, '') }}
```

---

## When Expressions Aren't Enough — Use the Code Node

Expressions handle 90% of cases. Use a Code node when:

- You need multi-step logic (multiple variables, loops, complex conditions)
- You need to transform the entire array of items (not just one item at a time)
- You need to call utility functions that span multiple lines
- You need to import JavaScript libraries (limited — see Lesson 10)

```javascript
// Too complex for an expression — use Code node:
const priceAfterTax = $json.price * (1 + $json.taxRate / 100);
const discount = $json.loyaltyPoints > 1000 ? 0.1 : 0;
const finalPrice = priceAfterTax * (1 - discount);
return { finalPrice: finalPrice.toFixed(2) };
```

---

## Summary

- `{{ expression }}` — n8n's expression syntax, pure JavaScript inside
- `$json` — current item's data from previous node
- `$('Node Name').item.json` — data from any named node
- `$now` / `$today` — Luxon DateTime objects for date work
- Use `?.` and `??` for safe access — never crash on missing fields
- Know your Luxon format strings — date formatting is constant work
- When logic gets complex, move it to a Code node

---

## Check Your Understanding — Q&A

### Q1. What's the difference between `||` and `??` for default values?

**Answer:** `||` treats any falsy value as "missing" — including `0`, `""`,
and `false`. `??` (nullish coalescing) only triggers on `null` and `undefined`.
Use `??` when `0` or `false` are valid values: `{{ $json.score ?? 0 }}` returns
`0` if score is null/undefined, but keeps `0` if score is actually `0`. Using
`||` would wrongly replace `0` with the default.

---

### Q2. You have an API that sometimes omits the `address` field. How do you safely access `address.city`?

**Answer:** Use optional chaining:
`{{ $json.address?.city ?? 'Unknown city' }}`
This returns `undefined` instead of throwing if `address` is missing, then
`??` catches the `undefined` and returns the default.

---

### Q3. How do you get data from a node that's 3 steps back in the workflow?

**Answer:** Use `$('Node Name').item.json.fieldName` — reference the node by
the name shown in the canvas. Node names can be customized (double-click the
node name to rename). Using descriptive node names makes this pattern much
more readable: `$('Fetch User Data').item.json.userId`.

---

## Next Lesson

**[Lesson 06 →](06-core-node-types.md)** — Deep dive into every important node
type: Schedule Trigger, Webhook, HTTP Request, Set, IF, Switch, Merge,
SplitInBatches, Code, Wait, and more.
