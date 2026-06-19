# PromptRails Tracing Guide

## Overview

OpenTelemetry-style distributed tracing that captures every step of agent execution.

Every execution generates a **trace** — a tree of **spans**:

```
[agent] Customer Support Bot (15ms)
  [guardrail] prompt_injection input scan (2ms)
  [prompt] Render main prompt (1ms)
  [llm] gpt-4o call (10ms, 450+120 tokens, $0.003)
  [guardrail] pii output scan (2ms)
```

## 17 Span Kinds

| Kind | Description |
|------|-------------|
| `agent` | Top-level agent execution |
| `llm` | LLM model call |
| `tool` | Tool invocation |
| `datasource` | Database/file query |
| `prompt` | Prompt template rendering |
| `guardrail` | Input/output guardrail scan |
| `chain` | Chain-type orchestration |
| `workflow` | Workflow step execution |
| `agent_step` | Step in multi-agent execution |
| `mcp_call` | Remote MCP server call |
| `preprocessing` | Input preprocessing |
| `postprocessing` | Output postprocessing |
| `embedding` | Vector embedding generation |
| `speech` | Text-to-speech / speech-to-text |
| `image` | Image generation/editing |
| `video` | Video generation |
| `storage` | Asset storage upload/download |

## Span Hierarchy

- **trace_id** — Groups all spans in one execution
- **span_id** — Unique span identifier
- **parent_span_id** — Links to parent (empty for root)

## Span Attributes

### LLM Span

```json
{
  "model": "gpt-4o",
  "provider": "openai",
  "temperature": 0.7,
  "prompt_tokens": 450,
  "completion_tokens": 120,
  "total_tokens": 570,
  "cost": 0.003
}
```

### Guardrail Span

```json
{
  "scanner_type": "prompt_injection",
  "direction": "input",
  "action": "block",
  "triggered": false
}
```

### Tool Span

```json
{
  "tool_name": "weather_api",
  "tool_type": "api",
  "parameters": { "location": "New York" }
}
```

## Querying Traces

```python
traces = client.traces.list(
    agent_id="agent-id",
    kind="llm",
    status="ok",
    model_name="gpt-4o",
    page=1, limit=50
)

trace = client.traces.get_by_trace_id("trace-id")
```

```typescript
const traces = await client.traces.list({
  agentId: "agent-id", kind: "llm", status: "ok",
});
```

## Status and Level

| Status | Description |
|--------|-------------|
| `ok` | Completed successfully |
| `error` | Encountered an error |

| Level | Description |
|-------|-------------|
| `debug` | Detailed diagnostics |
| `default` | Standard operational info |
| `warning` | Unexpected but non-fatal |
| `error` | Error occurred |

## Cost Tracking

Every LLM span includes `prompt_tokens`, `completion_tokens`, `total_tokens`, `cost`, and `model_name` for granular cost attribution.

## Error Information

Error spans include `error_message`, `error_type`, and `error_stack` for debugging.
