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

Each agent version carries a `config` object:

| Field | Description |
|-------|-------------|
| `system_prompt` | Default system prompt |
| `model` | LLM model identifier |
| `temperature` | Sampling temperature (0.0–1.0) |
| `max_tokens` | Maximum response tokens |
| `top_p` | Nucleus sampling parameter |
| `tools` | List of MCP tool IDs |
| `data_sources` | List of data source IDs |
| `guardrails` | Guardrail configurations |
| `memory_enabled` | Whether memory system is active |
| `approval_required` | Whether execution pauses for human approval |
| `checkpoint_name` | Name of the approval checkpoint |

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
