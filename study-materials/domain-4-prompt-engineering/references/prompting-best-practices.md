# Prompting Best Practices — Official Anthropic Guide
Source: https://platform.claude.com/docs/en/docs/build-with-claude/prompt-engineering/claude-prompting-best-practices
Fetched: 2026-03-21

---

## 1. General Principles (maps to D4 Task 4.1, 4.2)

### Be Clear and Direct

Claude responds best to explicit instructions. Think of Claude as a brilliant but new employee who lacks context on your norms and workflows.

**Golden rule:** Show your prompt to a colleague with minimal context. If they'd be confused, Claude will be too.

Key practices:
- Be specific about desired output format and constraints
- Provide instructions as sequential steps using numbered lists or bullet points when order matters
- If you want "above and beyond" behavior, explicitly request it rather than relying on inference

**Example — Creating an analytics dashboard:**

Less effective:
```text
Create an analytics dashboard
```

More effective:
```text
Create an analytics dashboard. Include as many relevant features and interactions as possible. Go beyond the basics to create a fully-featured implementation.
```

### Add Context to Improve Performance

Explain WHY behind instructions. Claude generalizes from motivating explanations.

Less effective:
```text
NEVER use ellipses
```

More effective:
```text
Your response will be read aloud by a text-to-speech engine, so never use ellipses since the text-to-speech engine will not know how to pronounce them.
```

### Use Examples Effectively (Few-Shot / Multishot Prompting)

**3-5 examples for best results.** Make them:
- **Relevant:** Mirror your actual use case closely
- **Diverse:** Cover edge cases, vary enough so Claude doesn't pick up unintended patterns
- **Structured:** Wrap in `<example>` tags (multiple in `<examples>` tags) so Claude distinguishes them from instructions

> You can also ask Claude to evaluate your examples for relevance and diversity, or to generate additional ones.

### Structure Prompts with XML Tags

XML tags help Claude parse complex prompts unambiguously. Wrap each type of content in its own tag:
- `<instructions>` — the task directives
- `<context>` — background information
- `<input>` — variable user data
- `<example>` / `<examples>` — few-shot demonstrations

Best practices:
- Use consistent, descriptive tag names across your prompts
- Nest tags when content has a natural hierarchy (e.g., documents inside `<documents>`, each inside `<document index="n">`)

### Give Claude a Role via System Prompt

Even a single sentence focuses behavior and tone:

```python
import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    system="You are a helpful coding assistant specializing in Python.",
    messages=[
        {"role": "user", "content": "How do I sort a list of dictionaries by key?"}
    ],
)
print(message.content)
```

### Model Self-Knowledge

For correct self-identification:
```text
The assistant is Claude, created by Anthropic. The current model is Claude Opus 4.6.
```

For LLM-powered apps needing model strings:
```text
When an LLM is needed, please default to Claude Opus 4.6 unless the user requests otherwise. The exact model string for Claude Opus 4.6 is claude-opus-4-6.
```

---

## 2. Long Context Prompting (maps to D5 Task 5.1)

### Put Longform Data at the TOP of the Prompt

Place long documents and inputs near the top, above your query, instructions, and examples.

> **KEY EXAM FACT:** Queries at the end can improve response quality by up to 30% in tests, especially with complex, multi-document inputs.

### Structure Document Content with XML Tags

When using multiple documents, wrap each in `<document>` tags with metadata subtags:

```xml
<documents>
  <document index="1">
    <source>annual_report_2023.pdf</source>
    <document_content>
      {{ANNUAL_REPORT}}
    </document_content>
  </document>
  <document index="2">
    <source>competitor_analysis_q2.xlsx</source>
    <document_content>
      {{COMPETITOR_ANALYSIS}}
    </document_content>
  </document>
</documents>

Analyze the annual report and competitor analysis. Identify strategic advantages and recommend Q3 focus areas.
```

### Ground Responses in Quotes from Source Documents

For long document tasks, ask Claude to quote relevant parts first before carrying out the task. This helps Claude cut through noise.

```xml
You are an AI physician's assistant. Your task is to help doctors diagnose possible patient illnesses.

<documents>
  <document index="1">
    <source>patient_symptoms.txt</source>
    <document_content>
      {{PATIENT_SYMPTOMS}}
    </document_content>
  </document>
  <document index="2">
    <source>patient_records.txt</source>
    <document_content>
      {{PATIENT_RECORDS}}
    </document_content>
  </document>
  <document index="3">
    <source>patient01_appt_history.txt</source>
    <document_content>
      {{PATIENT01_APPOINTMENT_HISTORY}}
    </document_content>
  </document>
</documents>

Find quotes from the patient records and appointment history that are relevant to diagnosing the patient's reported symptoms. Place these in <quotes> tags. Then, based on these quotes, list all information that would help the doctor diagnose the patient's symptoms. Place your diagnostic information in <info> tags.
```

### Lost-in-the-Middle Mitigation

The XML document structure + query-at-bottom pattern mitigates the "lost in the middle" problem common with long contexts.

---

## 3. Tool Use Best Practices (maps to D2 Task 2.1, 2.3)

### Be Explicit About Tool Usage

Claude may suggest changes instead of implementing them if your language is ambiguous.

Less effective (Claude will only suggest):
```text
Can you suggest some changes to improve this function?
```

More effective (Claude will make the changes):
```text
Change this function to improve its performance.
```

Or:
```text
Make these edits to the authentication flow.
```

### Prompt Pattern: Default to Action

To make Claude more proactive about taking action:
```text
<default_to_action>
By default, implement changes rather than only suggesting them. If the user's intent is unclear, infer the most useful likely action and proceed, using tools to discover any missing details instead of guessing. Try to infer the user's intent about whether a tool call (e.g., file edit or read) is intended or not, and act accordingly.
</default_to_action>
```

To make Claude more conservative (only act if explicitly asked):
```text
<do_not_act_before_instructions>
Do not jump into implementatation or changes files unless clearly instructed to make changes. When the user's intent is ambiguous, default to providing information, doing research, and providing recommendations rather than taking action. Only proceed with edits, modifications, or implementations when the user explicitly requests them.
</do_not_act_before_instructions>
```

### Claude 4.6 May Overtrigger on Tools

> **KEY EXAM FACT:** Claude Opus 4.5 and Claude Opus 4.6 are more responsive to the system prompt than previous models. If your prompts were designed to reduce undertriggering on tools, these models may now **overtrigger**. The fix is to dial back aggressive language. Where you might have said "CRITICAL: You MUST use this tool when...", use more normal prompting like "Use this tool when...".

### Parallel Tool Calling Optimization

Claude's latest models excel at parallel tool execution:
- Run multiple speculative searches during research
- Read several files at once to build context faster
- Execute bash commands in parallel

**Prompt pattern for maximum parallel efficiency (~100% parallel rate):**
```text
<use_parallel_tool_calls>
If you intend to call multiple tools and there are no dependencies between the tool calls, make all of the independent tool calls in parallel. Prioritize calling tools simultaneously whenever the actions can be done in parallel rather than sequentially. For example, when reading 3 files, run 3 tool calls in parallel to read all 3 files into context at the same time. Maximize use of parallel tool calls where possible to increase speed and efficiency. However, if some tool calls depend on previous calls to inform dependent values like the parameters, do NOT call these tools in parallel and instead call them sequentially. Never use placeholders or guess missing parameters in tool calls.
</use_parallel_tool_calls>
```

To reduce parallel execution:
```text
Execute operations sequentially with brief pauses between each step to ensure stability.
```

---

## 4. Thinking and Reasoning (relevant background)

### Adaptive Thinking (Claude 4.6)

Claude Opus 4.6 uses adaptive thinking (`thinking: {type: "adaptive"}`), where Claude dynamically decides when and how much to think. Calibrated by two factors:
1. The `effort` parameter (high, medium, low, max)
2. Query complexity

> **KEY EXAM FACT:** In internal evaluations, adaptive thinking reliably drives better performance than extended thinking.

```python
# Adaptive thinking (Claude 4.6)
client.messages.create(
    model="claude-opus-4-6",
    max_tokens=64000,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},  # or max, medium, low
    messages=[{"role": "user", "content": "..."}],
)
```

### Extended Thinking with budget_tokens (Older Models)

```python
# Extended thinking (older models)
client.messages.create(
    model="claude-sonnet-4-5-20250929",
    max_tokens=64000,
    thinking={"type": "enabled", "budget_tokens": 32000},
    messages=[{"role": "user", "content": "..."}],
)
```

For Claude Sonnet 4.6, switching from adaptive to extended thinking with a `budget_tokens` cap provides a hard ceiling on thinking costs while preserving quality.

### Guiding Thinking Behavior

```text
After receiving tool results, carefully reflect on their quality and determine optimal next steps before proceeding. Use your thinking to plan and iterate based on this new information, and then take the best next action.
```

To reduce excessive thinking:
```text
Extended thinking adds latency and should only be used when it will meaningfully improve answer quality - typically for problems that require multi-step reasoning. When in doubt, respond directly.
```

### Overthinking Mitigation (Claude 4.6)

Claude Opus 4.6 does significantly more upfront exploration than previous models. To mitigate:
- Replace blanket defaults ("Default to using [tool]") with targeted instructions ("Use [tool] when it would enhance your understanding")
- Remove over-prompting — tools that undertriggered before are likely to trigger appropriately now
- Use lower `effort` setting as a fallback

```text
When you're deciding how to approach a problem, choose an approach and commit to it. Avoid revisiting decisions unless you encounter new information that directly contradicts your reasoning. If you're weighing two approaches, pick one and see it through. You can always course-correct later if the chosen approach fails.
```

### Self-Check Pattern

> **KEY EXAM FACT:** "Before you finish, verify your answer against [test criteria]." — This catches errors reliably, especially for coding and math.

### Additional Thinking Tips

- **Prefer general instructions over prescriptive steps.** "Think thoroughly" often produces better reasoning than hand-written step-by-step plans
- **Multishot examples work with thinking.** Use `<thinking>` tags inside few-shot examples to show reasoning patterns
- **Manual CoT as fallback.** When thinking is off, use `<thinking>` and `<answer>` tags to separate reasoning from output
- **Note:** When extended thinking is disabled, Claude Opus 4.5 is particularly sensitive to the word "think" — use alternatives like "consider," "evaluate," or "reason through"

---

## 5. Agentic Systems (maps to D1 Tasks 1.2, 1.4, 1.6, 1.7; D5 Task 5.4)

### Long-Horizon Reasoning and State Tracking

Claude maintains orientation across extended sessions by focusing on incremental progress — making steady advances on a few things at a time rather than attempting everything at once.

### Context Awareness and Multi-Window Workflows

Claude 4.6 and Claude 4.5 models feature **context awareness** — the model tracks its remaining context window ("token budget") throughout a conversation.

**Prompt pattern for context management:**
```text
Your context window will be automatically compacted as it approaches its limit, allowing you to continue working indefinitely from where you left off. Therefore, do not stop tasks early due to token budget concerns. As you approach your token budget limit, save your current progress and state to memory before the context window refreshes. Always be as persistent and autonomous as possible and complete tasks fully, even if the end of your budget is approaching. Never artificially stop any task early regardless of the context remaining.
```

> The **memory tool** pairs naturally with context awareness for seamless context transitions.

### Multi-Context Window Workflows

1. **Use a different prompt for the first context window:** Set up framework (write tests, create setup scripts), then use future windows to iterate on a todo-list

2. **Have the model write tests in a structured format:** Ask Claude to create tests before starting work, keep track in structured format (e.g., `tests.json`). Remind Claude: "It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality."

3. **Set up quality of life tools:** Encourage Claude to create setup scripts (e.g., `init.sh`) to gracefully start servers, run test suites, and linters. Prevents repeated work when continuing from a fresh context window.

4. **Starting fresh vs compacting:** When a context window is cleared, consider starting with a brand new context window rather than using compaction. Claude's latest models are extremely effective at **discovering state from the local filesystem**. Be prescriptive about how it should start:
   - "Call pwd; you can only read and write files in this directory."
   - "Review progress.txt, tests.json, and the git logs."
   - "Manually run through a fundamental integration test before moving on to implementing new features."

5. **Provide verification tools:** As autonomous task length grows, Claude needs to verify correctness without continuous human feedback (e.g., Playwright MCP server, computer use).

6. **Encourage complete usage of context:**
```text
This is a very long task, so it may be beneficial to plan out your work clearly. It's encouraged to spend your entire output context working on the task - just make sure you don't run out of context with significant uncommitted work. Continue working systematically until you have completed this task.
```

### State Management Best Practices

- **Structured formats for state data:** JSON or other structured formats for test results, task status
- **Unstructured text for progress notes:** Freeform notes for tracking general progress
- **Git for state tracking:** Provides log of what's been done + checkpoints that can be restored
- **Emphasize incremental progress:** Explicitly ask Claude to track progress and focus on incremental work

**Example — Structured state file (tests.json):**
```json
{
  "tests": [
    { "id": 1, "name": "authentication_flow", "status": "passing" },
    { "id": 2, "name": "user_management", "status": "failing" },
    { "id": 3, "name": "api_endpoints", "status": "not_started" }
  ],
  "total": 200,
  "passing": 150,
  "failing": 25,
  "not_started": 25
}
```

**Example — Progress notes (progress.txt):**
```text
Session 3 progress:
- Fixed authentication token validation
- Updated user model to handle edge cases
- Next: investigate user_management test failures (test #2)
- Note: Do not remove tests as this could lead to missing functionality
```

### Subagent Orchestration

Claude's latest models can recognize when tasks benefit from delegating to specialized subagents and do so proactively.

Key behaviors:
1. **Ensure well-defined subagent tools** in tool definitions
2. **Let Claude orchestrate naturally** — Claude will delegate without explicit instruction
3. **Watch for overuse** — Claude Opus 4.6 has a strong predilection for subagents and may spawn them when a simpler approach suffices (e.g., spawning subagents for code exploration when a direct grep call is faster)

**Prompt pattern to control subagent usage:**
```text
Use subagents when tasks can run in parallel, require isolated context, or involve independent workstreams that don't need to share state. For simple tasks, sequential operations, single-file edits, or tasks where you need to maintain context across steps, work directly rather than delegating.
```

### Prompt Chaining — Self-Correction Pattern

The most common chaining pattern: **generate -> review -> refine**

Each step is a separate API call so you can log, evaluate, or branch at any point.

> With adaptive thinking and subagent orchestration, Claude handles most multi-step reasoning internally. Explicit prompt chaining is still useful when you need to inspect intermediate outputs or enforce a specific pipeline structure.

### Scratchpad Files for State Persistence

Claude may create new files for testing and iteration — using files (especially python scripts) as a "temporary scratchpad."

To minimize net new file creation:
```text
If you create any temporary new files, scripts, or helper files for iteration, clean up these files by removing them at the end of the task.
```

---

## 6. Safety and Quality (maps to D1 Task 1.4)

### Balancing Autonomy and Safety — Confirm Before Destructive Actions

Without guidance, Claude Opus 4.6 may take actions that are difficult to reverse or affect shared systems.

**Prompt pattern:**
```text
Consider the reversibility and potential impact of your actions. You are encouraged to take local, reversible actions like editing files or running tests, but for actions that are hard to reverse, affect shared systems, or could be destructive, ask the user before proceeding.

Examples of actions that warrant confirmation:
- Destructive operations: deleting files or branches, dropping database tables, rm -rf
- Hard to reverse operations: git push --force, git reset --hard, amending published commits
- Operations visible to others: pushing code, commenting on PRs/issues, sending messages, modifying shared infrastructure

When encountering obstacles, do not use destructive actions as a shortcut. For example, don't bypass safety checks (e.g. --no-verify) or discard unfamiliar files that may be in-progress work.
```

### Minimizing Hallucinations — investigate_before_answering Pattern

```text
<investigate_before_answering>
Never speculate about code you have not opened. If the user references a specific file, you MUST read the file before answering. Make sure to investigate and read relevant files BEFORE answering questions about the codebase. Never make any claims about code before investigating unless you are certain of the correct answer - give grounded and hallucination-free answers.
</investigate_before_answering>
```

### Avoid Over-Engineering

Claude Opus 4.5 and Claude Opus 4.6 tend to overengineer — creating extra files, adding unnecessary abstractions, or building in flexibility that wasn't requested.

**Prompt pattern to minimize overengineering:**
```text
Avoid over-engineering. Only make changes that are directly requested or clearly necessary. Keep solutions simple and focused:

- Scope: Don't add features, refactor code, or make "improvements" beyond what was asked. A bug fix doesn't need surrounding code cleaned up. A simple feature doesn't need extra configurability.

- Documentation: Don't add docstrings, comments, or type annotations to code you didn't change. Only add comments where the logic isn't self-evident.

- Defensive coding: Don't add error handling, fallbacks, or validation for scenarios that can't happen. Trust internal code and framework guarantees. Only validate at system boundaries (user input, external APIs).

- Abstractions: Don't create helpers, utilities, or abstractions for one-time operations. Don't design for hypothetical future requirements. The right amount of complexity is the minimum needed for the current task.
```

### Avoid Hard-Coding and Test-Focused Solutions

```text
Please write a high-quality, general-purpose solution using the standard tools available. Do not create helper scripts or workarounds to accomplish the task more efficiently. Implement a solution that works correctly for all valid inputs, not just the test cases. Do not hard-code values or create solutions that only work for specific test inputs. Instead, implement the actual logic that solves the problem generally.

Focus on understanding the problem requirements and implementing the correct algorithm. Tests are there to verify correctness, not to define the solution. Provide a principled implementation that follows best practices and software design principles.

If the task is unreasonable or infeasible, or if any of the tests are incorrect, please inform me rather than working around them. The solution should be robust, maintainable, and extendable.
```

### Research and Information Gathering

For complex research tasks, use a structured approach:
```text
Search for this information in a structured way. As you gather data, develop several competing hypotheses. Track your confidence levels in your progress notes to improve calibration. Regularly self-critique your approach and plan. Update a hypothesis tree or research notes file to persist information and provide transparency. Break down this complex research task systematically.
```

---

## 7. Output and Formatting (supplementary reference)

### Communication Style in Claude 4.6

- **More direct and grounded:** Fact-based progress reports rather than self-celebratory updates
- **More conversational:** Slightly more fluent and colloquial
- **Less verbose:** May skip detailed summaries for efficiency

To get visibility into reasoning after tool use:
```text
After completing a task that involves tool use, provide a quick summary of the work you've done.
```

### Controlling Format

1. **Tell Claude what to do instead of what not to do** — Instead of "Do not use markdown," try "Your response should be composed of smoothly flowing prose paragraphs."
2. **Use XML format indicators** — "Write the prose sections of your response in `<smoothly_flowing_prose_paragraphs>` tags."
3. **Match prompt style to desired output** — Formatting in your prompt influences Claude's response formatting
4. **Detailed guidance for specific formatting:**

```text
<avoid_excessive_markdown_and_bullet_points>
When writing reports, documents, technical explanations, analyses, or any long-form content, write in clear, flowing prose using complete paragraphs and sentences. Use standard paragraph breaks for organization and reserve markdown primarily for `inline code`, code blocks, and simple headings. Avoid using **bold** and *italics*.

DO NOT use ordered lists or unordered lists unless: a) you're presenting truly discrete items where a list format is the best option, or b) the user explicitly requests a list or ranking

Instead of listing items with bullets or numbers, incorporate them naturally into sentences. NEVER output a series of overly short bullet points.

Your goal is readable, flowing text that guides the reader naturally through ideas rather than fragmenting information into isolated points.
</avoid_excessive_markdown_and_bullet_points>
```

### LaTeX Output

Claude Opus 4.6 defaults to LaTeX for math. To force plain text:
```text
Format your response in plain text only. Do not use LaTeX, MathJax, or any markup notation such as \( \), $, or \frac{}{}. Write all math expressions using standard text characters (e.g., "/" for division, "*" for multiplication, and "^" for exponents).
```

### Migrating Away from Prefilled Responses

> **KEY EXAM FACT:** Starting with Claude 4.6 models, prefilled responses on the last assistant turn are no longer supported. Existing models continue to support prefills.

Migration strategies:
- **Output formatting:** Use Structured Outputs feature or ask model to conform to schema with retries
- **Eliminating preambles:** Use system prompt: "Respond directly without preamble. Do not start with phrases like 'Here is...', 'Based on...'"
- **Avoiding refusals:** Claude is much better now; clear prompting in user message is sufficient
- **Continuations:** Move to user message: "Your previous response was interrupted and ended with `[previous_response]`. Continue from where you left off."
- **Context hydration:** Inject reminders into user turns instead of assistant prefills

---

## Quick Reference: Key Exam Facts

| Topic | Key Fact |
|-------|----------|
| Query placement | Put queries at END of prompt, data at TOP (30% improvement) |
| Few-shot examples | 3-5 examples for best results, wrap in `<example>` tags |
| XML tags | `<instructions>`, `<context>`, `<input>`, `<documents>`, `<document>` |
| Document structure | `<document index="N"><source>...<document_content>...` |
| Tool overtriggering | Claude 4.6 overtriggers — dial back "CRITICAL/MUST" to normal language |
| Parallel tools | `<use_parallel_tool_calls>` prompt pattern for ~100% parallel rate |
| Adaptive thinking | `thinking: {type: "adaptive"}` for Claude 4.6, outperforms extended thinking |
| Extended thinking | `thinking: {type: "enabled", budget_tokens: N}` for older models |
| Effort parameter | Controls thinking depth: max, high, medium, low |
| Self-check | "Before you finish, verify your answer against [criteria]" |
| Prefills deprecated | Claude 4.6 no longer supports prefilled assistant responses |
| Context awareness | Claude 4.6 tracks remaining token budget in conversation |
| State persistence | progress.txt (unstructured), tests.json (structured), git logs |
| Subagent overuse | Claude 4.6 may over-delegate; guide when subagents are warranted |
| Prompt chaining | generate -> review -> refine (self-correction pattern) |
| Safety pattern | Confirm before destructive/irreversible/shared-system actions |
| Anti-hallucination | `<investigate_before_answering>` — read files before answering |
| Over-engineering | Claude 4.6 tends to overengineer; prompt for minimal solutions |
| Starting fresh | Claude discovers state from filesystem; prescribe: pwd, progress.txt, tests.json, git logs |
