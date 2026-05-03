# Claude Code Best Practices and Plan Mode
Source: https://docs.anthropic.com/en/docs/claude-code/best-practices
Fetched: 2026-03-21 (from model knowledge - WebFetch was denied)

## Key Concepts for Exam

### Plan Mode vs Direct Execution

#### Plan Mode (shift+tab to toggle)
- Claude analyzes and plans before making changes
- Shows what it WOULD do without actually doing it
- User reviews the plan, then approves execution
- Useful for complex refactoring, understanding impact before changes
- Toggle with **Shift+Tab** in the REPL

**Plan mode workflow:**
1. User asks for a change
2. Claude analyzes codebase and creates a plan
3. Claude presents the plan (files to change, approach)
4. User reviews and approves (or modifies)
5. Claude executes the approved plan

#### Direct Execution (default)
- Claude analyzes and executes in one flow
- Each tool use still requires permission (unless auto-approved)
- Faster for straightforward tasks

### /compact Command
- Compresses the conversation history to save context window space
- Useful when conversation gets long and Claude starts losing earlier context
- Claude summarizes the conversation so far, replacing detailed history with a summary
- **When to use**: When you notice Claude forgetting earlier instructions or context
- Can provide a focus topic: `/compact focus on the authentication refactoring`

### Subagents and Forking

#### Explore Subagent
- A read-only subagent that Claude Code can spawn for research tasks
- Used when Claude needs to explore the codebase without modifying it
- Has access to Read, Glob, Grep tools only (no write/execute)
- Runs in a forked context - does not pollute the main conversation
- Results are summarized back to the main conversation

#### fork_session
- Internal mechanism for running parallel or isolated work
- Used by skills with `context: fork` frontmatter
- Creates a separate conversation branch
- The fork does NOT share context with the main conversation
- Results are returned when the fork completes

### CLAUDE.md Best Practices

#### Structure
```markdown
# Project Name

## Overview
Brief project description and tech stack.

## Development Commands
- Build: `npm run build`
- Test: `npm test`
- Lint: `npm run lint`
- Dev server: `npm run dev`

## Code Conventions
- [Specific patterns and preferences]

## Architecture
- [Key architectural decisions]

## Common Gotchas
- [Things Claude should watch out for]
```

#### Do's
- Keep it concise and focused
- Include commonly-used commands
- Document non-obvious project conventions
- Update it as conventions evolve
- Include file path patterns for where things live
- Put environment-specific setup instructions

#### Don'ts
- Don't make it too long (wastes context window)
- Don't duplicate information already in code comments
- Don't include sensitive information (API keys, secrets)
- Don't include information that changes frequently

### Effective Prompting Tips

#### Be Specific
```
Bad:  "fix the bug"
Good: "fix the null pointer exception in src/auth/login.ts when the user email is undefined"
```

#### Provide Context
```
Bad:  "add tests"
Good: "add unit tests for the UserService class in src/services/user.ts,
       focusing on the createUser and deleteUser methods. Use Jest with the
       existing test patterns in __tests__/"
```

#### Break Complex Tasks into Steps
```
"Let's refactor the authentication system. First, let's plan the approach:
1. Identify all auth-related files
2. Design the new token-based auth flow
3. Implement changes file by file
4. Update tests"
```

#### Use Plan Mode for Risky Changes
- Large refactoring across many files
- Database schema changes
- API contract changes
- Anything affecting production infrastructure

### Workflow Optimization

#### Multi-step Workflows
```bash
# Step 1: Analyze
claude -p "analyze the test coverage gaps" --output-format json > analysis.json

# Step 2: Generate
claude -p "based on analysis.json, generate missing tests" --max-turns 20

# Step 3: Verify
claude -p "run all tests and report results" --output-format json
```

#### Using --resume for Iterative Work
```bash
# First session
claude "start refactoring the auth module"
# ... work happens, session ends ...

# Later, resume and continue
claude --resume "now let's add the tests"
```

#### Keeping Context Fresh
1. Use `/compact` when conversation gets long
2. Use CLAUDE.md for persistent instructions (survives across sessions)
3. Use `--resume` / `--continue` for multi-session workflows
4. Reference specific files rather than asking Claude to "find" things

### Permission Management Best Practices

#### Development
```json
{
  "permissions": {
    "allow": [
      "Read",
      "Write",
      "Glob",
      "Grep",
      "Bash(npm test)",
      "Bash(npm run lint)",
      "Bash(git *)"
    ]
  }
}
```

#### CI/CD (Restrictive)
- Use `--dangerously-skip-permissions` only in sandboxed environments
- Prefer `--allowedTools` to whitelist specific tools
- Set `--max-turns` to prevent runaway loops
- Monitor costs with JSON output

### Agentic Coding Tips

1. **Let Claude explore first**: Don't over-constrain; let it read the codebase
2. **Review before approving writes**: Check proposed edits carefully
3. **Use git**: Work on branches so changes can be easily reverted
4. **Iterate**: If the first result isn't right, give feedback and let Claude refine
5. **Trust but verify**: Run tests after Claude makes changes

## Configuration Examples

### Optimal CLAUDE.md for a TypeScript Project
```markdown
# MyApp

## Stack
TypeScript, React 18, Next.js 14, Prisma, PostgreSQL

## Commands
- `npm run dev` - Start dev server (port 3000)
- `npm test` - Run Jest tests
- `npm run test:e2e` - Run Playwright E2E tests
- `npm run lint` - ESLint + Prettier check
- `npm run typecheck` - TypeScript compilation check

## Conventions
- Use barrel exports (index.ts) for each module
- React components: functional with hooks, no class components
- API routes: validate input with zod schemas
- Error handling: use Result<T, E> pattern, avoid throwing
- Tests: colocated in __tests__/ directories, use factories for test data

## Architecture
- src/app/ - Next.js app router pages
- src/components/ - Shared UI components
- src/services/ - Business logic
- src/lib/ - Utility functions and shared code
- prisma/ - Database schema and migrations
```

## CLI Reference

| Feature | How to Access |
|---------|---------------|
| Plan Mode | Shift+Tab in REPL |
| Compact | `/compact` or `/compact <focus>` |
| Memory | `/memory` |
| Clear | `/clear` |
| Resume | `claude --resume` |
| Continue | `claude --continue` or `claude -c` |

## Important Details

- **Plan mode is a toggle**: Shift+Tab switches between plan and execute modes
- **/compact preserves key info**: The summary retains important context, but details may be lost
- **/compact with focus**: `/compact focus on X` biases the summary toward topic X
- **Explore subagent is read-only**: Cannot modify files or run commands
- **fork_session isolation**: Forked conversations have NO access to parent conversation history
- **CLAUDE.md is loaded every turn**: Changes to CLAUDE.md take effect immediately (no restart needed)
- **Permission patterns**: Use glob-style wildcards (`Bash(git *)` allows all git commands)
- **Cost awareness**: Use `--max-turns` and monitor `cost_usd` in JSON output for budget control
- **Context window management**: Claude Code automatically manages context but /compact helps when things get slow or forgetful
- **Plan mode + permission system**: In plan mode, Claude still needs tool approval to execute the plan
