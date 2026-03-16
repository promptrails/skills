# PromptRails Python SDK Reference

## Installation

```bash
pip install promptrails
```

## Client Initialization

```python
from promptrails import PromptRails

# Sync client
client = PromptRails(api_key="pr_key_...")

# Async client
from promptrails import AsyncPromptRails
async_client = AsyncPromptRails(api_key="pr_key_...")

# Context manager
with PromptRails(api_key="pr_key_...") as client:
    agents = client.agents.list()

# Async context manager
async with AsyncPromptRails(api_key="pr_key_...") as client:
    result = await client.agents.execute("agent-id", input={"query": "hello"})
```

### Configuration

| Option | Default | Description |
|--------|---------|-------------|
| `api_key` | required | API key (`pr_key_...`) |
| `base_url` | `https://api.promptrails.ai` | API base URL |
| `timeout` | `30.0` | Request timeout (seconds) |
| `max_retries` | `3` | Max retries on network/5xx errors |

## Resources

### Agents

```python
# List agents
agents = client.agents.list(page=1, limit=20)

# Get agent
agent = client.agents.get("agent-id")

# Create agent
agent = client.agents.create(name="My Agent", type="simple", description="...", labels=["prod"])

# Update agent
client.agents.update("agent-id", name="New Name")

# Delete agent
client.agents.delete("agent-id")

# Execute agent
result = client.agents.execute("agent-id", input={"query": "hello"}, metadata={"user_id": "123"})

# Versions
versions = client.agents.list_versions("agent-id")
client.agents.create_version("agent-id", config={...}, message="v2")

# Guardrails
guardrails = client.agents.list_guardrails("agent-id")
client.agents.create_guardrail("agent-id", scanner_type="prompt_injection", direction="input", action="block")

# Memory
memories = client.agents.list_memories("agent-id")
client.agents.create_memory("agent-id", type="fact", content="User prefers dark mode")
results = client.agents.search_memories("agent-id", query="preferences")
client.agents.delete_all_memories("agent-id")
```

### Prompts

```python
prompts = client.prompts.list()
prompt = client.prompts.get("prompt-id")
prompt = client.prompts.create(name="My Prompt", description="...")
client.prompts.update("prompt-id", name="Updated")
client.prompts.delete("prompt-id")

# Versions
versions = client.prompts.list_versions("prompt-id")
client.prompts.create_version("prompt-id",
    system_prompt="You are helpful.",
    user_prompt="Answer: {{ question }}",
    temperature=0.7,
    max_tokens=1024,
    cache_timeout=3600,
    message="Initial version"
)
```

### Executions

```python
executions = client.executions.list(agent_id="agent-id", status="completed")
execution = client.executions.get("execution-id")
```

### Credentials

```python
credentials = client.credentials.list()
cred = client.credentials.get("cred-id")
cred = client.credentials.create(name="OpenAI", category="llm", type="openai", value="sk-...")
client.credentials.set_default("cred-id")
client.credentials.check_connection("cred-id")
client.credentials.delete("cred-id")
```

### Data Sources

```python
sources = client.data_sources.list()
ds = client.data_sources.create(name="Orders DB", type="postgresql", description="...")
client.data_sources.create_version("ds-id",
    credential_id="cred-id",
    query_template="SELECT * FROM orders WHERE id = :id",
    parameters=[{"name": "id", "type": "string", "required": True}],
    cache_timeout=300,
    message="Initial query"
)
result = client.data_sources.query("ds-id", parameters={"id": "123"})
client.data_sources.test_connection("ds-id")
```

### Chat Sessions

```python
sessions = client.chat.list_sessions(agent_id="agent-id")
session = client.chat.create_session(agent_id="agent-id", title="Support Chat")
messages = client.chat.list_messages(session_id="session-id")
reply = client.chat.send_message(session_id="session-id", content="Hello")
client.chat.delete_session("session-id")
```

### Traces

```python
traces = client.traces.list(agent_id="agent-id", kind="llm", status="ok")
trace = client.traces.get_by_trace_id("trace-id")
```

### Costs

```python
summary = client.costs.get_summary()
agent_costs = client.costs.get_agent_summary(agent_id="agent-id")
```

### Scores

```python
scores = client.scores.list(execution_id="exec-id")
score = client.scores.create(execution_id="exec-id", config_id="config-id", value=0.95)

# Score configs
configs = client.scores.list_configs()
config = client.scores.create_config(name="Accuracy", type="numeric", min_value=0, max_value=1)
aggregates = client.scores.aggregates(config_id="config-id")
```

### MCP Tools

```python
tools = client.mcp_tools.list()
tool = client.mcp_tools.create(name="Weather", type="api", config={...})
client.mcp_tools.update("tool-id", name="Updated")
client.mcp_tools.delete("tool-id")
```

### Approvals

```python
approvals = client.approvals.list()
approval = client.approvals.get("approval-id")
client.approvals.decide("approval-id", decision="approved")  # or "rejected"
```

### Webhook Triggers

```python
triggers = client.webhook_triggers.list()
trigger = client.webhook_triggers.create(name="Deploy", agent_id="agent-id")
client.webhook_triggers.update("trigger-id", active=True)
client.webhook_triggers.delete("trigger-id")
```

### A2A Protocol

```python
card = client.a2a.get_agent_card("agent-id")
response = client.a2a.send_message("agent-id", message={...})
task = client.a2a.get_task("task-id")
tasks = client.a2a.list_tasks(agent_id="agent-id")
client.a2a.cancel_task("task-id")
```

### Media

```python
# List models
models = client.media_models.list(media_type="image")

# Generate
result = client.media.generate(
    provider="fal", media_type="image", model="fal-ai/flux/schnell",
    prompt="A sunset", config={"width": 1024, "height": 768}
)

# Assets
assets = client.assets.list(type="image")
signed = client.assets.get_signed_url("asset-id")
client.assets.delete("asset-id")
```

## Error Handling

```python
from promptrails import NotFoundError, ValidationError, RateLimitError, QuotaExceededError

try:
    result = client.agents.execute("agent-id", input={})
except QuotaExceededError:
    print("Execution limit reached")
except RateLimitError:
    print("Too many requests")
except NotFoundError as e:
    print(f"Not found: {e.message}")
except ValidationError as e:
    print(f"Validation error: {e.message}")
```
