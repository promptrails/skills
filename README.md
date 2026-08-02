# PromptRails Skills

AI coding assistant skills for [PromptRails](https://promptrails.ai) — the AI agent orchestration platform.

These skills teach AI coding assistants (Claude Code, Cursor, Windsurf, etc.) how to work with PromptRails SDKs, CLI, agents, prompts, tracing, and more.

## Installation

### Cursor

```
/add-plugin promptrails/skills
```

### Skills CLI

```bash
npx skills add promptrails/skills --skill "promptrails"
```

### Claude Code

Add to your project's `.claude/settings.json`:

```json
{
  "skills": ["path/to/skills/skills/promptrails/SKILL.md"]
}
```

Or symlink into your Claude Code skills directory.

### Manual

Clone this repository and point your AI assistant's configuration to the `skills/promptrails/SKILL.md` file.

## What's Included

The PromptRails skill teaches your AI assistant:

- **SDK Usage** — Python (`promptrails`), JavaScript (`@promptrails/sdk`), and Go (`github.com/promptrails/go-sdk`) SDK patterns
- **CLI Commands** — All `promptrails` CLI commands for agents, prompts, executions, credentials, and more
- **Agent Management** — Creating, configuring, versioning, and executing agents (two types: `agent`, `workflow`)
- **Prompt Engineering** — Content-only Jinja2 templating and versioning (model/sampling live on the agent version)
- **Tracing & Observability** — 18 span kinds, cost tracking, error debugging
- **MCP Tools** — External tool integration via Model Context Protocol
- **Data Sources** — Database connections with parameterized queries

## Structure

```
skills/
└── promptrails/
    ├── SKILL.md              # Main skill definition
    └── references/
        ├── cli.md            # CLI reference
        ├── python-sdk.md     # Python SDK reference
        ├── javascript-sdk.md # JavaScript/TypeScript SDK reference
        ├── go-sdk.md         # Go SDK reference
        ├── agents.md         # Agents guide
        ├── prompts.md        # Prompts guide
        ├── tracing.md        # Tracing guide
        ├── mcp-tools.md      # MCP tools guide
        └── data-sources.md   # Data sources guide
.cursor-plugin/
└── plugin.json               # Cursor IDE plugin config
```

## Documentation

Full PromptRails documentation is available at [promptrails.ai/docs](https://promptrails.ai/docs).

## License

MIT
