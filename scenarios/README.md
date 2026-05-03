# Exam Scenarios Reference

The exam uses 6 scenarios (4 are randomly selected per exam session). Each scenario frames a set of questions.

## Scenario 1: Customer Support Resolution Agent

You are building a customer support resolution agent using the Claude Agent SDK. The agent handles high-ambiguity requests like returns, billing disputes, and account issues. It has access to backend systems through custom MCP tools (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`). Target: 80%+ first-contact resolution while knowing when to escalate.

**Primary domains:** Domain 1, Domain 2, Domain 5

**Key topics tested:**
- Programmatic prerequisite enforcement (block refund until customer verified)
- Tool description quality for reliable selection
- Escalation criteria with few-shot examples
- Structured handoff summaries for human agents
- Hook-based compliance enforcement (block refunds above threshold)
- Multi-concern request decomposition

---

## Scenario 2: Code Generation with Claude Code

You are using Claude Code to accelerate software development. Your team uses it for code generation, refactoring, debugging, and documentation. Need to integrate it into development workflow with custom slash commands, CLAUDE.md configurations, and understand when to use plan mode vs direct execution.

**Primary domains:** Domain 3, Domain 5

**Key topics tested:**
- CLAUDE.md hierarchy (user vs project vs directory)
- .claude/rules/ with glob patterns for path-specific conventions
- Custom slash commands (.claude/commands/) vs skills (.claude/skills/)
- Plan mode for architectural decisions vs direct execution for simple changes
- context: fork, allowed-tools, argument-hint in SKILL.md
- @import syntax for modular CLAUDE.md

---

## Scenario 3: Multi-Agent Research System

You are building a multi-agent research system using the Claude Agent SDK. A coordinator agent delegates to specialized subagents: one searches the web, one analyzes documents, one synthesizes findings, one generates reports. The system researches topics and produces comprehensive, cited reports.

**Primary domains:** Domain 1, Domain 2, Domain 5

**Key topics tested:**
- Hub-and-spoke coordinator pattern
- Task decomposition breadth (avoiding overly narrow subtasks)
- Subagent context passing (explicit, not inherited)
- Parallel subagent spawning (multiple Task calls in single response)
- Error propagation with structured context
- Source attribution / claim-source mappings in synthesis
- Scoped tool access per agent (verify_fact tool for synthesis)
- Coverage gap annotation

---

## Scenario 4: Developer Productivity with Claude

You are building developer productivity tools using the Claude Agent SDK. The agent helps engineers explore unfamiliar codebases, understand legacy systems, generate boilerplate, and automate repetitive tasks. Uses built-in tools (Read, Write, Bash, Grep, Glob) and integrates with MCP servers.

**Primary domains:** Domain 2, Domain 3, Domain 1

**Key topics tested:**
- Built-in tool selection (Grep for content, Glob for file paths)
- Incremental codebase understanding (Grep → Read → follow imports)
- MCP server integration (.mcp.json with env vars)
- Subagent delegation for verbose exploration
- Scratchpad files for persisting findings
- Context degradation in extended sessions
- /compact for context management

---

## Scenario 5: Claude Code for Continuous Integration

You are integrating Claude Code into your CI/CD pipeline. The system runs automated code reviews, generates test cases, and provides feedback on pull requests. Design prompts that provide actionable feedback and minimize false positives.

**Primary domains:** Domain 3, Domain 4

**Key topics tested:**
- `-p` (`--print`) flag for non-interactive CI mode
- `--output-format json` and `--json-schema` for structured output
- Session context isolation (separate sessions for generation vs review)
- Explicit review criteria to reduce false positives
- Few-shot examples for consistent review output format
- Multi-pass review (per-file local + cross-file integration)
- Including prior review findings to avoid duplicate comments
- Batch API for non-blocking workflows (overnight reports) vs real-time for blocking (pre-merge)

---

## Scenario 6: Structured Data Extraction

You are building a structured data extraction system using Claude. The system extracts information from unstructured documents, validates output using JSON schemas, and maintains high accuracy. Must handle edge cases and integrate with downstream systems.

**Primary domains:** Domain 4, Domain 5

**Key topics tested:**
- tool_use with JSON schemas for guaranteed schema compliance
- tool_choice: "auto" vs "any" vs forced tool selection
- Schema design: nullable fields to prevent hallucination, enum + "other" + detail
- Validation-retry loops with specific error feedback
- When retries are ineffective (info absent from source)
- Few-shot examples for varied document structures
- Batch processing with Message Batches API (custom_id, failure handling)
- Human review routing with field-level confidence scores
- Stratified sampling for measuring error rates
- Accuracy by document type and field before automating
