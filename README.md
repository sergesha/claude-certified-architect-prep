# Claude Certified Architect — Foundations: Study Materials & Exam Skill

A self-study kit for the **Claude Certified Architect — Foundations** certification exam, plus a custom Claude Code skill (`/exam`) that drills you on the material with adaptive difficulty and persistent progress tracking.

## What's Inside

```
.
├── CLAUDE.md                      # Project instructions for Claude Code
├── .claude/
│   ├── skills/exam/SKILL.md       # The /exam skill (drill / test / stats modes)
│   └── settings.json        # Local permissions allowlist
├── study-materials/
│   ├── domain-1-agentic-architecture/   # 27% — Agentic loops, multi-agent orchestration
│   ├── domain-2-tool-design-mcp/        # 18% — Tool design & MCP integration
│   ├── domain-3-claude-code-config/     # 20% — Claude Code configuration & workflows
│   ├── domain-4-prompt-engineering/     # 20% — Prompt engineering & structured output
│   ├── domain-5-context-reliability/    # 15% — Context management & reliability
│   ├── external-resources.md            # Curated external reading list
│   └── external-practice-questions.md   # Public practice question banks
└── scenarios/
    └── README.md                  # Reference summaries for the 6 exam scenarios
```

Each `domain-*/README.md` is a long-form study guide with key concepts, code examples, and common exam traps. Sub-directories under `references/` contain authoritative source material distilled from Anthropic docs.

## Exam Quick Facts

- **5 domains, 6 scenarios** (4 of 6 selected randomly per sitting — know all 6)
- **60 questions, 120 minutes**, multiple choice, 1 correct answer of 4
- **Pass: 720/1000 (72%)**
- **Fee: $99 USD** (free for first 5,000 partner employees)
- Live proctored — no Claude, docs, or tools allowed during the exam

## Official Exam Guide — Where to Get It

The authoritative exam guide is published by **Anthropic, PBC** and distributed to candidates who register for the exam. It is not bundled in this repository — request your own copy through the official channels:

- **Anthropic Academy (Skilljar):** https://anthropic.skilljar.com/ — free self-paced preparation courses covering all five domains
- **Exam access request:** https://anthropic.skilljar.com/claude-certified-architect-foundations-access-request
- **Claude Partner Network membership** — required for the certification; free for partner organizations

The notes in `study-materials/` and `scenarios/` were prepared with the official guide as a reference, alongside public Anthropic documentation, the Agent SDK docs, the MCP specification, and Claude Code docs.

## Prerequisites

- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed locally
- One of the supported memory plugins (see below) — without it, the `/exam` skill cannot persist progress

## Required Plugins

The `/exam` skill writes per-domain accuracy, task-level stats, and semantic notes about your knowledge gaps to a persistent store via MCP tools. Install at least the **redis-memory** plugin; **mempalace** is a recommended companion for organizing study notes.

Both plugins are installed through Claude Code's plugin marketplace by passing the GitHub `owner/repo` shortname — no manual cloning needed. The only prerequisite is a working **Docker** daemon: the redis-memory plugin ships with a `docker-compose.yaml` that brings up Redis Stack + an embeddings service alongside the MCP server, so you don't need to install or run Redis yourself.

### 1. redis-memory (required)

Provides the MCP tools `mcp__plugin_redis-memory_redis-memory-mcp__{kv_get, kv_set, kv_list, mem_save, mem_search, mem_list}` used by the skill.

**Repository:** https://github.com/sergesha/redis-memory-mcp

Inside Claude Code, run:

```
/plugin marketplace add sergesha/redis-memory-mcp
/plugin install redis-memory-mcp
```

Make sure Docker is running. The plugin's docker-compose stack (Redis Stack + embeddings + MCP server) handles everything else — see the plugin repo for env vars and troubleshooting if the stack doesn't come up automatically.

### 2. mempalace (recommended)

A "memory palace" plugin for organizing notes across sessions. Useful if you want long-form qualitative reflections in addition to the numeric stats stored by redis-memory.

**Repository:** https://github.com/mempalace/mempalace

Inside Claude Code, run:

```
/plugin marketplace add mempalace/mempalace
/plugin install mempalace
```

### Verifying the plugins are active

Run `/mcp` inside Claude Code to confirm the tools appear. The tool names referenced in `.claude/skills/exam/SKILL.md` and `.claude/settings.json` are `mcp__plugin_redis-memory_redis-memory-mcp__*` and `mcp__plugin_mempalace_mempalace__*` — they must match what `/mcp` shows. If your install produced different namespaces, update those two files accordingly.

## Getting Started

1. **Clone the repo:**
   ```bash
   git clone https://github.com/sergesha/claude-certified-architect-prep.git
   cd claude-certified-architect-prep
   ```

2. **Install the plugins** (see [Required Plugins](#required-plugins) above).

3. **Open the project in Claude Code:**
   ```bash
   claude
   ```

4. **Try the skill:**
   ```
   /exam drill              # one adaptive question
   /exam drill domain-2     # focus on a specific domain
   /exam drill weak 5       # 5-question series on your weakest areas
   /exam test 15            # full simulated exam (15 questions)
   /exam stats              # show progress and recommendations
   ```

5. **Read the study material** in `study-materials/` alongside drilling. The skill pulls from these files when generating questions, so you can read the same source it uses.

## How the `/exam` Skill Works

- **`drill`** — interactive questions, one at a time, with immediate feedback. Adaptive: it reads your stats from redis and biases toward weak domains.
- **`test`** — simulated exam (default 15 questions, weighted by domain percentages). All answers collected first, then full review at the end.
- **`stats`** — per-domain and per-task accuracy with bar charts, weakest/strongest areas, and recommendations.

After every question, the skill delegates a background agent to update your stats in redis (KV counters + semantic notes). Questions you've already seen are tracked semantically so the skill avoids repeats and tests the same concept from new angles.

See `.claude/skills/exam/SKILL.md` for the full spec.

## Project Layout Conventions

- Long-form, narrative study material — not flash cards. Each domain README is meant to be read end-to-end at least once.
- Code examples throughout illustrate Claude Agent SDK patterns, MCP tool schemas, and prompt engineering techniques.
- "Exam traps" sections call out the common wrong answers and the reasoning behind the right one.

## Contributing / Forking

This is a personal study kit, but the structure is reusable. To adapt it:

1. Fork the repo.
2. Replace `study-materials/` with your own domain READMEs.
3. Update `CLAUDE.md` with your routing rules, domain weights, and scenario list.
4. Adjust `.claude/skills/exam/SKILL.md` if your scoring rubric or modes differ.

## License

The original study material, the `/exam` skill, and all repository scaffolding are released under the **MIT License** — see [`LICENSE`](LICENSE). You may copy, modify, and redistribute them freely with attribution.

This license **does not** cover:

- The official Anthropic exam guide PDF — © Anthropic, PBC; not bundled here, fetch your own copy from Anthropic Academy.
- Anthropic's documentation excerpts paraphrased in `study-materials/*/references/` — those remain under Anthropic's documentation terms. Use them for personal study.
- The "Claude" name, logos, and trademarks — owned by Anthropic, PBC.
