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

## Next Lesson

See [README](../README.md) for full curriculum.
