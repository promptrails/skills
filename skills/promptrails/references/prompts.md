# PromptRails Prompts Guide

## Overview

Prompts are versioned, **content-only** Jinja2 templates.

A prompt version consists of:
- **System prompt** — LLM role and behavior instructions
- **User prompt** — Jinja2 template that renders user input
- **Input schema** — JSON schema for validating template variables

Model + sampling (`model_config`), fallback, response caching
(`cache_timeout`), output schema, and tool/sub-agent attachments are **not**
part of the prompt — they live on the **agent version** that references the
prompt (see the [Agents Guide](agents.md)).

## Jinja2 Templating

### Variables

```jinja2
You are a {{ role }} assistant for {{ company_name }}.
Help the customer: {{ message }}
```

### Conditionals

```jinja2
{% if language == "spanish" %}
Respond in Spanish.
{% else %}
Respond in English.
{% endif %}
```

### Loops

```jinja2
{% for doc in documents %}
Document {{ loop.index }}: {{ doc.title }}
{{ doc.content }}
{% endfor %}
```

### Filters

```jinja2
{{ name | upper }}
{{ date | default("Unknown") }}
{{ text | truncate(200) }}
```

## Model, Sampling, and Caching

These are **not** prompt fields. The agent version that references the
prompt owns the model + fallback (`model_config`), sampling, and response
caching (`cache_timeout`). Supported providers: OpenAI, Anthropic, Google
Gemini, DeepSeek, Fireworks, xAI, OpenRouter. See the
[Agents Guide](agents.md) for `create_version` with `model_config` and
`cache_timeout`.

## Versioning

Immutable, content-only versions with promotion:

```python
# Create version
client.prompts.create_version("prompt-id",
    version="1",
    system_prompt="You are helpful.",
    user_prompt="Answer: {{ question }}",
    input_schema={"type": "object", "properties": {"question": {"type": "string"}}},
    message="v1"
)

# List versions
versions = client.prompts.list_versions("prompt-id")

# Promote a version
client.prompts.promote_version("prompt-id", "version-id")
```

## Testing

There is no standalone "run prompt" endpoint. To try prompt content
without saving a version, use the agent **playground** with an ad-hoc
`prompt_override` — the agent version supplies the runtime (model, tools),
and `prompt_override` carries `system_prompt` / `user_prompt` /
`input_schema`:

```python
result = client.agents.playground(
    "agent-id",
    input={"message": "I want a refund"},
    prompt_override={
        "system_prompt": "You are a support ticket classifier.",
        "user_prompt": "Classify: {{ message }}",
    },
)
```

```typescript
const result = await client.agents.playground("agent-id", {
  input: { message: "I want a refund" },
  prompt_override: {
    system_prompt: "You are a support ticket classifier.",
    user_prompt: "Classify: {{ message }}",
  },
});
```

```go
result, err := client.Agents.Playground(ctx, "agent-id", &promptrails.PlaygroundParams{
    Input:          map[string]any{"message": "I want a refund"},
    PromptOverride: map[string]any{"user_prompt": "Classify: {{ message }}"},
})
```

## Status

| Status | Description |
|--------|-------------|
| `active` | Available for use |
| `archived` | Hidden, cannot be executed |
