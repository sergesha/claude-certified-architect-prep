# Claude Code Sub-Agents
Source: https://code.claude.com/docs/en/sub-agents
Fetched: 2026-03-21

## Key Concepts for Exam

### Built-in Sub-Agents
1. **Explore** — Uses Haiku model, read-only, three thoroughness levels:
   - quick
   - medium
   - thorough
2. **Plan** — Read-only, used in plan mode for reasoning about approach
3. **General-purpose** — Has access to all tools, full capability

### Custom Sub-Agents
- **Project-level:** `.claude/agents/<name>.md`
- **User-level:** `~/.claude/agents/<name>.md`

### Agent Tool (formerly Task Tool)
- The **Agent tool** (renamed from Task tool) spawns sub-agents
- Use `Agent(agent_type)` syntax to restrict which sub-agents can be spawned
- **Sub-agents cannot spawn other sub-agents** (no recursive nesting)

### Execution Modes
- **Foreground** — blocks until complete, results returned inline
- **Background** — runs asynchronously, can be resumed later
- Resume background sub-agents with **SendMessage**

### Auto-Compaction
- Sub-agents auto-compact at **95% capacity** to manage context window

## Configuration Details

### Agent Frontmatter Fields
```yaml
---
name: my-agent
description: What this agent does
tools: [Read, Edit, Bash, Grep, Glob]
disallowedTools: [Write]
model: claude-sonnet-4-20250514
permissionMode: acceptEdits
maxTurns: 10
skills: [my-skill, another-skill]
mcpServers:
  server-name:
    command: npx
    args: ["-y", "some-mcp-server"]
hooks:
  pre-run: ...
  post-run: ...
memory: project
background: true
effort: medium
isolation: true
---
```

### Key Frontmatter Fields Explained
- **`tools`** — whitelist of tools the agent can use
- **`disallowedTools`** — blacklist of tools the agent cannot use
- **`permissionMode`** — how permissions are handled:
  - `default` — normal permission prompts
  - `acceptEdits` — auto-accept file edits
  - `dontAsk` — skip permission prompts (skip disallowed)
  - `bypassPermissions` — no permission checks at all
  - `plan` — read-only planning mode
- **`maxTurns`** — limit on conversation turns
- **`skills`** — preload specific skills into the sub-agent
- **`memory`** — persistent memory scopes: `user`, `project`, `local`
- **`background`** — run as background agent (true/false)
- **`isolation`** — isolate the agent's environment

### context: fork (Skills Integration)
- When a skill has `context: fork`, it runs inside a sub-agent
- The skill's instructions become the sub-agent's system context
- Useful for isolated, focused task execution

## Examples

### Custom Agent (.claude/agents/reviewer.md)
```yaml
---
name: reviewer
description: Code review specialist
tools: [Read, Grep, Glob]
model: claude-sonnet-4-20250514
permissionMode: default
maxTurns: 5
---
You are a code reviewer. Analyze code for bugs, style issues,
and potential improvements. Never modify files.
```

### Background Agent
```yaml
---
name: watcher
description: Monitors test results in background
background: true
tools: [Bash, Read]
maxTurns: 20
---
Run tests and report results.
```

### Agent with Preloaded Skills
```yaml
---
name: full-stack-dev
description: Full-stack development agent
skills: [review, deploy, test-runner]
permissionMode: acceptEdits
---
You are a full-stack developer with access to review, deploy, and test skills.
```

### Restricting Agent Spawning
```
Agent(reviewer)  — only allows spawning the "reviewer" sub-agent
Agent             — allows spawning any sub-agent
```

## Important for Exam (Mapping to Task Statements)

| Exam Topic | Key Facts |
|---|---|
| Built-in agents | Explore (Haiku, read-only), Plan (read-only), General-purpose (all tools) |
| Explore thoroughness | quick, medium, thorough |
| Custom agent location (project) | .claude/agents/<name>.md |
| Custom agent location (user) | ~/.claude/agents/<name>.md |
| Tool name change | Agent tool (renamed from Task tool) |
| Nesting restriction | Sub-agents CANNOT spawn other sub-agents |
| Agent spawning syntax | Agent(agent_type) to restrict |
| Background execution | Resume with SendMessage |
| Auto-compaction trigger | 95% capacity |
| Permission modes | default, acceptEdits, dontAsk, bypassPermissions, plan |
| Memory scopes | user, project, local |
| Skill integration | context: fork runs skill in sub-agent |
| Skills preloading | skills field in agent frontmatter |
