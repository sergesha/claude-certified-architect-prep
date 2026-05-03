# Claude Code Overview
Source: https://code.claude.com/docs (also docs.anthropic.com/en/docs/claude-code)
Fetched: 2026-03-21

## Key Concepts for Exam

### What is Claude Code?

- An agentic coding tool from Anthropic
- Available in: **terminal, IDE (VS Code / JetBrains), desktop, web, browser**
- Understands entire codebases, executes commands, edits files, searches, manages git workflows
- Uses Claude as the underlying model (Opus, Sonnet, Haiku)

### Core Feature Set

| Feature | Description |
|---------|-------------|
| **CLAUDE.md** | Read at start of every session for coding standards, project context, and conventions |
| **Auto memory** | Automatically remembers build commands, debugging insights, and project patterns |
| **Custom commands** | Repeatable workflows via slash commands (e.g., `/review-pr`, `/deploy-staging`) |
| **Hooks** | Shell commands that run before/after Claude Code actions (pre/post tool execution) |
| **Sub-agents** | Multiple Claude Code agents working simultaneously on different tasks |
| **Agent SDK** | Build custom agentic workflows programmatically |
| **CLI composability** | Pipe, script, and automate -- designed for Unix-style composition |
| **-p flag** | Non-interactive/print mode for CI/CD automation |
| **MCP integration** | Connect to external tools and data sources via Model Context Protocol |

### CLAUDE.md (Memory System)

- Read at the start of every session
- Contains coding standards, project conventions, build commands
- Can exist at multiple levels: `~/.claude/CLAUDE.md` (global), project root, subdirectories
- Auto-memory feature learns from interactions (build commands, debugging insights)

### Custom Commands (Skills)

- Defined as markdown files in `.claude/commands/` directory
- Invoked via slash commands (e.g., `/review-pr`, `/deploy-staging`)
- Can include parameters using `$ARGUMENTS` placeholder
- Project-level commands in `.claude/commands/`
- User-level commands in `~/.claude/commands/`

### Hooks

- Shell commands that execute before or after Claude Code actions
- Configured in settings files
- Can run on events like: PreToolUse, PostToolUse, Notification, Stop
- Use cases: auto-formatting, linting, custom validation, notifications

### Sub-agents

- Claude Code can spawn multiple agents working simultaneously
- Used for parallelizing research, code exploration, and multi-file tasks
- Explore subagent for research tasks; Task subagent for focused work

### Agent SDK

- Programmatic API for building custom agentic workflows
- Enables embedding Claude Code capabilities into custom tools and pipelines
- Supports streaming, tool management, and conversation control

### CLI Composability

- Pipe input/output to/from Claude Code
- Script with shell commands
- `-p` flag for non-interactive mode (print mode) -- essential for CI/CD
- Can read from stdin, write to stdout
- Exit codes for automation

### MCP Integration

- Claude Code can connect to MCP servers for external tools
- Configured via `claude mcp add` command or settings files
- Supports stdio and SSE transport types
- MCP servers provide tools, resources, and prompts to Claude Code

## Configuration Examples

### Installation
```bash
npm install -g @anthropic-ai/claude-code
```
Requires Node.js >= 18.

### Starting Claude Code
```bash
# Interactive mode
claude

# With initial prompt
claude "explain this codebase"

# Non-interactive / print mode (for CI/CD)
claude -p "what does this function do"

# Resume last conversation
claude --resume

# Continue last conversation
claude --continue
```

### Environment Variables
```bash
export ANTHROPIC_API_KEY="sk-ant-..."
export ANTHROPIC_MODEL="claude-sonnet-4-20250514"
export CLAUDE_CODE_USE_BEDROCK=1
export CLAUDE_CODE_USE_VERTEX=1
```

### Settings Files

Global settings at `~/.claude/settings.json`:
```json
{
  "permissions": {
    "allow": ["Bash(git *)", "Read", "Write"],
    "deny": ["Bash(rm -rf *)"]
  },
  "model": "claude-sonnet-4-20250514"
}
```

Project settings at `.claude/settings.json`:
```json
{
  "permissions": {
    "allow": ["Bash(npm test)", "Bash(npm run lint)"]
  }
}
```

### CLI Reference

| Command | Description |
|---------|-------------|
| `claude` | Start interactive REPL |
| `claude "prompt"` | Start with initial prompt |
| `claude -p "prompt"` | Print mode (non-interactive, for CI/CD) |
| `claude --resume` | Resume last conversation |
| `claude --continue` | Continue last conversation |
| `claude config` | Manage configuration |
| `claude mcp` | Manage MCP servers |

### Key Docs Pages to Reference

- `/en/memory` -- CLAUDE.md and auto-memory system
- `/en/skills` -- Custom commands and slash commands
- `/en/hooks` -- Pre/post action shell hooks
- `/en/sub-agents` -- Multi-agent parallel execution
- `/en/cli-reference` -- Full CLI flags and options
- `/en/best-practices` -- Recommended patterns for effective use
- `/en/github-actions` -- CI/CD integration with GitHub Actions
- `/en/settings` -- Configuration files and options

## Important for Exam (Mapping to Task Statements)

### Domain 3: Claude Code Configuration

**Task 3.1 -- CLAUDE.md Configuration**
- CLAUDE.md is read at the start of every session
- Contains coding standards, conventions, build/test commands
- Hierarchical: global (~/.claude/CLAUDE.md) -> project root -> subdirectories
- Auto-memory learns and stores insights automatically

**Task 3.2 -- Custom Commands (Skills)**
- Markdown files in `.claude/commands/` (project) or `~/.claude/commands/` (user)
- Invoked as slash commands
- `$ARGUMENTS` placeholder for parameterized commands

**Task 3.3 -- Hooks**
- Shell commands on events: PreToolUse, PostToolUse, Notification, Stop
- Configured in settings files
- Auto-formatting, linting, validation, notifications

**Task 3.4 -- Permissions and Settings**
- Three-tier: allow, deny, ask
- Wildcard patterns supported (e.g., `Bash(git *)`)
- Global settings: `~/.claude/settings.json`
- Project settings: `.claude/settings.json`
- Tool approval system: auto-approve, ask each time, deny

**Task 3.5 -- CI/CD Integration**
- `-p` flag for non-interactive/print mode
- Pipe input/output for scripting
- Exit codes for automation
- GitHub Actions integration
- Agent SDK for custom workflows

**Task 3.6 -- MCP Server Configuration**
- `claude mcp add` command
- Settings-based configuration
- stdio and SSE transport types
- Provides tools, resources, and prompts to Claude Code

### Architecture Notes

- Claude Code uses tools internally: Read, Edit, Write, Bash, Grep, Glob, etc.
- Permission system is critical -- controls what Claude Code can do
- Sessions stored locally, can be resumed with --resume / --continue
- Works with any programming language
- Git-aware: understands repository structure, branches, history
- Supports multiple AI providers: Anthropic API, Amazon Bedrock, Google Vertex AI
