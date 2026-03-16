# PromptRails CLI Reference

## Installation

```bash
# macOS / Linux (Homebrew)
brew tap promptrails/tap
brew install promptrails

# Go Install
go install github.com/promptrails/cli/cmd/promptrails@latest

# Binary download
curl -sL https://github.com/promptrails/cli/releases/latest/download/promptrails-OS-ARCH.tar.gz | tar xz
sudo mv promptrails /usr/local/bin/
```

## Authentication

```bash
# Interactive setup
promptrails init

# Non-interactive (CI/CD)
promptrails init --api-key pr_key_...

# Check current context
promptrails status
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `PROMPTRAILS_API_KEY` | API key (overrides stored credentials) |
| `PROMPTRAILS_WORKSPACE_ID` | Workspace ID override |
| `PROMPTRAILS_API_URL` | API base URL override (default: `https://api.promptrails.ai`) |

## Commands

### Agents

```bash
promptrails agent list                           # List agents
promptrails agent list --type simple             # Filter by type
promptrails agent get <id>                       # Get details
promptrails agent create --name "My Agent"       # Create
promptrails agent update <id> --name "New Name"  # Update
promptrails agent delete <id>                    # Delete
promptrails agent execute <id> --input '{"q": "hello"}'  # Execute
promptrails agent versions <id>                  # List versions
promptrails agent promote <id> <version-id>      # Set current version
```

### Prompts

```bash
promptrails prompt list
promptrails prompt get <id>
promptrails prompt create --name "My Prompt"
promptrails prompt run <id> --user-prompt "Hello"
promptrails prompt versions <id>
promptrails prompt promote <id> <version-id>
```

### Executions

```bash
promptrails execution list
promptrails execution list --agent <id>
promptrails execution list --status completed
promptrails execution get <id>
```

### Credentials

```bash
promptrails credential list
promptrails credential create --provider openai --name "Production"
promptrails credential delete <id>
promptrails credential check <id>
```

### API Keys

```bash
promptrails apikey list
promptrails apikey create --name "CI"
promptrails apikey delete <id>
```

### Webhook Triggers

```bash
promptrails webhook-trigger list
promptrails wt get <trigger-id>
promptrails wt create --name "GitHub" --agent-id <id>
promptrails wt update <trigger-id> --active
promptrails wt delete <trigger-id>
```

### Media Studio

```bash
promptrails media generate --provider stability --media-type image_gen --model sd3.5-large --prompt "A sunset"
promptrails media generate --provider elevenlabs --media-type tts --model eleven_multilingual_v2 --prompt "Hello"
```

### Assets

```bash
promptrails assets list
promptrails assets list --type image
promptrails assets get <id>
promptrails assets signed-url <id>
promptrails assets delete <id>
```

### Media Models

```bash
promptrails media-models list
promptrails media-models list --provider fal
promptrails media-models list --media-type tts
```

## Global Flags

| Flag | Description |
|------|-------------|
| `-o, --output` | Output format: `table` (default) or `json` |
| `--workspace` | Override workspace ID |
| `--api-url` | Override API base URL |
| `--no-color` | Disable color output |

## Shell Completions

```bash
promptrails completion bash > /etc/bash_completion.d/promptrails
promptrails completion zsh  > "${fpath[1]}/_promptrails"
promptrails completion fish > ~/.config/fish/completions/promptrails.fish
```

## CI/CD Usage

### GitHub Actions

```yaml
- name: Install CLI
  run: |
    curl -sL https://github.com/promptrails/cli/releases/latest/download/promptrails-linux-amd64.tar.gz | tar xz
    sudo mv promptrails /usr/local/bin/

- name: Execute agent
  env:
    PROMPTRAILS_API_KEY: ${{ secrets.PROMPTRAILS_API_KEY }}
  run: |
    promptrails agent execute ${{ vars.AGENT_ID }} \
      --input '{"branch": "${{ github.ref_name }}"}' \
      --output json
```

## Config Files

Stored in `~/.promptrails/`:

| File | Permissions | Contents |
|------|-------------|----------|
| `config.json` | `0644` | API URL, active workspace, output format |
| `credentials.json` | `0600` | API key |
