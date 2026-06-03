# Lesson 08 — Working With Data: Set, Merge, Split, Filter

## Goal
Master every data manipulation node in n8n. After this lesson you'll know
exactly how to clean, reshape, combine, expand, and filter data at any point
in a workflow — without reaching for a Code node unnecessarily.

---

## The Data Manipulation Nodes

```
┌─────────────────────────────────────────────────────────┐
│             Data Manipulation Nodes                      │
│                                                          │
│  Edit Fields (Set)  — pick, rename, compute fields      │
│  Split Out          — 1 item with array → many items    │
│  Aggregate          — many items → 1 item with array    │
│  Filter             — keep or remove items by condition  │
│  Remove Duplicates  — deduplicate by field              │
│  Sort               — reorder items                     │
│  Limit              — keep only first N items           │
│  Merge              — combine two streams of items      │
└─────────────────────────────────────────────────────────┘
```

---

## Edit Fields (Set) — The Field Shaper

The most used transform node. Creates a new output item with exactly the
fields you define.

### Mode: Manual Mapping

Click "Add field" for each output field you want:

```
Input item:
{
  "userId": 1,
  "id": 101,
  "title": "Buy groceries",
  "completed": false,
  "createdAt": "2026-06-03T10:00:00Z",
  "internalMeta": { ... }   ← you don't need this
}

Set node config:
  task_id    = {{ $json.id }}
  task_name  = {{ $json.title }}
  is_done    = {{ $json.completed }}
  user       = {{ $json.userId }}

Output item (Keep Only Set Fields: ON):
{
  "task_id": 101,
  "task_name": "Buy groceries",
  "is_done": false,
  "user": 1
}
```

**"Keep Only Set Fields" toggle:**
- **ON** → output contains ONLY the fields you defined (clean, recommended)
- **OFF** → output contains your new fields PLUS all original fields

Always turn it ON unless you explicitly need to carry forward all old fields.

### Common Set Node Patterns

**Rename a field:**
```
new_name = {{ $json.old_name }}
```

**Compute a value:**
```
tax        = {{ $json.price * 0.18 }}
full_name  = {{ $json.firstName + ' ' + $json.lastName }}
is_senior  = {{ $json.age >= 60 }}
```

**Add a constant:**
```
source     = "n8n"
processed  = true
env        = "production"
```

**Format a date:**
```
date_formatted = {{ DateTime.fromISO($json.createdAt).toFormat('dd MMM yyyy') }}
```

**Flatten a nested field:**
```
city       = {{ $json.address.city }}
country    = {{ $json.address.country }}
```

---

## Split Out — Expand Array Into Items

Use when an API returns **1 item containing an array** and you need to
process each element separately.

```
Before Split Out:
[
  {
    "company": "Acme",
    "employees": [
      { "id": 1, "name": "Alice" },
      { "id": 2, "name": "Bob" },
      { "id": 3, "name": "Charlie" }
    ]
  }
]
→ 1 item

After Split Out (Field: "employees"):
[
  { "id": 1, "name": "Alice" },
  { "id": 2, "name": "Bob" },
  { "id": 3, "name": "Charlie" }
]
→ 3 items
```

**Config:**
- Field to Split Out: `employees` (the array field name)
- Include other fields: toggle ON if you want `company` to stay on each item

**When to use:**
- API returns `{ users: [...] }` — split out `users`
- API returns `{ data: { items: [...] } }` — split out `data.items`
- You used Aggregate earlier and now want to re-expand

**When NOT to use:**
- If the HTTP Request node already returned multiple items directly (already split)
- If you just want to loop — use SplitInBatches instead

---

## Aggregate — Collapse Items Into One

The inverse of Split Out. Takes many items and puts them all into one item
as an array.

```
Before Aggregate:
[ {name:"Alice"}, {name:"Bob"}, {name:"Charlie"} ]
→ 3 items

After Aggregate:
[ { "data": [ {name:"Alice"}, {name:"Bob"}, {name:"Charlie"} ] } ]
→ 1 item
```

**When to use:**
- Before sending a **summary message** (one Slack message listing all items)
- Before writing to a **spreadsheet** (one row with all data)
- Before a **Code node** that needs to see all items together
- After processing items to collect results back into one

### Real Pattern: Send One Summary Slack Message

```
[200 todo items]
       │
       ▼
[Filter: completed = true]   ← keep only done items
       │
       ▼
[Aggregate]                  ← collapse all done items into 1
       │
       ▼
[Code: format summary]
const items = $json.data;
return {
  message: `✅ ${items.length} tasks completed today:\n` +
    items.map(i => `• ${i.title}`).join('\n')
};
       │
       ▼
[Slack: send message]        ← 1 message, not 200
```

---

## Filter — Keep or Remove Items

Removes items from the stream based on a condition. Items that don't match
are dropped entirely.

```
Input:  [ {score:95}, {score:40}, {score:72}, {score:55} ]
Filter: score >= 60
Output: [ {score:95}, {score:72} ]
```

**Filter vs IF node — know the difference:**

| | Filter | IF Node |
|--|--------|---------|
| Non-matching items | Dropped silently | Go to False branch |
| Use when | You want to discard non-matches | You need to handle both cases |

Use **Filter** when you just want to skip items.
Use **IF** when you need to do something with both matched and unmatched items.

---

## Remove Duplicates

Removes items where a specified field has the same value as a previous item.

```
Input:
[ {email:"a@x.com"}, {email:"b@x.com"}, {email:"a@x.com"} ]

Remove Duplicates (Field: email):
[ {email:"a@x.com"}, {email:"b@x.com"} ]
```

**Real use case:** A webhook fires twice for the same event. Deduplicate
by event ID before processing to avoid double-processing.

---

## Sort

Reorders items by a field value.

```
Input:  [ {score:40}, {score:95}, {score:72} ]
Sort:   score, Descending
Output: [ {score:95}, {score:72}, {score:40} ]
```

---

## Limit

Keeps only the first N items, discards the rest.

```
Input:  200 items
Limit:  10
Output: first 10 items only
```

**Real use case:** Fetch 200 results, sort by score descending, Limit to top 5.

---

## Merge — Combining Two Streams

Merge takes **two inputs** and combines them.

### Mode 1: Append (most common)

Stacks all items from both inputs into one stream.

```
Input A: [ {name:"Alice"}, {name:"Bob"} ]
Input B: [ {name:"Charlie"}, {name:"Dave"} ]
Output:  [ {name:"Alice"}, {name:"Bob"}, {name:"Charlie"}, {name:"Dave"} ]
```

Use when: reconnecting IF/Switch branches back together.

### Mode 2: Combine by Field (SQL JOIN)

Joins items from two inputs where a specified field matches.

```
Input A: [ {id:1, name:"Alice"}, {id:2, name:"Bob"} ]
Input B: [ {id:1, score:95},     {id:2, score:80}  ]

Merge by field: "id"

Output:  [ {id:1, name:"Alice", score:95}, {id:2, name:"Bob", score:80} ]
```

Use when: you called two different APIs and want to combine data for the
same record (enrich pattern).

### Mode 3: Combine by Position

Merges item 1 from A with item 1 from B, item 2 with item 2, etc.

```
Input A: [ {name:"Alice"}, {name:"Bob"} ]
Input B: [ {city:"NY"},    {city:"LA"}  ]
Output:  [ {name:"Alice", city:"NY"}, {name:"Bob", city:"LA"} ]
```

### Mode 4: Choose Branch

Waits for both inputs, then passes items from only ONE specified input.
Use when you need a parallel operation to finish before continuing.

---

## Hands-On Exercise: Full Data Pipeline

Build this workflow:

```
[Manual Trigger]
       │
       ▼
[HTTP Request: GET https://jsonplaceholder.typicode.com/users]
       │
       ▼ (10 users, auto-split into 10 items)
[Edit Fields: keep only id, name, email, company.name as company]
       │
       ▼
[Filter: id <= 5]           ← keep only first 5 users
       │
       ▼
[Sort: name, Ascending]     ← alphabetical order
       │
       ▼
[Aggregate]                 ← collapse to 1 item
       │
       ▼
[Code: format as summary]
const users = $json.data;
return {
  count: users.length,
  summary: users.map(u => `${u.name} (${u.email})`).join('\n')
};
```

Expected final output:
```json
{
  "count": 5,
  "summary": "Chelsey Dietrich (Lucio_Hettinger@annie.ca)\n..."
}
```

---

## Summary

| Node | Purpose | Key setting |
|------|---------|------------|
| Edit Fields (Set) | Pick/rename/compute fields | Keep Only Set Fields: ON |
| Split Out | Array inside item → many items | Field to split |
| Aggregate | Many items → 1 item with array | All Item Data |
| Filter | Remove non-matching items silently | Condition |
| Remove Duplicates | Deduplicate by field | Compare field |
| Sort | Reorder items | Field + direction |
| Limit | Keep first N items | Max items |
| Merge (Append) | Stack two streams | — |
| Merge (Combine by Field) | JOIN two streams on matching ID | Join field |

---

## Check Your Understanding — Q&A

### Q1. What is the difference between Filter node and IF node?

**Answer:** Filter silently drops non-matching items — they disappear.
IF node routes non-matching items to a False branch where you can do
something with them. Use Filter to discard. Use IF when you need to
handle both cases differently.

---

### Q2. You have 3 items and want to send ONE Slack message listing all 3. What nodes do you use?

**Answer:** Aggregate node (collapse 3 items into 1) → Code node (format
the message string from `$json.data`) → Slack node (sends 1 message).
Without Aggregate, Slack would send 3 separate messages.

---

### Q3. You call two APIs — one returns `{id, name}` and the other returns `{id, score}` for the same users. How do you combine them?

**Answer:** Merge node → Mode: **Combine by Field** → Field: `id`.
This joins items where `id` matches, producing `{id, name, score}` for each user.

---

## Next Lesson

**[Lesson 09 →](09-credentials-and-auth.md)** — Credentials and authentication:
creating API keys, OAuth2, Bearer tokens, and best practices for managing
secrets securely in n8n.
