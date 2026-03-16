# PromptRails JavaScript/TypeScript SDK Reference

## Installation

```bash
npm install @promptrails/sdk
# or
pnpm add @promptrails/sdk
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
const created = await client.agents.create({ name: "My Agent", type: "simple", labels: ["prod"] });
await client.agents.update("agent-id", { name: "New Name" });
await client.agents.delete("agent-id");

// Execute
const result = await client.agents.execute("agent-id", {
  input: { query: "hello" },
  metadata: { userId: "123" },
});

// Versions
const versions = await client.agents.listVersions("agent-id");
await client.agents.createVersion("agent-id", { config: {...}, message: "v2" });

// Guardrails
const guardrails = await client.agents.listGuardrails("agent-id");
await client.agents.createGuardrail("agent-id", {
  scannerType: "prompt_injection", direction: "input", action: "block"
});

// Memory
const memories = await client.agents.listMemories("agent-id");
await client.agents.createMemory("agent-id", { type: "fact", content: "User prefers dark mode" });
const results = await client.agents.searchMemories("agent-id", { query: "preferences" });
await client.agents.deleteAllMemories("agent-id");
```

### Prompts

```typescript
const prompts = await client.prompts.list();
const prompt = await client.prompts.get("prompt-id");
const created = await client.prompts.create({ name: "My Prompt" });
await client.prompts.update("prompt-id", { name: "Updated" });
await client.prompts.delete("prompt-id");

// Versions
const versions = await client.prompts.listVersions("prompt-id");
await client.prompts.createVersion("prompt-id", {
  systemPrompt: "You are helpful.",
  userPrompt: "Answer: {{ question }}",
  temperature: 0.7,
  maxTokens: 1024,
  message: "Initial version",
});
```

### Executions

```typescript
const executions = await client.executions.list({ agentId: "agent-id", status: "completed" });
const execution = await client.executions.get("execution-id");
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

### Traces

```typescript
const traces = await client.traces.list({ agentId: "agent-id", kind: "llm" });
const trace = await client.traces.getByTraceId("trace-id");
```

### Costs

```typescript
const summary = await client.costs.getSummary();
const agentCosts = await client.costs.getAgentSummary({ agentId: "agent-id" });
```

### Scores

```typescript
const scores = await client.scores.list({ executionId: "exec-id" });
const score = await client.scores.create({ executionId: "exec-id", configId: "config-id", value: 0.95 });
const configs = await client.scores.listConfigs();
const config = await client.scores.createConfig({ name: "Accuracy", type: "numeric" });
const aggregates = await client.scores.aggregates({ configId: "config-id" });
```

### MCP Tools

```typescript
const tools = await client.mcpTools.list();
const tool = await client.mcpTools.create({ name: "Weather", type: "api", config: {...} });
await client.mcpTools.update("tool-id", { name: "Updated" });
await client.mcpTools.delete("tool-id");
```

### Approvals

```typescript
const approvals = await client.approvals.list();
const approval = await client.approvals.get("approval-id");
await client.approvals.decide("approval-id", { decision: "approved" });
```

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

### Media

```typescript
// List models
const models = await client.mediaModels.list({ provider: "fal", media_type: "image" });

// Generate
const result = await client.media.generate({
  provider: "fal", media_type: "image", model: "fal-ai/flux/schnell",
  prompt: "A sunset", config: { width: 1024, height: 1024 },
});

// Assets
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
