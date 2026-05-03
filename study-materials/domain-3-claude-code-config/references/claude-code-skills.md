# Claude Code Skills
Source: https://code.claude.com/docs/en/skills
Fetched: 2026-03-21

## Key Concepts for Exam

### Skill File Location
- Project skills: `.claude/skills/<name>/SKILL.md`
- Personal skills: `~/.claude/skills/<name>/SKILL.md`
- Plugin skills (from plugins/extensions)
- Enterprise skills (organization-managed)

### Priority Order
**Enterprise > Personal > Project** (enterprise wins on conflict)

### Skills vs CLAUDE.md
- **Skills** load **on demand** (when invoked or triggered)
- **CLAUDE.md** is **always loaded** every session
- Skill **descriptions** are loaded into context so Claude knows what skills are available
- Use skills for specialized workflows; use CLAUDE.md for always-on instructions

### Backward Compatibility
- `.claude/commands/` directory is **merged into skills** (backward compatible)
- Old command format still works

## Configuration Details

### SKILL.md Frontmatter Fields
```yaml
---
name: my-skill
description: What this skill does (loaded into context for discovery)
argument-hint: <file-path>
disable-model-invocation: true    # Only user can invoke (not Claude)
user-invocable: false              # Only Claude can invoke (not user)
allowed-tools: [Read, Edit, Bash]  # Restrict available tools
model: claude-sonnet-4-20250514          # Override model
effort: medium                     # Effort level
context: fork                      # Runs in forked subagent context
agent: my-custom-agent             # Use specific agent
hooks:                             # Lifecycle hooks
  pre-run: ...
  post-run: ...
---
```

### Key Frontmatter Fields Explained
- **`context: fork`** — runs the skill in a **forked subagent context** (isolated)
- **`disable-model-invocation: true`** — only the **user** can invoke this skill (prevents Claude from calling it autonomously)
- **`user-invocable: false`** — only **Claude** can invoke (not available as slash command)
- **`description`** — loaded into context so Claude knows the skill exists and what it does

### Argument Substitutions
- `$ARGUMENTS` — entire argument string passed to the skill
- `$ARGUMENTS[N]` — Nth argument (zero-indexed)
- `$N` — shorthand for $ARGUMENTS[N] (e.g., $0, $1, $2)

### Dynamic Context Injection
- Use `` !`command` `` syntax to inject dynamic content
- The command runs and its output is included in the skill context
- Example: `` !`git diff --staged` `` injects current staged changes

### Supporting Files
- Place additional files alongside SKILL.md in the skill directory
- These files can be referenced by the skill for templates, examples, etc.

## Examples

### Basic Skill (.claude/skills/review/SKILL.md)
```yaml
---
name: review
description: Reviews code changes for quality and style
argument-hint: <file-path>
allowed-tools: [Read, Grep, Glob]
---
Review the code in $ARGUMENTS for:
1. Code quality issues
2. Style violations
3. Potential bugs
```

### User-Only Skill (No Auto-Invocation)
```yaml
---
name: deploy
description: Deploy to production
disable-model-invocation: true
---
Run the deployment pipeline for the current branch.
```

### Claude-Only Skill (Not User-Invocable)
```yaml
---
name: lint-check
description: Internal lint checking helper
user-invocable: false
---
Run linting and return results.
```

### Forked Context Skill
```yaml
---
name: analyze
description: Deep code analysis
context: fork
---
Perform thorough analysis of the codebase architecture.
```

## Important for Exam (Mapping to Task Statements)

| Exam Topic | Key Facts |
|---|---|
| Skill location (project) | .claude/skills/<name>/SKILL.md |
| Skill location (personal) | ~/.claude/skills/<name>/SKILL.md |
| Priority order | Enterprise > Personal > Project |
| On-demand loading | Skills load when invoked; CLAUDE.md always loaded |
| Description purpose | Loaded into context for Claude to discover available skills |
| Forked execution | context: fork runs in subagent |
| Invocation control | disable-model-invocation (user only), user-invocable: false (Claude only) |
| Argument access | $ARGUMENTS, $ARGUMENTS[N], $N |
| Dynamic context | !`command` syntax |
| Legacy commands | .claude/commands/ merged into skills |
| Tool restrictions | allowed-tools field in frontmatter |
