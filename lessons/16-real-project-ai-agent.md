# Lesson 16 — Real Project 3: AI Agent with Memory

## Goal
Build a production-ready AI assistant in n8n that accepts customer questions, remembers previous conversations, uses a knowledge base for context, and optionally dispatches other workflows as "tools." Applies: webhooks, OpenAI Chat, database-backed memory, code logic, conditional routing, and sub-workflows.

---

## Why This Project Matters
Modern automation systems need more than one-off triggers: they need context, persistence, and the ability to act like an agent. This lesson teaches how to wire n8n into an AI workflow that is:
- conversational, not stateless
- aware of prior customer interactions
- capable of deciding whether to answer directly or call another workflow
- resilient when the AI output is incomplete or requires fallback handling

---

## What We're Building

```
[Webhook: POST /ai-chat]
          │
          ▼
[HTTP Request / DB: load recent memory]
          │
          ▼
[Code: assemble prompt + tool metadata]
          │
          ▼
[OpenAI Chat: ask GPT-4]
          │
          ▼
[Code: parse response for tool call or direct answer]
      ┌───┴────────┐
      ▼            ▼
[Execute: workflow]  [Respond to Webhook]
      │            │
      ▼            ▼
[Write memory]   [Write memory]
```

The agent can answer a question directly, or return a structured tool call such as `lookup-kb` or `create-ticket`.

---

## Core Concepts

- `OpenAI Chat` node: Use system and user prompts to steer behavior
- Memory store: keep user + assistant messages in a database or Airtable
- Prompt engineering: include only recent history + relevant facts
- Tool call design: let the AI decide whether to invoke another workflow
- Webhook trigger: accept messages from your frontend or external system

---

## Step 1 — Create the Workflow

1. New workflow → name: **"AI Agent with Memory"**
2. Add a **Webhook** trigger
3. Method: `POST`
4. Path: `ai-chat`
5. Response Mode: **Using 'Respond to Webhook' Node**

Your test URL will look like:

```
Test:       http://localhost:5678/webhook-test/ai-chat
Production: http://localhost:5678/webhook/ai-chat
```

---

## Step 2 — Accept the User Message

Add a **Code** node after the webhook to normalize the incoming payload.

```javascript
const body = $json.body;
if (!body?.message) {
  throw new Error('Request body must contain a message field.');
}

return {
  message: body.message.trim(),
  userId: body.userId || body.user_id || 'anonymous',
  sessionId: body.sessionId || body.session_id || $now.toMillis().toString(),
  source: body.source || 'web',
  timestamp: $now.toISO()
};
```

This node ensures the workflow always has a clean `message`, `userId`, and `sessionId`.

---

## Step 3 — Load Recent Conversation Memory

The AI needs context. Use a database query or Airtable lookup to fetch the most recent messages for this user/session.

### Option A: Postgres / MySQL

Use an **HTTP Request** node if you access your database via an HTTP API, or a native DB node if available. Example query:

```
SELECT role, content, created_at
FROM ai_memory
WHERE session_id = {{ $json.sessionId }}
ORDER BY created_at DESC
LIMIT 10;
```

### Option B: Airtable

Use the Airtable node to search records where `sessionId` equals the incoming session, sort by created time, and limit to the last 10 messages.

### Code node to normalize memory:

```javascript
const rows = $json.records || $json.rows || [];
return rows
  .map(row => ({
    role: row.role || row.fields?.role || row[0],
    content: row.content || row.fields?.content || row[1],
    createdAt: row.created_at || row.fields?.createdAt || row[2]
  }))
  .sort((a, b) => new Date(a.createdAt) - new Date(b.createdAt));
```

The result should be an ordered array of prior messages, oldest first.

---

## Step 4 — Build the Chat Prompt

Add a **Code** node to construct the prompt payload for the OpenAI Chat node.

```javascript
const memory = $json.memory || [];
const userMessage = $json.message;

const systemPrompt = `You are a helpful customer support assistant.
You have access to two tools: lookup-kb and create-ticket.
If the user asks a factual question about product behavior or documentation,
answer directly using the knowledge base if available.
If the user asks for support action, return a JSON object with a tool call.
Do not invent tool calls unless the user explicitly requests an action.
`;

const messages = [
  { role: 'system', content: systemPrompt }
];

for (const item of memory) {
  messages.push({ role: item.role, content: item.content });
}

messages.push({ role: 'user', content: userMessage });

return {
  chatPayload: {
    model: 'gpt-4',
    temperature: 0.2,
    messages,
    max_tokens: 800
  }
};
```

Keep the system prompt short and concrete. This defines the agent’s behavior.

---

## Step 5 — Call OpenAI Chat

Add an **OpenAI Chat** node. Use your OpenAI credential and map the payload:

```
Model: GPT-4
Input: {{ $json.chatPayload.messages }}
Temperature: {{ $json.chatPayload.temperature }}
Max Tokens: 800
```

If your OpenAI node supports direct JSON, you can pass the structured `messages` array.

---

## Step 6 — Parse the AI Response

The AI may either answer directly or request a tool. Add a **Code** node to inspect the assistant reply.

```javascript
const assistantText = $json.choices?.[0]?.message?.content || $json.text || '';

const toolPattern = /```json\s*([\s\S]*?)\s*```/i;
const match = assistantText.match(toolPattern);
let toolCall = null;

if (match) {
  try {
    toolCall = JSON.parse(match[1]);
  } catch (error) {
    // If JSON parsing fails, keep it as a plain answer instead.
    toolCall = null;
  }
}

return {
  assistantText,
  toolCall,
  shouldCallTool: !!toolCall?.tool,
  toolName: toolCall?.tool,
  toolInput: toolCall?.input || null
};
```

This code looks for a fenced JSON block in the assistant response. If found, it will trigger a sub-workflow.

---

## Step 7 — Branch by Tool Call

Add an **IF** node after the parser:
- Condition: `shouldCallTool === true`

True branch: execute the tool workflow
False branch: respond directly

### Tool design
Use a separate workflow for each tool, for example:
- `lookup-kb` → searches your FAQ / knowledge base and returns a concise answer
- `create-ticket` → creates a support ticket in Zendesk/Airtable/Sheets

You can use an `Execute Workflow` node or a webhook to handle the tool call.

---

## Step 8 — Build a `lookup-kb` Sub-Workflow

Create a second workflow named **"AI Tool: lookup-kb"** with a `When Called By Another Workflow` trigger.

Nodes:
1. `When Called By Another Workflow`
2. `Code` node to normalize tool input
3. `Airtable` / `Google Sheets` / `HTTP Request` node to search the KB
4. `Set` node to return `{ answer, found }`

Example normalization:

```javascript
return {
  query: $json.toolInput?.query || $json.query || ''
};
```

Example result output:

```javascript
return {
  answer: 'The product supports up to 10 workflows per account.',
  found: true
};
```

Then link this workflow via an `Execute Workflow` node from the main flow.

---

## Step 9 — Return the Final Response

### Direct answer path
If no tool is required, respond with the assistant text.

### Tool path
If a tool was called, respond with the tool result plus a short assistant confirmation message.

Example code for the response node:

```javascript
const direct = $json.assistantText;
const toolResult = $json.toolResult?.answer || null;
const toolName = $json.toolName;

return {
  reply: toolResult
    ? `I found this for you with ${toolName}: ${toolResult}`
    : direct,
  status: toolResult ? 'tool' : 'direct',
  toolName
};
```

Use `Respond to Webhook` to send the final JSON back to the caller.

---

## Step 10 — Persist the Conversation

Never lose context. After your final response, store both the user message and the assistant reply in your memory store.

### Memory record example
- sessionId
- userId
- role: `user` or `assistant`
- content
- createdAt

### Using Airtable / Postgres
Append one record for the user message and one for the assistant response.

This gives the next request enough context to continue the conversation.

---

## Step 11 — Manage Context Size

Large histories can blow your token budget. Use one of these strategies:
- keep only the last 5–10 turns for the same `sessionId`
- summarize older conversation in a single note
- store metadata separately (e.g. customer name, product, plan)

### Summarization pattern
If the history grows too large, add a summary step:
1. fetch older messages beyond your retention window
2. call OpenAI with `Please summarize this conversation into 4 sentences.`
3. store the summary as a single `assistant` message with role `system` or `assistant`
4. delete or archive older raw turns

---

## Step 12 — Example Use Cases

### Use Case 1: Customer support question
Request:
```json
{ "message": "How do I reset my API key?", "userId": "user123" }
```
Expected behavior:
- load recent memory
- answer directly if the KB contains instructions
- save the reply for later context

### Use Case 2: Knowledge base lookup
Request:
```json
{ "message": "Check the docs for password policy", "userId": "user123" }
```
Expected behavior:
- trigger `lookup-kb`
- return a focused answer from the KB
- persist the action and result

### Use Case 3: Create a ticket
Request:
```json
{ "message": "My webhook is failing, please create a support ticket", "userId": "user123" }
```
Expected behavior:
- parse a tool call to `create-ticket`
- call the ticket workflow
- return confirmation text to the user

---

## Best Practices

- keep the system prompt stable and explicit
- avoid “hallucinated” tool calls by giving the AI a strict JSON-only fallback format
- separate the agent logic from tool execution in modular workflows
- store only what you need for future context
- use a persistent `sessionId` so memory stays tied to the right conversation

---

## Troubleshooting

- If GPT responds with plain text instead of JSON tool instructions, tighten the prompt and add examples
- If memory loads too slowly, reduce the number of rows or store a lighter representation
- If the agent repeats itself, ensure you append both user and assistant turns in order
- If the webhook fails, inspect the raw payload in the n8n execution log

---

## What You've Applied

| Concept | Where |
|---------|-------|
| Webhook trigger | Entry point for all chat messages |
| Respond to Webhook | Return reply to the caller |
| Code node | Normalize input, build prompt, parse AI response |
| OpenAI Chat node | Send message history to GPT-4 |
| IF node | Branch on tool call vs. direct answer |
| Execute Workflow | Dispatch to tool sub-workflows |
| Sub-workflow trigger | Receive tool inputs from the main agent |
| Airtable / Postgres | Persist conversation memory |
| Context window management | Limit + summarize history to stay under token budget |
| System prompt engineering | Define agent behavior and tool contract |

---

## Check Your Understanding — Q&A

### Q1. The AI sometimes returns plain text instead of a JSON tool block when it should call a tool. What causes this and how do you fix it?

**Answer:** GPT models follow natural language instructions probabilistically — a vague system prompt like "use a tool if needed" leaves room for free-text improvisation. Fix it at three levels: (1) **Prompt constraint** — tell the model exactly when to return a tool block and show a concrete example: `When you need to call a tool, respond ONLY with a JSON code block: \`\`\`json\n{"tool": "lookup-kb", "input": {"query": "..."}}\n\`\`\``. (2) **Few-shot examples** — include one user/assistant pair in the prompt that demonstrates a tool call. (3) **Validation fallback** — in your parse Code node, if `toolCall` is null but the message mentions a tool name, flag it and re-prompt or treat it as a direct answer rather than a silent failure. Models follow tight examples more reliably than abstract rules.

---

### Q2. Two users send messages at the same time. Both requests load memory, build a prompt, and try to write the response to memory simultaneously. What goes wrong and how do you prevent it?

**Answer:** The race is in the **read-modify-write** cycle: both executions read the same last 10 messages, build independent prompts, then both try to append to the memory store. If they finish writing at the same time, one write may overwrite the other, or both messages may land without ordering, breaking the conversation sequence for the next request. Fix: use a database (Postgres) with an `INSERT` that includes a `created_at` timestamp — the DB serializes writes and the next query just sorts by that column. With Airtable you cannot enforce this atomically, so either (a) accept the race for low-volume use and sort by `createdAt` on every read, or (b) switch to a DB and use a unique `(sessionId, createdAt)` index. The session-scoped writes means different `sessionId` values never conflict — this race only happens within the same active session.

---

### Q3. After 30 conversation turns the OpenAI request starts throwing `context_length_exceeded` errors. Walk through the full context management strategy.

**Answer:** The cause is that your raw message history exceeds the model's token limit (8k for GPT-4, 128k for GPT-4 Turbo). The fix is layered: (1) **Hard limit** — only load the most recent N turns from memory (start with 10). This is the cheapest fix and handles most cases. (2) **Token counting** — instead of a fixed turn count, estimate token usage before calling OpenAI (`(role + content length) / 4 ≈ tokens`). If the estimate exceeds 6000 tokens, trim further. (3) **Summarization** — when history is trimmed, add a summary step: call OpenAI separately to compress older turns into a single `{"role": "system", "content": "Prior context summary: ..."}` message. Store the summary back; next requests include the summary instead of raw old turns. (4) **Metadata injection** — keep a separate record of stable facts (user name, plan, product) and inject them as a single system message rather than keeping them in the conversational history. This keeps the history window purely for recent back-and-forth.

---

### Q4. Why build tool logic (lookup-kb, create-ticket) as separate sub-workflows instead of inline Code nodes in the main agent workflow?

**Answer:** Three reasons matter at engineering scale: (1) **Independent testability** — you can trigger `lookup-kb` directly with test data without running the full agent, making the KB search logic easy to iterate. (2) **Reuse** — other workflows (a Telegram bot, a support dashboard) can call the same `lookup-kb` sub-workflow without duplicating the search logic. (3) **Failure isolation** — if `create-ticket` throws a 500 from Zendesk, only the tool sub-workflow fails. The agent can catch that error from `Execute Workflow`, log it, and reply "I couldn't create the ticket right now" rather than the entire agent execution dying. Inline Code nodes mix concerns — the agent becomes monolithic, hard to test, and fragile when any one tool changes.

---

### Q5. Your agent works but always responds in a flat tone regardless of what users ask. You want it to adapt its tone based on the incoming source (web, mobile, internal). How do you implement this?

**Answer:** Parameterize the system prompt using the `source` field you already extract in Step 2. In the prompt-building Code node:

```javascript
const toneMap = {
  web:      'Be helpful and professional. Use clear, concise language.',
  mobile:   'Be brief. Responses should be 1-2 sentences maximum.',
  internal: 'Be technical and precise. Include relevant field names and IDs.'
};

const tone = toneMap[$json.source] || toneMap.web;

const systemPrompt = `You are a customer support assistant. ${tone}
You have access to two tools: lookup-kb and create-ticket.
...`;
```

This keeps tone-specific instructions out of the main prompt body, making them easy to update, and uses the source value you already have — no extra node or API call needed. The same pattern applies to language: add a `lang` field from the request and inject `Respond in ${lang}.` into the system prompt.

---

## Next Lesson

See [README](../README.md) for full curriculum.
