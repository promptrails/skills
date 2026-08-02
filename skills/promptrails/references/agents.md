# PromptRails Agents Guide

## Agent Types

PromptRails has exactly two agent types:

### Agent

A prompt plus optional tools and sub-agents. On its own it is a single
prompt → LLM → output loop; attach MCP tools and it can call them; attach
sub-agents and it becomes a **supervisor** that delegates to (or hands off
to) other agents. Best for question-answering, classification, tool-using
assistants, and agents-as-tools orchestration.

```
Input -> Prompt Rendering -> LLM Call (+ tools / sub-agents) -> Output
```

### Workflow

A deterministic DAG of nodes. Each node can be a prompt, tool call, data
source query, or sub-agent, wired by `depends_on` edges. Best when you need
explicit, reproducible control flow instead of letting the model decide.

```
Input -> Node A -> [Node B, Node C] -> Node D -> Output
```

## Agent Configuration

Each agent version carries a typed `config` whose shape depends on the
agent type. All three SDKs expose the config as a discriminated union of
two variants — the SDK injects the `type` discriminator automatically so
you never build the JSON by hand.

`config` describes **structure only**. Model + sampling, run budget,
approval policy, cache TTL, VFS/masking toggles, and the attached tools /
sub-agents / guardrails are **version-scoped siblings of `config`** passed
alongside it to `create_version` — they are not part of the config payload.

### Typed Variants

| Variant | Type | Required fields | Notes |
|---------|------|-----------------|-------|
| `PromptAgentConfig` | `agent` | `prompt_id` | Behavior comes from the version's model config, tools, and sub-agents |
| `WorkflowAgentConfig` | `workflow` | `nodes: WorkflowNode[]` | Deterministic DAG; each node wires `depends_on` edges |

`WorkflowNode` carries `{id, depends_on, prompt_id?, agent_id?, tool_id?, ...}`
— see each SDK's `agent_config` module for the full per-node fields.

### Version-Scoped Runtime (siblings of `config`)

| Field | Purpose |
|-------|---------|
| `model_config` | Model + sampling: `model_id`, `fallback_model_id`, `temperature`, `top_p`, `top_k`, `max_tokens` (all optional; unset inherits the model default) |
| `run_budget` | Bounds the whole execution tree, enforced at the root: `max_cost`, `max_total_tokens`, `max_tool_calls`, `max_children`, `max_depth` |
| `approval_policy` | Who may approve/deny a parked call: `mode` (`admins` \| `assigned` \| `any_member`) + optional `member_ids` |
| `cache_timeout` | Response cache TTL in seconds |
| `vfs_enabled` / `masking_enabled` | Version-scoped VFS and PII-masking overrides |
| `tools` | List of `ToolAttachment` (`mcp_tool_id`, `requires_approval`, `no_retry`, `sort_order`) |
| `sub_agents` | List of `SubAgentAttachment` (`agent_id`, `alias`, `mode` = `delegate` \| `handoff`, `context_mode` = `task` \| `window`, `requires_approval`) |
| `guardrails` | List of `GuardrailSpec` attachments |

### Building a Version (Python)

```python
from promptrails import (
    PromptAgentConfig,
    ModelConfig,
    RunBudget,
    ApprovalPolicy,
    ToolAttachment,
    SubAgentAttachment,
)

client.agents.create_version(
    "agent-id",
    version="2",
    config=PromptAgentConfig(prompt_id="p1"),   # .to_dict() injects "type": "agent"
    model_config=ModelConfig(model_id="gpt-4o", temperature=0.3),
    run_budget=RunBudget(max_cost=1.0, max_tool_calls=20),
    approval_policy=ApprovalPolicy(mode="admins"),
    tools=[ToolAttachment(mcp_tool_id="tool-1", requires_approval=True)],
    sub_agents=[SubAgentAttachment(agent_id="researcher", alias="research", mode="delegate")],
    set_current=True,
    message="v2",
)

# Deterministic workflow
from promptrails import WorkflowAgentConfig, WorkflowNode

workflow = WorkflowAgentConfig(
    nodes=[
        WorkflowNode(id="extract", prompt_id="p1"),
        WorkflowNode(id="summarize", prompt_id="p2", depends_on=["extract"]),
    ],
)
```

### Building a Version (TypeScript)

```typescript
import { PromptAgentConfig, ModelConfig } from "@promptrails/sdk";

const config: PromptAgentConfig = { type: "agent", prompt_id: "p1" };

await client.agents.createVersion("agent-id", {
  version: "2",
  config,
  model_config: { model_id: "gpt-4o", temperature: 0.3 },
  run_budget: { max_cost: 1.0, max_tool_calls: 20 },
  tools: [{ mcp_tool_id: "tool-1", requires_approval: true }],
  set_current: true,
  message: "v2",
});
```

### Building a Version (Go)

Implement `promptrails.AgentConfig` with `PromptAgentConfig` or
`WorkflowAgentConfig` — each one's `MarshalJSON` writes the correct `type`
discriminator.

```go
_, err := client.Agents.CreateVersion(ctx, "agent-id", &promptrails.CreateVersionParams{
    Version:     "2",
    Config:      promptrails.PromptAgentConfig{PromptID: "p1"},
    ModelConfig: &promptrails.ModelConfig{ModelID: "gpt-4o", Temperature: ptr.Float64(0.3)},
    RunBudget:   &promptrails.RunBudget{MaxCost: ptr.Float64(1.0)},
    Tools:       []promptrails.ToolAttachment{{MCPToolID: "tool-1", RequiresApproval: true}},
    SetCurrent:  true,
    Message:     "v2",
})
```

## Agent Status

| Status | Description |
|--------|-------------|
| `active` | Available for execution |
| `archived` | Hidden, cannot be executed (soft delete) |

## Versioning

Agents use immutable versioning:

- Each version captures the full configuration (structure + runtime)
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

Agents carry arbitrary string labels (returned on the agent) for organizing
and filtering:

```python
agent = client.agents.create(name="My Agent", type="agent")
print(agent.labels)
```

## Execution

```python
result = client.agents.execute("agent-id",
    input={"message": "Help with order #12345"},
    metadata={"user_id": "customer-456", "channel": "web"}
)

print(result.status)       # "completed", "failed", "waiting_approval", "cancel_requested", "cancelled"
print(result.output)       # Agent response
print(result.cost)         # Cost in USD
print(result.duration_ms)  # Execution time
```

### Executions Are Trees

A supervisor delegation, a handoff continuation, or a workflow node run
starts a **child** execution with `parent_execution_id` set. The root
execution's `run_budget` bounds the whole tree. Fetch the full tree, or
cancel a running tree cooperatively:

```python
tree = client.executions.tree("execution-id")   # root + nested children
for child in tree.children:
    print(child.agent_id, child.status)

client.executions.cancel("execution-id")         # -> status "cancel_requested"
```

### Human-in-the-Loop Approvals

A tool call or sub-agent whose attachment sets `requires_approval` parks
its execution at `waiting_approval` and waits for a decision governed by the
version's `approval_policy`. Approvals are **execution-scoped**:

```python
inbox = client.executions.approval_inbox()        # runs parked at waiting_approval
client.executions.approve("execution-id", reason="looks safe")   # resume
client.executions.deny("execution-id", reason="not allowed")     # resume with denial
```

## Guardrails

Built-in scanner types for input/output validation (toxicity, PII,
prompt injection, and more). Attach them on the agent version via
`guardrails=[GuardrailSpec(...)]`, or manage them with the
`agents.create_guardrail` / `agents.list_guardrails` endpoints. Each
guardrail runs as `block`, `redact`, or `log`.
