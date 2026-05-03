# External Practice Questions (ClaudeCertifications.com)

25 free questions with correct answers. Use for quick self-check on key concepts.

## Domain 1: Agentic Architecture (7 questions)

**Q1.** How should the agentic loop determine when to stop?
- **Correct:** Check `stop_reason` field — `"end_turn"` means done, `"tool_use"` means continue

**Q2.** How should a coordinator pass context to subagents?
- **Correct:** Pass explicit, relevant context — subagents do NOT inherit parent context

**Q3.** How to enforce a $500 refund limit deterministically?
- **Correct:** PostToolUse hook (programmatic enforcement, not prompt instructions)

**Q13.** How to explore an alternative approach without affecting the current session?
- **Correct:** `fork_session` — creates independent branch from shared baseline

**Q17.** What context to pass to a market analysis subagent?
- **Correct:** Only the specific research question + relevant parameters (minimal necessary context)

**Q23.** How to ensure an agent NEVER reveals pricing formulas?
- **Correct:** PreToolUse hook to filter/redact sensitive data before it reaches the model

**Q24 (from D3).** Most cost-effective approach for nightly code quality audits?
- **Correct:** Message Batches API (50% savings, non-blocking, up to 24h SLA acceptable for nightly runs)

## Domain 2: Tool Design & MCP (4 questions)

**Q4.** Agent has 18 tools, frequently selects the wrong one. Fix?
- **Correct:** Reduce to 4-5 role-specific tools per agent (scoped `allowedTools`)

**Q5.** How should an MCP tool report an API key expired error?
- **Correct:** Structured error with `isError: true`, error category, `isRetryable` flag

**Q14.** How to store a Jira API token for an MCP server?
- **Correct:** `${JIRA_TOKEN}` env var expansion in `.mcp.json` (never hardcode secrets)

**Q18.** Which tool to use for modifying line 42 of a TypeScript file?
- **Correct:** Edit tool (not Write, Bash/sed, or Read)

## Domain 3: Claude Code Configuration (5 questions)

**Q6.** How to share team-wide coding standards?
- **Correct:** `.claude/CLAUDE.md` at project level (version-controlled, shared via VCS)

**Q7.** How to integrate Claude Code into CI/CD?
- **Correct:** `-p` flag for non-interactive + `--output-format json` for structured output

**Q15.** Where to put personal terminal color scheme preferences?
- **Correct:** `~/.claude/CLAUDE.md` user-level (personal, not shared with team)

**Q19.** How to create a reusable, isolated refactoring behavior?
- **Correct:** Skill with `context: fork` and `allowed-tools` in frontmatter

## Domain 4: Prompt Engineering & Structured Output (5 questions)

**Q8.** How to reduce false positives in automated code review?
- **Correct:** Explicit, measurable criteria (not vague "be careful" instructions)

**Q9.** How to guarantee JSON structure in Claude's output?
- **Correct:** `tool_use` with JSON schema (NOT json_mode — Anthropic has no json_mode)

**Q16.** How to classify sarcastic product reviews correctly?
- **Correct:** 2-4 few-shot examples including edge cases (sarcasm, mixed sentiment)

**Q20.** tool_use guarantees JSON structure — do you still need validation?
- **Correct:** Yes — tool_use guarantees STRUCTURE only, NOT semantic correctness. Still need validation for content accuracy.

**Q25.** Validation-retry loop keeps failing on same error. Fix?
- **Correct:** Replace generic retry message with SPECIFIC error details (what failed, where, expected vs actual)

## Domain 5: Context Management & Reliability (4 questions)

**Q10.** Long session quality degradation. Fix?
- **Correct:** `/compact` + scratchpad files + delegate verbose work to subagents

**Q11.** 95% overall accuracy but poor performance on invoices. How to diagnose?
- **Correct:** Track per-document-type metrics (stratified accuracy, not just aggregate)

**Q12.** Subagent gets API permission error. How to report to coordinator?
- **Correct:** Structured error distinguishing "couldn't check" (access failure) vs "found nothing" (valid empty)

**Q22.** Two data sources show conflicting revenue figures. What should the agent do?
- **Correct:** Information provenance — trust the verified/authoritative source (e.g., DB over scraped report)
