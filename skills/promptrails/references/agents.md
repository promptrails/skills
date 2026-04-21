# PromptRails Agents Guide

## Agent Types

PromptRails supports five agent types:

### Simple

Single prompt → LLM → output. Best for question-answering, classification, text generation.

```
Input -> Prompt Rendering -> LLM Call -> Output
```

### Chain

Sequential multi-step execution where each step's output feeds into the next.

```
Input -> Prompt 1 -> LLM -> Prompt 2 -> LLM -> ... -> Output
```

### Multi-Agent

Parallel or routed execution across multiple sub-agents.

```
Input -> [Agent A, Agent B, Agent C] -> Aggregation -> Output
```

### Workflow

Directed graph with conditional branching. Each step can be a prompt, tool call, data source query, or sub-agent.

```
Input -> Step 1 -> Condition -> Step 2a or Step 2b -> ... -> Output
```

### Composite

Combines multiple agent types into a single orchestration unit.

## Agent Configuration

Each agent version carries a typed `config` whose shape depends on the
agent type. All three SDKs expose the config as a discriminated union
of five variants — the SDK injects the `type` discriminator automatically
so you never build the JSON by hand. Fields like guardrails, memory,
and mcp tool attachments are managed via their own endpoints (see the
`agents.create_guardrail`, `agents.create_memory`, and `mcp_tools`
resources), not the config payload.

### Typed Variants

| Variant | Required fields | Notes |
|---------|-----------------|-------|
| `SimpleAgentConfig` | `prompt_id` | Optional `approval_required`, `approval_checkpoint_name`, `max_tokens`, `temperature`, `llm_model_id` |
| `ChainAgentConfig` | `prompt_ids: PromptLink[]` | Sequential steps; optional approval fields |
| `MultiAgentConfig` | `prompt_ids: PromptLink[]` | Parallel / routed sub-agents |
| `WorkflowAgentConfig` | `nodes: WorkflowNode[]` | Graph of prompt, media, or tool nodes |
| `CompositeAgentConfig` | `steps: CompositeStep[]` | Each step references another agent |

`PromptLink` carries `{prompt_id, role, sort_order}`. `WorkflowNode` and
`CompositeStep` have the per-type fields documented in each SDK's
`agent_config` module.

### Building a Config (Python)

```python
from promptrails import SimpleAgentConfig, ChainAgentConfig, PromptLink

cfg = SimpleAgentConfig(
    prompt_id="p1",
    temperature=0.3,
    approval_required=True,
    approval_checkpoint_name="pii_review",
)

client.agents.create_version(
    "agent-id",
    config=cfg,          # .to_dict() injects "type": "simple" for you
    set_current=True,
    message="v2",
)

# Chain across multiple prompts
chain = ChainAgentConfig(
    prompt_ids=[
        PromptLink(prompt_id="p1", role="extract", sort_order=0),
        PromptLink(prompt_id="p2", role="summarize", sort_order=1),
    ],
)
```

### Building a Config (TypeScript)

```typescript
import { SimpleAgentConfig, ChainAgentConfig } from "@promptrails/sdk";

const cfg: SimpleAgentConfig = {
  type: "simple",
  prompt_id: "p1",
  temperature: 0.3,
};

await client.agents.createVersion("agent-id", {
  config: cfg,
  message: "v2",
});
```

### Building a Config (Go)

Implement `promptrails.AgentConfig` by using one of the concrete types —
each one's `MarshalJSON` writes the correct `type` discriminator.

```go
cfg := promptrails.SimpleAgentConfig{
    PromptID:    "p1",
    Temperature: ptr.Float64(0.3),
}

_, err := client.Agents.CreateVersion(ctx, "agent-id", &promptrails.CreateVersionParams{
    Config:     cfg,
    Message:    "v2",
    SetCurrent: true,
})
```

### Migration from prompt_version_id

Earlier SDK versions used `prompt_version_id` on simple and chain
configs. That field was renamed to `prompt_id` — the agent executor now
always resolves to the current version of a prompt. Legacy configs are
migrated server-side, but new writes MUST use `prompt_id`.

## Agent Status

| Status | Description |
|--------|-------------|
| `active` | Available for execution |
| `archived` | Hidden, cannot be executed (soft delete) |

## Versioning

Agents use immutable versioning:

- Each version captures the full configuration
- One version per agent is marked as "current"
- Promote versions to make them active
- Roll back by promoting a previous version

## Input/Output Schemas

Agent versions can define JSON schemas for structured validation:

```json
{
  "input_schema": {
    "type": "object",
    "properties": {
      "message": { "type": "string" },
      "language": { "type": "string", "default": "en" }
    },
    "required": ["message"]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "response": { "type": "string" },
      "confidence": { "type": "number" }
    }
  }
}
```

## Labels

Agents support arbitrary string labels for filtering:

```python
agent = client.agents.create(name="My Agent", type="simple", labels=["production", "v2"])
```

## Execution

```python
result = client.agents.execute("agent-id",
    input={"message": "Help with order #12345"},
    metadata={"user_id": "customer-456", "channel": "web"}
)

print(result.status)       # "completed", "failed", "awaiting_approval", "rejected"
print(result.output)       # Agent response
print(result.cost)         # Cost in USD
print(result.duration_ms)  # Execution time
```

## Guardrails

14 built-in scanner types for input/output validation:

- Toxicity detection
- PII filtering
- Prompt injection prevention
- And more

Configure per agent with `block`, `redact`, or `log` actions.

## Memory

Five memory types for context-aware agents:

- **conversation** — Chat history
- **fact** — Learned facts
- **procedure** — Workflows and processes
- **episodic** — Past events and experiences
- **semantic** — Conceptual knowledge with vector embeddings
