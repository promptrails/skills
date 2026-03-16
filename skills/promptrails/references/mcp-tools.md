# PromptRails MCP Tools Guide

## Overview

PromptRails provides first-class Model Context Protocol (MCP) support, enabling agents to invoke external tools, query data sources, call built-in functions, and connect to remote MCP servers.

## Tool Types

### API Tools (`api`)

HTTP requests to external REST APIs:

```python
tool = client.mcp_tools.create(
    name="Weather API",
    type="api",
    config={
        "url": "https://api.weather.com/v1/current",
        "method": "GET",
        "headers": {"X-API-Key": "{{credential}}"},
        "parameters": {
            "location": {"type": "string", "description": "City name", "required": True}
        }
    },
    credential_id="weather-credential-id"
)
```

### Data Source Tools (`datasource`)

Execute queries against connected databases:

```python
tool = client.mcp_tools.create(
    name="Customer Lookup",
    type="datasource",
    config={
        "data_source_id": "your-data-source-id",
        "parameters": {
            "email": {"type": "string", "required": True}
        }
    }
)
```

### Built-in Tools (`builtin`)

Predefined functions running within the platform:

```python
tool = client.mcp_tools.create(
    name="JSON Formatter",
    type="builtin",
    config={"function": "json_format"}
)
```

### Remote MCP Tools (`remote_mcp`)

Connect to external MCP-compatible servers:

```python
tool = client.mcp_tools.create(
    name="GitHub Operations",
    type="remote_mcp",
    config={"server_url": "https://mcp.github.example.com", "transport": "sse"},
    credential_id="github-mcp-credential-id"
)
```

## Tool Schema

Define JSON schemas for tool parameters:

```json
{
  "type": "object",
  "properties": {
    "query": {"type": "string", "description": "Search query"},
    "max_results": {"type": "integer", "default": 10}
  },
  "required": ["query"]
}
```

## Adding Tools to Agents

Link tools via agent version config:

```python
client.agents.create_version("agent-id",
    config={"tools": ["tool-id-1", "tool-id-2"], "temperature": 0.7},
    message="Added tools"
)
```

## Tool Invocation Flow

1. LLM generates a tool call with parameters
2. PromptRails validates parameters against schema
3. Tool is invoked (API call, DB query, MCP request)
4. Results returned to LLM for response generation
5. Entire call recorded as `mcp_call` trace span

## Tool Status

| Status | Description |
|--------|-------------|
| `active` | Available for use |
| `inactive` | Disabled but not removed |
| `error` | Connection error |
| `archived` | Permanently disabled |

## MCP Templates

Pre-configured tool definitions for common integrations (Slack, GitHub, Jira, etc.):

```python
templates = client.mcp_templates.list()
tool = client.mcp_tools.create_from_template(template_id="template-id", name="My Slack")
```
