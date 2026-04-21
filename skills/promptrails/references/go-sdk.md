# PromptRails Go SDK Reference

Current release: **v0.3.1** — streaming chat & executions via
`*ChatStream`, typed `AgentConfig` interface, `promptrails.Version`
constant. Requires Go 1.21+.

## Installation

```bash
go get github.com/promptrails/go-sdk@v0.3.1
```

## Client Initialization

```go
import promptrails "github.com/promptrails/go-sdk"

client := promptrails.NewClient("pr_key_...")

// With options
client := promptrails.NewClient("pr_key_...",
    promptrails.WithBaseURL("http://localhost:8082"),
    promptrails.WithTimeout(10 * time.Second),
    promptrails.WithMaxRetries(5),
)
```

### Configuration

| Option | Default | Description |
|--------|---------|-------------|
| `WithBaseURL` | `https://api.promptrails.ai` | API base URL |
| `WithTimeout` | `30s` | Request timeout |
| `WithMaxRetries` | `3` | Max retries on network/5xx errors |

## Resources

### Agents

```go
ctx := context.Background()

// List
agents, err := client.Agents.List(ctx, &promptrails.ListAgentsParams{Page: 1, Limit: 20})

// Get
agent, err := client.Agents.Get(ctx, "agent-id")

// Create
agent, err := client.Agents.Create(ctx, &promptrails.CreateAgentParams{
    Name: "My Agent", Type: "simple", Description: "...",
})

// Update
err := client.Agents.Update(ctx, "agent-id", &promptrails.UpdateAgentParams{Name: "New Name"})

// Delete
err := client.Agents.Delete(ctx, "agent-id")

// Execute
result, err := client.Agents.Execute(ctx, "agent-id", &promptrails.ExecuteAgentParams{
    Input: map[string]any{"query": "hello"},
})

// Versions
versions, err := client.Agents.ListVersions(ctx, "agent-id")
err := client.Agents.CreateVersion(ctx, "agent-id", &promptrails.CreateVersionParams{...})
err := client.Agents.PromoteVersion(ctx, "agent-id", "version-id")

// Guardrails
guardrails, err := client.Agents.ListGuardrails(ctx, "agent-id")
err := client.Agents.CreateGuardrail(ctx, "agent-id", &promptrails.CreateGuardrailParams{...})

// Memory
memories, err := client.Agents.ListMemories(ctx, "agent-id")
err := client.Agents.CreateMemory(ctx, "agent-id", &promptrails.CreateMemoryParams{
    Type: "fact", Content: "User prefers dark mode",
})
results, err := client.Agents.SearchMemories(ctx, "agent-id", "preferences")
err := client.Agents.DeleteAllMemories(ctx, "agent-id")
```

### Prompts

```go
prompts, err := client.Prompts.List(ctx, &promptrails.ListPromptsParams{})
prompt, err := client.Prompts.Get(ctx, "prompt-id")
prompt, err := client.Prompts.Create(ctx, &promptrails.CreatePromptParams{Name: "My Prompt"})
err := client.Prompts.Update(ctx, "prompt-id", &promptrails.UpdatePromptParams{Name: "Updated"})
err := client.Prompts.Delete(ctx, "prompt-id")

// Versions
versions, err := client.Prompts.ListVersions(ctx, "prompt-id")
err := client.Prompts.CreateVersion(ctx, "prompt-id", &promptrails.CreatePromptVersionParams{
    SystemPrompt: "You are helpful.",
    UserPrompt:   "Answer: {{ question }}",
    Temperature:  0.7,
    Message:      "Initial version",
})
err := client.Prompts.PromoteVersion(ctx, "prompt-id", "version-id")

// Run prompt directly
result, err := client.Prompts.Run(ctx, "prompt-id", &promptrails.RunPromptParams{
    Input: map[string]any{"question": "What is Go?"},
})
```

### Executions

```go
executions, err := client.Executions.List(ctx, &promptrails.ListExecutionsParams{AgentID: "agent-id"})
execution, err := client.Executions.Get(ctx, "execution-id")
```

### Credentials

```go
creds, err := client.Credentials.List(ctx)
cred, err := client.Credentials.Create(ctx, &promptrails.CreateCredentialParams{
    Name: "OpenAI", Category: "llm", Type: "openai", Value: "sk-...",
})
err := client.Credentials.SetDefault(ctx, "cred-id")
err := client.Credentials.CheckConnection(ctx, "cred-id")
err := client.Credentials.Delete(ctx, "cred-id")
```

### Data Sources

```go
sources, err := client.DataSources.List(ctx, &promptrails.ListDataSourcesParams{})
ds, err := client.DataSources.Create(ctx, &promptrails.CreateDataSourceParams{
    Name: "Orders DB", Type: "postgresql",
})
err := client.DataSources.CreateVersion(ctx, "ds-id", &promptrails.CreateDataSourceVersionParams{...})
result, err := client.DataSources.Query(ctx, "ds-id", map[string]any{"id": "123"})
err := client.DataSources.TestConnection(ctx, "ds-id")
```

### Chat

```go
sessions, err := client.Chat.ListSessions(ctx, &promptrails.ListSessionsParams{AgentID: "agent-id"})
session, err := client.Chat.CreateSession(ctx, &promptrails.CreateSessionParams{AgentID: "agent-id"})
messages, err := client.Chat.ListMessages(ctx, "session-id")
reply, err := client.Chat.SendMessage(ctx, "session-id", &promptrails.SendMessageParams{Content: "Hello"})
err := client.Chat.DeleteSession(ctx, "session-id")
```

### Streaming

`Chat.SendMessageStream` and `Executions.Stream` return a `*ChatStream`
that iterates typed events on one HTTP connection. Always `defer
stream.Close()`, and cancel mid-stream by cancelling `ctx`.

```go
session, err := client.Chat.CreateSession(ctx, &promptrails.CreateSessionParams{
    AgentID: "agent-id",
})
if err != nil { log.Fatal(err) }

stream, err := client.Chat.SendMessageStream(ctx, session.ID, &promptrails.SendMessageParams{
    Content: "Hello",
})
if err != nil { log.Fatal(err) }
defer stream.Close()

for stream.Next() {
    switch e := stream.Event().(type) {
    case *promptrails.ExecutionEvent:
        log.Printf("execution_id: %s", e.ExecutionID)
    case *promptrails.ThinkingEvent:
        log.Printf("[thinking] %s", e.Content)
    case *promptrails.ToolStartEvent:
        log.Printf("[tool_start] %s", e.Name)
    case *promptrails.ToolEndEvent:
        log.Printf("[tool_end] %s — %s", e.Name, e.Summary)
    case *promptrails.ContentEvent:
        fmt.Print(e.Content)
    case *promptrails.DoneEvent:
        fmt.Printf("\n[done] %d tokens\n", e.TokenUsage.TotalTokens)
    case *promptrails.ErrorEvent:
        log.Fatalf("[error] %s", e.Message)
    }
}
if err := stream.Err(); err != nil {
    log.Fatal(err)
}
```

For an execution started outside chat (e.g. `Agents.Execute`), subscribe
to its live SSE stream with `client.Executions.Stream(ctx, executionID)`.

### Traces

```go
traces, err := client.Traces.List(ctx, &promptrails.ListTracesParams{AgentID: "agent-id", Kind: "llm"})
trace, err := client.Traces.GetByTraceID(ctx, "trace-id")
```

### Costs

```go
summary, err := client.Costs.GetSummary(ctx)
agentCosts, err := client.Costs.GetAgentSummary(ctx, "agent-id")
```

### Scores

```go
scores, err := client.Scores.List(ctx, &promptrails.ListScoresParams{ExecutionID: "exec-id"})
score, err := client.Scores.Create(ctx, &promptrails.CreateScoreParams{...})
configs, err := client.Scores.ListConfigs(ctx)
aggregates, err := client.Scores.Aggregates(ctx, "config-id")
```

### MCP Tools

```go
tools, err := client.MCPTools.List(ctx)
tool, err := client.MCPTools.Create(ctx, &promptrails.CreateMCPToolParams{Name: "Weather", Type: "api"})
err := client.MCPTools.Update(ctx, "tool-id", &promptrails.UpdateMCPToolParams{})
err := client.MCPTools.Delete(ctx, "tool-id")
```

### Approvals

```go
approvals, err := client.Approvals.List(ctx)
approval, err := client.Approvals.Get(ctx, "approval-id")
err := client.Approvals.Decide(ctx, "approval-id", "approved")
```

### Webhook Triggers

```go
triggers, err := client.WebhookTriggers.List(ctx)
trigger, err := client.WebhookTriggers.Create(ctx, &promptrails.CreateWebhookTriggerParams{...})
err := client.WebhookTriggers.Delete(ctx, "trigger-id")
```

### A2A Protocol

```go
card, err := client.A2A.GetAgentCard(ctx, "agent-id")
resp, err := client.A2A.SendMessage(ctx, "agent-id", &promptrails.A2AMessageParams{...})
task, err := client.A2A.GetTask(ctx, "task-id")
tasks, err := client.A2A.ListTasks(ctx, &promptrails.ListTasksParams{AgentID: "agent-id"})
err := client.A2A.CancelTask(ctx, "task-id")
```

### Media

```go
models, err := client.MediaModels.List(ctx, &promptrails.ListMediaModelsParams{Provider: "fal"})

result, err := client.Media.Generate(ctx, &promptrails.GenerateMediaParams{
    Provider: "fal", MediaType: "image", Model: "fal-ai/flux/schnell",
    Prompt: "A sunset", Config: map[string]any{"width": 1024},
})

assets, err := client.Assets.List(ctx, &promptrails.ListAssetsParams{MediaType: "image"})
signed, err := client.Assets.GetSignedURL(ctx, "asset-id")
err := client.Assets.Delete(ctx, "asset-id")
```

## Error Handling

```go
import "errors"

result, err := client.Agents.Get(ctx, "missing-id")
if err != nil {
    var notFound *promptrails.NotFoundError
    var rateLimit *promptrails.RateLimitError
    var quota *promptrails.QuotaExceededError

    switch {
    case errors.As(err, &notFound):
        fmt.Println("Not found")
    case errors.As(err, &rateLimit):
        fmt.Println("Rate limited")
    case errors.As(err, &quota):
        fmt.Println("Quota exceeded")
    default:
        fmt.Printf("Error: %v\n", err)
    }
}
```
