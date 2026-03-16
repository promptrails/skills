# PromptRails Data Sources Guide

## Overview

Data sources allow agents to query external databases and files during execution using versioned, parameterized query templates.

## Supported Databases

| Type | Identifier | Description |
|------|------------|-------------|
| PostgreSQL | `postgresql` | Standard PostgreSQL |
| MySQL | `mysql` | MySQL and MariaDB |
| BigQuery | `bigquery` | Google BigQuery |
| Snowflake | `snowflake` | Snowflake cloud platform |
| Redshift | `redshift` | Amazon Redshift |
| MSSQL | `mssql` | Microsoft SQL Server |
| ClickHouse | `clickhouse` | ClickHouse analytics DB |
| Static File | `static_file` | CSV, JSON, or other files |

## Query Templates

Parameterized SQL with `:param` placeholders:

```sql
SELECT order_id, status, total_amount, created_at
FROM orders
WHERE customer_id = :customer_id
  AND status = :status
ORDER BY created_at DESC
LIMIT :limit
```

### Parameters

```json
[
  {"name": "customer_id", "type": "string", "required": true},
  {"name": "status", "type": "string", "required": false, "default": "active"},
  {"name": "limit", "type": "integer", "required": false, "default": "10"}
]
```

## Creating a Data Source

```python
ds = client.data_sources.create(name="Customer Orders", type="postgresql")

client.data_sources.create_version("ds-id",
    credential_id="pg-cred-id",
    query_template="SELECT * FROM orders WHERE customer_id = :customer_id LIMIT :limit",
    parameters=[
        {"name": "customer_id", "type": "string", "required": True},
        {"name": "limit", "type": "integer", "required": False, "default": "10"}
    ],
    cache_timeout=300,
    message="Initial query"
)
```

## Versioning

Same immutable pattern as agents and prompts:

```python
versions = client.data_sources.list_versions("ds-id")
client.data_sources.promote_version("ds-id", "version-id")
```

## Cache Timeout

| Value | Behavior |
|-------|----------|
| `0` | No caching |
| `300` | Cache for 5 minutes |
| `3600` | Cache for 1 hour (default) |

Cache key = rendered query after parameter substitution.

## Output Format

- `json` (default) — Query results as JSON arrays
- `csv` — Query results as CSV text

## Testing

```python
result = client.data_sources.query("ds-id", parameters={"customer_id": "test", "limit": 5})
print(result.status, result.duration_ms, result.result)
```

## Using in Agents

Data sources are linked via agent version config. During execution:

1. Agent receives input
2. Parameters mapped to query parameters
3. Query executed, results returned
4. Results injected into prompt context for LLM
