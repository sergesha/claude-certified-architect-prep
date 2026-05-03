# Claude Code CLI Reference
Source: https://code.claude.com/docs/en/cli-reference
Fetched: 2026-03-21

## Key Concepts for Exam

### Non-Interactive / SDK Mode (CRITICAL for CI/CD)
- **`-p` / `--print`** — runs in non-interactive (SDK) mode
- Essential for CI/CD pipelines, scripting, automation
- No interactive prompts — runs and exits with output
- Combine with output format flags for structured results

### Output Formats
- **`--output-format text`** — plain text output (default for --print)
- **`--output-format json`** — structured JSON output
- **`--output-format stream-json`** — streaming JSON (line-delimited)
- **`--json-schema`** — validated JSON output against a provided schema

### Session Management
- **`--resume` / `-r`** — resume a previous session by ID
- **`--fork-session`** — create a new session branching from an existing one
- **`--name` / `-n`** — name a session for easy reference
- **`--continue` / `-c`** — continue the **most recent** conversation

### Model & Agent Selection
- **`--model`** — select which model to use
- **`--agent`** — run as a specific sub-agent (from .claude/agents/)
- **`--agents`** — inline sub-agent definitions via JSON (no file needed)
- **`--effort`** — set effort level (low, medium, high)

### Tool Control
- **`--tools`** — restrict which tools are available
- **`--allowedTools`** — whitelist specific tools
- **`--disallowedTools`** — blacklist specific tools

### Prompt Customization
- **`--system-prompt`** — replace the system prompt entirely
- **`--append-system-prompt`** — add to the existing system prompt

### Permission Modes
- **`--permission-mode`** — set permission handling:
  - `default` — normal interactive permission prompts
  - `acceptEdits` — auto-accept file edits
  - `dontAsk` — skip permission prompts (skip disallowed)
  - `bypassPermissions` — no permission checks
  - `plan` — read-only planning mode

### Additional Configuration
- **`--mcp-config`** — load MCP server configuration file
- **`--add-dir`** — add additional directories to the session context
- **`--max-turns`** — limit the number of conversation turns
- **`--max-budget-usd`** — set a spending limit in USD

## Configuration Details

### CI/CD Pipeline Usage Pattern
```bash
# Basic non-interactive usage
claude -p "Explain this error" < error.log

# JSON output for parsing
claude -p --output-format json "List all TODO comments"

# With schema validation
claude -p --json-schema '{"type":"object","properties":{"todos":{"type":"array"}}}' "Find TODOs"

# With budget limit
claude -p --max-turns 5 --max-budget-usd 0.50 "Fix the lint errors"
```

### Session Management Pattern
```bash
# Start a named session
claude -n "feature-work"

# Continue most recent conversation
claude -c

# Resume a specific session
claude -r session-id-here

# Fork from existing session
claude --fork-session session-id-here
```

### Tool Restriction Pattern
```bash
# Only allow read operations
claude --tools Read,Grep,Glob -p "Analyze the codebase"

# Disallow destructive operations
claude --disallowedTools Bash,Write -p "Review the code"
```

## Examples

### Running as a Sub-Agent
```bash
# Use a defined agent
claude --agent reviewer -p "Review the latest changes"

# Inline agent definition
claude --agents '{"name":"linter","tools":["Bash"],"maxTurns":3}' -p "Run linting"
```

### Custom System Prompt
```bash
# Replace system prompt
claude --system-prompt "You are a security auditor" -p "Check for vulnerabilities"

# Append to system prompt
claude --append-system-prompt "Focus on performance" -p "Review this code"
```

### MCP Server Configuration
```bash
# Load MCP config
claude --mcp-config ./mcp-servers.json -p "Query the database"
```

### Multi-Directory Context
```bash
# Add additional directories
claude --add-dir ../shared-lib --add-dir ../common -p "Find shared utilities"
```

## Important for Exam (Mapping to Task Statements)

| Exam Topic | Key Facts |
|---|---|
| CI/CD mode | -p / --print (non-interactive, CRITICAL) |
| Output formats | text, json, stream-json |
| Schema validation | --json-schema for validated JSON output |
| Session resume | --resume / -r (by ID), --continue / -c (most recent) |
| Session branching | --fork-session |
| Session naming | --name / -n |
| Model override | --model flag |
| Agent selection | --agent (file-based), --agents (inline JSON) |
| Tool whitelisting | --allowedTools |
| Tool blacklisting | --disallowedTools |
| Tool restriction | --tools |
| System prompt replace | --system-prompt |
| System prompt append | --append-system-prompt |
| Permission modes | default, acceptEdits, dontAsk, bypassPermissions, plan |
| MCP config loading | --mcp-config |
| Extra directories | --add-dir |
| Turn limit | --max-turns |
| Budget limit | --max-budget-usd |
| Effort level | --effort |
| Continue conversation | --continue / -c (most recent) |
