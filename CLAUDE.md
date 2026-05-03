# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

This repository contains study materials and exam preparation tools for the **Claude Certified Architect — Foundations** certification exam.

## Structure

- `study-materials/domain-{1..5}-*/` — study guides with key concepts, code examples, and exam traps per domain
- `scenarios/` — reference material for the 6 exam scenarios
- `.claude/skills/exam/` — the `/exam` skill for interactive practice (drill, test, stats modes)

## Exam Skill Routing

**CRITICAL:** When the user asks for exam practice in ANY form — formal or informal — ALWAYS invoke the `/exam` skill with the correct parameters. NEVER generate exam questions manually.

Routing rules for informal requests (English and other languages alike):
- "mock exam", "another test exam", "trial exam", "practice exam", "full exam" → `/exam test 15`
- "drill", "practice", "question on domain 2", "quiz me" → `/exam drill [domain-N|weak]`
- "show stats", "progress", "how am I doing" → `/exam stats`
- "hard exam", "harder", "focus on weak areas" → `/exam test 15` (difficulty is adaptive from redis stats)
- If the user specifies a question count, use it; otherwise default is 15

```
/exam drill              # one question, adaptive topic selection
/exam drill domain-2     # focus on specific domain
/exam drill weak         # focus on weakest areas from redis stats
/exam drill weak 5       # series of 5 questions on weak areas (reads redis once)
/exam drill domain-3 10  # series of 10 questions on domain 3
/exam test               # simulate exam (15 questions)
/exam test 20            # simulate exam (20 questions)
/exam stats              # show progress and recommendations
```

## Required Plugins

The `/exam` skill stores progress in a persistent memory layer via MCP tools. Install **at least one** of the following Claude Code plugins before running the skill:

- **redis-memory** (required for full `/exam` functionality) — provides the `mcp__plugin_redis-memory_redis-memory-mcp__*` tools (`kv_get`, `kv_set`, `kv_list`, `mem_save`, `mem_search`, `mem_list`) used by the skill for stats and semantic notes. Repository: https://github.com/sergesha/redis-memory-mcp
- **mempalace** (recommended companion) — adds a structured "memory palace" for organizing study notes and cross-session knowledge. Repository: https://github.com/mempalace/mempalace

If neither plugin is installed, `/exam` will not be able to track progress between sessions. See `README.md` for installation instructions.

## Redis Memory Schema

Stats are stored via the redis-memory MCP plugin:
- **KV** (`exam:stats:*`) — structured counters for domains, tasks, overall progress
- **MEM** — semantic notes about knowledge gaps and strengths

## Exam Domains & Weights

1. Agentic Architecture & Orchestration — **27%**
2. Tool Design & MCP Integration — **18%**
3. Claude Code Configuration & Workflows — **20%**
4. Prompt Engineering & Structured Output — **20%**
5. Context Management & Reliability — **15%**

Passing score: 720/1000 (72%). All questions are multiple choice with 1 correct answer from 4 options.

## Exam Format Details

- **60 questions**, 120 minutes, live proctored (no Claude/docs/tools allowed during exam)
- **4 of 6 scenarios** randomly selected per sitting — must know ALL 6
- Fee: $99 USD (first 5,000 partner employees free)
- Results within 2 business days
- Access requires Claude Partner Network membership (free for organizations)

## Core Mental Models (Drive Most Answers)

1. **Programmatic enforcement > prompt guidance** — when something MUST happen, use hooks/gates, not prompts
2. **Tool descriptions drive selection** — not system prompts or function names
3. **Subagents do NOT inherit context** — everything must be explicitly passed
4. **"Lost in the middle" effect** — key info at top/bottom of context, not middle
5. **Batch API = offline (50% savings, up to 24h, no tool loop) vs Real-time API = interactive (tool calling, user-facing)**

## 7 Anti-Patterns (Appear as Wrong Answers)

1. Few-shot examples for tool ordering → use programmatic prerequisites
2. Self-reported LLM confidence scores for routing → use structured schemas
3. Batch API for interactive workflows → Batch has no SLA, no tool loop
4. Larger context window solves attention → it does NOT
5. Empty results on subagent failures → return structured error context
6. 18+ tools per agent → reduce to 4-5 role-specific
7. Prompt-only validation → use JSON schema + retry loops

## Content Style

When generating study material summaries or cheat sheets, provide detailed context — not compressed tables. Include explanations, analogies, code examples, and reasoning. Think "study guide section", not "flash card".

## Memory Policy

Do NOT save to auto-memory anything that is already in CLAUDE.md, SKILL.md, or derivable from the codebase. Before creating a memory file, check: is this already codified? If yes — don't duplicate it. Memory is only for information that has no other persistent home (e.g., user context not related to this project).
