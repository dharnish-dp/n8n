# Lesson 04 — The Data Model: Items, JSON & Pinning

## Goal
Understand exactly how data moves through n8n. This is the single most important
concept in n8n. Every confusing behavior, every "why isn't this working" moment
traces back to not fully understanding the data model.

---

## The Item — n8n's Fundamental Unit

Every piece of data in n8n is wrapped in an **item**. An item looks like this:

```json
{
  "json": {
    "name": "Alice",
    "email": "alice@example.com",
    "score": 95
  },
  "binary": {},
  "pairedItem": { "item": 0 }
}
```

Three parts:
- `json` — the actual data (a plain object, any shape)
- `binary` — file data (images, PDFs, etc. — rarely used until you need it)
- `pairedItem` — which input item this came from (used for merging — Lesson 08)

**When you write `$json.name`, you're accessing `item.json.name`.**
n8n drops the `.json` part for you in expressions.

---

## Workflows Always Process Arrays of Items

Every node receives an **array** of items. Every node outputs an **array**.

```
Node Input:  [ item1, item2, item3 ]
                 │       │       │
                 ▼       ▼       ▼
              (process each item)
                 │       │       │
Node Output: [ item1', item2', item3' ]
```

This is the single most important thing to understand:

> **n8n nodes process ALL items.** If 10 items arrive at a Slack node, it sends
> 10 Slack messages. If 100 items arrive at an HTTP Request node, it makes 100
> HTTP requests.

This is both the power and the trap of n8n.

---

## What Each Node Type Does to Items

### Action nodes (Slack, Gmail, HTTP Request, etc.)

Execute once **per item**:

```
Input:  [ {name: "Alice"}, {name: "Bob"}, {name: "Charlie"} ]
Node:   Send Slack message "Hello {{ $json.name }}"
Result: 3 Slack messages sent: "Hello Alice", "Hello Bob", "Hello Charlie"
```

### Transform nodes (Set, Code)

Transform each item, output same count:

```
Input:  [ {temp: 22.3}, {temp: 18.1} ]
Node:   Set → temp_f = {{ $json.temp * 9/5 + 32 }}
Output: [ {temp_f: 72.14}, {temp_f: 64.58} ]
```

### Filter nodes (IF, Filter)

Some items pass, some don't:

```
Input:  [ {score: 95}, {score: 40}, {score: 88} ]
Node:   IF → score >= 60
True:   [ {score: 95}, {score: 88} ]
False:  [ {score: 40} ]
```

### Aggregation nodes (Merge, Aggregate)

Multiple items become one (or fewer):

```
Input:  [ {a: 1}, {a: 2}, {a: 3} ]
Node:   Aggregate → all into one item
Output: [ {data: [{a: 1}, {a: 2}, {a: 3}]} ]
```

---

## The "One Item vs. Many Items" Mental Model

**The most common n8n mistake:** Not knowing how many items you have.

When you make one HTTP request that returns a list, n8n gives you ONE item
containing that list:

```json
Input after HTTP Request node:
[
  {
    "json": {
      "users": [
        {"id": 1, "name": "Alice"},
        {"id": 2, "name": "Bob"},
        {"id": 3, "name": "Charlie"}
      ]
    }
  }
]
```

That's **1 item**. The `users` array is inside `$json.users`.

If you connect a Slack node here, it sends **1 Slack message** — not 3.

**To process each user separately**, you need to "split out" the array using the
**"Items from an Array"** option or a **Split Out** node:

```json
After splitting $json.users:
[
  { "json": {"id": 1, "name": "Alice"} },
  { "json": {"id": 2, "name": "Bob"} },
  { "json": {"id": 3, "name": "Charlie"} }
]
```

Now connecting a Slack node sends **3 messages**.

This "1 item with array" vs "many items" confusion is the #1 source of bugs
for beginners. Draw a mental picture of your item count at each node.

---

## Hands-On: See Item Counts Live

### Exercise 1 — One item with an array inside

1. Create a new workflow
2. Add a **Code** node (no trigger needed for testing)
3. In the Code node, set mode to **"Run Once for All Items"** and paste:

```javascript
return [
  {
    json: {
      total: 3,
      users: [
        { id: 1, name: "Alice" },
        { id: 2, name: "Bob" },
        { id: 3, name: "Charlie" }
      ]
    }
  }
];
```

4. Execute it. Count the items in the output panel. **Answer: 1 item.**

### Exercise 2 — Split into multiple items

1. Add a **Split Out** node after the Code node
2. Configure: **Field to Split Out** = `users`
3. Execute. Count the items. **Answer: 3 items.**

Now each item has `$json.id` and `$json.name` directly accessible.

---

## $json — The Expression Variable You Use Most

`$json` in an expression refers to the **current item's json property** from
the immediately previous node.

```
{{ $json }}                          → the entire item object
{{ $json.name }}                     → "Alice"
{{ $json.address.city }}             → nested access
{{ $json.tags[0] }}                  → first element of an array
{{ $json.tags.length }}              → count of array elements
{{ Object.keys($json) }}             → all field names
{{ JSON.stringify($json) }}          → the object as a string
```

### Accessing data from NON-previous nodes

Sometimes you need data from 2 nodes back, or from a specific named node.
Use `$('Node Name').item.json`:

```
{{ $('HTTP Request').item.json.temperature }}
{{ $('Manual Trigger').item.json.userId }}
```

Or reference all items from a node:
```
{{ $('HTTP Request').all() }}         → array of all items
{{ $('HTTP Request').first().json }}  → first item's json
{{ $('HTTP Request').last().json }}   → last item's json
```

This is how you "reach back" in a workflow when you've branched or
merged and lost access to earlier data.

---

## Pinning Data — The Critical Dev Workflow Technique

**The problem:** Every time you click "Execute node" on a mid-workflow node,
n8n has to re-run all upstream nodes to get fresh input data. This is:
- Slow (re-fetches APIs)
- Costly (burns API rate limits)
- Unpredictable (live data changes)

**The solution: Pin data.**

Pinning "freezes" the output of a node so that downstream nodes always receive
that exact data during development, without re-running upstream.

### How to pin data:

1. Execute a node (any way — single execute or full workflow)
2. In the output panel, click the **"Pin"** button (pushpin icon) in the top right
3. The node now shows a pin icon
4. Future executions will use the pinned data instead of re-running the node

### When to pin:

- During development of downstream nodes when upstream is an API call
- When testing with a specific edge-case response you captured
- When the trigger requires an external event (webhook, email) you can't
  easily re-fire during development

### How to un-pin:

Click the pin icon again. The node will go back to live execution.

**Important:** Pinned data is stored in the workflow JSON. If you share a
workflow, the pinned data comes with it — useful for sharing examples.

---

## Binary Data

For files (images, PDFs, CSVs), n8n uses the `binary` property of an item:

```json
{
  "json": {
    "filename": "report.pdf"
  },
  "binary": {
    "data": {
      "data": "base64encodedcontent...",
      "mimeType": "application/pdf",
      "fileName": "report.pdf",
      "fileSize": 12345
    }
  }
}
```

Nodes that handle files (Google Drive, Email, HTTP Request with file response)
populate the `binary` property. You reference binary data with `$binary.data`
in expressions, or configure nodes to look for it by name.

We'll use binary data in Lesson 07 (HTTP Request) when downloading files.

---

## Item Pairing — How Merge Nodes Know What to Combine

When you have two branches that merge back together, n8n uses `pairedItem`
to know which items belong together:

```
Original item: { id: 1, name: "Alice" }
    │                              │
    ├──── Branch A ────▶ { id: 1, enriched: {...} }
    └──── Branch B ────▶ { id: 1, score: 95 }

Merge node combines using pairedItem:
Result: { id: 1, name: "Alice", enriched: {...}, score: 95 }
```

You rarely configure this manually — n8n tracks it automatically. But knowing
it exists explains why merge operations sometimes produce unexpected results
when item counts don't match between branches.

---

## The Complete Item Lifecycle

Let's trace a real item through our weather workflow:

```
Manual Trigger output:
[ { json: {}, pairedItem: {item: 0} } ]

After HTTP Request node:
[ { json: {latitude: 40.7, current: {temperature_2m: 22.3, weathercode: 3}}, pairedItem: {item: 0} } ]

After Set node (extracted fields):
[ { json: {temperature: 22.3, weathercode: 3, wind_speed: 15.2, city: "New York"}, pairedItem: {item: 0} } ]

After IF node (True branch, weathercode < 51 is False → goes to False branch):
[ { json: {temperature: 22.3, weathercode: 3, wind_speed: 15.2, city: "New York"}, pairedItem: {item: 0} } ]

After final Set node:
[ { json: {message: "☀️ No rain in New York today. Temperature: 22.3°C. Enjoy!"}, pairedItem: {item: 0} } ]
```

The item evolves as it flows through nodes. Each node adds, removes, or
transforms the `json` property. The `pairedItem` tracks lineage.

---

## Summary

- Everything in n8n is an item: `{ json: {}, binary: {}, pairedItem: {} }`
- Nodes receive an **array of items** and output an **array of items**
- Most action nodes execute **once per item** — 10 items = 10 API calls
- `$json` = current item's json from the previous node
- Use `$('Node Name').item.json` to access data from non-adjacent nodes
- **Pin data** during development to avoid re-running expensive upstream nodes
- The "one item with array" vs "many items" distinction is the #1 source of bugs

---

## Check Your Understanding — Q&A

### Q1. You call an API that returns a list of 50 users in one response. How many items does n8n have after the HTTP Request node?

**Answer:** **1 item.** The entire API response (including the 50-user array)
is wrapped in one n8n item. To process each user separately (e.g., send a
Slack message per user), you must use a **Split Out** node to expand the array
into 50 separate items.

---

### Q2. You have 20 items flowing into a Slack node. What happens?

**Answer:** n8n sends **20 Slack messages** — one per item. Every action node
executes once per input item. This is why understanding item count is critical
before connecting action nodes.

---

### Q3. What is data pinning and when should you use it?

**Answer:** Pinning freezes the output of a node. Downstream nodes use the
pinned data instead of re-running the upstream node. Use it when developing
downstream nodes that depend on an API response — so you don't burn API rate
limits or wait for slow upstream calls on every test.

---

### Q4. How do you access data from a node two steps back in your workflow?

**Answer:** Use `$('Node Name').item.json.fieldName` — referencing the node by
its name in the canvas. You can also use `$('Node Name').first().json` or
`$('Node Name').all()` to access all items from that node.

---

## Next Lesson

**[Lesson 05 →](05-expressions-and-dynamic-data.md)** — The full expression
language: `$json`, `$node`, `$workflow`, `$now`, `$vars`, date handling,
JavaScript functions, and how to write expressions that never break.
