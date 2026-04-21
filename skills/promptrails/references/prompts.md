# PromptRails Prompts Guide

## Overview

Prompts are versioned Jinja2 templates with model assignment, caching, and structured schemas.

A prompt consists of:
- **System prompt** — LLM role and behavior instructions
- **User prompt** — Jinja2 template that renders user input
- **Model assignment** — Primary model + optional fallback
- **Parameters** — Temperature, max tokens, top_p
- **Input/output schemas** — JSON schemas for validation
- **Cache timeout** — Response caching duration

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

## Model Assignment

Each version specifies:
- **Primary model** — Default for execution
- **Fallback model** — Used if primary fails

Supported providers: OpenAI, Anthropic, Google Gemini, DeepSeek, Fireworks, xAI, OpenRouter.

## Caching

Set `cache_timeout` (seconds) on a version to cache identical inputs:

```python
client.prompts.create_version("prompt-id",
    system_prompt="...",
    user_prompt="Translate '{{ text }}' to {{ target_language }}.",
    cache_timeout=3600,  # 1 hour
    message="Added caching"
)
```

Cache key = rendered prompt content (after template substitution).

## Versioning

Immutable versions with promotion:

```python
# Create version
client.prompts.create_version("prompt-id",
    system_prompt="You are helpful.",
    user_prompt="Answer: {{ question }}",
    temperature=0.7,
    message="v1"
)

# List versions
versions = client.prompts.list_versions("prompt-id")

# Promote a version
client.prompts.promote_version("prompt-id", "version-id")
```

## Testing

Run a prompt directly without an agent. The request body carries the
rendered prompt body and the target model; the `input` map supplies
variables for Jinja templating.

```python
response = client.prompts.run_prompt(
    "prompt-id",
    data={
        "system_prompt": "You are a support ticket classifier.",
        "user_prompt": "Classify: {{ message }}",
        "llm_model_id": "gpt-4o",
        "temperature": 0.3,
        "input": {"message": "I want a refund"},
    },
)
print(response.content)
print(response.token_usage, response.cost)
```

Other SDKs expose the same operation:

```typescript
const response = await client.prompts.runPrompt("prompt-id", {
  system_prompt: "You are a support ticket classifier.",
  user_prompt: "Classify: {{ message }}",
  llm_model_id: "gpt-4o",
  input: { message: "I want a refund" },
});
```

```go
response, err := client.Prompts.Run(ctx, "prompt-id", &promptrails.RunPromptParams{
    UserPrompt: "Classify: {{ message }}",
    LLMModelID: "gpt-4o",
    Input:      map[string]any{"message": "I want a refund"},
})
```

## Status

| Status | Description |
|--------|-------------|
| `active` | Available for use |
| `archived` | Hidden, cannot be executed |
