# Lesson 03 — Your First Workflow

## Goal
Build a complete, working workflow from scratch. By the end, you'll have
triggered a workflow, fetched real API data, transformed it, and routed it
based on conditions. You'll understand the edit → execute → debug loop that
defines all n8n work.

---

## What We're Building

**A weather alert workflow:**
1. Trigger manually (later we'll make it run on a schedule)
2. Fetch the current weather for your city via a free public API
3. Transform the data to extract only what we need
4. Check if it's going to rain
5. Display different output based on the result

This is a real, complete workflow — not a "hello world" toy.

```
[Manual Trigger]
       │
       ▼
[HTTP Request → OpenWeatherMap API]
       │
       ▼
[Set Node → Extract temperature + condition]
       │
       ▼
[IF Node → Is it raining?]
       │                    │
       ▼ (yes)              ▼ (no)
[Set → "Bring umbrella"]  [Set → "No rain today"]
```

---

## Step 1 — Create a New Workflow

1. Open **http://localhost:5678**
2. Click **"+ New workflow"** (top-left or center of screen)
3. Click the workflow name at the top and rename it: **"Weather Alert"**
4. Click **Save**

You now have an empty canvas with a **Manual Trigger** node.

---

## Step 2 — Understand the Manual Trigger Node

The Manual Trigger is always your starting point during development. It lets you
run a workflow by clicking a button instead of waiting for a real trigger.

Click the **Manual Trigger** node. Observe:
- Right panel opens with node settings
- This node has no configuration — it just starts the workflow when you click Execute
- It outputs **one empty item**: `[{}]`

That empty item `{}` is the "seed" data — the first item flowing into the
pipeline. Right now it's empty, but once we add more nodes, data will accumulate.

---

## Step 3 — Add an HTTP Request Node

Click the **"+"** button that appears to the right of the Manual Trigger node,
or click **"+ Add node"** in the bottom bar and search for **"HTTP Request"**.

Connect it to the Manual Trigger if it isn't already.

### Configure the HTTP Request node:

We'll use the Open-Meteo API — it's completely free, no API key required:

```
URL: https://api.open-meteo.com/v1/forecast

Query Parameters (click "Add Parameter" for each):
  latitude     = 40.7128
  longitude    = -74.0060
  current      = temperature_2m,weathercode,windspeed_10m
  timezone     = America/New_York
```

This fetches current weather for New York City.

**What these parameters mean:**
- `latitude/longitude` — coordinates (New York City)
- `current` — which current weather variables to return
- `timezone` — for correct local time

Your full URL will look like:
```
https://api.open-meteo.com/v1/forecast?latitude=40.7128&longitude=-74.0060&current=temperature_2m,weathercode,windspeed_10m&timezone=America/New_York
```

### Test it first — click "Execute node"

After configuring, click **"Execute node"** (the play button on the node itself,
not the big Execute Workflow button).

You'll see output data in the right panel. It'll look like:

```json
{
  "latitude": 40.710335,
  "longitude": -73.99822,
  "current": {
    "time": "2026-05-22T14:00",
    "temperature_2m": 22.3,
    "weathercode": 3,
    "windspeed_10m": 15.2
  }
}
```

**This is your first live API response in n8n.** The HTTP Request node wrapped
the entire HTTP call, handled JSON parsing, and handed you the result as an
n8n item.

---

## Step 4 — Understand What Just Happened to Your Data

Before moving on, look at what the HTTP Request node output.

The entire API response was wrapped in a single n8n item:

```json
[
  {
    "json": {
      "latitude": 40.710335,
      "longitude": -73.99822,
      "current": {
        "time": "2026-05-22T14:00",
        "temperature_2m": 22.3,
        "weathercode": 3,
        "windspeed_10m": 15.2
      }
    }
  }
]
```

n8n wraps every piece of data in a `json` property. This is the n8n item
structure. We'll cover this deeply in Lesson 04. For now: when you want to
access `temperature_2m`, you reference it as `$json.current.temperature_2m`.

---

## Step 5 — Add a Set Node to Extract Key Fields

The full API response has a lot of data. We only want:
- The temperature
- The weather code
- Whether it's raining

Add a **Set** node after the HTTP Request.

**What the Set node does:** Creates a new object with exactly the fields you
specify. Think of it as a `pick` or field selector.

### Configure the Set node:

Mode: **Manual mapping**

Add these assignments:

| Name | Value |
|------|-------|
| `temperature` | `{{ $json.current.temperature_2m }}` |
| `weathercode` | `{{ $json.current.weathercode }}` |
| `wind_speed` | `{{ $json.current.windspeed_10m }}` |
| `city` | `New York` |

The `{{ }}` syntax is n8n's expression syntax. `$json` refers to the current
item's JSON data from the previous node. We'll cover expressions in depth in
Lesson 05.

### Execute the Set node (click its play button)

Output should look like:
```json
{
  "temperature": 22.3,
  "weathercode": 3,
  "wind_speed": 15.2,
  "city": "New York"
}
```

Now you have clean, structured data. The Set node transformed the raw API
response into exactly what you need downstream.

---

## Step 6 — Add an IF Node to Check for Rain

The Open-Meteo weather code (`weathercode`) uses WMO Weather Codes:
- `0` = Clear sky
- `1-3` = Partly cloudy
- `51-67` = Drizzle
- `61-67` = Rain
- `71-77` = Snow
- `80-82` = Rain showers
- `95-99` = Thunderstorm

Rain is happening when `weathercode >= 51`.

Add an **IF** node after the Set node.

### Configure the IF node:

Condition:
```
Value 1:  {{ $json.weathercode }}
Operation: >=  (greater than or equal)
Value 2:  51
```

This creates two output branches:
- **True branch** (left output): weathercode >= 51 → it's raining
- **False branch** (right output): weathercode < 51 → no rain

---

## Step 7 — Add Output Nodes for Each Branch

### True branch (rain):

Add a **Set** node connected to the **True** output of the IF node.

Add field:
```
message = "☔ It's raining in {{ $json.city }}! Temperature: {{ $json.temperature }}°C. Bring an umbrella."
```

### False branch (no rain):

Add another **Set** node connected to the **False** output.

Add field:
```
message = "☀️ No rain in {{ $json.city }} today. Temperature: {{ $json.temperature }}°C. Enjoy!"
```

---

## Step 8 — Execute the Full Workflow

Click **"Execute Workflow"** (the big button at the bottom).

Watch what happens:
1. Manual Trigger fires → outputs `[{}]`
2. HTTP Request node runs → fetches weather data → outputs `[{json: {...}}]`
3. Set node runs → extracts fields → outputs `[{temperature, weathercode, ...}]`
4. IF node runs → evaluates condition → routes to True or False branch
5. The correct Set node runs → outputs the message

You'll see the data flowing visually — nodes light up green as they succeed.

**Click any node after execution** to see:
- Left panel: what data came IN to the node
- Right panel: what data went OUT

This is the core debug workflow. You'll do this hundreds of times.

---

## Step 9 — View the Execution in History

Click **"Executions"** in the top bar.

You'll see your "Weather Alert" execution listed with:
- Timestamp
- Status: Success
- Duration

Click it. You get the **full execution view** — every node, its input, its
output, its timing. This is what production debugging looks like.

---

## Step 10 — Convert to a Scheduled Trigger

Manual execution is fine for development. Let's make this run automatically
every morning at 8am.

1. Click the **Manual Trigger** node
2. At the top of the right panel, click **"Change trigger"**  
   (or delete the Manual Trigger and add a **Schedule Trigger** node instead)
3. Configure the Schedule Trigger:
   - Mode: **Cron**
   - Cron expression: `0 8 * * *`  (8:00 AM every day)
   - Or use the visual: Every Day → Hour: 8 → Minute: 0

**Activate the workflow** by clicking the toggle in the top-right: `Active`

Now the workflow will run automatically every morning at 8am. n8n's scheduler
checks every minute and fires workflows whose cron expressions match.

---

## Understanding Expressions — A Preview

You used `{{ $json.temperature }}` in this lesson. Let's be precise:

```
{{ $json.current.temperature_2m }}
 ↑    ↑     ↑          ↑
 │    │     │          └── the field name
 │    │     └── the path into the JSON
 │    └── refers to the current item's JSON data
 └── expression delimiter (everything inside is evaluated as JavaScript)
```

You can use any JavaScript expression inside `{{ }}`:

```
{{ $json.temperature > 30 ? "Hot!" : "Cool" }}
{{ $json.city.toLowerCase() }}
{{ new Date().toISOString() }}
{{ Math.round($json.temperature) }}
```

Lesson 05 covers this fully. For now: `{{ $json.fieldName }}` gets you the
value of any field from the previous node's output.

---

## Common Mistakes at This Stage

### Mistake 1: Forgetting to save

n8n does NOT auto-save. Always click **Save** before closing. The keyboard
shortcut is **Cmd+S**.

### Mistake 2: Testing a node without the previous node's data

When you click "Execute node" on a single node, it needs data from upstream.
If you haven't run the upstream nodes yet (or pinned their output — Lesson 04),
the node will run with empty input.

**Fix:** Run the whole workflow first, or pin data on the upstream node.

### Mistake 3: Confusing the two execute buttons

- **"Execute node"** (play button on the node) = runs just that one node
- **"Execute Workflow"** (bottom bar) = runs the whole workflow from the trigger

### Mistake 4: The IF node's true/false routing

The IF node has TWO outputs. You must connect BOTH if you want both branches
to do something. A branch with no connected node simply ends — the items
traveling that path are dropped.

---

## Summary

- Built a complete weather alert workflow: fetch → transform → branch → output
- Learned the edit → execute → debug loop: click nodes, inspect input/output
- Saw how the Set node extracts and renames fields
- Used the IF node to branch based on a condition
- Converted to a scheduled trigger for automatic execution
- Every execution is recorded — click Executions to see full history

---

## Exercise: Extend the Workflow

Extend your weather alert to also include snow conditions:

1. The IF node currently only checks for rain (code >= 51)
2. Add a second IF node on the True branch to check if it's specifically snow
   (weathercode between 71 and 77)
3. Create three output messages: "Raining", "Snowing", "Clear"

**Hint:** Chain IF nodes — the True output of the first IF feeds into a second
IF that checks the narrower condition.

---

## Check Your Understanding — Q&A

### Q1. What does the Set node do?

**Answer:** The Set node creates a new item with exactly the fields you specify.
It's a field picker/transformer — you use it to clean up API responses, rename
fields, add computed values, or slim down data to only what downstream nodes need.

---

### Q2. How do you access data from the previous node?

**Answer:** Using `$json.fieldName` inside an expression (`{{ }}`). `$json`
always refers to the current item being processed from the immediately previous
node. For nested fields: `$json.parent.child.value`.

---

### Q3. What does the IF node output?

**Answer:** Two outputs — True (items where the condition was true) and False
(items where it was false). Each output can connect to a different downstream
node. Items that go down the True branch are NOT on the False branch and vice
versa.

---

### Q4. Where do you see what data a node received and produced?

**Answer:** Click any node after execution — the panel shows "Input" (what it
received) on the left and "Output" (what it produced) on the right. Alternatively,
open the execution from the Executions tab to see the full historical record for
any past run.

---

## Next Lesson

**[Lesson 04 →](04-the-data-model.md)** — The data model in depth: exactly how
items work, what happens when a node processes multiple items, what "pinning"
is, and why understanding this separates beginners from experts.
