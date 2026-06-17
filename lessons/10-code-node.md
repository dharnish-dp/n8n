# Lesson 10 — The Code Node: JavaScript & Python Power

## Goal
Master the Code node — n8n's escape hatch for anything too complex for
expressions. Know when to use it, both execution modes, all available
built-ins, and real production patterns.

---

## When to Use the Code Node

Expressions (`{{ }}`) handle 90% of cases. Use Code node when:

| Situation | Use Code Node |
|-----------|--------------|
| Multi-step logic (temp variables) | ✅ |
| Loop over array and transform | ✅ |
| Manipulate ALL items together | ✅ |
| Complex string parsing (regex, split/join) | ✅ |
| Mathematical computations | ✅ |
| Filter items based on complex conditions | ✅ |
| Call utility functions spanning multiple lines | ✅ |
| Simple field access or rename | ❌ (use Set node) |
| Simple condition | ❌ (use IF node) |

---

## Two Execution Modes

### Mode 1: Run Once Per Item (default)

```
Input: [ item1, item2, item3 ]
Runs: once for item1, once for item2, once for item3
```

The code receives the current item as `$json`. Return one object (not an array):

```javascript
// Access current item
const name = $json.name;
const score = $json.score;

// Return the output object
return {
  name: name.toUpperCase(),
  passed: score >= 60,
  grade: score >= 90 ? 'A' : score >= 70 ? 'B' : score >= 60 ? 'C' : 'F'
};
```

If you return multiple objects (array), each becomes a separate item:
```javascript
// Split one item into multiple
const tags = $json.tags.split(',');
return tags.map(tag => ({ tag: tag.trim(), source: $json.id }));
// 1 item in → N items out
```

### Mode 2: Run Once for All Items

```
Input: [ item1, item2, item3 ]
Runs: ONCE, receives all items
```

Use `$input.all()` to get all items. Must return an array of objects:

```javascript
// Get all input items
const items = $input.all();

// Compute aggregate
const total = items.reduce((sum, item) => sum + item.json.amount, 0);
const avg = total / items.length;

// Return one item with summary
return [{ json: { total, avg, count: items.length } }];
```

Or return a transformed version of all items:
```javascript
const items = $input.all();

// Filter + transform all at once
const processed = items
  .filter(i => i.json.status === 'active')
  .map(i => ({
    json: {
      id: i.json.id,
      name: i.json.name,
      revenue: i.json.mrr * 12
    }
  }));

return processed;
```

---

## Return Format — Important

### Per Item mode
Return a plain object (not wrapped in `{ json: ... }`):
```javascript
return { name: "Alice", score: 95 };  // ✅ correct
return { json: { name: "Alice" } };   // ❌ wrong — creates { json: { name: "Alice" } }
```

### All Items mode
Return an array of objects wrapped in `{ json: ... }`:
```javascript
return [
  { json: { name: "Alice", score: 95 } },   // ✅ correct
  { json: { name: "Bob", score: 80 } },
];
```

---

## Available Variables in the Code Node

```javascript
// Current item (Per Item mode)
$json                      // same as $input.item.json
$binary                    // current item's binary data

// All items
$input.all()               // array of all input items
$input.first()             // first input item
$input.last()              // last input item
$input.item                // current item (in Per Item mode)
$input.item.json           // current item's json

// Reference other nodes
$('Node Name').all()       // all items from a named node
$('Node Name').first()     // first item from a named node
$('Node Name').item.json   // current item from a named node

// Workflow context
$workflow.name             // workflow name
$workflow.id               // workflow ID
$execution.id              // current execution ID
$execution.mode            // 'manual' | 'trigger' | 'webhook'

// Date (Luxon)
$now                       // current DateTime
$today                     // today at midnight
DateTime                   // Luxon class for date construction

// Variables
$vars.MY_VAR               // workflow variables
$env.ENV_VAR               // environment variables (if allowed)
```

---

## Real Code Node Patterns

### Pattern 1: Complex string extraction

```javascript
// Extract domain from email, handle edge cases
const email = $json.email || '';
const match = email.match(/@([^@]+)$/);
const domain = match ? match[1].toLowerCase() : 'unknown';

return {
  ...$json,
  domain,
  isPersonal: ['gmail.com', 'yahoo.com', 'hotmail.com'].includes(domain)
};
```

### Pattern 2: Flatten nested structure

```javascript
// API returns nested, you want flat
return {
  id: $json.id,
  name: $json.profile.name,
  city: $json.profile.address?.city ?? 'Unknown',
  country: $json.profile.address?.country ?? 'Unknown',
  plan: $json.subscription?.plan ?? 'free',
  mrr: $json.subscription?.amount ?? 0
};
```

### Pattern 3: Group items by field (All Items mode)

```javascript
const items = $input.all();

// Group by status
const grouped = {};
for (const item of items) {
  const key = item.json.status;
  if (!grouped[key]) grouped[key] = [];
  grouped[key].push(item.json);
}

return [{ json: grouped }];
// Output: { active: [...], pending: [...], cancelled: [...] }
```

### Pattern 4: Deduplicate items (All Items mode)

```javascript
const items = $input.all();
const seen = new Set();
const unique = [];

for (const item of items) {
  const key = item.json.email;
  if (!seen.has(key)) {
    seen.add(key);
    unique.push(item);
  }
}

return unique;
```

### Pattern 5: Compute rolling totals / running sums (All Items mode)

```javascript
const items = $input.all();
let runningTotal = 0;

return items.map(item => ({
  json: {
    ...item.json,
    runningTotal: (runningTotal += item.json.amount)
  }
}));
```

### Pattern 6: Parse CSV string

```javascript
const csvText = $json.csvContent;
const lines = csvText.trim().split('\n');
const headers = lines[0].split(',').map(h => h.trim());

return lines.slice(1).map(line => {
  const values = line.split(',');
  const obj = {};
  headers.forEach((h, i) => { obj[h] = values[i]?.trim(); });
  return { json: obj };
});
```

---

## Python in the Code Node

> ⚠️ **Common Error:** `Python runner unavailable: Virtual environment is missing`
>
> This means n8n can't find a Python venv on your system.
> **Easiest fix:** switch the Code node language to **JavaScript** — same power, zero setup.
>
> If you specifically need Python, create the venv:
> ```bash
> cd ~/.npm-global/lib/node_modules/n8n
> python3 -m venv .venv
> ```
> Then restart n8n. All course lessons use JavaScript — Python is optional.

Switch language to Python. Note: fewer built-ins than JavaScript.

```python
# Per Item mode
name = _json["name"].upper()
score = _json.get("score", 0)
grade = "A" if score >= 90 else "B" if score >= 70 else "C" if score >= 60 else "F"

return {
    "name": name,
    "score": score,
    "grade": grade
}
```

Python built-ins available: `_json`, `_execution_id`, `_workflow_id`.
Less access to n8n context than JS. Use JavaScript for full power.

---

## Debugging the Code Node

When your code fails:
1. The error shows the line number and message
2. Add `console.log(JSON.stringify($json))` to see what data you're receiving
3. Logs appear in the n8n server terminal output and in node's "Error" output

```javascript
// Debug: log what you're receiving
console.log('Input data:', JSON.stringify($json, null, 2));
console.log('Item count:', $input.all().length);

return { debug: true };  // placeholder while testing
```

---

## Summary

- Two modes: Per Item (most common) vs All Items (for aggregation/filtering)
- Per Item: return plain object; All Items: return array of `{json: ...}` objects
- Access all built-in variables: `$json`, `$input`, `$now`, `$workflow`, etc.
- Code node is the escape hatch — use it when Set/IF/expressions aren't enough
- Don't over-use it: simple transforms belong in Set/expressions, not Code

---

## Check Your Understanding — Q&A

### Q1. You're in Per Item mode and you return an array of 3 objects from one input item. How many output items do you get?

**Answer:** 3 output items — one per object in the returned array. Per Item mode treats a returned array as "split this item into multiple." This is how you expand one item into N items inside a Code node, without using a Split Out node.

---

### Q2. What is wrong with this All Items mode code?

```javascript
const items = $input.all();
return items.map(i => ({ name: i.json.name }));
```

**Answer:** The return format is wrong. All Items mode requires `{ json: {...} }` wrappers. The correct version:
```javascript
const items = $input.all();
return items.map(i => ({ json: { name: i.json.name } }));
```
Without the `json` wrapper, n8n throws a runtime error or produces unexpected output.

---

### Q3. You need to deduplicate 1,000 items by email, then enrich each unique email via an API. Where does the Code node fit and which mode?

**Answer:** Code node in **Run Once for All Items** mode — placed before the HTTP Request node. It receives all 1,000 items, deduplicates using a `Set`, returns only unique items. The HTTP Request then fires once per unique item. Per Item mode can't deduplicate because it processes items independently — it has no view of other items.

```javascript
const items = $input.all();
const seen = new Set();
return items
  .filter(i => {
    if (seen.has(i.json.email)) return false;
    seen.add(i.json.email);
    return true;
  });
// Note: no { json: wrap } needed when returning existing item objects
```

---

### Q4. What variables are available in the Code node that are NOT available in a Set node expression?

**Answer:** `$input.all()` (access all items at once), `require()` for Node.js built-ins like `crypto`, multi-line logic with intermediate variables, loops (`for`, `map`, `reduce`), and `console.log()` for debugging. Set node expressions are single-line — no loops, no intermediate state, no `require`.

---

### Q5. When should you NOT use a Code node even though you technically could?

**Answer:** When a dedicated node already does it: field picking → Set node, branching → IF/Switch, deduplication → Remove Duplicates node, sorting → Sort node, simple HTTP call → HTTP Request node. Code nodes are harder to read at a glance, don't show visual config in the panel, and require reading JS to understand. Use them only when no built-in node covers the logic. Three similar transform lines in a Set node is better than a Code node with a three-line map.

---

## Next Lesson

**[Lesson 11 →](11-error-handling.md)** — Error handling, retry patterns, Error
Workflows, and building production-grade resilient automation.
