# Domain 1: Agentic Architecture & Orchestration (27%)

This is the highest-weighted domain on the Claude Certified Architect -- Foundations exam. It covers the design and implementation of agentic systems using the Claude Agent SDK, including agentic loops, multi-agent orchestration, hooks, task decomposition, and session management.

---

## Table of Contents

- [Task 1.1: Agentic Loop Implementation](#task-11-agentic-loop-implementation)
- [Task 1.2: Multi-Agent Orchestration (Coordinator-Subagent)](#task-12-multi-agent-orchestration-coordinator-subagent)
- [Task 1.3: Subagent Invocation and Context Passing](#task-13-subagent-invocation-and-context-passing)
- [Task 1.4: Multi-step Workflows with Enforcement](#task-14-multi-step-workflows-with-enforcement)
- [Task 1.5: Agent SDK Hooks](#task-15-agent-sdk-hooks)
- [Task 1.6: Task Decomposition Strategies](#task-16-task-decomposition-strategies)
- [Task 1.7: Session State Management](#task-17-session-state-management)

---

## Task 1.1: Agentic Loop Implementation

**Exam Reference:** "Design and implement agentic loops for autonomous task execution"

### Key Concepts (What You Need to KNOW)

1. **The Agentic Loop Lifecycle:**
   - Send a request to Claude with tools available
   - Inspect the response's `stop_reason`:
     - `"tool_use"` -- Claude wants to call a tool; execute it and return the result
     - `"end_turn"` -- Claude has finished; present the final response to the user
   - Tool results are appended to conversation history so the model can reason about what to do next
   - The loop continues until Claude decides it is done (returns `"end_turn"`)

2. **Model-Driven Decision-Making:**
   - Claude reasons about which tool to call next based on context -- this is NOT a pre-configured decision tree
   - The model decides the sequence of actions dynamically based on what it discovers
   - You do NOT hardcode tool call sequences; the model adapts based on intermediate results

3. **Tool Result Handling:**
   - Each tool result is added back to the conversation as a `tool_result` content block
   - The model uses these results to inform its next action
   - The complete conversation history (including all tool calls and results) is sent with each API request

### Practical Skills (What You Need to DO)

- Implement loop control flow that continues on `stop_reason == "tool_use"` and terminates on `stop_reason == "end_turn"`
- Correctly append tool results to conversation context between iterations
- Avoid all three anti-patterns listed below

### Code Example: Basic Agentic Loop in Python

```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_customer",
        "description": "Retrieves customer information by ID or email. Returns customer profile including name, email, account status, and verified customer ID. Use this BEFORE any order or refund operations to verify customer identity.",
        "input_schema": {
            "type": "object",
            "properties": {
                "customer_id": {"type": "string", "description": "Customer ID or email address"},
            },
            "required": ["customer_id"],
        },
    },
    {
        "name": "lookup_order",
        "description": "Retrieves order details by order number. Requires a verified customer ID from get_customer. Returns order items, status, shipping info, and refund eligibility.",
        "input_schema": {
            "type": "object",
            "properties": {
                "order_id": {"type": "string"},
                "customer_id": {"type": "string", "description": "Verified customer ID from get_customer"},
            },
            "required": ["order_id", "customer_id"],
        },
    },
]

def execute_tool(tool_name: str, tool_input: dict) -> str:
    """Execute a tool and return the result as a string."""
    if tool_name == "get_customer":
        return '{"customer_id": "C-12345", "name": "Jane Doe", "status": "active"}'
    elif tool_name == "lookup_order":
        return '{"order_id": "ORD-999", "status": "delivered", "total": 59.99}'
    return '{"error": "Unknown tool"}'

def run_agentic_loop(user_message: str) -> str:
    """Run the agentic loop until the model returns end_turn."""
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        )

        # KEY DECISION POINT: Check stop_reason
        if response.stop_reason == "end_turn":
            # Model is done -- extract and return the final text
            final_text = ""
            for block in response.content:
                if block.type == "text":
                    final_text += block.text
            return final_text

        elif response.stop_reason == "tool_use":
            # Model wants to call tools -- execute them and continue
            # First, add the assistant's response to history
            messages.append({"role": "assistant", "content": response.content})

            # Execute each tool call and collect results
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    })

            # Add tool results to conversation history
            messages.append({"role": "user", "content": tool_results})
            # Loop continues -- next iteration sends updated history to Claude
```

### Anti-Patterns to Avoid (EXAM TRAPS)

| Anti-Pattern | Why It Is Wrong | Correct Approach |
|---|---|---|
| **Parsing natural language for loop termination** (e.g., checking if assistant says "I'm done") | Unreliable; the model's text is not a structured signal | Use `stop_reason == "end_turn"` |
| **Arbitrary iteration caps as the primary stopping mechanism** (e.g., `max_iterations = 5`) | Masks underlying issues; may cut off the agent before task completion | Let `stop_reason` drive termination; use iteration caps only as a safety net, not the primary mechanism |
| **Checking for assistant text content as a completion indicator** (e.g., `if response has text, stop`) | A response can contain BOTH text AND tool_use blocks; text alone does not mean completion | Always check `stop_reason`, not content block types |

---

## Task 1.2: Multi-Agent Orchestration (Coordinator-Subagent)

**Exam Reference:** "Orchestrate multi-agent systems with coordinator-subagent patterns"

### Key Concepts (What You Need to KNOW)

1. **Hub-and-Spoke Architecture:**
   - A single **coordinator agent** manages all communication with subagents
   - Subagents do NOT communicate directly with each other
   - All inter-subagent communication flows through the coordinator
   - The coordinator handles error handling, information routing, and task assignment

2. **Isolated Subagent Context:**
   - Subagents do NOT inherit the coordinator's conversation history automatically
   - Each subagent starts with a fresh context containing only what is explicitly provided
   - This is a critical exam concept -- subagents have NO implicit access to parent state

3. **Coordinator Responsibilities:**
   - **Task decomposition**: Analyzing a complex query and breaking it into subtasks
   - **Delegation**: Choosing which subagent(s) to invoke based on query complexity
   - **Dynamic selection**: NOT always routing through the full pipeline -- selecting only the needed subagents
   - **Result aggregation**: Combining subagent outputs into a coherent response
   - **Iterative refinement**: Evaluating synthesis output for gaps, re-delegating with targeted queries

4. **Risks of Overly Narrow Task Decomposition:**
   - If the coordinator decomposes "impact of AI on creative industries" into only visual arts subtasks, it misses music, writing, film
   - Narrow decomposition leads to incomplete coverage of broad topics
   - The coordinator must ensure subtasks collectively cover the full scope

### Practical Skills (What You Need to DO)

- Design coordinators that dynamically select subagents (not always using all of them)
- Partition research scope to minimize duplication (e.g., assign distinct subtopics to each subagent)
- Implement iterative refinement: coordinator evaluates results, re-delegates to fill gaps
- Route ALL communication through the coordinator for observability

### Code Example: Coordinator + Subagents (Conceptual SDK Pattern)

```typescript
// Agent SDK multi-agent setup (conceptual TypeScript)
import { Agent, AgentDefinition } from "@anthropic-ai/agent-sdk";

// Define subagent types with restricted tool sets
const webSearchAgent: AgentDefinition = {
  name: "web_search_agent",
  description: "Searches the web for current information on assigned topics. Returns structured findings with source URLs and publication dates.",
  systemPrompt: `You are a web research specialist. For each assigned topic:
    1. Search for authoritative sources
    2. Return structured findings with: claim, evidence, source_url, publication_date
    3. If a search times out, return partial results with error context`,
  allowedTools: ["web_search", "fetch_url"],
};

const analysisAgent: AgentDefinition = {
  name: "document_analysis_agent",
  description: "Analyzes documents and extracts structured insights with source attribution.",
  systemPrompt: `You are a document analysis specialist. Extract key findings as structured data. Include source document name, page numbers, and relevant excerpts.`,
  allowedTools: ["read_document", "extract_data"],
};

const synthesisAgent: AgentDefinition = {
  name: "synthesis_agent",
  description: "Combines findings from multiple sources into a coherent report with citations.",
  systemPrompt: `You are a synthesis specialist. Combine findings preserving source attribution. Flag conflicting data with both sources rather than choosing one.`,
  allowedTools: ["verify_fact"],  // Scoped cross-role tool for simple lookups
};

// Coordinator agent -- has access to Task tool for spawning subagents
const coordinatorAgent: AgentDefinition = {
  name: "coordinator",
  description: "Orchestrates research by decomposing topics, delegating to specialists, and ensuring comprehensive coverage.",
  systemPrompt: `You are a research coordinator. Your job:
    1. Decompose the research topic into comprehensive subtasks covering ALL relevant domains
    2. Delegate to specialist subagents via the Task tool
    3. Evaluate results for coverage gaps
    4. Re-delegate with targeted queries if gaps are found
    5. Invoke synthesis only when coverage is sufficient

    IMPORTANT: Ensure subtasks cover the FULL scope of the topic.
    For example, "creative industries" includes visual arts, music, writing, film, gaming, etc.`,
  allowedTools: ["Task"],  // Must include "Task" to spawn subagents
};
```

### Exam Trap: Overly Narrow Task Decomposition (Sample Question Pattern)

The exam guide includes a sample question where the coordinator decomposes "impact of AI on creative industries" into only visual arts subtasks. The correct answer is that **the coordinator's task decomposition is too narrow** -- NOT that downstream agents are failing. The subagents executed correctly within their assigned scope; the problem is what they were assigned.

---

## Task 1.3: Subagent Invocation and Context Passing

**Exam Reference:** "Configure subagent invocation, context passing, and spawning"

### Key Concepts (What You Need to KNOW)

1. **The Task Tool:**
   - The mechanism for spawning subagents in the Claude Agent SDK
   - The coordinator's `allowedTools` MUST include `"Task"` for it to invoke subagents
   - Each Task call creates a new subagent with its own isolated context

2. **Explicit Context Passing:**
   - Subagents do NOT automatically inherit parent context or share memory
   - All context must be explicitly provided in the subagent's prompt
   - When passing results between agents, include the complete findings directly in the prompt
   - Use structured data formats to separate content from metadata (source URLs, document names, page numbers)

3. **AgentDefinition Configuration:**
   - `name`: Identifier for the subagent type
   - `description`: What the subagent does (used by coordinator for selection)
   - `systemPrompt`: Instructions for the subagent's behavior
   - `allowedTools`: Restricted set of tools for the subagent's role

4. **fork_session:**
   - Creates independent branches from a shared analysis baseline
   - Used for exploring divergent approaches (e.g., comparing two refactoring strategies)
   - Each fork starts from the same point but proceeds independently

5. **Parallel Subagent Spawning:**
   - The coordinator can emit **multiple Task tool calls in a single response**
   - This spawns subagents in parallel rather than sequentially
   - Significantly reduces latency compared to spawning across separate turns

### Practical Skills (What You Need to DO)

- Include complete prior findings in subagent prompts (not just summaries)
- Use structured data to preserve attribution when passing context
- Spawn parallel subagents via multiple Task calls in one response
- Design coordinator prompts that specify **goals and quality criteria** rather than step-by-step procedures

### Code Example: Context Passing Between Agents

```python
# Coordinator passes findings to synthesis subagent via Task tool
# This shows what the coordinator's Task tool call looks like

# Step 1: Coordinator receives web search results from search subagent
web_search_results = {
    "findings": [
        {
            "claim": "AI art generators increased digital art production by 300% in 2024",
            "evidence": "Survey of 500 digital artists showed...",
            "source_url": "https://example.com/ai-art-survey",
            "publication_date": "2024-06-15",
        },
        {
            "claim": "Music composition AI tools are used by 45% of producers",
            "evidence": "Industry report indicates...",
            "source_url": "https://example.com/music-ai-report",
            "publication_date": "2024-08-20",
        }
    ],
    "coverage": ["visual_arts", "music"],
    "gaps": ["writing", "film"],
}

# Step 2: Coordinator passes COMPLETE findings to synthesis subagent
# Note: Context is EXPLICITLY included -- not inherited
synthesis_task_prompt = f"""
Synthesize the following research findings into a coherent report.

## Research Findings
{json.dumps(web_search_results["findings"], indent=2)}

## Coverage Status
- Covered topics: {web_search_results["coverage"]}
- Known gaps: {web_search_results["gaps"]}

## Requirements
- Preserve ALL source attributions (URLs, dates)
- Flag any conflicting statistics with both sources
- Clearly note which topics have gaps in coverage
- Structure output with sections for well-supported vs. limited findings
"""

# Step 3: Coordinator emits this as a Task tool call
# The SDK handles spawning the subagent with this prompt
```

### Parallel Subagent Spawning Pattern

```python
# Coordinator emits MULTIPLE Task calls in a SINGLE response
# This spawns subagents in parallel for reduced latency

# In a single assistant turn, the coordinator returns tool_use blocks like:
# [
#   {tool_use: Task, input: {agent: "web_search_agent", prompt: "Research AI in visual arts..."}},
#   {tool_use: Task, input: {agent: "web_search_agent", prompt: "Research AI in music..."}},
#   {tool_use: Task, input: {agent: "document_analysis_agent", prompt: "Analyze report on AI in film..."}}
# ]
# All three subagents run in parallel
```

### Common Exam Traps

- **Trap:** Assuming subagents can access the coordinator's conversation history. They CANNOT.
- **Trap:** Passing only a summary instead of complete findings to downstream subagents -- this loses critical details and attribution.
- **Trap:** Spawning subagents one at a time across separate turns when they could run in parallel.

---

## Task 1.4: Multi-step Workflows with Enforcement

**Exam Reference:** "Implement multi-step workflows with enforcement and handoff patterns"

### Key Concepts (What You Need to KNOW)

1. **Programmatic Enforcement vs. Prompt-Based Guidance:**

   | Approach | Guarantee Level | Use When |
   |---|---|---|
   | **Programmatic enforcement** (hooks, prerequisite gates) | Deterministic -- 100% guaranteed | Business-critical rules with financial/safety consequences |
   | **Prompt-based guidance** (system prompt instructions) | Probabilistic -- high but NOT 100% | Soft preferences, style guidelines, general behavior |

   **Critical exam insight:** When deterministic compliance is required (e.g., identity verification before financial operations), prompt instructions alone have a **non-zero failure rate** and are insufficient.

2. **Prerequisite Gates:**
   - Block downstream tool calls until prerequisite steps have completed
   - Example: Block `process_refund` until `get_customer` has returned a verified customer ID
   - This is programmatic, not prompt-based -- it cannot be bypassed by the model

3. **Structured Handoff Protocols:**
   - When escalating to a human agent, include: customer ID, root cause analysis, recommended actions, refund amount
   - Human agents may NOT have access to the conversation transcript
   - The handoff summary must be self-contained and actionable

4. **Multi-Concern Request Decomposition:**
   - A single customer message may contain multiple issues
   - The agent should decompose into distinct items, investigate each (potentially in parallel), then synthesize a unified resolution

### Practical Skills (What You Need to DO)

- Implement programmatic prerequisites that block tool calls until preconditions are met
- Compile structured handoff summaries for human escalation
- Decompose multi-concern requests into parallel investigation tracks

### Code Example: Prerequisite Blocking

```python
# Programmatic enforcement: block process_refund until get_customer completes

class ToolPrerequisiteEnforcer:
    """Enforces tool call ordering with deterministic guarantees."""

    def __init__(self):
        self.completed_tools = {}  # Maps tool_name -> result

    # Define prerequisites: {tool: [required_prior_tools]}
    PREREQUISITES = {
        "lookup_order": ["get_customer"],
        "process_refund": ["get_customer", "lookup_order"],
        "escalate_to_human": [],  # No prerequisites -- always allowed
        "get_customer": [],        # No prerequisites -- entry point
    }

    def check_prerequisites(self, tool_name: str) -> tuple[bool, str]:
        """Check if all prerequisites for a tool have been completed.
        Returns (allowed, reason)."""
        required = self.PREREQUISITES.get(tool_name, [])
        missing = [t for t in required if t not in self.completed_tools]

        if missing:
            return False, (
                f"Cannot call {tool_name}: requires {missing} to complete first. "
                f"Completed so far: {list(self.completed_tools.keys())}"
            )
        return True, "OK"

    def record_completion(self, tool_name: str, result: dict):
        """Record that a tool has completed successfully."""
        self.completed_tools[tool_name] = result

    def intercept_tool_call(self, tool_name: str, tool_input: dict) -> dict | None:
        """Intercept a tool call. Returns error response if blocked, None if allowed."""
        allowed, reason = self.check_prerequisites(tool_name)
        if not allowed:
            # Return an error to the model so it knows to call prerequisites first
            return {
                "error": reason,
                "errorCategory": "validation",
                "isRetryable": False,
                "required_action": f"Call {self.PREREQUISITES[tool_name]} first",
            }
        return None  # Tool call is allowed to proceed


# Usage in the agentic loop:
enforcer = ToolPrerequisiteEnforcer()

# When processing tool calls:
for block in response.content:
    if block.type == "tool_use":
        # Check prerequisites BEFORE executing
        error = enforcer.intercept_tool_call(block.name, block.input)
        if error:
            # Return error to model -- it will learn to call prerequisites first
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": json.dumps(error),
                "is_error": True,
            })
        else:
            # Execute the tool normally
            result = execute_tool(block.name, block.input)
            enforcer.record_completion(block.name, json.loads(result))
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": result,
            })
```

### Structured Handoff Example

```python
# When escalating to a human agent, compile a self-contained summary
handoff_summary = {
    "customer_id": "C-12345",
    "customer_name": "Jane Doe",
    "root_cause": "Customer requests price match against competitor, but policy only covers own-site price adjustments",
    "attempted_resolution": "Offered standard return and repurchase; customer declined",
    "refund_amount_requested": 45.00,
    "policy_gap": "No policy exists for competitor price matching",
    "recommended_action": "Approve $45 goodwill credit OR escalate to policy team for new guideline",
    "conversation_summary": "Customer purchased item at $199, found same item at competitor for $154, requests $45 adjustment",
}
```

### Exam Trap (From Sample Question 1 in the Guide)

**Question pattern:** "Production data shows the agent skips `get_customer` in 12% of cases..."

**Correct answer:** Add a **programmatic prerequisite** that blocks `lookup_order` and `process_refund` until `get_customer` has returned a verified customer ID.

**Why the other options are wrong:**
- Enhancing the system prompt = probabilistic, not deterministic
- Adding few-shot examples = still probabilistic
- Implementing a routing classifier = addresses tool availability, not tool ordering

---

## Task 1.5: Agent SDK Hooks

**Exam Reference:** "Apply Agent SDK hooks for tool call interception and data normalization"

### Key Concepts (What You Need to KNOW)

1. **PostToolUse Hooks:**
   - Intercept tool results AFTER execution but BEFORE the model processes them
   - Used for **data normalization**: converting heterogeneous formats into consistent ones
   - Example: Normalizing Unix timestamps, ISO 8601 dates, and numeric status codes from different MCP tools into a uniform format

2. **Tool Call Interception Hooks:**
   - Intercept outgoing tool calls BEFORE execution
   - Used for **compliance enforcement**: blocking policy-violating actions
   - Example: Blocking refunds above $500 and redirecting to human escalation
   - Provides **deterministic guarantees** that the business rule will be enforced

3. **Hooks vs. Prompt-Based Enforcement:**

   | Aspect | Hooks | Prompt Instructions |
   |---|---|---|
   | Guarantee | Deterministic (100%) | Probabilistic (high but not 100%) |
   | Use case | Business rules requiring guaranteed compliance | General behavioral guidance |
   | Bypass risk | Cannot be bypassed by the model | Model may occasionally ignore |
   | Implementation | Code-level enforcement | Natural language instructions |

   **Exam insight:** Choose hooks over prompt-based enforcement when business rules require **guaranteed compliance** (financial thresholds, identity verification, regulatory requirements).

### Practical Skills (What You Need to DO)

- Implement PostToolUse hooks for normalizing heterogeneous data formats
- Implement tool call interception hooks that block policy-violating actions and redirect to alternative workflows
- Know when to choose hooks over prompt-based enforcement

### Code Example: PostToolUse Hook for Data Normalization

```python
from datetime import datetime, timezone

def post_tool_use_hook(tool_name: str, tool_input: dict, tool_result: dict) -> dict:
    """Normalize tool results before the model processes them.

    This hook intercepts tool results and converts heterogeneous
    data formats into a consistent format the agent can reliably process.
    """
    normalized = dict(tool_result)

    # Normalize timestamps: convert Unix timestamps to ISO 8601
    for key in ["created_at", "updated_at", "order_date", "refund_date"]:
        if key in normalized and isinstance(normalized[key], (int, float)):
            # Unix timestamp -> ISO 8601
            dt = datetime.fromtimestamp(normalized[key], tz=timezone.utc)
            normalized[key] = dt.isoformat()

    # Normalize status codes: convert numeric codes to human-readable strings
    status_map = {
        0: "pending",
        1: "processing",
        2: "completed",
        3: "failed",
        4: "cancelled",
    }
    if "status" in normalized and isinstance(normalized["status"], int):
        normalized["status"] = status_map.get(
            normalized["status"], f"unknown_status_{normalized['status']}"
        )

    # Normalize currency: ensure consistent decimal format
    for key in ["amount", "total", "refund_amount"]:
        if key in normalized and isinstance(normalized[key], (int, float)):
            normalized[key] = round(float(normalized[key]), 2)

    return normalized
```

### Code Example: Tool Call Interception for Compliance

```python
def pre_tool_call_hook(tool_name: str, tool_input: dict) -> dict | None:
    """Intercept tool calls to enforce business rules.

    Returns None if the call is allowed.
    Returns an error dict if the call should be blocked.
    """
    # Rule: Block refunds above $500 -- redirect to human escalation
    if tool_name == "process_refund":
        amount = tool_input.get("amount", 0)
        if amount > 500:
            return {
                "blocked": True,
                "reason": f"Refund amount ${amount} exceeds $500 threshold",
                "errorCategory": "policy_violation",
                "isRetryable": False,
                "required_action": "escalate_to_human",
                "message": (
                    f"This refund of ${amount} exceeds the automated processing "
                    f"limit of $500. Please use escalate_to_human to route this "
                    f"to a supervisor for approval."
                ),
            }

    # Rule: Block order modifications on shipped orders
    if tool_name == "modify_order":
        order_status = tool_input.get("current_status")
        if order_status in ["shipped", "delivered"]:
            return {
                "blocked": True,
                "reason": f"Cannot modify order in '{order_status}' status",
                "errorCategory": "validation",
                "isRetryable": False,
                "required_action": "Inform customer that shipped orders cannot be modified",
            }

    return None  # Call is allowed
```

### Common Exam Traps

- **Trap:** Thinking prompt instructions are sufficient for financial compliance rules. They are NOT -- hooks provide deterministic guarantees.
- **Trap:** Confusing PostToolUse (normalizes results AFTER execution) with pre-call interception (blocks calls BEFORE execution). Know which direction each hook operates.
- **Trap:** Not understanding that hooks operate at the code level outside the model's control -- the model cannot override or bypass a hook.

---

## Task 1.6: Task Decomposition Strategies

**Exam Reference:** "Design task decomposition strategies for complex workflows"

### Key Concepts (What You Need to KNOW)

1. **Two Decomposition Patterns:**

   | Pattern | Description | Use When |
   |---|---|---|
   | **Fixed sequential pipeline (prompt chaining)** | Pre-defined sequence of steps, each feeding into the next | Predictable, multi-aspect reviews; known structure |
   | **Dynamic adaptive decomposition** | Generate subtasks based on what is discovered at each step | Open-ended investigation; unknown structure |

2. **Prompt Chaining (Fixed Sequential):**
   - Break a review into sequential steps: analyze each file individually, then run a cross-file integration pass
   - Appropriate for code reviews, data extraction pipelines, multi-aspect document analysis
   - Steps are known in advance; the structure does not change based on findings

3. **Dynamic Adaptive Decomposition:**
   - Generate an investigation plan, execute initial steps, then adapt based on discoveries
   - Example: "Add comprehensive tests to a legacy codebase" -- first map structure, identify high-impact areas, then create a prioritized plan that adapts as dependencies are discovered
   - The value is that subtasks are generated based on what is found, not pre-specified

4. **Attention Dilution Problem:**
   - When processing many files in a single pass, the model gives inconsistent depth
   - Splitting into per-file analysis + cross-file integration prevents this
   - This is the rationale for prompt chaining in code reviews

### Practical Skills (What You Need to DO)

- Select the appropriate decomposition pattern based on the workflow
- Split large code reviews into per-file local analysis + separate cross-file integration pass
- Decompose open-ended tasks by first mapping structure, then creating an adaptive plan

### Code Example: Prompt Chaining for Code Review

```python
import anthropic

client = anthropic.Anthropic()

def review_pull_request(changed_files: list[dict]) -> dict:
    """Multi-pass code review using prompt chaining.

    Pass 1: Per-file local analysis (run for each file individually)
    Pass 2: Cross-file integration analysis (run once with all findings)
    """

    # ---- PASS 1: Per-file local analysis ----
    per_file_findings = []

    for file_info in changed_files:
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            system="""You are a code reviewer. Analyze this single file for:
            - Bugs and logic errors
            - Security vulnerabilities
            - Performance issues
            - Code style violations

            Return findings as structured JSON:
            {"findings": [{"file": str, "line": int, "severity": str, "issue": str, "suggestion": str}]}""",
            messages=[{
                "role": "user",
                "content": f"File: {file_info['path']}\n\nDiff:\n{file_info['diff']}",
            }],
        )
        per_file_findings.append({
            "file": file_info["path"],
            "analysis": response.content[0].text,
        })

    # ---- PASS 2: Cross-file integration analysis ----
    integration_response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        system="""You are a code reviewer performing cross-file integration analysis.
        You have already received per-file analyses. Now examine:
        - Data flow consistency across files
        - API contract violations between modules
        - Shared state mutation risks
        - Import/dependency issues
        - Contradictions (same pattern flagged in one file but approved in another)""",
        messages=[{
            "role": "user",
            "content": f"Per-file analyses:\n{json.dumps(per_file_findings, indent=2)}",
        }],
    )

    return {
        "per_file_findings": per_file_findings,
        "integration_findings": integration_response.content[0].text,
    }
```

### Code Example: Dynamic Adaptive Decomposition

```python
# Dynamic decomposition: the plan adapts based on discoveries
# Used for open-ended tasks like "add comprehensive tests to a legacy codebase"

def adaptive_test_generation(codebase_path: str):
    """
    Step 1: Map the codebase structure (discover what exists)
    Step 2: Identify high-impact areas (analyze what was found)
    Step 3: Create prioritized test plan (adapt based on findings)
    Step 4: Execute plan, adapting as dependencies are discovered
    """
    # Each step's output feeds into the next step's prompt
    # The plan is NOT pre-defined -- it emerges from analysis

    # Step 1 output might reveal: "This codebase has 3 modules,
    # module A has 0% coverage, module B has 80% coverage"

    # Step 2 then focuses on module A and discovers:
    # "Module A has complex business logic with no tests,
    # but it depends on module C which also has no tests"

    # Step 3 adapts: "Test module C first (dependency),
    # then module A (high impact), skip module B (already covered)"

    # This adaptation could not have been planned upfront
    pass
```

### Exam Trap (From Sample Question 12 in the Guide)

**Question pattern:** "A single-pass review of 14 files produces inconsistent, contradictory results..."

**Correct answer:** Split into focused per-file passes + a separate integration pass.

**Why the other options are wrong:**
- Requiring developers to split PRs = shifts burden without improving the system
- Using a larger context model = larger context does NOT solve attention quality issues
- Running 3 passes and flagging consensus = suppresses real bugs that are only caught intermittently

---

## Task 1.7: Session State Management

**Exam Reference:** "Manage session state, resumption, and forking"

### Key Concepts (What You Need to KNOW)

1. **Named Session Resumption (`--resume`):**
   - Use `--resume <session-name>` to continue a specific prior conversation
   - Preserves full conversation history including tool calls and results
   - Useful for long-running investigations that span multiple work sessions

2. **fork_session:**
   - Creates independent branches from a shared analysis baseline
   - Each fork explores a different approach from the same starting point
   - Example: Comparing two testing strategies or two refactoring approaches
   - Forks proceed independently -- changes in one do not affect the other

3. **When to Resume vs. Start Fresh:**

   | Approach | Use When |
   |---|---|
   | **Resume** (`--resume`) | Prior context is mostly valid; no major code changes since last session |
   | **Start fresh with summary** | Prior tool results are stale (e.g., code has changed significantly since last session) |

   **Key insight:** Starting a new session with a structured summary is MORE RELIABLE than resuming with stale tool results. If files have changed since the last session, the tool results in the conversation history are outdated and can mislead the model.

4. **Informing Resumed Sessions About Changes:**
   - When resuming after code modifications, explicitly tell the agent which files changed
   - This enables targeted re-analysis rather than requiring full re-exploration
   - Example: "Since our last session, I've refactored `auth.py` and `middleware.py`. Please re-analyze those files."

5. **/compact for Context Management:**
   - Use `/compact` during extended exploration sessions when context fills with verbose discovery output
   - Reduces context usage by summarizing prior conversation
   - Essential for long-running sessions that accumulate many tool results

### Practical Skills (What You Need to DO)

- Use `--resume` with session names to continue named investigation sessions
- Use `fork_session` to create parallel exploration branches
- Choose between resumption and fresh start based on context staleness
- Inform resumed sessions about specific file changes for targeted re-analysis
- Use `/compact` to manage context during extended sessions

### Usage Examples

```bash
# Start a named session for a long investigation
claude --session "auth-refactor" "Analyze the authentication module for security issues"

# Resume the session later
claude --resume "auth-refactor"
# Then tell it about changes: "Since our last session, I've updated auth.py lines 45-80"

# Fork a session to explore two approaches
# From within a session, use fork_session to create branches:
# Branch A: "Refactor auth using JWT tokens"
# Branch B: "Refactor auth using session cookies"
# Each branch has the shared analysis baseline but proceeds independently

# Compact context when it gets too long
# Within a session, use: /compact
# This summarizes the conversation to free up context window space
```

### Decision Framework: Resume vs. Fresh Start

```
Has significant code changed since the last session?
├── NO  → Resume with --resume (prior context is valid)
│         └── Optionally mention small changes for targeted re-analysis
├── YES → Start fresh with injected summary
│         └── Summarize key findings from prior session
│         └── Include specific file paths and decisions made
│         └── Do NOT include stale tool results
└── PARTIALLY → Resume but explicitly inform about changed files
              └── "Files X, Y, Z have changed. Please re-analyze those."
```

### Common Exam Traps

- **Trap:** Resuming a session after major code changes without informing the agent. The model will reference stale tool results and produce incorrect analysis.
- **Trap:** Not knowing that `fork_session` creates independent branches -- changes in one fork do NOT propagate to the other.
- **Trap:** Continuing an extended session without using `/compact` when context is full of verbose discovery output -- this leads to context degradation where the model starts referencing "typical patterns" rather than actual findings.
- **Trap:** Thinking `--resume` and `fork_session` are the same thing. Resume continues the SAME conversation; fork creates a NEW independent branch from a shared baseline.

---

## Cross-Cutting Exam Themes for Domain 1

### Theme 1: Programmatic vs. Probabilistic Enforcement

This is perhaps the single most important concept in Domain 1. Multiple questions test whether you understand that:
- **Prompt instructions** = probabilistic (model usually complies, but not always)
- **Hooks / programmatic gates** = deterministic (guaranteed enforcement)
- For business-critical rules with financial or safety consequences, ALWAYS choose programmatic enforcement

### Theme 2: Hub-and-Spoke Architecture

All inter-agent communication flows through the coordinator. Subagents never talk to each other directly. This provides:
- Observability (all communication is logged at the coordinator)
- Consistent error handling
- Controlled information flow

### Theme 3: Explicit Over Implicit

- Subagent context must be EXPLICITLY provided (no automatic inheritance)
- Session changes must be EXPLICITLY communicated (no automatic detection)
- Coverage requirements must be EXPLICITLY decomposed (the model may narrowly interpret broad topics)

### Theme 4: Structured Data for Inter-Agent Communication

When passing information between agents, always use structured formats that separate:
- Content (the actual findings)
- Metadata (source URLs, dates, page numbers, document names)
- Coverage annotations (what was found vs. what has gaps)

This preserves attribution through the pipeline and enables accurate downstream synthesis.

---

## Quick Reference: Technologies Tested (Domain 1)

From the exam guide's appendix:

| Technology | Key Concepts |
|---|---|
| **Claude Agent SDK** | Agent definitions, agentic loops, `stop_reason` handling, hooks (PostToolUse, tool call interception), subagent spawning via Task tool, `allowedTools` configuration |
| **Claude API** | `tool_use` with JSON schemas, `tool_choice` options (`"auto"`, `"any"`, forced tool selection), `stop_reason` values (`"tool_use"`, `"end_turn"`), `max_tokens`, system prompts |
| **Claude Code** | `--resume`, `fork_session`, `/compact`, Explore subagent |
| **Session Management** | Session resumption, `fork_session`, named sessions, session context isolation |

---

## Sample Exam Questions (From the Official Guide)

### Question 1 (Programmatic Enforcement)
**Scenario:** Agent skips `get_customer` in 12% of cases, leading to misidentified accounts.
**Answer:** Programmatic prerequisite that blocks downstream tools until `get_customer` completes (NOT prompt enhancement, NOT few-shot examples).

### Question 7 (Narrow Task Decomposition)
**Scenario:** Reports on "AI in creative industries" cover only visual arts, missing music/writing/film. Coordinator decomposed into only visual arts subtasks.
**Answer:** Coordinator's task decomposition is too narrow (NOT synthesis agent failure, NOT search agent limitation).

### Question 8 (Error Propagation)
**Scenario:** Web search subagent times out. How should failure flow back to coordinator?
**Answer:** Return structured error context (failure type, attempted query, partial results, alternatives) -- NOT generic "search unavailable", NOT empty results as success, NOT terminate entire workflow.

### Question 9 (Scoped Cross-Role Tools)
**Scenario:** Synthesis agent needs frequent simple fact-checks, causing 40% latency increase from round-trips through coordinator.
**Answer:** Give synthesis agent a scoped `verify_fact` tool for simple lookups; complex verifications still go through coordinator (NOT full web search access, NOT batching, NOT speculative caching).

---

## Supplemental Coverage: Gap-Filling Additions

The sections below add depth and code examples for exam requirements that were underrepresented above. They do NOT replace anything above; they supplement it.

---

### Task 1.2 Supplement: Iterative Refinement Loop (Code Example)

The coordinator must evaluate synthesis output for completeness and re-delegate when gaps are found. This is NOT a one-shot pipeline -- the coordinator loops until quality criteria are met.

```python
import anthropic
import json

client = anthropic.Anthropic()

def coordinator_iterative_refinement(topic: str, max_refinement_rounds: int = 3):
    """
    Coordinator evaluates subagent outputs, identifies coverage gaps,
    and re-delegates with targeted queries until quality criteria are met.

    max_refinement_rounds is a safety net, NOT the primary stopping mechanism.
    The coordinator stops when it judges coverage as sufficient.
    """
    all_findings = []
    covered_subtopics = set()
    expected_subtopics = None  # Discovered during first decomposition

    for round_num in range(max_refinement_rounds):
        if round_num == 0:
            # Initial decomposition: coordinator determines subtopics
            decomposition_prompt = f"""
            Decompose the research topic "{topic}" into comprehensive subtopics.
            Return JSON: {{"subtopics": ["subtopic1", "subtopic2", ...],
                          "rationale": "why these cover the full scope"}}
            IMPORTANT: Ensure FULL scope coverage. For broad topics,
            include ALL relevant domains, not just the most obvious ones.
            """
            # (Coordinator calls Claude to get decomposition, then spawns subagents)
            # After receiving results:
            expected_subtopics = {"visual_arts", "music", "writing", "film", "gaming"}
        else:
            # Re-delegation round: target ONLY the gaps
            gaps = expected_subtopics - covered_subtopics
            if not gaps:
                break  # Coverage is sufficient -- stop refining

            # Coordinator spawns NEW subagents for gap areas only
            # These are targeted queries, not a full re-run
            gap_prompt = f"""
            Previous research covered: {covered_subtopics}
            MISSING coverage for: {gaps}
            Research ONLY the missing subtopics. Provide findings with
            the same structured format as previous results.
            """
            # (Coordinator spawns subagents for gap areas)

        # After each round, coordinator evaluates:
        # - Are all expected subtopics covered?
        # - Are findings substantive (not just surface-level)?
        # - Are there contradictions that need resolution?

    # Only after coverage is judged sufficient does coordinator invoke synthesis
    return {"findings": all_findings, "coverage": covered_subtopics}
```

**Key exam point:** The coordinator evaluates synthesis quality and re-delegates for gaps. This iterative loop is what distinguishes a well-designed multi-agent system from a simple one-shot pipeline.

### Task 1.2 Supplement: Observability Through Hub-and-Spoke Routing

All communication between subagents MUST be routed through the coordinator. This is not just architectural preference -- it provides:

```python
# Why ALL communication flows through the coordinator:

# 1. OBSERVABILITY: Every inter-agent message is visible at the coordinator
coordinator_log = []

def route_message(from_agent: str, to_agent: str, message: dict):
    """All inter-agent communication goes through coordinator."""
    # Log for observability -- every message is captured
    coordinator_log.append({
        "from": from_agent,
        "to": to_agent,
        "timestamp": "2025-01-15T10:30:00Z",
        "message_summary": message.get("summary", ""),
        "token_count": len(str(message)),
    })
    # Coordinator can inspect, filter, or transform before forwarding
    return message

# 2. ERROR HANDLING: Coordinator can catch and handle subagent failures
# 3. INFORMATION CONTROL: Coordinator decides what context each subagent receives
# 4. AUDIT TRAIL: Complete record of all decisions and data flow

# ANTI-PATTERN: Direct subagent-to-subagent communication
# - Loses observability
# - Makes debugging impossible
# - Breaks the coordinator's ability to manage information flow
```

---

### Task 1.3 Supplement: Task Tool Configuration and fork_session (Code Examples)

#### Task Tool Schema and allowedTools Requirement

The coordinator MUST have `"Task"` in its `allowedTools` to spawn subagents. The Task tool itself accepts an agent name and a prompt:

```typescript
// The Task tool call structure as emitted by the coordinator
// This is what the coordinator's tool_use block looks like:

{
  "type": "tool_use",
  "id": "toolu_01ABC",
  "name": "Task",
  "input": {
    "agent": "web_search_agent",     // Which AgentDefinition to use
    "prompt": "Research the impact of AI on music composition. \
Return findings as structured JSON with fields: claim, evidence, \
source_url, publication_date. Focus on quantitative data from 2023-2025."
  }
}

// CRITICAL: The coordinator's allowedTools MUST include "Task"
const coordinatorAgent: AgentDefinition = {
  name: "coordinator",
  description: "Orchestrates research tasks across specialist agents",
  systemPrompt: "You are a research coordinator...",
  allowedTools: ["Task"],  // <-- WITHOUT THIS, coordinator CANNOT spawn subagents
};

// Subagents' allowedTools should NOT include "Task" unless they need
// to spawn their own subagents (nested orchestration)
const searchAgent: AgentDefinition = {
  name: "web_search_agent",
  description: "Searches the web for current information. Returns structured \
findings with source URLs and publication dates.",
  systemPrompt: "You are a web research specialist...",
  allowedTools: ["web_search", "fetch_url"],  // No "Task" -- leaf agent
};
```

#### AgentDefinition Configuration Details

Each field in `AgentDefinition` serves a specific purpose:

```typescript
const agentDef: AgentDefinition = {
  // name: Used to reference this agent when spawning via Task tool
  name: "document_analysis_agent",

  // description: Used by the COORDINATOR to decide WHICH agent to invoke.
  // This is critical for dynamic subagent selection -- the coordinator reads
  // these descriptions to choose the right specialist for each subtask.
  description: "Analyzes documents and extracts structured insights. \
Best for PDF reports, research papers, and long-form content. \
Returns findings with page numbers and section references.",

  // systemPrompt: Instructions for the subagent's behavior.
  // Should specify GOALS and QUALITY CRITERIA, not step-by-step procedures.
  systemPrompt: `You are a document analysis specialist.

GOAL: Extract actionable insights from the provided document.
QUALITY CRITERIA:
- Every claim must include the source page number
- Quantitative data must include exact figures and units
- Conflicting statements must be flagged with both references
- Coverage assessment: note which sections were analyzed vs skipped

Do NOT follow a rigid procedure. Adapt your analysis approach
based on the document type and content.`,

  // allowedTools: Restricts which tools this agent can use.
  // Principle of least privilege -- only grant what the role needs.
  allowedTools: ["read_document", "extract_data"],
};
```

**Exam point:** Coordinator prompts should specify research goals and quality criteria, NOT step-by-step procedures. Step-by-step instructions prevent the agent from adapting its approach based on what it discovers.

#### fork_session for Divergent Approaches

```python
# fork_session creates independent branches from a shared baseline.
# Use case: comparing two different approaches to the same problem.

# Example: Evaluating two refactoring strategies for an auth module

# Step 1: Shared analysis baseline (both forks start from here)
# claude --session "auth-analysis" "Analyze the auth module architecture"
# ... agent analyzes and builds understanding ...

# Step 2: Fork into two independent branches
# Fork A: JWT-based approach
# fork_session creates a NEW session that starts with the same
# conversation history as the parent, then proceeds independently.

# In the Agent SDK, fork_session is used like this:
# fork_session("auth-analysis-jwt")
#   -> "Refactor the auth module to use JWT tokens.
#       Implement the changes and run tests."

# fork_session("auth-analysis-sessions")
#   -> "Refactor the auth module to use server-side sessions.
#       Implement the changes and run tests."

# CRITICAL PROPERTIES:
# - Both forks share the SAME analysis baseline (the parent's history)
# - Changes in fork A do NOT affect fork B (fully independent)
# - Each fork has its own conversation history going forward
# - The parent session is NOT modified by either fork

# This is different from --resume:
# --resume CONTINUES the same conversation (linear)
# fork_session BRANCHES into independent parallel explorations (divergent)
```

#### Parallel Spawning: Multiple Task Calls in One Response

```typescript
// When the coordinator needs multiple subagents to work simultaneously,
// it emits MULTIPLE Task tool_use blocks in a SINGLE assistant response.

// The coordinator's response.content looks like:
[
  {
    "type": "text",
    "text": "I'll research all three subtopics in parallel..."
  },
  {
    "type": "tool_use",
    "id": "toolu_01A",
    "name": "Task",
    "input": {
      "agent": "web_search_agent",
      "prompt": "Research AI impact on visual arts. Return structured JSON..."
    }
  },
  {
    "type": "tool_use",
    "id": "toolu_01B",
    "name": "Task",
    "input": {
      "agent": "web_search_agent",
      "prompt": "Research AI impact on music composition. Return structured JSON..."
    }
  },
  {
    "type": "tool_use",
    "id": "toolu_01C",
    "name": "Task",
    "input": {
      "agent": "document_analysis_agent",
      "prompt": "Analyze the attached industry report on AI in film..."
    }
  }
]

// All three subagents run IN PARALLEL -- significant latency reduction
// vs. spawning them one at a time across three separate turns.
// The coordinator receives all three results at once in the next turn.
```

#### Structured Data Separating Content from Metadata

```python
# When passing results between agents, separate content from metadata
# so downstream agents can use content while preserving attribution.

subagent_result = {
    # CONTENT: the actual findings for downstream processing
    "content": {
        "findings": [
            {
                "claim": "AI-generated art sales grew 300% in 2024",
                "evidence": "Based on survey of 500 digital artists",
                "confidence": "high",
            }
        ],
        "summary": "AI is significantly accelerating digital art production",
    },
    # METADATA: attribution info that must be preserved but is not
    # part of the analytical content itself
    "metadata": {
        "source_url": "https://example.com/ai-art-survey-2024",
        "source_type": "peer-reviewed survey",
        "publication_date": "2024-06-15",
        "author": "Digital Arts Research Institute",
        "access_date": "2025-01-10",
        "search_query_used": "AI impact digital art production statistics 2024",
    },
    # COVERAGE: what this result does and does not address
    "coverage": {
        "topics_addressed": ["digital_art", "generative_art"],
        "topics_not_found": ["traditional_art_market"],
        "geographic_scope": "global",
        "time_range": "2023-2024",
    },
}

# This structure enables:
# 1. Synthesis agent uses "content" for analysis
# 2. Attribution is preserved in "metadata" for citations
# 3. Coordinator uses "coverage" to identify gaps for re-delegation
```

---

### Task 1.4 Supplement: Multi-Concern Request Decomposition (Code Example)

When a customer message contains multiple distinct issues, the agent decomposes them into parallel investigation tracks:

```python
# Customer message: "My order ORD-123 arrived damaged AND I was charged twice
# for subscription renewal AND my loyalty points disappeared"

# The agent decomposes this into three independent concerns:

concerns = [
    {
        "id": "concern_1",
        "type": "damaged_order",
        "details": "Order ORD-123 arrived damaged",
        "required_tools": ["get_customer", "lookup_order", "check_damage_policy"],
        "resolution_path": "refund_or_replace",
    },
    {
        "id": "concern_2",
        "type": "billing_error",
        "details": "Double charged for subscription renewal",
        "required_tools": ["get_customer", "get_billing_history", "check_subscription"],
        "resolution_path": "refund_duplicate_charge",
    },
    {
        "id": "concern_3",
        "type": "loyalty_issue",
        "details": "Loyalty points disappeared",
        "required_tools": ["get_customer", "get_loyalty_balance", "get_loyalty_history"],
        "resolution_path": "restore_points_or_explain",
    },
]

# KEY POINTS:
# 1. Each concern is investigated INDEPENDENTLY (can run in parallel)
# 2. All three share the get_customer prerequisite (run once, share result)
# 3. After parallel investigation, results are SYNTHESIZED into a
#    single coherent response addressing all three issues
# 4. If any concern requires escalation, the handoff summary includes
#    the status of ALL concerns (not just the escalated one)

# The agent's response addresses ALL concerns in a unified message,
# not three separate replies.
```

---

### Task 1.5 Supplement: Hook Registration Patterns in the Agent SDK

The Agent SDK provides specific hook points. Here is how hooks are registered and how they differ from prompt-based approaches:

```typescript
// Agent SDK hook registration pattern (TypeScript)
import { Agent, Hook, HookEvent } from "@anthropic-ai/agent-sdk";

// PostToolUse hook: fires AFTER tool execution, BEFORE model sees result
const dataNormalizationHook: Hook = {
  event: HookEvent.PostToolUse,
  handler: async (context) => {
    const { toolName, toolResult } = context;

    // Normalize Unix timestamps to ISO 8601 across ALL tools
    if (toolResult.created_at && typeof toolResult.created_at === "number") {
      toolResult.created_at = new Date(toolResult.created_at * 1000).toISOString();
    }

    // Normalize numeric status codes to readable strings
    const statusMap: Record<number, string> = {
      0: "pending", 1: "processing", 2: "completed",
      3: "failed", 4: "cancelled"
    };
    if (typeof toolResult.status === "number") {
      toolResult.status = statusMap[toolResult.status] ?? `unknown_${toolResult.status}`;
    }

    return toolResult; // Normalized result is what the model sees
  },
};

// PreToolUse hook: fires BEFORE tool execution for compliance checks
const complianceHook: Hook = {
  event: HookEvent.PreToolUse,
  handler: async (context) => {
    const { toolName, toolInput } = context;

    // Block refunds > $500, redirect to human
    if (toolName === "process_refund" && toolInput.amount > 500) {
      return {
        blocked: true,
        error: `Refund of $${toolInput.amount} exceeds $500 automated limit. ` +
               `Use escalate_to_human with the refund details.`,
      };
    }

    return null; // Allow the call to proceed
  },
};

// Register hooks on the agent
const agent = new Agent({
  definition: coordinatorAgentDef,
  hooks: [dataNormalizationHook, complianceHook],
});
```

**Why hooks beat prompts for compliance (exam-critical comparison):**

```
Scenario: "Block refunds above $500"

PROMPT APPROACH (probabilistic):
  System prompt: "Never process refunds above $500. Escalate to human instead."
  Result: Works ~95% of the time. In 5% of cases, the model may process
  the refund anyway due to complex conversation context, jailbreak-like
  user messages, or edge cases in reasoning.

HOOK APPROACH (deterministic):
  PreToolUse hook checks amount > 500 and returns blocked=true.
  Result: Works 100% of the time. The model NEVER sees the tool execute.
  The hook runs at the code level, outside the model's control entirely.
  Even if the model "decides" to process the refund, the hook blocks it.
```

---

### Task 1.6 Supplement: Open-Ended Task Decomposition Pattern

The three-phase pattern for open-ended tasks:

```python
# Phase 1: MAP STRUCTURE -- discover what exists before planning
def phase_1_map_structure(codebase_path: str) -> dict:
    """
    Use tools to discover the codebase structure.
    DO NOT plan yet -- just observe.
    """
    # Agent uses file listing, grep, and read tools to discover:
    # - Module structure and dependencies
    # - Existing test coverage (which modules have tests, which don't)
    # - Code complexity indicators
    # - Entry points and critical paths
    return {
        "modules": ["auth", "billing", "notifications", "api_gateway"],
        "test_coverage": {"auth": 0, "billing": 80, "notifications": 45, "api_gateway": 20},
        "dependencies": {"auth": [], "billing": ["auth"], "notifications": ["auth", "billing"]},
        "complexity_scores": {"auth": "high", "billing": "medium", "notifications": "low", "api_gateway": "high"},
    }

# Phase 2: IDENTIFY HIGH-IMPACT AREAS -- analyze what was found
def phase_2_identify_priorities(structure: dict) -> list:
    """
    Based on discovered structure, identify where effort has highest impact.
    This step CANNOT be done upfront -- it requires Phase 1's output.
    """
    # Agent reasons about the structure to find:
    # - auth: 0% coverage + high complexity + dependency root = HIGHEST PRIORITY
    # - api_gateway: 20% coverage + high complexity = HIGH PRIORITY
    # - notifications: 45% coverage + low complexity = LOW PRIORITY
    # - billing: 80% coverage + depends on auth = DEFER (already well-tested)
    return [
        {"module": "auth", "priority": 1, "rationale": "Zero coverage, high complexity, dependency root"},
        {"module": "api_gateway", "priority": 2, "rationale": "Low coverage, high complexity"},
        {"module": "notifications", "priority": 3, "rationale": "Medium coverage, low complexity"},
    ]

# Phase 3: PRIORITIZED PLAN -- adapts as execution proceeds
def phase_3_execute_adaptive(priorities: list):
    """
    Execute the plan, adapting as new information emerges.
    Example: While testing auth, discover it has an undocumented
    dependency on a config module -- add config tests first.
    """
    # This adaptation COULD NOT have been planned upfront.
    # The plan changes based on what is discovered during execution.
    pass
```

---

### Task 1.7 Supplement: Session Management Decision Details

#### --resume with Named Sessions

```bash
# Create a named session
claude --session "security-audit-v2" "Begin a security audit of the payments module"

# ... work happens, you close the terminal ...

# Resume the EXACT same session later (full history preserved)
claude --resume "security-audit-v2"

# The resumed session has ALL prior tool calls and results in context.
# You can pick up exactly where you left off.

# IMPORTANT: If you've made code changes since the last session,
# TELL the agent explicitly:
claude --resume "security-audit-v2"
# First message: "Since our last session, I've updated payments/processor.py
# (lines 45-80, refactored the retry logic) and added payments/validator.py
# (new file). Please re-analyze these specific files."
```

#### fork_session Detailed Workflow

```bash
# Scenario: You've analyzed the auth module and want to compare
# two different refactoring approaches.

# Step 1: Build shared analysis baseline
claude --session "auth-refactor" "Analyze the auth module: architecture,
dependencies, pain points, and test coverage"
# ... agent builds deep understanding of the module ...

# Step 2: Fork into approach A
# fork_session("auth-refactor-jwt")
# Prompt: "Based on our analysis, refactor auth to use JWT tokens.
#          Implement the changes and run the test suite."

# Step 3: Fork into approach B (from the SAME baseline, not from fork A)
# fork_session("auth-refactor-sessions")
# Prompt: "Based on our analysis, refactor auth to use server-side sessions.
#          Implement the changes and run the test suite."

# Properties:
# - Fork A and Fork B share the SAME analysis baseline
# - Fork A's changes are INVISIBLE to Fork B (and vice versa)
# - You can compare results from both forks to make a decision
# - The original session "auth-refactor" is UNCHANGED
```

#### When to Resume vs Start Fresh (Detailed Decision Matrix)

```
RESUME (--resume "session-name") when:
  ✓ Code has NOT changed significantly since last session
  ✓ You need the full context of prior discoveries
  ✓ The investigation is ongoing and sequential
  ✓ Tool results in history are still valid

START FRESH with injected summary when:
  ✓ Significant code changes make prior tool results STALE
  ✓ You want a clean context without accumulated noise
  ✓ The prior session's findings can be summarized concisely
  ✓ You need the agent to re-read files with fresh eyes

  Example fresh-start prompt:
  "Previous investigation found: [summary of key findings].
   Files analyzed: auth.py, middleware.py, routes.py.
   Decision made: Use JWT-based auth.
   Since then, auth.py has been completely rewritten.
   Please re-analyze auth.py with the new implementation
   and verify it aligns with our JWT decision."

USE /compact when:
  ✓ Mid-session, context is filling with verbose tool output
  ✓ You want to continue the SAME session but need space
  ✓ Early exploration output is no longer relevant
  ✓ /compact summarizes prior conversation, freeing context window
```

---

## Source Links

The following official Anthropic documentation pages are relevant to Domain 1 topics. Note: URLs are based on the current documentation site structure and should be verified for availability.

- [Anthropic Agent SDK Overview](https://docs.anthropic.com/en/docs/agents-and-tools/agent-sdk)
- [Building Agentic Loops (Tool Use)](https://docs.anthropic.com/en/docs/build-with-claude/agentic-tool-use)
- [Tool Use (Function Calling)](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview)
- [Claude Code Overview](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview)
- [Claude Code Sessions and Resumption](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/sessions)
- [Multi-Agent Orchestration Patterns](https://docs.anthropic.com/en/docs/build-with-claude/agentic-tool-use#multi-agent-orchestration)
- [Agent SDK Hooks and Middleware](https://docs.anthropic.com/en/docs/agents-and-tools/agent-sdk/hooks)
- [Prompt Engineering for Agents](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [Anthropic Cookbook: Agent Patterns](https://github.com/anthropics/anthropic-cookbook/tree/main/misc/prompt_caching)
- [Claude Code CLI Reference](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/cli-reference)
