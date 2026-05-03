# Domain 5: Context Management & Reliability (15%)

This domain covers how to maintain coherence, accuracy, and reliability in Claude-powered systems as conversations grow, errors propagate, and decisions require human oversight. It spans 6 task statements addressing context preservation, escalation, error handling, codebase exploration, human review, and provenance tracking.

---

## Task 5.1: Conversation Context Preservation

### Key Concepts

**Progressive Summarization Risks**

When conversations grow long, systems often summarize earlier turns to stay within context limits. This creates a critical failure mode: condensing specific transactional facts -- numbers, dates, percentages, account IDs, dollar amounts -- into vague summaries.

Example of dangerous summarization:
```
Original: "Customer reported $4,287.53 charged on 2025-03-14 from merchant ID MRC-99201"
Bad summary: "Customer reported an overcharge recently"
```

The summary loses every actionable detail. When a downstream agent or later turn needs to issue a refund, it has no amount, no date, and no merchant reference.

**"Lost in the Middle" Effect**

Large language models reliably process information at the beginning and end of context but may fail to attend to information in the middle of long contexts. This is a well-documented phenomenon in transformer architectures. Implications:

- Critical facts placed in the middle of a 50-turn conversation may be overlooked
- System prompts (beginning) and recent turns (end) get disproportionate attention
- Long tool result blocks in the middle are particularly vulnerable

**Tool Result Token Accumulation**

Tool calls often return full API responses with 40+ fields when only 5 are relevant. Over multiple tool calls, these verbose responses consume enormous context:

```json
// Raw CRM tool result: ~800 tokens
{
  "customer_id": "C-12345",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "created_at": "2020-01-15T...",
  "updated_at": "2025-03-14T...",
  "internal_score": 87,
  "segment": "enterprise",
  "lifecycle_stage": "active",
  "marketing_preferences": { ... },
  "billing_history": [ ... ],  // 20+ entries
  "support_tickets": [ ... ],  // 15+ entries
  "metadata": { ... }
  // ... 30 more fields
}
```

After 10 tool calls, you may have consumed 8,000+ tokens on data that is largely irrelevant to the current task.

**Passing Complete Conversation History**

For multi-turn interactions, passing the complete conversation history (all user and assistant messages) to each API call is the default and recommended approach for maintaining coherence. The Messages API is stateless -- each call must include the full context needed.

### Practical Skills

1. **Extract transactional facts into a persistent "case facts" block.** Maintain a structured block at the top of the system prompt or in a dedicated message that captures key facts as they emerge. For multi-issue sessions, persist structured issue data so that each issue's state (status, key facts, resolution) is tracked independently:

```python
# Multi-issue session tracking
case_facts = {
    "customer_id": "C-12345",
    "issues": [
        {
            "issue_id": 1,
            "type": "duplicate_charge",
            "amount": "$4,287.53",
            "date": "2025-03-14",
            "merchant_id": "MRC-99201",
            "status": "resolved",
            "resolution": "refund_issued"
        },
        {
            "issue_id": 2,
            "type": "subscription_cancellation",
            "plan": "Enterprise Annual",
            "renewal_date": "2025-04-01",
            "status": "pending_escalation",
            "resolution": None
        }
    ]
}
```

This pattern ensures that when the conversation addresses issue #2, the facts from issue #1 are not lost or conflated.

The single-issue variant of this pattern:

```python
# Case facts block pattern
case_facts = {
    "customer_id": "C-12345",
    "issue": "duplicate_charge",
    "amount": "$4,287.53",
    "transaction_date": "2025-03-14",
    "merchant_id": "MRC-99201",
    "status": "investigating",
    "opened": "2025-03-15T10:23:00Z"
}

# Inject into system prompt or prepend to conversation
system_prompt = f"""You are a support agent.

## Active Case Facts
{json.dumps(case_facts, indent=2)}

## Instructions
..."""
```

2. **Trim verbose tool outputs to relevant fields.** Process tool results before injecting them into conversation context:

```python
def trim_tool_result(raw_result, relevant_fields):
    """Extract only relevant fields from tool output."""
    return {k: v for k, v in raw_result.items() if k in relevant_fields}

# Example: CRM lookup only needs these fields for a refund workflow
relevant = trim_tool_result(crm_response, [
    "customer_id", "name", "email", "account_status", "recent_transactions"
])
```

3. **Place key findings at beginning; use section headers.** Structure long contexts with clear headers and place the most critical information at the start:

```xml
<key_findings>
  - Account C-12345 has a duplicate charge of $4,287.53
  - Transaction occurred on 2025-03-14
  - Customer is enterprise tier, eligible for immediate refund
</key_findings>

<investigation_details>
  ## Step 1: Account Lookup
  ...
  ## Step 2: Transaction History
  ...
</investigation_details>
```

4. **Require metadata (dates, sources) in structured outputs.** When agents produce summaries or reports, mandate that they retain metadata:

```python
output_schema = {
    "findings": [
        {
            "claim": "str",
            "source": "str",
            "date": "str (ISO 8601)",
            "confidence": "high | medium | low"
        }
    ]
}
```

5. **Return structured data instead of verbose content from upstream agents.** In multi-agent systems, sub-agents should return concise structured results, not full prose:

```python
# Bad: sub-agent returns 2000-token narrative
# Good: sub-agent returns structured result
{
    "status": "found",
    "matching_records": 1,
    "customer": {
        "id": "C-12345",
        "name": "Jane Smith",
        "account_status": "active"
    },
    "relevant_transactions": [
        {"date": "2025-03-14", "amount": 4287.53, "merchant": "MRC-99201"}
    ]
}
```

### Configuration Example: Conversation Summarization with Fact Extraction

```python
import anthropic

client = anthropic.Anthropic()

def manage_long_conversation(messages, system_prompt, case_facts):
    """
    Manage a conversation that may exceed context limits.
    Preserves case facts while summarizing older turns.
    """
    # Always inject case facts at the top
    enhanced_system = f"""## Active Case Facts (DO NOT LOSE THESE)
{json.dumps(case_facts, indent=2)}

{system_prompt}"""

    # If conversation is getting long, summarize older middle turns
    # but keep first 2 and last 5 turns intact
    if count_tokens(messages) > 150000:
        preserved_start = messages[:2]
        preserved_end = messages[-5:]
        middle = messages[2:-5]

        summary = summarize_turns(middle, case_facts)
        messages = (
            preserved_start
            + [{"role": "user", "content": f"[Summary of prior conversation: {summary}]"}]
            + preserved_end
        )

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=8096,
        system=enhanced_system,
        messages=messages
    )
    return response
```

### Common Exam Traps

- **Trap:** Believing that summarizing the full conversation is always safe. It is not -- progressive summarization destroys transactional facts.
- **Trap:** Assuming Claude equally attends to all parts of a long context. The "lost in the middle" effect means middle content gets less attention.
- **Trap:** Passing raw tool results directly into context without trimming. This wastes tokens and buries relevant data.
- **Trap:** Thinking that simply increasing the context window size solves all problems. Larger windows help but do not eliminate attention degradation.
- **Trap:** Treating context management as optional. For production systems, it is a reliability requirement, not an optimization.

---

## Task 5.2: Escalation and Ambiguity Resolution

### Key Concepts

**Escalation Triggers**

Three primary categories of escalation triggers:

1. **Explicit customer request** -- Customer says "I want to speak to a human" or "transfer me to a manager"
2. **Policy exceptions/gaps** -- The situation falls outside defined policies or the policy is ambiguous
3. **Inability to progress** -- The agent cannot resolve the issue after reasonable attempts

The cardinal rule: **escalate immediately when a customer explicitly demands a human agent.** Never attempt to convince them to stay with the AI, never delay, never add friction.

**Unreliable Escalation Proxies**

Two commonly proposed but unreliable escalation signals:

- **Sentiment-based escalation:** Detecting anger/frustration via sentiment analysis is unreliable. Customers may be calmly describing a serious issue, or may express frustration casually. Sentiment is a noisy signal that produces both false positives and false negatives.
- **Self-reported confidence scores:** Asking the model "how confident are you?" produces poorly calibrated results. Models tend to express high confidence even when wrong, and low confidence does not reliably correlate with actual error rates.

**Multiple Customer Matches**

When a lookup returns multiple possible customer records:
- **Never** use heuristics to guess which customer the user means
- **Never** pick the "most likely" match based on partial data
- **Always** ask for additional identifying information

```
Bad: "I found a Jane Smith in our system. Let me pull up that account..."
     (There were 3 Jane Smiths; agent picked one silently)

Good: "I found multiple customers named Jane Smith. Could you provide
       your email address or account number so I can locate the correct account?"
```

### Practical Skills

1. **Define explicit escalation criteria with few-shot examples.** Do not rely on the model's judgment alone. Provide concrete rules:

```xml
<escalation_policy>
## Immediate Escalation (no delay)
- Customer explicitly requests human agent (e.g., "let me talk to a person",
  "transfer me", "I want a human", "get me a manager")
- Legal threats or regulatory complaints
- Transactions over $10,000 requiring manual approval
- Account security incidents (compromised credentials, unauthorized access)

## Conditional Escalation
- Three failed resolution attempts on the same issue
- Policy gap: no documented procedure for the customer's request
- Customer provides information that contradicts system records

## Do NOT Escalate (handle directly)
- Routine information requests
- Standard refund requests under $500
- Password resets
- FAQ-type questions

## Examples
User: "This is ridiculous, just connect me to someone who can help"
Action: ESCALATE immediately. Acknowledge frustration, then transfer.

User: "I'm a bit frustrated but let's try one more thing"
Action: DO NOT ESCALATE. Continue troubleshooting.

User: "I want to cancel my enterprise contract effective immediately"
Action: ESCALATE. Contract cancellations require human review.
</escalation_policy>
```

2. **Honor explicit human-agent requests immediately:**

```python
ESCALATION_PHRASES = [
    "talk to a human", "speak to a person", "transfer me",
    "get me a manager", "real person", "human agent",
    "speak to someone", "talk to a representative"
]

def check_immediate_escalation(user_message):
    message_lower = user_message.lower()
    for phrase in ESCALATION_PHRASES:
        if phrase in message_lower:
            return {
                "action": "escalate",
                "reason": "explicit_customer_request",
                "response": "I understand you'd like to speak with a human agent. "
                           "Let me transfer you right away."
            }
    return None
```

3. **Acknowledge frustration while offering resolution:**

```xml
<response_pattern>
When a customer expresses frustration:
1. Acknowledge the emotion: "I understand this is frustrating."
2. Do NOT apologize excessively or grovel.
3. Immediately pivot to resolution: "Here's what I can do to help..."
4. Only escalate if the customer reiterates their desire for a human
   AFTER you have offered a concrete resolution path.
   - First expression of frustration -> acknowledge + offer resolution
   - Customer reiterates "no, get me a person" -> escalate immediately
5. If resolution is not possible at all, escalate directly: "Let me connect
   you with a specialist who can resolve this."
</response_pattern>
```

The key distinction: frustration alone does not trigger escalation. Offer a resolution first. If the customer rejects the resolution and reiterates a request for a human, then escalate. This avoids both (a) over-escalating on every frustrated message and (b) stonewalling customers who genuinely want a human.

4. **Escalate when policy is ambiguous or silent.** This includes situations where a customer's request falls into a gap not addressed by existing policy. For example, if your return policy only addresses price matching against your own website but a customer asks for a competitor price match, the policy is silent on this scenario -- escalate rather than improvise:

```python
# In the agent's tool definitions
def handle_policy_gap(situation_description):
    """
    When the agent encounters a situation not covered by policy,
    it should escalate rather than improvise.

    Examples of policy gaps:
    - Customer requests competitor price matching, but policy only
      covers own-site price matching
    - Customer asks for an exception to a policy that has no
      documented exception process
    - New product/service not yet covered by support playbooks
    """
    return {
        "action": "escalate",
        "reason": "policy_gap",
        "context": situation_description,
        "attempted_resolution": None,
        "message_to_human": f"Customer situation not covered by current policy: "
                           f"{situation_description}. Requires human judgment."
    }
```

5. **Ask for additional identifiers on multiple matches:**

```python
def handle_customer_lookup(results):
    if len(results) == 0:
        return {"action": "ask_retry", "message": "No matching customer found. "
                "Could you double-check the spelling or try a different identifier?"}
    elif len(results) == 1:
        return {"action": "proceed", "customer": results[0]}
    else:
        # NEVER guess. ALWAYS ask for disambiguation.
        return {
            "action": "disambiguate",
            "match_count": len(results),
            "message": f"I found {len(results)} customers matching that name. "
                      f"Could you provide your email address, phone number, "
                      f"or account number so I can find the right account?"
        }
```

### Common Exam Traps

- **Trap:** Using sentiment analysis as a primary escalation trigger. Sentiment is unreliable -- use explicit criteria instead.
- **Trap:** Asking the model to self-assess confidence as an escalation signal. Self-reported confidence is poorly calibrated.
- **Trap:** Adding any friction or delay when a customer explicitly requests a human. The correct action is immediate transfer.
- **Trap:** Picking the "most likely" customer from multiple matches. Always ask for additional identifiers.
- **Trap:** Failing to escalate on policy gaps. If the policy does not address the situation, escalate -- do not improvise policy.

---

## Task 5.3: Error Propagation in Multi-Agent Systems

### Key Concepts

**Structured Error Context**

When an agent in a multi-agent pipeline fails, it must communicate rich error context -- not just "something went wrong." The coordinator agent needs to understand what happened to decide on next steps.

Four components of structured error context:
1. **Failure type** -- Categorized error (timeout, auth_failure, rate_limit, invalid_input, not_found, etc.)
2. **Attempted query** -- What was the agent trying to do
3. **Partial results** -- Any data successfully retrieved before failure
4. **Suggested alternatives** -- Fallback actions the coordinator could take

**Access Failures vs. Valid Empty Results**

This distinction is critical and frequently confused:

```
Access failure:    Database connection timed out -> we DON'T KNOW if records exist
Valid empty result: Query succeeded, zero records match -> we KNOW no records exist
```

These require completely different handling. An access failure should trigger retries or fallbacks. A valid empty result is a definitive answer.

**Generic Error Messages Hide Context**

When a sub-agent returns `"search unavailable"` to the coordinator, the coordinator cannot distinguish between:
- Temporary network issue (should retry)
- Authentication expired (should re-auth)
- Invalid query parameters (should reformulate)
- Service permanently deprecated (should use alternative)

The coordinator is forced to either give up or blindly retry -- both suboptimal.

**Anti-Patterns**

1. **Silently suppressing errors:** A sub-agent encounters an error and returns an empty result or a fabricated response instead of reporting the failure. This is the most dangerous anti-pattern because the coordinator proceeds as if the data is valid.

2. **Terminating entire workflow on single failure:** One sub-agent fails and the entire pipeline aborts, even though other sub-agents completed successfully and their results are still valuable.

### Practical Skills

1. **Return structured errors with failure type, attempts, and partial results:**

```python
class AgentError:
    """Structured error for multi-agent communication."""

    def __init__(self, failure_type, query, partial_results=None,
                 alternatives=None, retry_eligible=True):
        self.failure_type = failure_type  # "timeout", "auth", "rate_limit", etc.
        self.query = query
        self.partial_results = partial_results or {}
        self.alternatives = alternatives or []
        self.retry_eligible = retry_eligible
        self.attempts = 0

    def to_dict(self):
        return {
            "status": "error",
            "failure_type": self.failure_type,
            "attempted_query": self.query,
            "attempts_made": self.attempts,
            "partial_results": self.partial_results,
            "suggested_alternatives": self.alternatives,
            "retry_eligible": self.retry_eligible
        }

# Example: sub-agent reporting a partial failure
def search_financial_data(query):
    try:
        sec_data = fetch_sec_filings(query)  # succeeds
    except TimeoutError:
        sec_data = None

    try:
        market_data = fetch_market_data(query)  # succeeds
    except AuthError:
        market_data = None

    if sec_data is None and market_data is None:
        return AgentError(
            failure_type="partial_failure",
            query=query,
            partial_results={},
            alternatives=["Try narrower date range", "Use cached data from last week"]
        ).to_dict()

    return {
        "status": "partial_success" if (sec_data is None or market_data is None) else "success",
        "sec_filings": sec_data,
        "market_data": market_data,
        "coverage_note": "SEC data unavailable due to timeout" if sec_data is None else None
    }
```

2. **Distinguish access failures from valid empty results:**

```python
def search_customer_records(name):
    try:
        results = database.query("SELECT * FROM customers WHERE name = %s", name)

        if len(results) == 0:
            return {
                "status": "success",          # Query executed successfully
                "result_type": "empty",        # No matching records
                "data": [],
                "message": "No customers found matching that name.",
                "is_definitive": True          # This IS the answer
            }
        else:
            return {
                "status": "success",
                "result_type": "found",
                "data": results,
                "is_definitive": True
            }

    except ConnectionError as e:
        return {
            "status": "error",                 # Query did NOT execute
            "result_type": "access_failure",   # We don't know the answer
            "data": [],
            "message": f"Database connection failed: {e}",
            "is_definitive": False,            # This is NOT the answer
            "retry_eligible": True
        }
```

3. **Local recovery for transient failures; propagate only unresolvable errors:**

```python
import time

def resilient_agent_call(agent_fn, query, max_retries=3):
    """
    Retry transient failures locally. Only propagate
    persistent or non-retryable failures to coordinator.
    """
    TRANSIENT_ERRORS = {"timeout", "rate_limit", "connection_reset"}

    for attempt in range(max_retries):
        result = agent_fn(query)

        if result["status"] == "success":
            return result

        if result.get("failure_type") not in TRANSIENT_ERRORS:
            # Non-transient error: propagate immediately
            result["attempts_made"] = attempt + 1
            return result

        # Transient error: retry with backoff
        time.sleep(2 ** attempt)

    # All retries exhausted: propagate with full context
    result["attempts_made"] = max_retries
    result["message"] = f"Failed after {max_retries} retries: {result.get('message')}"
    return result
```

4. **Coverage annotations in synthesis output:**

```python
def synthesize_report(sub_agent_results):
    """
    Coordinator synthesizes results from multiple sub-agents,
    annotating coverage gaps.
    """
    report = {
        "findings": [],
        "coverage": {
            "complete": [],
            "partial": [],
            "unavailable": []
        },
        "confidence_note": ""
    }

    for source, result in sub_agent_results.items():
        if result["status"] == "success":
            report["findings"].extend(result["data"])
            report["coverage"]["complete"].append(source)
        elif result["status"] == "partial_success":
            report["findings"].extend(result.get("data", []))
            report["coverage"]["partial"].append({
                "source": source,
                "note": result.get("coverage_note", "Some data unavailable")
            })
        else:
            report["coverage"]["unavailable"].append({
                "source": source,
                "reason": result.get("message", "Unknown error"),
                "retry_eligible": result.get("retry_eligible", False)
            })

    # Annotate overall confidence based on coverage
    if report["coverage"]["unavailable"]:
        report["confidence_note"] = (
            f"Warning: {len(report['coverage']['unavailable'])} data source(s) "
            f"were unavailable. Findings may be incomplete."
        )

    return report
```

### Common Exam Traps

- **Trap:** Returning a generic "search unavailable" message instead of structured error context. The coordinator needs failure type, partial results, and alternatives.
- **Trap:** Treating access failures and valid empty results the same way. They have opposite meanings: "we don't know" vs. "we know there are none."
- **Trap:** Terminating an entire multi-agent workflow because one sub-agent failed. Partial results from successful agents are still valuable.
- **Trap:** Silently returning empty data on failure. This is the worst anti-pattern -- it causes the coordinator to make decisions on incomplete data without knowing it.
- **Trap:** Always propagating errors to the coordinator. Transient failures (timeouts, rate limits) should be retried locally first.

---

## Task 5.4: Large Codebase Exploration

### Key Concepts

**Context Degradation Signals**

When Claude's context window fills up during codebase exploration, several degradation signals appear:

- **Inconsistent answers:** Giving different answers about the same code when asked again
- **"Typical patterns" substitution:** Saying "this likely follows the typical repository pattern" or "based on common conventions" instead of citing specific classes, methods, or files
- **Hallucinated file paths or class names:** Referencing files that do not exist
- **Vague references:** "the main service class" instead of `OrderProcessingService.java`

When you observe these signals, context is saturated and findings are unreliable.

**Scratchpad Files for Persisting Findings**

Since context resets between sessions (and degrades within long sessions), persisting findings to disk is essential:

```markdown
<!-- /tmp/exploration-scratchpad.md -->
# Codebase Exploration: Payment Service

## Architecture
- Monorepo with 4 services: api-gateway, payment-service, user-service, notification-service
- Payment service uses hexagonal architecture
- Entry point: src/payment/application/PaymentApplicationService.java

## Key Classes Found
- PaymentProcessor (src/payment/domain/PaymentProcessor.java) - core domain logic
- StripeAdapter (src/payment/infrastructure/StripeAdapter.java) - Stripe integration
- PaymentRepository (src/payment/domain/ports/PaymentRepository.java) - port interface

## Open Questions
- [ ] How are refunds handled? (Check RefundService if it exists)
- [ ] What retry policy is used for failed charges?

## Session 2 Findings
- RefundService found at src/payment/domain/RefundService.java
- Uses exponential backoff: max 3 retries, base delay 1s
```

**Subagent Delegation**

For verbose exploration tasks (e.g., "read every file in this directory and summarize"), delegate to a subagent. The subagent does the heavy token-consuming exploration and returns a concise structured summary. The parent agent's context stays clean.

**Structured State Persistence for Crash Recovery (Manifests)**

For long-running automated tasks (migrations, refactors), maintain a manifest file that tracks progress:

```json
{
  "task": "migrate_api_v1_to_v2",
  "started": "2025-03-15T10:00:00Z",
  "total_files": 47,
  "completed": [
    {"file": "src/api/users.py", "status": "migrated", "timestamp": "..."},
    {"file": "src/api/orders.py", "status": "migrated", "timestamp": "..."}
  ],
  "pending": [
    "src/api/payments.py",
    "src/api/inventory.py"
  ],
  "failed": [
    {"file": "src/api/legacy.py", "error": "Circular dependency", "timestamp": "..."}
  ],
  "current": null
}
```

If the process crashes or context resets, the next session reads the manifest and resumes from where it left off.

**The `/compact` Command**

In Claude Code, `/compact` reduces context usage by summarizing the current conversation. This is valuable when:
- Context is filling up during a long exploration session
- You need to continue working but are hitting token limits
- You want to preserve key findings while discarding verbose intermediate outputs

It can also accept an optional prompt to guide the summarization focus (e.g., `/compact focus on the database schema findings`).

### Practical Skills

1. **Spawn subagents for specific questions:**

```python
# Parent agent delegates verbose exploration
def explore_codebase_architecture(repo_path):
    """
    Instead of reading 100 files in the parent context,
    spawn a subagent to do the exploration.
    """
    subagent_task = f"""
    Explore the repository at {repo_path} and return a structured summary:
    1. Top-level directory structure
    2. Primary language(s) and framework(s)
    3. Entry points (main files, API routes)
    4. Key abstractions (services, repositories, controllers)
    5. Configuration files and their purpose

    Return as JSON. Do NOT include file contents, only paths and descriptions.
    """

    # Subagent consumes tokens reading files, but only returns
    # a concise structured summary to the parent
    result = spawn_subagent(subagent_task)
    return result  # Compact structured data, not raw file contents
```

2. **Maintain scratchpad files:**

```python
# Claude Code usage pattern
SCRATCHPAD = "/tmp/project-exploration.md"

def update_scratchpad(section, content):
    """Append findings to persistent scratchpad file."""
    with open(SCRATCHPAD, "a") as f:
        f.write(f"\n## {section}\n")
        f.write(f"_Updated: {datetime.now().isoformat()}_\n\n")
        f.write(content)
        f.write("\n")

# At start of new session, read scratchpad to recover state
def load_scratchpad():
    try:
        with open(SCRATCHPAD, "r") as f:
            return f.read()
    except FileNotFoundError:
        return "No previous exploration data found."
```

3. **Summarize findings before spawning next phase agents:**

```
Phase 1 Agent: Explore directory structure -> writes summary to scratchpad
Phase 2 Agent: Reads scratchpad + explores specific module -> updates scratchpad
Phase 3 Agent: Reads scratchpad + implements changes based on accumulated knowledge
```

Each agent starts with a concise summary from prior phases rather than re-exploring from scratch.

4. **Crash recovery via manifests:**

```python
import json

MANIFEST_PATH = "/tmp/migration-manifest.json"

def load_or_create_manifest(task_name, files):
    """Load existing manifest or create new one."""
    try:
        with open(MANIFEST_PATH, "r") as f:
            manifest = json.load(f)
            print(f"Resuming: {len(manifest['completed'])}/{manifest['total_files']} done")
            return manifest
    except FileNotFoundError:
        manifest = {
            "task": task_name,
            "total_files": len(files),
            "completed": [],
            "pending": files,
            "failed": [],
            "current": None
        }
        save_manifest(manifest)
        return manifest

def save_manifest(manifest):
    with open(MANIFEST_PATH, "w") as f:
        json.dump(manifest, f, indent=2)

def process_next_file(manifest):
    """Process next pending file with crash recovery."""
    if not manifest["pending"]:
        return None

    current_file = manifest["pending"][0]
    manifest["current"] = current_file
    save_manifest(manifest)  # Save BEFORE processing

    try:
        result = migrate_file(current_file)
        manifest["completed"].append({
            "file": current_file,
            "status": "migrated"
        })
    except Exception as e:
        manifest["failed"].append({
            "file": current_file,
            "error": str(e)
        })

    manifest["pending"].remove(current_file)
    manifest["current"] = None
    save_manifest(manifest)  # Save AFTER processing
```

### Common Exam Traps

- **Trap:** Assuming Claude can explore an entire large codebase in a single context window. Context degrades; use subagents and scratchpads.
- **Trap:** Not recognizing degradation signals. When Claude says "this likely follows the typical pattern," it may be fabricating rather than citing actual code.
- **Trap:** Restarting exploration from scratch after a crash. Use manifests to track progress and resume.
- **Trap:** Having the parent agent do all file reading. Delegate verbose exploration to subagents and receive structured summaries.
- **Trap:** Forgetting that `/compact` exists. It is the built-in mechanism for reclaiming context in Claude Code.

---

## Task 5.5: Human Review Workflows and Confidence Calibration

### Key Concepts

**Aggregate Accuracy Masks Specific Failures**

A system with 97% overall accuracy may have:
- 99.5% accuracy on standard invoices
- 94% accuracy on handwritten forms
- 78% accuracy on multi-currency transactions
- 65% accuracy on scanned receipts with poor image quality

If you only measure aggregate accuracy, you will automate document types that have unacceptable error rates. This is one of the most common and costly mistakes in production AI systems.

**Stratified Random Sampling**

To measure true error rates, you must sample across meaningful strata:

- **By document type:** invoices, contracts, receipts, forms
- **By field:** amounts, dates, names, addresses, account numbers
- **By confidence level:** high-confidence vs. low-confidence extractions
- **By source quality:** clean digital vs. scanned vs. photographed

Random sampling without stratification will over-represent common document types and under-represent rare but error-prone types.

**Field-Level Confidence Scores**

Rather than a single confidence score per document, assign confidence per field. This enables surgical human review:

```
Document: Invoice INV-2025-0891
  - vendor_name:   "Acme Corp"        confidence: 0.99  -> auto-accept
  - invoice_date:  "2025-03-14"       confidence: 0.97  -> auto-accept
  - total_amount:  "$12,847.00"       confidence: 0.95  -> auto-accept
  - line_item_3:   "Consulting svc"   confidence: 0.62  -> HUMAN REVIEW
  - tax_id:        "XX-XXXXXXX"       confidence: 0.41  -> HUMAN REVIEW
```

**Validate Accuracy by Document Type AND Field Before Automating**

Before enabling full automation for any document type, you must validate accuracy at both the document-type level and the individual field level. A document type should only be automated when ALL critical fields meet their accuracy thresholds for that specific type. For example, even if "standard invoices" have 99.5% overall accuracy, if the `tax_id` field only achieves 89% accuracy on standard invoices, that field must still be routed to human review until the extraction pipeline is improved.

**Calibration with Labeled Validation Sets**

Confidence scores are only useful if they are calibrated. A score of 0.95 should mean "this is correct 95% of the time." To achieve this:

1. Build a labeled validation set (ground truth data)
2. Run the model on the validation set
3. Group predictions by confidence bucket (0.90-0.95, 0.95-1.00, etc.)
4. Measure actual accuracy in each bucket
5. If 0.95 confidence predictions are only correct 88% of the time, the threshold needs adjustment

### Practical Skills

1. **Stratified sampling of high-confidence extractions:**

```python
def stratified_sample(extractions, sample_size_per_stratum=50):
    """
    Sample extractions for human review, stratified by document type
    and confidence level. Ensures all strata are represented.
    """
    strata = {}
    for ext in extractions:
        key = (ext["document_type"], confidence_bucket(ext["confidence"]))
        strata.setdefault(key, []).append(ext)

    sample = []
    for stratum_key, items in strata.items():
        n = min(sample_size_per_stratum, len(items))
        sample.extend(random.sample(items, n))

    return sample

def confidence_bucket(score):
    if score >= 0.95: return "high"
    elif score >= 0.80: return "medium"
    else: return "low"
```

2. **Analyze accuracy by document type and field:**

```python
def accuracy_report(validation_results):
    """
    Break down accuracy by document type AND field to identify
    specific failure modes hidden by aggregate metrics.
    """
    report = {}
    for result in validation_results:
        doc_type = result["document_type"]
        if doc_type not in report:
            report[doc_type] = {"total": 0, "correct": 0, "fields": {}}

        report[doc_type]["total"] += 1
        all_fields_correct = True

        for field, value in result["extracted_fields"].items():
            if field not in report[doc_type]["fields"]:
                report[doc_type]["fields"][field] = {"total": 0, "correct": 0}

            report[doc_type]["fields"][field]["total"] += 1
            if value == result["ground_truth"].get(field):
                report[doc_type]["fields"][field]["correct"] += 1
            else:
                all_fields_correct = False

        if all_fields_correct:
            report[doc_type]["correct"] += 1

    # Calculate rates
    for doc_type, data in report.items():
        data["accuracy"] = data["correct"] / data["total"]
        for field, fdata in data["fields"].items():
            fdata["accuracy"] = fdata["correct"] / fdata["total"]

    return report

# Example output:
# {
#   "standard_invoice": {"accuracy": 0.995, "fields": {"amount": {"accuracy": 0.998}, ...}},
#   "handwritten_form": {"accuracy": 0.78, "fields": {"amount": {"accuracy": 0.82}, ...}},
#   "scanned_receipt":  {"accuracy": 0.65, "fields": {"amount": {"accuracy": 0.71}, ...}}
# }
```

3. **Field-level confidence with calibrated thresholds:**

```python
# Calibrated thresholds per field (determined from validation set)
FIELD_THRESHOLDS = {
    "vendor_name":   {"auto_accept": 0.92, "auto_reject": 0.30},
    "invoice_date":  {"auto_accept": 0.95, "auto_reject": 0.40},
    "total_amount":  {"auto_accept": 0.97, "auto_reject": 0.50},  # Higher bar for money
    "line_items":    {"auto_accept": 0.90, "auto_reject": 0.35},
    "tax_id":        {"auto_accept": 0.98, "auto_reject": 0.60},  # Very high bar for IDs
}

def route_extraction(extraction):
    """Route each field to auto-accept, human review, or auto-reject."""
    routing = {}
    for field, value_info in extraction["fields"].items():
        conf = value_info["confidence"]
        thresholds = FIELD_THRESHOLDS.get(field, {"auto_accept": 0.95, "auto_reject": 0.40})

        if conf >= thresholds["auto_accept"]:
            routing[field] = "auto_accept"
        elif conf <= thresholds["auto_reject"]:
            routing[field] = "auto_reject"
        else:
            routing[field] = "human_review"

    return routing
```

4. **Route low confidence or ambiguous extractions to human review:**

```python
def create_review_queue(extractions):
    """
    Build a prioritized human review queue.
    High-value low-confidence items first.
    """
    review_items = []

    for ext in extractions:
        routing = route_extraction(ext)
        fields_needing_review = [f for f, r in routing.items() if r == "human_review"]

        if fields_needing_review:
            priority = calculate_review_priority(ext, fields_needing_review)
            review_items.append({
                "document_id": ext["document_id"],
                "document_type": ext["document_type"],
                "fields_to_review": fields_needing_review,
                "priority": priority,
                "context": ext  # Full extraction for reviewer context
            })

    return sorted(review_items, key=lambda x: x["priority"], reverse=True)

def calculate_review_priority(extraction, fields):
    """Higher priority for: financial fields, low confidence, high-value documents."""
    priority = 0
    financial_fields = {"total_amount", "tax_amount", "line_item_amount"}
    for field in fields:
        if field in financial_fields:
            priority += 10  # Financial fields are high priority
        conf = extraction["fields"][field]["confidence"]
        priority += (1 - conf) * 5  # Lower confidence = higher priority
    return priority
```

### Common Exam Traps

- **Trap:** Relying on aggregate accuracy (e.g., 97%) to justify full automation. Always break down by document type and field.
- **Trap:** Using random sampling without stratification. Rare document types with high error rates will be undersampled.
- **Trap:** Trusting raw confidence scores without calibration. Scores must be validated against labeled data.
- **Trap:** Applying the same confidence threshold to all fields. Financial amounts and tax IDs need higher thresholds than names.
- **Trap:** Only reviewing low-confidence outputs. You must also sample high-confidence outputs to verify calibration remains accurate over time.

---

## Task 5.6: Information Provenance and Uncertainty

### Key Concepts

**Source Attribution Lost During Summarization**

When a sub-agent summarizes multiple sources into a single narrative, source attribution is typically lost:

```
Sub-agent input:
  - Source A (WHO 2024): Global prevalence is 8.2%
  - Source B (CDC 2023): US prevalence is 11.3%
  - Source C (Lancet 2024): Prevalence ranges from 6-14% depending on region

Sub-agent output (bad):
  "The prevalence is approximately 8-11%"
  -> Which source? What geography? What year? All lost.
```

**Structured Claim-Source Mappings**

Every factual claim should be traceable to its source:

```json
{
  "claim": "Global diabetes prevalence is 8.2%",
  "source": "WHO Global Diabetes Report",
  "publication_date": "2024-06-15",
  "data_collection_period": "2021-2023",
  "geographic_scope": "global",
  "confidence": "high",
  "url": "https://who.int/..."
}
```

**Handling Conflicting Statistics**

When sources disagree, the correct approach is to present all values with source attribution, not to arbitrarily select one or average them:

```
WRONG: "The market size is approximately $45 billion."
       (Silently picked one of three conflicting estimates)

RIGHT: "Market size estimates vary by source:
        - Gartner (2025): $42.3 billion
        - IDC (2024): $47.1 billion
        - McKinsey (2025): $44.8 billion
        Note: Differences may reflect varying scope definitions
        (Gartner excludes services; IDC includes consulting)."
```

**Temporal Data Requirements**

Data without dates is unreliable. Always require:
- **Publication date:** When was the source published?
- **Data collection date:** When was the underlying data gathered? (Often years before publication)
- **Temporal scope:** Does this represent a point in time or a range?

### Practical Skills

1. **Require claim-source mappings from subagents:**

```python
# System prompt for research sub-agent
RESEARCH_AGENT_PROMPT = """
You are a research sub-agent. For every factual claim in your output,
you MUST provide a source mapping.

Output format (JSON):
{
  "findings": [
    {
      "claim": "The exact factual statement",
      "value": "The specific number or fact",
      "source": {
        "name": "Source document name",
        "author": "Author or organization",
        "publication_date": "YYYY-MM-DD",
        "data_collection_date": "YYYY-MM-DD or range",
        "url": "Full URL to source",
        "excerpt": "Relevant verbatim excerpt from the source supporting this claim",
        "type": "primary_research | secondary_analysis | estimate | projection"
      },
      "confidence": "high | medium | low",
      "notes": "Any caveats or context"
    }
  ],
  "conflicts": [
    {
      "topic": "What the disagreement is about",
      "values": [
        {"value": "X", "source": "Source A"},
        {"value": "Y", "source": "Source B"}
      ],
      "possible_explanation": "Why sources may differ"
    }
  ]
}

NEVER summarize away source information.
NEVER silently pick one value when sources conflict.
ALWAYS include publication dates.
"""
```

2. **Distinguish well-established from contested findings:**

```python
def classify_finding_certainty(finding):
    """
    Classify whether a finding is well-established, emerging,
    contested, or speculative.
    """
    source_count = len(finding.get("corroborating_sources", []))
    has_conflicts = len(finding.get("conflicting_sources", [])) > 0
    source_quality = finding.get("primary_source_type", "unknown")

    if source_count >= 3 and not has_conflicts and source_quality == "primary_research":
        return "well_established"
    elif source_count >= 2 and not has_conflicts:
        return "emerging_consensus"
    elif has_conflicts:
        return "contested"
    elif source_count <= 1:
        return "preliminary"
    else:
        return "uncertain"

# Usage in output
def format_finding(finding):
    certainty = classify_finding_certainty(finding)
    labels = {
        "well_established": "",
        "emerging_consensus": "[Emerging] ",
        "contested": "[Contested - see conflicting sources] ",
        "preliminary": "[Preliminary - single source] ",
        "uncertain": "[Uncertain] "
    }
    return f"{labels[certainty]}{finding['claim']}"
```

3. **Include conflicting values with annotations:**

```python
def format_conflicting_data(topic, values_by_source):
    """
    Present conflicting data points with full attribution
    rather than silently selecting one.
    """
    output = {
        "topic": topic,
        "note": "Multiple sources report different values. "
                "All are included with attribution.",
        "values": []
    }

    for source, value in values_by_source.items():
        output["values"].append({
            "value": value["number"],
            "source": source,
            "date": value["date"],
            "methodology": value.get("methodology", "not specified"),
            "scope": value.get("scope", "not specified")
        })

    # Add reconciliation note if possible
    scopes = set(v.get("scope") for v in values_by_source.values())
    if len(scopes) > 1:
        output["reconciliation_note"] = (
            "Differences may be explained by varying geographic or "
            "definitional scope across sources."
        )

    return output
```

4. **Render content types appropriately:**

```python
def render_output(findings, output_type):
    """
    Different content types require different rendering:
    - Financial data -> tables with precise numbers
    - News/events -> prose with inline citations
    - Technical specs -> structured lists
    - Statistical comparisons -> tables with source columns
    """
    if output_type == "financial":
        return render_financial_table(findings)
    elif output_type == "news":
        return render_prose_with_citations(findings)
    elif output_type == "comparison":
        return render_comparison_table(findings)
    else:
        return render_structured_list(findings)

def render_financial_table(findings):
    """Financial data must be in tables with exact figures, not prose."""
    headers = ["Metric", "Value", "Source", "Date", "Notes"]
    rows = []
    for f in findings:
        rows.append([
            f["claim"],
            f["value"],          # Exact number, never rounded
            f["source"]["name"],
            f["source"]["publication_date"],
            f.get("notes", "")
        ])
    return {"format": "table", "headers": headers, "rows": rows}

def render_prose_with_citations(findings):
    """News content uses inline citations."""
    paragraphs = []
    for f in findings:
        citation = f"({f['source']['name']}, {f['source']['publication_date']})"
        paragraphs.append(f"{f['claim']} {citation}")
    return {"format": "prose", "content": "\n\n".join(paragraphs)}
```

5. **Require and validate publication dates to prevent temporal misinterpretation.** Reject or flag findings that lack temporal context, since undated statistics can be silently stale:

```python
def validate_temporal_metadata(finding):
    """
    Reject findings without publication dates.
    Warn on findings where data collection date is significantly
    older than publication date.
    """
    source = finding.get("source", {})
    pub_date = source.get("publication_date")
    collection_date = source.get("data_collection_date")

    if not pub_date:
        return {
            "valid": False,
            "reason": "Missing publication date -- cannot assess currency of data"
        }

    if pub_date and collection_date:
        # Warn if data is more than 2 years older than publication
        pub_year = int(pub_date[:4])
        coll_year = int(collection_date[:4])
        if pub_year - coll_year > 2:
            return {
                "valid": True,
                "warning": f"Data collected in {coll_year} but published in {pub_year}. "
                          f"Underlying data may be outdated."
            }

    return {"valid": True}
```

### Common Exam Traps

- **Trap:** Allowing sub-agents to summarize without maintaining source attribution. Every claim must be traceable.
- **Trap:** Silently picking one value when sources conflict. Present all values with attribution and explain differences.
- **Trap:** Reporting statistics without publication or data collection dates. Temporal context is mandatory.
- **Trap:** Rendering all content types the same way. Financial data needs tables with exact figures; narrative content needs prose with citations.
- **Trap:** Treating all sources as equally authoritative. Distinguish primary research from secondary analysis, estimates, and projections.
- **Trap:** Averaging conflicting numbers as a "compromise." Each number may reflect a different methodology or scope -- averaging is meaningless.

---

## Cross-Cutting Themes for Domain 5

### The Reliability Stack

Domain 5's six tasks form a coherent reliability stack for production AI systems:

```
Layer 6: Provenance & Uncertainty    (Task 5.6) - Trust in outputs
Layer 5: Human Review & Calibration  (Task 5.5) - Accuracy assurance
Layer 4: Codebase Exploration        (Task 5.4) - Working with large state
Layer 3: Error Propagation           (Task 5.3) - Failure resilience
Layer 2: Escalation & Ambiguity      (Task 5.2) - Knowing limits
Layer 1: Context Preservation        (Task 5.1) - Information integrity
```

### Key Principles Across All Tasks

1. **Structure over prose.** Structured data (JSON, typed errors, claim-source mappings) is more reliable than natural language for inter-agent communication.

2. **Explicit over implicit.** Explicit escalation criteria beat sentiment analysis. Explicit error types beat generic messages. Explicit source attribution beats embedded prose.

3. **Specific over aggregate.** Field-level confidence over document-level. Per-document-type accuracy over aggregate accuracy. Specific file references over "typical patterns."

4. **Preserve over summarize.** Transactional facts must survive summarization. Source attribution must survive synthesis. Error context must survive propagation.

5. **Partial success over total failure.** Report what worked alongside what failed. Return partial results with coverage annotations. Degrade gracefully.

### Quick Reference: Anti-Patterns to Recognize on the Exam

| Anti-Pattern | Task | Correct Approach |
|---|---|---|
| Summarizing away dollar amounts and dates | 5.1 | Extract into persistent case facts block |
| Using sentiment for escalation decisions | 5.2 | Use explicit rule-based escalation criteria |
| Returning "search unavailable" without context | 5.3 | Return structured error with type, partial results, alternatives |
| Reading 100 files in parent agent context | 5.4 | Delegate to subagent, receive structured summary |
| Reporting 97% aggregate accuracy | 5.5 | Break down by document type, field, and confidence bucket |
| Averaging conflicting statistics | 5.6 | Present all values with source attribution |
| Guessing which customer from multiple matches | 5.2 | Ask for additional identifying information |
| Silently suppressing sub-agent errors | 5.3 | Propagate structured errors to coordinator |
| Trusting uncalibrated confidence scores | 5.5 | Validate against labeled data per confidence bucket |
| Losing source attribution in summaries | 5.6 | Require claim-source mappings from all agents |

---

## Exam Preparation Checklist

- [ ] Can you explain why progressive summarization is dangerous for transactional data?
- [ ] Can you describe the "lost in the middle" effect and how to mitigate it?
- [ ] Can you list three categories of escalation triggers?
- [ ] Can you explain why sentiment-based escalation is unreliable?
- [ ] Can you distinguish between access failures and valid empty results?
- [ ] Can you design a structured error return format for multi-agent systems?
- [ ] Can you recognize context degradation signals in codebase exploration?
- [ ] Can you explain the role of scratchpad files and manifests?
- [ ] Can you explain why aggregate accuracy can be misleading?
- [ ] Can you design a stratified sampling strategy for human review?
- [ ] Can you explain field-level confidence calibration?
- [ ] Can you design a claim-source mapping schema?
- [ ] Can you explain the correct way to handle conflicting statistics?
- [ ] Can you identify all 10 anti-patterns in the quick reference table above?
- [ ] Can you explain why accuracy must be validated by document type AND field before automating?
- [ ] Can you design a multi-issue session tracking schema?
- [ ] Can you explain the reiteration pattern for frustration-based escalation?

---

## Preparation Exercise 1: Multi-Tool Agent with Escalation Logic

**Scenario:** Design a customer support agent that handles order inquiries, refund processing, and account management. The agent has access to tools for CRM lookup, order history, and refund processing.

**Requirements to implement:**

1. **Escalation triggers with few-shot examples.** Define explicit escalation criteria in the system prompt covering:
   - Immediate escalation: explicit human requests, legal threats, transactions over $10,000
   - Conditional escalation: 3+ failed resolution attempts, policy gaps (e.g., competitor price match requests when policy only covers own-site matching), contradictory system records
   - Non-escalation: routine inquiries, standard refunds under threshold, FAQ questions

2. **Multi-match disambiguation.** When CRM lookup returns multiple customers, request additional identifiers rather than selecting heuristically.

3. **Frustration handling with reiteration logic.** On first frustration expression, acknowledge and offer resolution. Escalate only if the customer reiterates a request for a human after receiving a resolution offer.

4. **Tool result trimming.** Strip CRM and order history responses to relevant fields before injecting into context.

5. **Case facts persistence.** Maintain a structured case facts block that survives conversation summarization, including multi-issue tracking when customers raise more than one concern.

**Design pattern:**

```python
import anthropic
import json

client = anthropic.Anthropic()

def build_support_agent(case_facts, escalation_policy):
    """
    Build a support agent with escalation logic, tool trimming,
    and case facts persistence.
    """
    system_prompt = f"""You are a customer support agent.

## Active Case Facts (PRESERVE THESE EXACTLY)
{json.dumps(case_facts, indent=2)}

## Escalation Policy
{escalation_policy}

## Rules
- If the customer explicitly asks for a human, escalate IMMEDIATELY
- If frustrated but not requesting a human, acknowledge and offer resolution
- If the customer reiterates human request after your resolution offer, escalate
- If the situation is not covered by policy, escalate as policy_gap
- When multiple customers match a lookup, ask for additional identifiers
- NEVER guess which customer from multiple matches
- Update case facts as new information emerges
"""

    tools = [
        {
            "name": "crm_lookup",
            "description": "Look up customer by name, email, or account number",
            "input_schema": {
                "type": "object",
                "properties": {
                    "query": {"type": "string"},
                    "identifier_type": {"type": "string",
                                       "enum": ["name", "email", "account_number"]}
                },
                "required": ["query", "identifier_type"]
            }
        },
        {
            "name": "process_refund",
            "description": "Process a refund for a specific transaction",
            "input_schema": {
                "type": "object",
                "properties": {
                    "transaction_id": {"type": "string"},
                    "amount": {"type": "number"},
                    "reason": {"type": "string"}
                },
                "required": ["transaction_id", "amount", "reason"]
            }
        },
        {
            "name": "escalate_to_human",
            "description": "Transfer the conversation to a human agent",
            "input_schema": {
                "type": "object",
                "properties": {
                    "reason": {"type": "string",
                              "enum": ["explicit_request", "policy_gap",
                                      "resolution_failure", "high_value",
                                      "security_incident"]},
                    "context": {"type": "string"},
                    "case_facts": {"type": "object"}
                },
                "required": ["reason", "context", "case_facts"]
            }
        }
    ]

    return system_prompt, tools


def handle_tool_result(tool_name, raw_result):
    """Trim tool results to relevant fields before returning to context."""
    RELEVANT_FIELDS = {
        "crm_lookup": ["customer_id", "name", "email", "account_status",
                       "account_tier", "recent_transactions"],
        "process_refund": ["refund_id", "status", "amount", "estimated_arrival"],
    }
    fields = RELEVANT_FIELDS.get(tool_name, None)
    if fields and isinstance(raw_result, dict):
        return {k: v for k, v in raw_result.items() if k in fields}
    return raw_result
```

**What this exercise tests:**
- Integration of escalation criteria (Tasks 5.1, 5.2)
- Tool result management (Task 5.1)
- Policy gap recognition (Task 5.2)
- Multi-match disambiguation (Task 5.2)
- Case facts preservation across turns (Task 5.1)

---

## Preparation Exercise 4: Multi-Agent Research Pipeline with Provenance

**Scenario:** Design a multi-agent research system where a coordinator dispatches research sub-agents to gather information from different source types (academic databases, news APIs, financial data providers), then synthesizes results with full provenance tracking.

**Requirements to implement:**

1. **Claim-source mappings from sub-agents.** Each sub-agent must return structured findings with source name, URL, excerpt, publication date, and data collection date.

2. **Error propagation with partial results.** When one sub-agent fails (e.g., academic database timeout), the coordinator continues with results from other sub-agents and annotates which sources were unavailable.

3. **Conflicting data handling.** When sub-agents return conflicting statistics, present all values with source attribution and possible explanations for discrepancies.

4. **Temporal validation.** Reject or flag findings that lack publication dates. Warn when data collection dates are significantly older than publication dates.

5. **Coverage annotations.** The final synthesis must indicate which findings are well-supported (multiple concordant sources) vs. gaps (single source or contested).

6. **Content-type-appropriate rendering.** Financial data rendered as tables, news as prose with citations, technical specs as structured lists.

**Design pattern:**

```python
import anthropic
import json

client = anthropic.Anthropic()

# Sub-agent prompt template enforcing provenance
SUBAGENT_PROMPT = """You are a research sub-agent specializing in {domain}.

For EVERY factual claim, return a structured mapping:
{{
  "claim": "exact statement",
  "value": "specific number or fact",
  "source": {{
    "name": "document or source name",
    "url": "full URL",
    "excerpt": "verbatim excerpt supporting this claim",
    "publication_date": "YYYY-MM-DD",
    "data_collection_date": "YYYY-MM-DD or range",
    "type": "primary_research | secondary_analysis | estimate | projection"
  }},
  "confidence": "high | medium | low"
}}

NEVER omit publication dates.
NEVER summarize away source details.
If sources conflict, return ALL values with separate source mappings.
"""


def run_research_pipeline(query, sources):
    """
    Coordinator dispatches sub-agents and synthesizes with provenance.
    """
    results = {}

    for source_type, config in sources.items():
        try:
            agent_result = dispatch_subagent(
                prompt=SUBAGENT_PROMPT.format(domain=source_type),
                query=query,
                config=config
            )
            results[source_type] = {
                "status": "success",
                "findings": agent_result["findings"],
                "conflicts": agent_result.get("conflicts", [])
            }
        except TimeoutError:
            results[source_type] = {
                "status": "error",
                "failure_type": "timeout",
                "attempted_query": query,
                "partial_results": {},
                "retry_eligible": True
            }
        except Exception as e:
            results[source_type] = {
                "status": "error",
                "failure_type": "unknown",
                "attempted_query": query,
                "message": str(e),
                "retry_eligible": False
            }

    return synthesize_with_provenance(results)


def synthesize_with_provenance(results):
    """
    Synthesize findings from all sub-agents with full provenance,
    coverage annotations, and conflict reporting.
    """
    synthesis = {
        "findings": [],
        "conflicts": [],
        "coverage": {
            "well_supported": [],   # Multiple concordant sources
            "single_source": [],    # Only one source (preliminary)
            "contested": [],        # Sources disagree
            "gaps": []              # Sources unavailable
        },
        "temporal_warnings": [],
        "source_inventory": []
    }

    all_findings = []
    for source_type, result in results.items():
        if result["status"] == "success":
            for finding in result["findings"]:
                finding["source_type"] = source_type
                # Validate temporal metadata
                temporal_check = validate_temporal_metadata(finding)
                if not temporal_check["valid"]:
                    synthesis["temporal_warnings"].append({
                        "finding": finding["claim"],
                        "warning": temporal_check["reason"]
                    })
                elif "warning" in temporal_check:
                    synthesis["temporal_warnings"].append({
                        "finding": finding["claim"],
                        "warning": temporal_check["warning"]
                    })
                all_findings.append(finding)

            synthesis["conflicts"].extend(result.get("conflicts", []))
            synthesis["source_inventory"].append({
                "type": source_type, "status": "available"
            })
        else:
            synthesis["coverage"]["gaps"].append({
                "source_type": source_type,
                "reason": result.get("message", result.get("failure_type")),
                "retry_eligible": result.get("retry_eligible", False)
            })
            synthesis["source_inventory"].append({
                "type": source_type, "status": "unavailable",
                "reason": result.get("failure_type")
            })

    # Classify support level for each finding
    for finding in all_findings:
        corroborating = count_corroborating_sources(finding, all_findings)
        if corroborating >= 2:
            synthesis["coverage"]["well_supported"].append(finding["claim"])
        else:
            synthesis["coverage"]["single_source"].append(finding["claim"])
        synthesis["findings"].append(finding)

    # Add contested findings from conflicts
    for conflict in synthesis["conflicts"]:
        synthesis["coverage"]["contested"].append(conflict["topic"])

    return synthesis


def render_synthesis(synthesis, content_types):
    """Render each section of the synthesis in the appropriate format."""
    rendered = {}
    for content_type, findings in content_types.items():
        if content_type == "financial":
            rendered[content_type] = render_financial_table(findings)
        elif content_type == "news":
            rendered[content_type] = render_prose_with_citations(findings)
        elif content_type == "technical":
            rendered[content_type] = render_structured_list(findings)
    return rendered
```

**What this exercise tests:**
- Claim-source mapping enforcement (Task 5.6)
- Error propagation with structured context (Task 5.3)
- Access failures vs. valid empty results (Task 5.3)
- Partial results with coverage annotations (Task 5.3)
- Temporal validation (Task 5.6)
- Conflicting data presentation (Task 5.6)
- Content-type-appropriate rendering (Task 5.6)
- Well-established vs. contested classification (Task 5.6)

---

## Source Links

- Anthropic Messages API documentation -- conversation history management: https://docs.anthropic.com/en/api/messages
- Anthropic extended thinking and context window guidance: https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking
- Anthropic prompt engineering -- long context window tips: https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/long-context-tips
- Claude Code /compact documentation: https://docs.anthropic.com/en/docs/claude-code/overview
- Anthropic multi-agent orchestration patterns: https://docs.anthropic.com/en/docs/build-with-claude/agentic-systems
- Anthropic human-in-the-loop patterns: https://docs.anthropic.com/en/docs/build-with-claude/human-in-the-loop
- Anthropic tool use and error handling: https://docs.anthropic.com/en/docs/build-with-claude/tool-use
- "Lost in the Middle" research (Liu et al., 2023): https://arxiv.org/abs/2307.03172
- Claude Agent SDK documentation: https://github.com/anthropics/anthropic-sdk-python
