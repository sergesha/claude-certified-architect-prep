# Claude Code Memory & CLAUDE.md
Source: https://code.claude.com/docs/en/memory
Fetched: 2026-03-21

## Key Concepts for Exam

### CLAUDE.md Hierarchy (Priority Order)
1. **Managed policy** (enterprise-level, highest priority)
2. **Project instructions** — `./CLAUDE.md` or `./.claude/CLAUDE.md` (repo root)
3. **User instructions** — `~/.claude/CLAUDE.md` (personal, NOT shared via version control)

- User-level CLAUDE.md is **not** shared via version control — it lives in the home directory
- Project-level CLAUDE.md **is** committed to the repo for team sharing

### @import Syntax
- Use `@path/to/import` to include other files into CLAUDE.md
- Paths are **relative** to the importing file
- Maximum **5 hops** depth (import chain limit)
- Allows modular organization of instructions

### .claude/rules/ Directory (Path-Specific Rules)
- Place rule files in `.claude/rules/` directory
- Use YAML frontmatter with `paths` field to specify which files the rule applies to
- Rules **load only when editing matching files** — reduces noise, saves context
- Glob patterns for path matching:
  - `**/*.ts` — all TypeScript files
  - `src/**/*.*` — all files under src/
  - Standard glob pattern syntax

### User-Level Rules
- `~/.claude/rules/` — applies to **every project** the user works on
- Personal coding preferences, style guides, etc.

### Auto Memory
- Claude writes notes itself during conversations
- Stored in `~/.claude/projects/<project>/memory/`
- `MEMORY.md` — first **200 lines** loaded every session automatically

### Key Commands
- `/memory` — view all currently loaded memory files
- `/init` — generate a starting CLAUDE.md for a project

## Configuration Details

### claudeMdExcludes
- Used for **monorepos** to exclude certain CLAUDE.md files
- Prevents loading irrelevant project instructions from sub-packages

### Effective Instructions Best Practices
- Keep under **200 lines** total
- Be **specific** — vague instructions waste context
- Be **consistent** — avoid contradictory rules
- Path-specific rules reduce noise by loading only when relevant

## Examples

### Path-Specific Rule File (.claude/rules/typescript-style.md)
```yaml
---
paths:
  - "**/*.ts"
  - "**/*.tsx"
---
Use strict TypeScript. No `any` types. Prefer interfaces over type aliases.
```

### @import in CLAUDE.md
```
@.claude/rules/coding-standards
@.claude/rules/testing-guidelines
```

### User-Level CLAUDE.md (~/.claude/CLAUDE.md)
```
Always use descriptive variable names.
Prefer functional programming patterns.
```

## Important for Exam (Mapping to Task Statements)

| Exam Topic | Key Facts |
|---|---|
| Memory hierarchy | Managed > Project > User |
| Version control | User-level (~/.claude/) is NOT committed |
| Path-specific rules | YAML frontmatter `paths` field, glob patterns |
| Auto memory location | ~/.claude/projects/<project>/memory/ |
| MEMORY.md loading | First 200 lines, every session |
| Import depth limit | 5 hops maximum |
| Monorepo config | claudeMdExcludes |
| Rule loading behavior | Path-specific rules load only when editing matching files |
| Generating CLAUDE.md | /init command |
| Viewing loaded memory | /memory command |
