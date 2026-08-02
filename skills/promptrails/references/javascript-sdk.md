# PromptRails JavaScript/TypeScript SDK Reference

Current release: **v0.9.0** — API v2: discriminated `AgentConfig` union
(2 agent types: `agent` | `workflow`), version-scoped `ModelConfig` /
`RunBudget` / `ApprovalPolicy`, execution trees with cancel + HITL
approvals, and `traces.getSummary`. See the
[CHANGELOG](https://github.com/promptrails/javascript-sdk/blob/main/CHANGELOG.md)
for the full history.

## Installation

```bash
npm install @promptrails/sdk@^0.9.0
# or
pnpm add @promptrails/sdk@^0.9.0
```

Requires Node.js 18+ (uses native `fetch`). Ships as both ESM and CJS.

## Client Initialization

```typescript
import { PromptRails } from "@promptrails/sdk";

const client = new PromptRails({ apiKey: "pr_key_..." });
```

### Configuration

| Option | Default | Description |
|--------|---------|-------------|
| `apiKey` | required | API key (`pr_key_...`) |
| `baseUrl` | `https://api.promptrails.ai` | API base URL |
| `timeout` | `30000` | Request timeout (ms) |
| `maxRetries` | `3` | Max retries on network/5xx errors |

## Resources

### Agents

```typescript
const agents = await client.agents.list({ page: 1, limit: 20 });
const agent = await client.agents.get("agent-id");
// type is "agent" or "workflow"
const created = await client.agents.create({ name: "My Agent", type: "agent" });
await client.agents.update("agent-id", { name: "New Name" });
await client.agents.delete("agent-id");

// Execute
const result = await client.agents.execute("agent-id", {
  input: { query: "hello" },
  metadata: { userId: "123" },
});

// Versions — config is structure; model/budget/tools are version-scoped siblings
const versions = await client.agents.listVersions("agent-id");
await client.agents.createVersion("agent-id", {
  version: "2",
  config: { type: "agent", prompt_id: "p1" },
  model_config: { model_id: "gpt-4o", temperature: 0.3 },
  run_budget: { max_cost: 1.0, max_tool_calls: 20 },
  tools: [{ mcp_tool_id: "tool-1", requires_approval: true }],
  set_current: true,
  message: "v2",
});
await client.agents.promoteVersion("agent-id", "version-id");

// Playground — run with an ad-hoc prompt override, no saved version
await client.agents.playground("agent-id", {
  input: { query: "hello" },
  prompt_override: { user_prompt: "Answer: {{ query }}" },
});

// Guardrails
const guardrails = await client.agents.listGuardrails("agent-id");
await client.agents.createGuardrail("agent-id", {
  type: "input", scannerType: "prompt_injection", action: "block"
});
```

### Prompts

```typescript
const prompts = await client.prompts.list();
const prompt = await client.prompts.get("prompt-id");
const created = await client.prompts.create({ name: "My Prompt" });
await client.prompts.update("prompt-id", { name: "Updated" });
await client.prompts.delete("prompt-id");

// Versions — prompts are content-only (model/sampling/cache live on the agent version)
const versions = await client.prompts.listVersions("prompt-id");
await client.prompts.createVersion("prompt-id", {
  version: "1",
  systemPrompt: "You are helpful.",
  userPrompt: "Answer: {{ question }}",
  inputSchema: { type: "object", properties: { question: { type: "string" } } },
  message: "Initial version",
});
```

### Executions

```typescript
const executions = await client.executions.list({ agentId: "agent-id", status: "completed" });
const execution = await client.executions.get("execution-id");

// Executions form a tree (supervisor delegation, handoff, workflow nodes)
const tree = await client.executions.tree("execution-id");
await client.executions.cancel("execution-id");

// Human-in-the-loop approvals (execution-scoped)
const inbox = await client.executions.approvalInbox();
await client.executions.approve("execution-id", { reason: "ok" });
await client.executions.deny("execution-id", { reason: "not allowed" });
```

### Credentials

```typescript
const credentials = await client.credentials.list();
const cred = await client.credentials.create({
  name: "OpenAI", category: "llm", type: "openai", value: "sk-..."
});
await client.credentials.setDefault("cred-id");
await client.credentials.checkConnection("cred-id");
await client.credentials.delete("cred-id");
```

### Data Sources

```typescript
const sources = await client.dataSources.list();
const ds = await client.dataSources.create({ name: "Orders DB", type: "postgresql" });
await client.dataSources.createVersion("ds-id", {
  credentialId: "cred-id",
  queryTemplate: "SELECT * FROM orders WHERE id = :id",
  parameters: [{ name: "id", type: "string", required: true }],
  cacheTimeout: 300,
  message: "Initial query",
});
const result = await client.dataSources.query("ds-id", { parameters: { id: "123" } });
await client.dataSources.testConnection("ds-id");
```

### Chat Sessions

```typescript
const sessions = await client.chat.listSessions({ agentId: "agent-id" });
const session = await client.chat.createSession({ agentId: "agent-id", title: "Support" });
const messages = await client.chat.listMessages({ sessionId: "session-id" });
const reply = await client.chat.sendMessage({ sessionId: "session-id", content: "Hello" });
await client.chat.deleteSession("session-id");
```

### Streaming

`chat.sendMessageStream` and `executions.stream` are async generators
that yield typed `StreamEvent` frames on the same connection. Abort
with an `AbortController` signal. Unknown event types are dropped so
the client survives backward-compatible server additions.

```typescript
import { PromptRails, StreamEvent } from "@promptrails/sdk";

const client = new PromptRails({ apiKey: "pr_key_..." });
const session = await client.chat.createSession({ agentId: "agent-id" });

const controller = new AbortController();
for await (const event of client.chat.sendMessageStream(
  session.id,
  { content: "Hello" },
  { signal: controller.signal },
)) {
  switch (event.type) {
    case "execution":
      console.log("execution_id:", event.executionId);
      break;
    case "thinking":
      console.log("[thinking]", event.content);
      break;
    case "tool_start":
      console.log("[tool_start]", event.name);
      break;
    case "tool_end":
      console.log("[tool_end]", event.name, event.summary);
      break;
    case "content":
      process.stdout.write(event.content);
      break;
    case "done":
      console.log("\n[done]", event.tokenUsage?.total_tokens, "tokens");
      break;
    case "error":
      throw new Error(event.message);
  }
}
```

Subscribe to an execution that was started outside chat (e.g.
`agents.execute`) via `client.executions.stream(executionId, { signal })`.

### Traces

```typescript
const traces = await client.traces.list({ agentId: "agent-id", kind: "llm" });
const trace = await client.traces.getByTraceId("trace-id");

// Aggregate cost / token / latency stats over a filtered set of traces
const summary = await client.traces.getSummary({ agentId: "agent-id" });
```

### MCP Tools

```typescript
const tools = await client.mcpTools.list();
const tool = await client.mcpTools.create({ name: "Weather", type: "api", config: {...} });
await client.mcpTools.update("tool-id", { name: "Updated" });
await client.mcpTools.delete("tool-id");
```

### Approvals

Approvals are execution-scoped — see `executions.approvalInbox` /
`executions.approve` / `executions.deny` under **Executions**.

### Webhook Triggers

```typescript
const triggers = await client.webhookTriggers.list();
const trigger = await client.webhookTriggers.create({ name: "Deploy", agentId: "agent-id" });
await client.webhookTriggers.update("trigger-id", { active: true });
await client.webhookTriggers.delete("trigger-id");
```

### A2A Protocol

```typescript
const card = await client.a2a.getAgentCard("agent-id");
const response = await client.a2a.sendMessage("agent-id", { message: {...} });
const task = await client.a2a.getTask("task-id");
const tasks = await client.a2a.listTasks({ agentId: "agent-id" });
await client.a2a.cancelTask("task-id");
```

### Assets

```typescript
const assets = await client.assets.list({ type: "image" });
const { url } = await client.assets.getSignedUrl("asset-id");
await client.assets.delete("asset-id");
```

## Error Handling

```typescript
import {
  NotFoundError, ValidationError, RateLimitError, QuotaExceededError,
} from "@promptrails/sdk";

try {
  const result = await client.agents.execute("agent-id", { input: {} });
} catch (e) {
  if (e instanceof QuotaExceededError) console.log("Limit reached");
  else if (e instanceof RateLimitError) console.log("Rate limited");
  else if (e instanceof NotFoundError) console.log(`Not found: ${e.message}`);
}
```
