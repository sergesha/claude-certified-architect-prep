# Domain 3: Claude Code Configuration & Workflows (20%)

This domain tests your ability to configure Claude Code for team and individual productivity,
design effective workflows, and integrate Claude Code into CI/CD pipelines.

---

## Task 3.1: CLAUDE.md Configuration Hierarchy

### Key Concepts

Claude Code uses a layered memory system based on `CLAUDE.md` files at multiple levels.
These files are loaded automatically when Claude Code starts and provide persistent instructions,
project context, and conventions.

**Loading order (all levels are merged, not overridden):**

1. **User-level**: `~/.claude/CLAUDE.md`
   - Personal preferences and global instructions
   - NOT shared with the team (not in any repo)
   - Example: preferred coding style, editor quirks, personal tool preferences

2. **Project-level** (two equivalent locations):
   - `CLAUDE.md` in the repository root
   - `.claude/CLAUDE.md` in the repository root
   - Shared via git with the entire team
   - Example: project architecture, coding standards, build commands

3. **Directory-level**: `CLAUDE.md` in any subdirectory
   - Loaded only when Claude is working on files in or below that directory
   - Useful for module-specific or package-specific conventions
   - Example: `frontend/CLAUDE.md` with React-specific rules

**All levels are additive** -- they combine rather than override each other. A user-level
file does not replace a project-level file; both apply simultaneously.

### The `@import` Syntax

CLAUDE.md files can reference external files using the `@import` directive:

```markdown
# Project Instructions

@docs/coding-standards.md
@.claude/conventions/api-design.md
@../shared-config/CLAUDE.md
```

- Paths are relative to the CLAUDE.md file containing the import
- Imported files are inlined into the context as if their content were in the CLAUDE.md itself
- Useful for breaking large instruction sets into manageable, topic-specific files
- Can reference files outside the `.claude/` directory

#### Using `@import` to Selectively Include Standards per Package

A key use case for `@import` is selectively including different standards files depending
on the package or module. Each package maintainer can import only the standards relevant
to their domain:

```markdown
# packages/payments/CLAUDE.md
# Payment team's directory-level CLAUDE.md imports only payment-relevant standards

@../../.claude/standards/api-conventions.md
@../../.claude/standards/security-requirements.md
@../../.claude/standards/pci-compliance.md
```

```markdown
# packages/frontend-ui/CLAUDE.md
# Frontend team imports only UI-relevant standards

@../../.claude/standards/react-conventions.md
@../../.claude/standards/accessibility.md
@../../.claude/standards/css-guidelines.md
```

```markdown
# packages/data-pipeline/CLAUDE.md
# Data engineering team imports data-specific standards

@../../.claude/standards/sql-conventions.md
@../../.claude/standards/etl-patterns.md
@../../.claude/standards/data-validation.md
```

This pattern avoids loading irrelevant context (e.g., PCI compliance rules when editing
a CSS component) and keeps token usage efficient. Each package's maintainer controls
which standards apply to their domain.

### The `.claude/rules/` Directory

The `.claude/rules/` directory provides an alternative to putting everything in a single
CLAUDE.md file:

- Each file in `.claude/rules/` is a standalone rule file
- Files can have YAML frontmatter for path-scoping (see Task 3.3)
- Files without frontmatter are loaded globally (like project-level CLAUDE.md)
- Easier to manage in version control than one monolithic file
- Shared with the team via git

#### Splitting a Large CLAUDE.md into `.claude/rules/` Files

When a monolithic CLAUDE.md grows too large, split it into topic-specific rule files:

```
# Before: one massive CLAUDE.md with 500+ lines covering everything

# After: organized topic-specific rule files
.claude/
  rules/
    testing.md              # Testing conventions, frameworks, coverage requirements
    api-conventions.md      # REST API design patterns, error formats, versioning
    deployment.md           # Deployment procedures, environment configs, rollback steps
    code-style.md           # Formatting, naming conventions, import ordering
    database.md             # Migration patterns, query conventions, indexing rules
    security.md             # Auth patterns, input validation, secrets handling
```

Example `testing.md`:
```markdown
# Testing Standards

- Use Jest for unit tests, Playwright for E2E
- Minimum 80% code coverage for new modules
- Test file naming: `<module>.test.ts` co-located with source
- Always test error paths, not just happy paths
- Use factories (not fixtures) for test data generation
- Mock external services at the HTTP boundary, not at the module level
```

Example `api-conventions.md`:
```markdown
# API Design Conventions

- Use RESTful resource naming: plural nouns, no verbs in URLs
- Version APIs via URL path prefix: /api/v1/
- Error responses follow RFC 7807 Problem Details format
- All endpoints require authentication except /health and /docs
- Pagination: cursor-based for lists, with `next_cursor` field
```

Benefits of splitting:
- Each file is independently reviewable in PRs
- Team members can own specific rule files
- Path-scoped rules (Task 3.3) only work in `.claude/rules/`, not in monolithic CLAUDE.md
- Reduces merge conflicts when multiple people update conventions simultaneously

### The `/memory` Command

Use the `/memory` command inside Claude Code to:
- See which CLAUDE.md files are currently loaded
- Verify which memory files are active in the current session
- Debug why instructions are or are not being applied
- Check the effective configuration hierarchy
- **Diagnose inconsistent behavior**: If Claude sometimes follows a convention and sometimes
  does not, run `/memory` to check whether the relevant CLAUDE.md or rule file is actually
  loaded. Inconsistent behavior often means a directory-level or path-scoped rule is only
  loading in some contexts (e.g., when editing files in one directory but not another).
- The `/memory` output lists each loaded file by path, making it easy to spot when a file
  is missing from the active set

### Practical Skills

**Diagnosing why a team member does not get instructions:**

1. Run `/memory` to see loaded files
2. Check if the file is at the correct path (e.g., `CLAUDE.md` vs `.claude/CLAUDE.md`)
3. Verify the file is committed to git and the team member has pulled
4. Check if the file is in `.gitignore`
5. For user-level files: confirm `~/.claude/CLAUDE.md` exists on their machine
6. For directory-level files: confirm Claude is working on files in/below that directory
7. For `@import` references: verify the imported file path is correct and the file exists

**Key scenario -- new team member missing instructions:**

A common exam scenario: An experienced developer's Claude Code follows all project conventions,
but a new team member's Claude Code does not. The most likely cause is that the conventions
were placed in the experienced developer's **user-level** `~/.claude/CLAUDE.md` (personal,
not shared via version control) instead of the **project-level** `.claude/CLAUDE.md` or
root `CLAUDE.md` (shared via git). The fix: move shared conventions from `~/.claude/CLAUDE.md`
to the project-level file so all team members get them automatically.

**Diagnosing inconsistent behavior with `/memory`:**

If Claude Code sometimes follows a rule and sometimes ignores it:
1. Run `/memory` in a session where it works and note the loaded files
2. Run `/memory` in a session where it does not work and compare
3. Look for directory-level CLAUDE.md files or path-scoped rules that only load in certain contexts
4. Check if `@import` targets exist at the expected relative paths from each CLAUDE.md location

### Configuration Example

```
my-project/
  CLAUDE.md                          # Project-level (shared)
  .claude/
    CLAUDE.md                        # Also project-level (shared, alternative location)
    rules/
      code-style.md                  # Global rule file (shared)
      terraform.md                   # Path-scoped rule (shared, see Task 3.3)
  frontend/
    CLAUDE.md                        # Directory-level, loaded for frontend/ work
  backend/
    CLAUDE.md                        # Directory-level, loaded for backend/ work

~/.claude/
  CLAUDE.md                          # User-level (personal, not shared)
```

### Common Exam Traps

- **Trap**: Thinking project-level CLAUDE.md overrides user-level. It does NOT -- all levels are merged/additive.
- **Trap**: Confusing `.claude/CLAUDE.md` (project config) with `~/.claude/CLAUDE.md` (user config). The tilde (`~`) means home directory = personal.
- **Trap**: Assuming directory-level CLAUDE.md files load globally. They only load when working on files in or below that directory.
- **Trap**: Thinking `@import` can only reference files inside `.claude/`. It can reference any file reachable by a relative path.
- **Trap**: Forgetting that `.claude/rules/` files are shared via git, while `~/.claude/` is personal.

---

## Task 3.2: Custom Slash Commands and Skills

### Key Concepts

Claude Code supports two mechanisms for reusable workflows: **custom slash commands** and **skills**.

#### Custom Slash Commands

Slash commands are Markdown files that define reusable prompts.

**Project commands** (shared with team):
```
.claude/commands/
  review.md          -> invoked as /project:review
  deploy-check.md    -> invoked as /project:deploy-check
```

**Personal commands** (not shared):
```
~/.claude/commands/
  my-review.md       -> invoked as /user:my-review
```

Command files contain the prompt template. They can use the `$ARGUMENTS` placeholder
to accept user input:

```markdown
<!-- .claude/commands/review.md -->
Review the following code for security vulnerabilities and performance issues.
Focus on: $ARGUMENTS
```

Usage: `/project:review the authentication module`

#### Skills (Advanced Reusable Workflows)

Skills are more powerful than slash commands. They live in `.claude/skills/` (project)
or `~/.claude/skills/` (personal) and use a `SKILL.md` file with YAML frontmatter.

**SKILL.md Frontmatter Fields:**

| Field | Description |
|-------|-------------|
| `context: fork` | Runs the skill in an isolated sub-agent context. The sub-agent cannot see or modify the parent conversation. Output is summarized back. |
| `allowed-tools` | Restricts which tools the skill can use (e.g., only Read + Grep, no file writes) |
| `argument-hint` | Describes what parameter the user should provide; shown in autocomplete |

#### Why `context: fork` for Verbose/Exploratory Output

When a skill performs verbose exploration (searching many files, reading large code sections,
producing lengthy analysis), `context: fork` is critical because:

- The sub-agent runs in an **isolated conversation context** -- all the verbose discovery
  output stays in the fork and does not pollute the parent conversation
- Only a **summary** is returned to the parent context, keeping it clean and focused
- This is ideal for skills that do broad codebase searches, architecture analysis, or
  dependency audits where the intermediate output would be thousands of lines
- Without `context: fork`, all that verbose output would consume tokens in the main
  conversation and reduce the effective context window for subsequent work

```yaml
# .claude/skills/architecture-review/SKILL.md
---
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
argument-hint: "Module or directory to analyze (e.g., src/auth/)"
---

Analyze the architecture of the specified module. Read all files,
map dependencies, identify patterns and anti-patterns.
Return a concise summary of: structure, key abstractions, coupling issues,
and recommendations.
```

#### Personal Skills with Different Names

Personal skills in `~/.claude/skills/` can have names that differ from project skills.
This allows individual developers to create their own workflow variations:

```
# Project skill (shared, same for everyone):
.claude/skills/code-review/SKILL.md       -> invoked as a project skill

# Personal skill (only on your machine, your own workflow):
~/.claude/skills/my-deep-review/SKILL.md  -> invoked as a personal skill
~/.claude/skills/quick-scan/SKILL.md      -> another personal skill
```

Personal skills are useful for individual productivity workflows that may not suit the
entire team (e.g., a skill that uses a personal tool preference or outputs in a specific
format for your note-taking system).

### Practical Skills

**When to use skills vs CLAUDE.md:**

| Use Case | Mechanism |
|----------|-----------|
| Always-on project conventions | CLAUDE.md |
| On-demand reusable workflow | Slash command or skill |
| Isolated investigation that should not pollute context | Skill with `context: fork` |
| Workflow needing restricted tool access | Skill with `allowed-tools` |
| Simple prompt template | Slash command |
| Complex multi-step workflow | Skill |

### Configuration Example

```yaml
# .claude/skills/security-audit/SKILL.md
---
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
argument-hint: "Path to the module to audit (e.g., src/auth/)"
---

# Security Audit Skill

Perform a thorough security audit of the code at the specified path.

## Steps
1. Search for common vulnerability patterns (SQL injection, XSS, CSRF)
2. Check authentication and authorization logic
3. Review input validation and sanitization
4. Examine dependency usage for known vulnerabilities
5. Report findings with severity ratings

Focus on: $ARGUMENTS
```

```markdown
<!-- .claude/commands/fix-lint.md -->
Run the linter, then fix all reported issues in: $ARGUMENTS
If no path is specified, fix issues in all staged files.
```

### Common Exam Traps

- **Trap**: Confusing command paths. Project commands = `.claude/commands/`, personal = `~/.claude/commands/`. Project skills = `.claude/skills/`, personal = `~/.claude/skills/`.
- **Trap**: Thinking `context: fork` means the skill can modify files seen by the parent. Fork creates an isolated sub-agent; its file changes are real, but its conversation context is separate.
- **Trap**: Forgetting that `allowed-tools` is a restriction (allowlist), not a blocklist. Only listed tools are available.
- **Trap**: Mixing up slash commands (simple Markdown prompt templates) with skills (structured SKILL.md with frontmatter and isolation capabilities).
- **Trap**: Not knowing the invocation syntax: `/project:command-name` for project commands, `/user:command-name` for personal commands.

---

## Task 3.3: Path-Specific Rules

### Key Concepts

Files in `.claude/rules/` can include YAML frontmatter with a `paths` field containing
glob patterns. When the `paths` field is present, the rule file is only loaded when Claude
is editing files that match one of the specified glob patterns.

**Rule file with path scoping:**

```yaml
# .claude/rules/terraform.md
---
paths:
  - "terraform/**/*"
---

When modifying Terraform files:
- Always run `terraform fmt` after changes
- Use variables for all configurable values
- Include description fields on all variables and outputs
- Follow the standard module structure: main.tf, variables.tf, outputs.tf
```

```yaml
# .claude/rules/react-tests.md
---
paths:
  - "**/*.test.tsx"
  - "**/*.test.ts"
  - "**/*.spec.tsx"
---

When writing or modifying React tests:
- Use React Testing Library, not Enzyme
- Prefer userEvent over fireEvent
- Test behavior, not implementation details
- Always include accessibility assertions
```

### Path-Scoped Rules vs Directory-Level CLAUDE.md

| Feature | `.claude/rules/` with `paths` | Directory-level `CLAUDE.md` |
|---------|-------------------------------|----------------------------|
| Location | `.claude/rules/` (centralized) | In each subdirectory |
| Scoping | Glob patterns (flexible, cross-directory) | Directory tree (hierarchical) |
| Use case | File-type conventions (e.g., all test files) | Module-specific conventions |
| Cross-directory | Yes (e.g., `**/*.proto`) | No (only that directory and below) |
| Discoverability | All rules in one place | Distributed across the repo |

### Practical Skills

- Use `.claude/rules/` with glob patterns when conventions apply to a **file type** regardless of location (e.g., all `.test.tsx` files, all `.proto` files)
- Use directory-level `CLAUDE.md` when conventions apply to a **module/package** regardless of file type (e.g., everything in `frontend/`)
- Glob patterns support standard syntax: `*` (single segment), `**` (recursive), `?` (single char), `{a,b}` (alternation)
- Rules without a `paths` field in `.claude/rules/` are loaded globally

### Configuration Example

```
.claude/
  rules/
    general.md              # No frontmatter -> always loaded
    terraform.md            # paths: ["terraform/**/*"] -> only for terraform files
    api-design.md           # paths: ["src/api/**/*"] -> only for API code
    testing.md              # paths: ["**/*.test.*", "**/*.spec.*"] -> all test files
    documentation.md        # paths: ["docs/**/*", "**/*.md"] -> docs and markdown
```

### Common Exam Traps

- **Trap**: Thinking rules without a `paths` field are never loaded. They are ALWAYS loaded (global rules).
- **Trap**: Confusing the `paths` field glob with file-system globs. The paths are relative to the project root.
- **Trap**: Using directory-level CLAUDE.md when the convention spans multiple directories by file type. Use `.claude/rules/` with glob patterns instead.
- **Trap**: Forgetting that path-scoped rules are only loaded when Claude is **editing** matching files, not just reading them.
- **Trap**: Placing path-scoped YAML frontmatter in a CLAUDE.md file. The `paths` frontmatter only works in `.claude/rules/` files.

---

## Task 3.4: Plan Mode vs Direct Execution

### Key Concepts

Claude Code supports different interaction modes optimized for different task types.

#### Plan Mode

Plan mode instructs Claude to think through a problem and propose a plan before making changes.
Claude will analyze the codebase, outline its approach, and wait for approval before executing.

**Best for:**
- Complex, multi-file architectural changes
- Large-scale refactoring (e.g., library migrations touching 45+ files)
- Tasks requiring investigation before action
- Situations where multiple valid approaches exist and the choice matters
- Architectural decisions (e.g., microservice restructuring, monolith decomposition)
- Changes that affect many parts of the system

**Why plan mode prevents costly rework:**
Without planning, Claude might commit to an approach early, make changes across dozens of
files, and then discover a fundamental problem. Plan mode forces exploration before commitment,
identifying issues (circular dependencies, missing interfaces, breaking changes) before any
code is modified. For a library migration touching 45+ files, discovering the wrong migration
strategy after 20 files are changed means reverting all of them.

**How to activate:**
- Use the `plan` keyword or shift+tab to toggle plan mode
- Ask Claude to "plan" or "think about" a change
- Claude will output a structured plan and wait for confirmation

#### Direct Execution

In direct execution mode, Claude immediately begins making changes.

**Best for:**
- Simple, well-scoped changes (fix a typo, add a field)
- Changes where the approach is obvious
- Following up on an already-approved plan
- Small bug fixes with clear solutions

#### Explore Sub-agent

The explore sub-agent is a built-in Claude Code feature for isolating verbose discovery
output from the main conversation. It is conceptually similar to `context: fork` in skills
but is a built-in capability rather than a user-defined skill.

**How it works:**
1. Claude spawns an explore sub-agent in a **forked/isolated context**
2. The sub-agent performs verbose investigation: reading many files, running searches,
   tracing call chains, analyzing dependencies
3. All intermediate output (file contents, search results, analysis) stays in the sub-agent
4. The sub-agent **returns only a summary** to the main conversation
5. The main conversation context remains clean and focused

**Key characteristics:**
- Runs investigation in a forked context (isolated from parent)
- Reads files, searches code, analyzes architecture
- Returns a concise summary to the main conversation
- Keeps the main context clean, preserving tokens for implementation work
- Useful for "find all usages of X", "understand how Y works", "map the dependency graph"

**Difference from skills with `context: fork`:**
- Explore sub-agent is **built-in** and invoked automatically or via prompting
- Skills with `context: fork` are **user-defined** reusable workflows with specific instructions
- Both use the same isolation mechanism (forked context with summary return)

#### Combining Modes

A common effective pattern:

1. **Plan mode** for investigation and architecture decisions
2. Review and approve the plan
3. **Direct execution** for implementing the approved plan
4. Use **explore sub-agent** for targeted investigation during implementation

### Practical Skills

**Scenario decision matrix:**

| Scenario | Recommended Mode | Why |
|----------|-----------------|-----|
| "Restructure monolith into microservices" | Plan mode | Architectural decision with multiple valid decomposition strategies |
| "Migrate from Moment.js to date-fns across 45+ files" | Plan mode | Large-scale change; need to map all usage patterns before committing |
| "Refactor the auth module to use OAuth2" | Plan mode | Multi-file, multiple approaches, security implications |
| "Fix the typo in the error message on line 42" | Direct execution | Single-file, obvious change |
| "Add a new API endpoint for user preferences" | Plan mode (multi-file) | Requires route, controller, model, tests |
| "Update the copyright year in the footer" | Direct execution | Trivial, well-scoped |
| "Migrate from REST to GraphQL" | Plan mode | Fundamental architecture change |
| "How does the payment processing pipeline work?" | Explore sub-agent | Investigation only, verbose output |
| "Add a missing null check in the parser" | Direct execution | Single-file bug fix, clear solution |
| "Restructure the monorepo into packages" | Plan mode | Architectural, many files affected |
| "Add a validation conditional to the email field" | Direct execution | Single, well-scoped change |
| "Find all places that call the deprecated API" | Explore sub-agent | Discovery task, summary needed |

### Configuration Example

```
# In CLAUDE.md, you can set default behavior:

## Workflow Preferences
- For changes touching more than 3 files, always use plan mode first
- For database schema changes, always plan and get explicit approval
- Use explore sub-agent for codebase investigation before proposing changes
```

### Common Exam Traps

- **Trap**: Thinking plan mode and direct execution are separate tools or CLI flags. Plan mode is a conversation-level toggle (shift+tab), not a CLI argument.
- **Trap**: Confusing the explore sub-agent with skills using `context: fork`. The explore sub-agent is a built-in feature for investigation; skills are user-defined workflows.
- **Trap**: Assuming plan mode prevents all file modifications. Claude may still read files during planning; it delays *writes* until the plan is approved.
- **Trap**: Not recognizing that combining modes is the recommended practice for complex tasks, not choosing one exclusively.

---

## Task 3.5: Iterative Refinement Techniques

### Key Concepts

Claude Code works best with iterative refinement -- progressively improving outputs through
structured feedback patterns.

#### Input/Output Examples

When prose instructions lead to inconsistent results, provide concrete input/output examples:

```markdown
# In CLAUDE.md or a prompt:

## Error Message Format
When generating error messages, follow this pattern:

Input: User not found in database
Output: [ERR-404] UserNotFoundError: The requested user does not exist in the database. Please verify the user ID and try again.

Input: Invalid email format
Output: [ERR-400] ValidationError: The provided email address is not in a valid format. Expected: user@domain.tld

Input: Database connection timeout
Output: [ERR-503] ConnectionTimeoutError: Unable to establish a database connection within the configured timeout period. Check database status and network connectivity.
```

This is more reliable than describing the format in prose because Claude can pattern-match
from examples.

#### Test-Driven Iteration

Write tests first, then have Claude iterate until they pass:

1. Write or specify the test cases that define correct behavior
2. Ask Claude to implement the code
3. Run the tests
4. If tests fail, share the failure output with Claude
5. Claude fixes the implementation
6. Repeat until all tests pass

This creates a tight feedback loop with objective success criteria.

**Providing specific test cases with example input and expected output for edge cases:**

When asking Claude to implement a function, include concrete edge case examples:

```markdown
Implement a parsePhoneNumber(input: string) function.

## Expected behavior with examples:

Normal cases:
- Input: "+1 (555) 123-4567" -> Output: { country: "1", number: "5551234567" }
- Input: "555-123-4567" -> Output: { country: null, number: "5551234567" }

Edge cases:
- Input: "" -> Output: null (empty string)
- Input: "abc" -> Output: null (no digits)
- Input: "+44 20 7946 0958" -> Output: { country: "44", number: "2079460958" } (international)
- Input: "   +1-555-123-4567  " -> Output: { country: "1", number: "5551234567" } (whitespace)
- Input: "+1 (555) 123-4567 ext 89" -> Output: { country: "1", number: "5551234567", ext: "89" }
```

Providing 2-3 concrete I/O examples for transformation requirements eliminates ambiguity
far more effectively than prose descriptions like "handle edge cases appropriately."

#### Interview Pattern

Instead of providing all requirements upfront, instruct Claude to ask clarifying questions:

```markdown
# In CLAUDE.md:
Before implementing any new feature, ask me the following questions:
1. What are the edge cases to handle?
2. Should this be backwards-compatible?
3. What error handling strategy should we use?
4. Are there performance requirements?

Do not proceed until I have answered.
```

This prevents Claude from making incorrect assumptions on ambiguous requirements.

#### Single Message vs Sequential Messages

| Pattern | When to Use |
|---------|-------------|
| **Single message** (all at once) | Problems that interact with each other; changes that must be consistent across files; refactoring where one change affects others |
| **Sequential messages** (one at a time) | Independent problems; unrelated bug fixes; separate features that do not interact |

**Why this matters**: When problems interact, giving them in a single message lets Claude
consider all constraints simultaneously. Sequential messages for interacting problems risk
Claude making a change that conflicts with a later request.

### Practical Skills

- Start with broad instructions, then refine based on output quality
- Use "show me, don't tell me" -- examples beat descriptions
- Leverage test suites as objective quality gates
- Use the interview pattern for ambiguous or complex features
- Batch related changes; separate unrelated ones

### Configuration Example

```markdown
# In CLAUDE.md:

## Iteration Workflow
- When implementing a new feature, always write tests first
- If the first implementation does not pass tests, analyze the failure
  and fix the root cause (do not just make the test pass with hacks)
- For API design, ask me about edge cases before implementing
- When I provide examples, match the exact format shown

## Input/Output Examples for Logging
Input: user login successful
Output: {"level":"info","event":"auth.login","status":"success","timestamp":"<ISO8601>"}

Input: payment processing failed
Output: {"level":"error","event":"payment.process","status":"failure","timestamp":"<ISO8601>"}
```

### Common Exam Traps

- **Trap**: Thinking sequential messages are always better because they are simpler. For interacting problems, a single message is superior because Claude can reason about dependencies.
- **Trap**: Assuming the interview pattern slows things down. It actually prevents costly rework from incorrect assumptions.
- **Trap**: Providing only prose descriptions when examples would be clearer. The exam may present a scenario where output format is inconsistent and ask for the best fix -- the answer is input/output examples.
- **Trap**: Running tests only once. Test-driven iteration implies a loop: implement, test, fix, test again.
- **Trap**: Batching unrelated changes into one message "for efficiency." Independent changes should be sequential to keep context focused.

---

## Task 3.6: CI/CD Integration

### Key Concepts

Claude Code can run in non-interactive (headless) mode for CI/CD pipelines, code review
automation, and other automated workflows.

#### The `-p` / `--print` Flag

The `-p` (or `--print`) flag runs Claude Code in non-interactive mode:

```bash
# Basic non-interactive usage
claude -p "Review this PR for security issues"

# Pipe input
git diff HEAD~1 | claude -p "Review this diff for bugs"

# With specific model
claude -p --model claude-sonnet-4-20250514 "Summarize the changes in this repo"
```

- No interactive prompts or confirmations
- Output goes to stdout
- Suitable for scripting and CI/CD pipelines
- Exits after completing the task

#### Structured Output

For programmatic consumption of Claude Code output:

```bash
# JSON output format
claude -p --output-format json "List all TODO comments in src/"

# With JSON schema for structured extraction
claude -p --output-format json --json-schema '{"type":"object","properties":{"issues":{"type":"array","items":{"type":"object","properties":{"file":{"type":"string"},"line":{"type":"number"},"severity":{"type":"string"},"message":{"type":"string"}}}}}}' "Review src/ for code quality issues"
```

- `--output-format json`: Returns output as JSON
- `--json-schema`: Constrains the JSON output to match a specific schema
- Combine both for reliable, parseable CI/CD output

#### CLAUDE.md in CI Context

CLAUDE.md files are loaded in CI just as in interactive mode:

- Project-level CLAUDE.md provides build commands, test commands, project structure
- Ensures Claude has the same context in CI as developers have locally
- Can include CI-specific instructions (e.g., "In CI, do not modify files, only report")

#### Session Context Isolation

**Critical concept for CI/CD:**

- When Claude reviews code it just wrote **in the same session**, it is less effective at
  finding issues (confirmation bias in context)
- For code review in CI, use a **fresh session** (separate `claude -p` invocation)
- This ensures the reviewer has no prior context that might bias the review

#### Avoiding Duplicate Findings

When integrating Claude into PR review pipelines:

- **Include prior review findings** in the prompt to avoid duplicate comments
- Pass previous review comments as context: "Previous review already noted: [comments]. Find NEW issues only."
- **Provide existing test file paths** to prevent Claude from suggesting tests that already exist
- **Provide existing test files as context** so Claude does not suggest creating tests that
  are already covered. Example: `"Existing tests: $(find src/ -name '*.test.ts' | head -50)"`

#### Documenting Testing Standards in CLAUDE.md for CI

When Claude runs in CI for test generation or code review, it needs to know what constitutes
a **valuable** test vs. a redundant one. Document these criteria in your project-level CLAUDE.md:

```markdown
# CLAUDE.md -- Testing Standards for CI

## What Makes a Valuable Test
- Tests behavior and outcomes, not implementation details
- Covers error paths and edge cases, not just happy paths
- Tests integration points between modules (not just unit-level)
- Regression tests for previously reported bugs (reference the issue number)

## What to Avoid in Tests
- Do not duplicate existing test coverage (check __tests__/ directories first)
- Do not test trivial getters/setters
- Do not test framework behavior (e.g., React rendering basics)
- Do not create snapshot tests for frequently changing UI components

## Test Fixtures and Factories
- Shared test fixtures are in test/fixtures/
- Use the factory pattern in test/factories/ for generating test data
- Database seed data is in test/seeds/ -- do not duplicate it in individual tests
- Mock API responses are in test/mocks/ -- reuse existing mocks before creating new ones

## Test File Organization
- Unit tests: co-located as <module>.test.ts
- Integration tests: test/integration/
- E2E tests: test/e2e/ using Playwright
- Test utilities and helpers: test/utils/
```

This ensures that CI-invoked Claude Code writes tests that match the team's standards
and does not generate duplicate or low-value test suggestions.

### Practical Skills

**CI/CD Pipeline Example (GitHub Actions):**

```yaml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get diff
        run: git diff origin/main...HEAD > /tmp/pr-diff.txt

      - name: Get previous review comments
        id: prev-comments
        run: |
          gh pr view ${{ github.event.pull_request.number }} \
            --json comments -q '.comments[].body' > /tmp/prev-comments.txt

      - name: Claude Review
        run: |
          cat /tmp/pr-diff.txt | claude -p \
            --output-format json \
            --json-schema '{"type":"object","properties":{"comments":{"type":"array","items":{"type":"object","properties":{"file":{"type":"string"},"line":{"type":"number"},"comment":{"type":"string"},"severity":{"type":"string","enum":["critical","warning","suggestion"]}}}}}}' \
            "Review this diff. Previous comments already made: $(cat /tmp/prev-comments.txt). Only report NEW issues."
```

**Structured Output for Test Generation:**

```bash
claude -p --output-format json \
  --json-schema '{"type":"object","properties":{"test_file":{"type":"string"},"test_code":{"type":"string"}}}' \
  "Generate a test for src/utils/parser.ts. Existing tests: $(ls src/utils/__tests__/)"
```

### Configuration Example

```markdown
# CLAUDE.md (project-level, used in CI)

## CI/CD Context
- Build command: npm run build
- Test command: npm test
- Lint command: npm run lint
- When running in CI (non-interactive), do not attempt interactive commands
- Test files are in __tests__/ directories, co-located with source
- Existing test coverage report: coverage/lcov-report/index.html
```

### Common Exam Traps

- **Trap**: Thinking `-p` and `--print` are different flags. They are aliases for the same thing.
- **Trap**: Forgetting session context isolation. The exam may ask: "Why might Claude miss bugs in a CI review?" Answer: if it is reviewing code it wrote in the same session.
- **Trap**: Not including prior review findings. The exam may describe a scenario where Claude keeps posting duplicate review comments and ask how to fix it.
- **Trap**: Assuming `--output-format json` alone guarantees a specific structure. You need `--json-schema` to enforce a particular JSON shape.
- **Trap**: Forgetting that CLAUDE.md loads in CI too. If the question asks how to give Claude project context in a CI pipeline, the answer is CLAUDE.md (not command-line flags for every detail).
- **Trap**: Using `--json-schema` without `--output-format json`. The schema flag requires JSON output format to be enabled.

---

## Quick Reference: Key CLI Flags for CI/CD

| Flag | Purpose |
|------|---------|
| `-p` / `--print` | Non-interactive mode, output to stdout |
| `--output-format json` | Return structured JSON output |
| `--json-schema <schema>` | Constrain JSON output to a specific schema |
| `--model <model>` | Specify which Claude model to use |
| `--max-turns <n>` | Limit the number of agentic turns |
| `--allowedTools <tools>` | Restrict which tools Claude can use |

---

## Cross-Cutting Exam Themes for Domain 3

1. **Shared vs Personal**: `.claude/` directory = shared (git). `~/.claude/` = personal (local).
   This pattern is consistent across CLAUDE.md, commands, skills, and rules.

2. **Scope and Loading**: User-level = always. Project-level = always (in that project).
   Directory-level = only when working in that subtree. Path-scoped rules = only when
   editing matching files.

3. **Isolation**: `context: fork` in skills, explore sub-agent for investigation, fresh
   sessions for CI review -- all solve the same problem of keeping contexts clean and unbiased.

4. **Structured vs Unstructured**: Use `--output-format json` + `--json-schema` for CI/CD.
   Use prose for human-facing interactive sessions.

5. **Additive Configuration**: CLAUDE.md files merge, they do not override. Rules stack.
   This is different from many config systems where deeper configs override shallower ones.

---

## Practice Questions

**Q1**: A developer reports that coding standards defined in the project's CLAUDE.md are
not being applied when they use Claude Code. The `/memory` command shows only their
`~/.claude/CLAUDE.md`. What is the most likely cause?

**A**: The project CLAUDE.md is either not committed to git, the developer has not pulled
the latest changes, or the file is listed in `.gitignore`.

**Q2**: You want all Terraform files across your entire monorepo to follow specific
conventions, regardless of which directory they are in. What is the best approach?

**A**: Create a file in `.claude/rules/` with YAML frontmatter `paths: ["**/*.tf", "terraform/**/*"]`.
This applies the rules whenever Claude edits any matching file, regardless of directory.

**Q3**: Your CI pipeline runs Claude to generate code and then immediately uses Claude to
review that code. Reviews rarely find issues. How do you improve review quality?

**A**: Use a separate session (separate `claude -p` invocation) for the review step.
Same-session review suffers from context bias -- Claude is less critical of code it just wrote.

**Q4**: You need Claude Code to output a structured list of issues found during a code review
in CI, with each issue having a file path, line number, and severity. What flags do you use?

**A**: `claude -p --output-format json --json-schema '<schema>'` where the schema defines
the required structure with file, line, and severity fields.

**Q5**: A team wants some Claude Code prompts shared across the team and others kept personal.
Where should each type be placed?

**A**: Shared prompts go in `.claude/commands/` (invoked as `/project:name`). Personal prompts
go in `~/.claude/commands/` (invoked as `/user:name`). Same pattern for skills: `.claude/skills/`
vs `~/.claude/skills/`.

**Q6**: A monorepo has packages maintained by different teams. The payments team needs PCI
compliance rules, the frontend team needs accessibility rules, and neither team wants the
other's rules loaded. How do you configure this?

**A**: Use `@import` in each package's directory-level CLAUDE.md to selectively import only
the relevant standards files. For example, `packages/payments/CLAUDE.md` imports
`@../../.claude/standards/pci-compliance.md` while `packages/frontend-ui/CLAUDE.md` imports
`@../../.claude/standards/accessibility.md`. Each team only gets the standards relevant to
their domain.

**Q7**: You need to audit the security of a large module, but you do not want the verbose
file-reading output to consume your main conversation context. What is the best approach?

**A**: Create a skill with `context: fork` and `allowed-tools` restricted to Read, Grep, and
Glob. The skill runs in an isolated sub-agent, performs verbose investigation, and returns
only a summary to the main conversation. Alternatively, use the built-in explore sub-agent
for ad-hoc investigation.

**Q8**: Claude Code sometimes follows your testing conventions and sometimes ignores them.
How do you diagnose this?

**A**: Run `/memory` in both a working and non-working session to compare which files are
loaded. The testing rules may be in a path-scoped `.claude/rules/testing.md` that only loads
when editing test files, or in a directory-level CLAUDE.md that only loads in certain
subdirectories.

---

## Source Links

The following official Anthropic documentation pages cover the topics in this domain:

- [Claude Code Memory and CLAUDE.md](https://docs.anthropic.com/en/docs/claude-code/memory) -- CLAUDE.md hierarchy, @import syntax, .claude/rules/, /memory command
- [Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code/skills) -- SKILL.md frontmatter, context: fork, allowed-tools, argument-hint, custom commands
- [Claude Code CLI Usage](https://docs.anthropic.com/en/docs/claude-code/cli-usage) -- --print flag, --output-format json, --json-schema, plan mode, non-interactive mode
- [Claude Code Best Practices](https://docs.anthropic.com/en/docs/claude-code/best-practices) -- Iterative refinement, test-driven iteration, interview pattern, I/O examples
- [Claude Code Overview](https://docs.anthropic.com/en/docs/claude-code/overview) -- General Claude Code features and capabilities
- [Claude Code CI/CD Integration](https://docs.anthropic.com/en/docs/claude-code/ci-cd) -- CI/CD pipeline setup, session isolation, structured output
- [Claude Code GitHub Actions](https://docs.anthropic.com/en/docs/claude-code/github-actions) -- GitHub Actions integration examples
