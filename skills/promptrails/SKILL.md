---
name: promptrails
description: >
  Use when working with PromptRails — the AI agent orchestration platform.
  Triggers when code imports promptrails SDK (Python, JavaScript, or Go),
  uses the promptrails CLI, or the user asks about PromptRails agents,
  prompts, tracing, MCP tools, data sources, or executions.
---

# PromptRails Skill

You are an expert at working with [PromptRails](https://promptrails.ai), the AI agent orchestration platform for building, deploying, and monitoring LLM-powered applications.

## When to Use This Skill

- Code imports `promptrails` (Python), `@promptrails/sdk` (JavaScript/TypeScript), or `github.com/promptrails/go-sdk` (Go)
- User asks about PromptRails agents, prompts, executions, tracing, guardrails, MCP tools, or data sources
- User wants to use the `promptrails` CLI
- User is integrating PromptRails into their application

## Core Workflows

### 1. Execute an Agent

The most common operation — run an agent and get results.

**Python:**
```python
from promptrails import PromptRails

client = PromptRails(api_key="pr_key_...")
result = client.agents.execute("agent-id", input={"query": "Hello"})
print(result.output)
```

**JavaScript/TypeScript:**
```typescript
import { PromptRails } from "@promptrails/sdk";

const client = new PromptRails({ apiKey: "pr_key_..." });
const result = await client.agents.execute("agent-id", {
  input: { query: "Hello" },
});
```

**Go:**
```go
client := promptrails.NewClient("pr_key_...")
result, err := client.Agents.Execute(ctx, "agent-id", &promptrails.ExecuteAgentParams{
    Input: map[string]any{"query": "Hello"},
})
```

**CLI:**
```bash
promptrails agent execute <agent-id> --input '{"query": "Hello"}'
```

### 2. Manage Agents and Prompts

Create, update, version, and promote agents and prompts through any SDK or the CLI.

- Agents support five types: `simple`, `chain`, `multi_agent`, `workflow`, `composite`
- Each SDK exposes typed `AgentConfig` classes (`SimpleAgentConfig`,
  `ChainAgentConfig`, `MultiAgentConfig`, `WorkflowAgentConfig`,
  `CompositeAgentConfig`) that inject the `type` discriminator
  automatically — don't build the config JSON by hand
- Prompts use Jinja2 templating with versioning and model assignment;
  call `prompts.run_prompt()` (not `execute()`) to test one without an
  agent
- Both support input/output JSON schemas

### 3. Stream Live Output

Chat turns and execution progress stream over SSE in all three SDKs.
Iterate typed events and dispatch per event kind — `execution`,
`thinking`, `tool_start`, `tool_end`, `content`, `done`, `error`.

**Python:**
```python
for event in client.chat.send_message_stream(session_id, content="Hello"):
    if isinstance(event, ContentEvent):
        print(event.content, end="", flush=True)
```

**JavaScript:**
```typescript
for await (const event of client.chat.sendMessageStream(sessionId, { content: "Hello" })) {
  if (event.type === "content") process.stdout.write(event.content);
}
```

**Go:**
```go
stream, _ := client.Chat.SendMessageStream(ctx, sessionID, params)
defer stream.Close()
for stream.Next() {
    if e, ok := stream.Event().(*promptrails.ContentEvent); ok {
        fmt.Print(e.Content)
    }
}
```

For an execution started outside chat (e.g. `agents.execute`), subscribe
to its live stream with `executions.stream(execution_id)`.

### 4. Observe and Debug

Use tracing to understand execution flow, costs, and errors:

```python
traces = client.traces.list(agent_id="agent-id", kind="llm")
```

17 span kinds track every step: `agent`, `llm`, `tool`, `datasource`, `prompt`, `guardrail`, `chain`, `workflow`, `agent_step`, `mcp_call`, `preprocessing`, `postprocessing`, `embedding`, `speech`, `image`, `video`, `storage`.

### 5. Look Up Documentation

Fetch the latest PromptRails documentation:

```bash
# Browse documentation index
curl https://promptrails.ai/llms.txt

# Fetch a specific doc page as markdown
curl https://promptrails.ai/docs/<topic>.md

# Available topics: agents, prompts, executions, tracing, guardrails,
# mcp-tools, data-sources, scoring-and-evaluation, approvals,
# cli, python-sdk, javascript-sdk, go-sdk, quickstart, and more
```

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Agent** | Execution unit combining prompts, tools, data sources, and guardrails |
| **Prompt** | Versioned Jinja2 template with model assignment and caching |
| **Execution** | A single agent run with status, output, cost, and trace |
| **Trace** | Tree of spans recording every step of an execution |
| **MCP Tool** | External tool connected via Model Context Protocol (API, datasource, builtin, remote_mcp) |
| **Data Source** | Database connection with versioned parameterized queries |
| **Guardrail** | Input/output scanner (toxicity, PII, prompt injection, etc.) |
| **Score** | Execution quality metric (numeric, categorical, or boolean) |
| **Credential** | Encrypted provider credentials (OpenAI, Anthropic, Gemini, etc.) |

## SDKs and Tools

| Tool | Install | Current | Docs |
|------|---------|---------|------|
| Python SDK | `pip install "promptrails>=0.3.0"` | 0.3.0 | [Reference](references/python-sdk.md) |
| JavaScript SDK | `npm install @promptrails/sdk@^0.3.1` | 0.3.1 | [Reference](references/javascript-sdk.md) |
| Go SDK | `go get github.com/promptrails/go-sdk@v0.3.1` | 0.3.1 | [Reference](references/go-sdk.md) |
| CLI | `brew install promptrails/tap/promptrails` | 0.3.0 | [Reference](references/cli.md) |

## Important Patterns

1. **Always use API keys** — all SDK clients require a `pr_key_...` API key
2. **Workspace-scoped** — all resources belong to a workspace; the CLI and SDKs scope by API key
3. **Versioned resources** — agents, prompts, and data sources use immutable versions with promotion
4. **KSUID identifiers** — all IDs are K-Sortable Unique Identifiers (27 chars)
5. **Error handling** — SDKs provide typed errors: `NotFoundError`, `ValidationError`, `RateLimitError`, `QuotaExceededError`
6. **Async support** — Python SDK has `AsyncPromptRails`, JS SDK is async-native

## Reference Materials

For detailed information on specific topics, see the reference files:

- [CLI Reference](references/cli.md) — All CLI commands and usage
- [Python SDK Reference](references/python-sdk.md) — Full Python SDK API
- [JavaScript SDK Reference](references/javascript-sdk.md) — Full JS/TS SDK API
- [Go SDK Reference](references/go-sdk.md) — Full Go SDK API
- [Agents Guide](references/agents.md) — Agent types, configuration, and execution
- [Prompts Guide](references/prompts.md) — Prompt management and Jinja2 templating
- [Tracing Guide](references/tracing.md) — Distributed tracing and observability
- [MCP Tools Guide](references/mcp-tools.md) — Tool integration via MCP
- [Data Sources Guide](references/data-sources.md) — Database connections and queries
