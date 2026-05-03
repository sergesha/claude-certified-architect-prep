# Domain 2: Tool Design & MCP Integration (18%)

> **Exam Weight:** 18% — the second-heaviest domain. Expect 9-10 questions covering tool description quality, MCP error handling, agent tool scoping, MCP server configuration, and built-in tool selection.

---

## Task 2.1: Tool Interface Design

### Key Concepts

**Tool descriptions are the primary mechanism by which the LLM selects which tool to call.** Claude does not "see" your code or understand tool internals — it reads the `name`, `description`, and `input_schema` fields in the tool definition JSON and decides based solely on that text.

#### Why Descriptions Matter

| Description Quality | Outcome |
|---|---|
| Minimal / vague (`"Analyzes stuff"`) | Unreliable selection; Claude guesses or picks the wrong tool |
| Verbose but ambiguous | Claude hesitates or calls multiple tools unnecessarily |
| Clear, differentiated, action-oriented | Reliable, deterministic tool selection |

**Core principle:** If two tools have overlapping descriptions, Claude cannot reliably distinguish them. You must either:
1. **Rename** one tool to make its purpose self-evident from the name alone
2. **Split** one tool into more specific sub-tools
3. **Merge** the tools if they truly do the same thing
4. **Differentiate descriptions** by specifying the exact input type, use case, and output format

#### Differentiating Similar Tools

Consider `analyze_content` vs `analyze_document`:
- Bad: Both described as "Analyzes content and returns insights"
- Good: `analyze_content` → "Analyzes short-form text (tweets, comments, chat messages) for sentiment and tone. Input: raw text string under 1000 chars."
- Good: `analyze_document` → "Analyzes structured documents (PDFs, reports, contracts) for key entities, clauses, and summaries. Input: document URL or base64-encoded file."

#### System Prompt Keyword Sensitivity

Claude's tool selection is influenced by keywords in the system prompt. If the system prompt says "Always analyze documents thoroughly," Claude is more likely to reach for a tool with "analyze" and "document" in its name/description. This can be leveraged intentionally or can cause unintended bias.

**Best practices:**
- Align system prompt terminology with your preferred tool names
- Avoid accidentally "priming" Claude toward the wrong tool by using its exact name in unrelated instructions
- Use the system prompt to provide explicit tool-selection guidance when ambiguity exists (e.g., "Use `search_web` for real-time information and `search_knowledge_base` for internal company data")

### Code Example: Good vs Bad Tool Descriptions

```json
{
  "tools": [
    {
      "name": "get_weather",
      "description": "Gets weather",
      "input_schema": {
        "type": "object",
        "properties": {
          "loc": { "type": "string" }
        }
      }
    }
  ]
}
```
**Why this is BAD:**
- `"Gets weather"` — no detail on what kind of weather data, format, or time range
- `"loc"` — cryptic parameter name; Claude may pass city name, ZIP code, or coordinates inconsistently
- No `required` array
- No property descriptions

```json
{
  "tools": [
    {
      "name": "get_current_weather",
      "description": "Retrieves the current weather conditions for a specific city. Returns temperature (Fahrenheit), humidity (%), wind speed (mph), and a short text summary. Use this for present-moment weather; use get_weather_forecast for future predictions.",
      "input_schema": {
        "type": "object",
        "properties": {
          "city": {
            "type": "string",
            "description": "The city name, e.g. 'San Francisco' or 'London, UK'. Must be a recognizable city name, not a ZIP code or coordinates."
          },
          "units": {
            "type": "string",
            "enum": ["fahrenheit", "celsius"],
            "description": "Temperature unit preference. Defaults to fahrenheit if not specified."
          }
        },
        "required": ["city"]
      }
    }
  ]
}
```
**Why this is GOOD:**
- Name is specific: `get_current_weather` (not just `get_weather`)
- Description states exactly what is returned (temperature, humidity, wind, summary)
- Description explicitly differentiates from a sibling tool (`get_weather_forecast`)
- Parameter names are self-documenting (`city` not `loc`)
- Each property has a description with examples and constraints
- `required` array is specified
- Enum constrains valid values

#### Description Checklist: What to Include

Every tool description should address these dimensions:

1. **Input formats** — what types/shapes of data does it accept? (e.g., "Accepts a JSON object with `url` string field")
2. **Example queries** — when should the LLM call this? (e.g., "Use when the user asks for current stock prices")
3. **Edge cases** — what happens with unusual input? (e.g., "Returns null if the ticker symbol is not found")
4. **Boundaries** — what does this tool NOT do? (e.g., "Does not support historical price data — use `get_price_history` instead")

```json
{
  "name": "search_products",
  "description": "Searches the product catalog by keyword. Input: JSON with 'query' (string, 1-200 chars) and optional 'category' (enum: electronics, clothing, food). Returns top 10 matches sorted by relevance. Example: {\"query\": \"wireless headphones\", \"category\": \"electronics\"}. Edge case: returns empty array if no matches found (not an error). Does NOT search order history — use search_orders for that.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Search keyword, 1-200 characters. E.g., 'wireless headphones'."
      },
      "category": {
        "type": "string",
        "enum": ["electronics", "clothing", "food"],
        "description": "Optional category filter."
      }
    },
    "required": ["query"]
  }
}
```

#### Resolving Ambiguous/Overlapping Descriptions — Skills Checklist

When tool mis-selection occurs due to overlapping descriptions, apply these remediation skills in order:

1. **Differentiate descriptions** — add explicit "Use this when... NOT for..." language
2. **Rename tools** — make the name self-evident (e.g., `analyze` -> `analyze_sentiment` vs `analyze_contract`)
3. **Split tools** — break a broad tool into narrow, purpose-specific tools
4. **Review system prompts for keyword issues** — search for words that match tool names/descriptions and may unintentionally bias selection. For example, a system prompt saying "analyze each customer request carefully" primes Claude toward any tool with "analyze" in its name, even if that tool is irrelevant.

```python
# Example: System prompt keyword audit
system_prompt = "You help customers analyze their spending and find savings."
tool_names = ["analyze_spending", "search_products", "create_budget"]

# Problem: "analyze" in system prompt biases toward analyze_spending
# even when user asks "find me a good deal" (should route to search_products)

# Fix: Rephrase system prompt to neutral language
system_prompt_fixed = "You help customers understand their spending and find savings."
# Or add explicit routing guidance:
system_prompt_fixed = (
    "You help customers with their finances. "
    "Use analyze_spending ONLY when the user asks about past transactions. "
    "Use search_products when the user wants to find or compare items."
)
```

### Practical Skills

- Write tool descriptions that pass the "substitution test": could another tool reasonably match this description? If yes, make it more specific.
- When you have 2+ tools that overlap, always add a "Use this tool when... Use [other tool] when..." sentence.
- Parameter descriptions should include: data type, format, example value, and constraints.
- Audit system prompts for keywords that overlap with tool names — rephrase or add explicit routing instructions.
- Include input formats, example queries, edge cases, and boundaries in every tool description.

### Common Exam Traps

1. **Trap:** "Adding more parameters makes tool selection more reliable." — **False.** Claude selects tools based on the top-level `description`, not parameter count.
2. **Trap:** "Tool names don't matter if descriptions are good." — **False.** Names are the first signal Claude uses; a well-named tool with a mediocre description often outperforms a poorly-named tool with a great description.
3. **Trap:** "You should always provide the most detailed description possible." — **Nuanced.** Overly long descriptions can dilute the key differentiator. Be precise, not exhaustive.
4. **Trap:** "Keyword overlap between system prompts and tool descriptions doesn't matter." — **False.** System prompt keywords create unintended tool associations. If the system prompt says "analyze every request," Claude is primed toward any tool with "analyze" in its name.

---

## Task 2.2: Structured Error Responses for MCP Tools

### Key Concepts

#### The MCP `isError` Flag

In the Model Context Protocol, when a tool returns a result, it includes an `isError` boolean field:

```json
{
  "content": [
    {
      "type": "text",
      "text": "Error details here"
    }
  ],
  "isError": true
}
```

- `isError: true` → signals to the LLM that the tool call failed and the content describes the error
- `isError: false` (or omitted) → signals success; content is the actual result
- **Critical distinction:** When `isError` is `true`, the LLM knows to attempt recovery, retry, or inform the user — rather than treating error text as a valid result

#### Error Categories

Structured errors should be categorized so the LLM can take appropriate action:

| Category | Description | LLM Action |
|---|---|---|
| **transient** | Temporary failure (timeout, rate limit, network blip) | Retry after brief delay |
| **validation** | Invalid input parameters | Fix parameters and retry |
| **business** | Business logic violation (insufficient funds, duplicate entry) | Inform user, do not retry |
| **permission** | Authentication/authorization failure | Inform user, do not retry; may need credential refresh |

#### Why Uniform Errors Prevent Recovery

If every error returns `"Operation failed"` with no structured metadata, the LLM cannot:
- Distinguish retryable from permanent failures
- Know which parameter was invalid
- Suggest a corrective action to the user
- Decide whether to try an alternative tool

**Uniform errors force the LLM into a generic "sorry, that didn't work" response** — the worst possible user experience.

### Code Example: Structured Error Response in MCP Tool

```typescript
// MCP Tool handler implementation
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "create_invoice") {
    try {
      // Validation
      if (!args.customer_id) {
        return {
          content: [
            {
              type: "text",
              text: JSON.stringify({
                error: "Missing required field: customer_id",
                errorCategory: "validation",
                isRetryable: true,
                field: "customer_id",
                humanDescription: "Please provide a customer ID to create the invoice."
              })
            }
          ],
          isError: true
        };
      }

      // Business logic check
      const customer = await db.getCustomer(args.customer_id);
      if (customer.status === "suspended") {
        return {
          content: [
            {
              type: "text",
              text: JSON.stringify({
                error: "Customer account is suspended",
                errorCategory: "business",
                isRetryable: false,
                customerId: args.customer_id,
                humanDescription: "This customer's account is suspended. Contact billing support to resolve."
              })
            }
          ],
          isError: true
        };
      }

      // Transient failure
      const result = await externalApi.createInvoice(args);
      return {
        content: [{ type: "text", text: JSON.stringify(result) }],
        isError: false
      };

    } catch (err) {
      if (err.code === "RATE_LIMITED") {
        return {
          content: [
            {
              type: "text",
              text: JSON.stringify({
                error: "Rate limited by billing API",
                errorCategory: "transient",
                isRetryable: true,
                retryAfterSeconds: err.retryAfter || 5,
                humanDescription: "The billing service is temporarily busy. Please try again in a few seconds."
              })
            }
          ],
          isError: true
        };
      }

      // Permission error
      if (err.code === "UNAUTHORIZED") {
        return {
          content: [
            {
              type: "text",
              text: JSON.stringify({
                error: "Authentication token expired",
                errorCategory: "permission",
                isRetryable: false,
                humanDescription: "The API credentials have expired. Please re-authenticate."
              })
            }
          ],
          isError: true
        };
      }

      // Fallback — still structured, never just "Operation failed"
      return {
        content: [
          {
            type: "text",
            text: JSON.stringify({
              error: err.message,
              errorCategory: "transient",
              isRetryable: true,
              humanDescription: `An unexpected error occurred: ${err.message}. It may be worth retrying.`
            })
          }
        ],
        isError: true
      };
    }
  }
});
```

#### Business Violations: `isRetryable: false` + Customer-Friendly Explanations

Business logic errors are a special category. They are **never retryable** by the agent because the problem is in the business state, not in the request. The error response must include a customer-friendly explanation that the LLM can relay directly to the user:

```typescript
// Business violation example: duplicate order
return {
  content: [{
    type: "text",
    text: JSON.stringify({
      error: "Duplicate order detected",
      errorCategory: "business",
      isRetryable: false,  // Critical: agent must NOT retry
      humanDescription: "An order with these exact items was already placed 5 minutes ago. If this is intentional, please confirm and we will process it as a separate order.",
      suggestedAction: "confirm_duplicate_order"
    })
  }],
  isError: true
};
```

The LLM receives `isRetryable: false` and knows to present the `humanDescription` to the user rather than silently retrying.

#### Local Error Recovery in Subagents

In multi-agent architectures, subagents should handle recoverable errors locally and only propagate unresolvable errors to the orchestrator:

```typescript
// Subagent: research_agent handles transient errors locally
async function researchAgentLoop(task: string) {
  const maxRetries = 3;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    const result = await callTool("search_web", { query: task });
    const parsed = JSON.parse(result.content[0].text);

    if (!result.isError) {
      return result; // Success — return to orchestrator
    }

    if (parsed.errorCategory === "transient" && parsed.isRetryable) {
      // Local recovery: retry without bothering the orchestrator
      await sleep(parsed.retryAfterSeconds || 2);
      continue;
    }

    if (parsed.errorCategory === "validation") {
      // Local recovery: fix the query and retry
      task = await reformulateQuery(task, parsed);
      continue;
    }

    // Unresolvable errors (business, permission) — propagate up
    return {
      status: "error",
      errorCategory: parsed.errorCategory,
      humanDescription: parsed.humanDescription,
      originalError: parsed
    };
  }

  // All retries exhausted — propagate to orchestrator
  return {
    status: "error",
    errorCategory: "transient",
    humanDescription: "Search service unavailable after multiple attempts."
  };
}
```

**Key principle:** The orchestrator should not see transient errors that the subagent can handle. Only business logic violations, permission failures, and exhausted-retry scenarios should bubble up.

#### Distinguishing Access Failures from Valid Empty Results

A critical design pattern: **a 403 Forbidden is NOT the same as an empty result set.** If your tool conflates these, the LLM cannot tell the user why they got nothing back.

```typescript
// BAD: Both cases return the same thing
return { content: [{ type: "text", text: "No results found" }], isError: false };

// GOOD: Access failure is an error with proper category
if (response.status === 403) {
  return {
    content: [{
      type: "text",
      text: JSON.stringify({
        error: "Access denied to this resource",
        errorCategory: "permission",
        isRetryable: false,
        humanDescription: "You do not have permission to access this dataset. Contact your admin for access."
      })
    }],
    isError: true  // This IS an error
  };
}

// GOOD: Empty result is a successful response, not an error
if (results.length === 0) {
  return {
    content: [{
      type: "text",
      text: JSON.stringify({
        results: [],
        resultCount: 0,
        message: "No matching records found for the given criteria."
      })
    }],
    isError: false  // This is NOT an error — the query succeeded, just found nothing
  };
}
```

### Practical Skills

- Always set `isError: true` on failures — never return error text as a "successful" result
- Include `errorCategory` so the LLM knows the class of failure
- Include `isRetryable` so the LLM knows whether to attempt again
- Include `humanDescription` for user-facing messaging
- For validation errors, specify which field(s) failed and why
- For business violations, always set `isRetryable: false` and provide a customer-friendly explanation
- In subagents, handle transient and validation errors locally; only propagate business and permission errors to the orchestrator
- Never conflate access failures (403) with valid empty results (200 with empty list) — they require different `isError` values and different user messaging

### Common Exam Traps

1. **Trap:** "Returning error messages in the content text with `isError: false` is fine because the LLM can read it." — **False.** Without `isError: true`, the LLM may treat error text as a valid response and present it as a result.
2. **Trap:** "All errors should be retryable." — **False.** Business logic violations and permission errors should NOT be retried automatically.
3. **Trap:** "MCP error handling is the same as HTTP status codes." — **False.** MCP uses `isError` as a boolean flag in the tool result, not numeric status codes. The error categorization is in the content payload.
4. **Trap:** "Subagents should always propagate errors to the orchestrator." — **False.** Transient and validation errors should be recovered locally within the subagent. Only unresolvable errors (business, permission, exhausted retries) should propagate.
5. **Trap:** "An empty result set and an access-denied response can be handled the same way." — **False.** Access denied (`isError: true`, category `permission`) is fundamentally different from an empty result (`isError: false`, successful query that returned zero items).

---

## Task 2.3: Tool Distribution Across Agents

### Key Concepts

#### The Tool Overload Problem

Research and practice show that **giving an agent too many tools degrades selection reliability**:

| Tool Count | Selection Reliability |
|---|---|
| 4-5 tools | High — Claude can reliably differentiate and select |
| 8-10 tools | Moderate — starts to confuse similar tools |
| 15-18+ tools | Low — frequent mis-selection, hallucinated parameters |

**Why?** Each tool definition consumes context. With 18 tools, Claude must read and compare 18 descriptions on every turn. Overlapping descriptions compound the problem.

#### Scoped Tool Access Per Agent Role

The solution is to **distribute tools across specialized agents**, each with a focused set:

- **Research Agent:** `search_web`, `search_knowledge_base`, `read_document` (3 tools)
- **Data Agent:** `query_database`, `run_analysis`, `create_chart` (3 tools)
- **Communication Agent:** `send_email`, `send_slack`, `schedule_meeting` (3 tools)
- **Orchestrator Agent:** `delegate_to_research`, `delegate_to_data`, `delegate_to_communication` (3 tools)

Each agent sees only its own tools, keeping the count in the reliable 3-5 range.

#### `tool_choice` Options

The `tool_choice` parameter in the Anthropic API controls how Claude interacts with tools:

| Value | Behavior | Use When |
|---|---|---|
| `{"type": "auto"}` | Claude decides whether to call a tool or respond with text | Default for most conversational flows; Claude may choose not to use any tool |
| `{"type": "any"}` | Claude MUST call one of the available tools (cannot respond with text only) | When every turn must produce a tool call (e.g., in an agentic loop where text-only responses are invalid) |
| `{"type": "tool", "name": "specific_tool"}` | Claude MUST call this specific tool | When you know exactly which tool should be used next (e.g., after validation, force a `submit_form` call); eliminates selection ambiguity |

**Important behavioral notes:**
- `auto` is the default if `tool_choice` is not specified
- `any` forces a tool call but Claude still chooses WHICH tool — **use this to guarantee a tool call over conversational text** (e.g., in an agentic loop where a text-only response would break the pipeline)
- Forced tool (`type: "tool"`) bypasses tool selection entirely — useful for deterministic pipelines
- With forced tool, Claude still generates the parameters based on conversation context

#### Cross-Specialization Misuse

When agents have access to tools outside their domain, they may misuse them. For example, a "Research Agent" given access to `send_email` might decide to email a source for information instead of searching the web. This is **cross-specialization misuse** — an agent using tools that belong to another agent's domain.

**Prevention:** Each agent should only have tools scoped to its role. A limited set of cross-role tools (e.g., `log_activity`) may be shared, but action tools should be isolated per specialization.

#### Replace Generic Tools with Constrained Alternatives

Generic tools invite misuse. Replace them with purpose-scoped alternatives:

| Generic Tool | Problem | Constrained Alternative |
|---|---|---|
| `fetch_url` | Agent may fetch arbitrary URLs, including malicious ones | `load_document(doc_id)` — only loads from approved document store |
| `run_query` | Agent may run any SQL, including DROP TABLE | `get_customer_orders(customer_id)` — parameterized, read-only |
| `call_api` | Agent may call any endpoint | `get_inventory(sku)` — scoped to a single safe operation |

```typescript
// BAD: Generic tool that can do anything
const genericTools = [{
  name: "fetch_url",
  description: "Fetches any URL and returns the content",
  input_schema: {
    type: "object",
    properties: {
      url: { type: "string", description: "Any URL to fetch" }
    },
    required: ["url"]
  }
}];

// GOOD: Constrained alternative scoped to approved sources
const constrainedTools = [{
  name: "load_document",
  description: "Loads a document from the approved company document store by ID. Use for retrieving internal policies, procedures, and reference materials. Does NOT fetch arbitrary URLs.",
  input_schema: {
    type: "object",
    properties: {
      doc_id: {
        type: "string",
        description: "Document ID from the company doc store, e.g. 'DOC-2024-0142'"
      }
    },
    required: ["doc_id"]
  }
}];
```

#### Pipeline Pattern: Forced First Step, Then Auto

A common pattern is to force the first tool call (to ensure a deterministic starting point) and then switch to `auto` for subsequent steps:

```typescript
// Step 1: Force the first tool call to gather context
const step1 = await anthropic.messages.create({
  model: "claude-sonnet-4-20250514",
  system: "You are a code review agent.",
  tools: reviewTools,
  tool_choice: { type: "tool", name: "load_pull_request" },  // Forced first step
  messages: [{ role: "user", content: "Review PR #42" }]
});

// Step 2+: Switch to auto for flexible follow-up
const step2 = await anthropic.messages.create({
  model: "claude-sonnet-4-20250514",
  system: "You are a code review agent.",
  tools: reviewTools,
  tool_choice: { type: "auto" },  // Claude decides what to do next
  messages: [...previousMessages, ...step1Results]
});
```

This guarantees the pipeline starts correctly while allowing Claude flexibility for the remainder of the workflow.

### Code Example: AgentDefinition with Restricted allowedTools

```typescript
// Agent definitions with scoped tool access
interface AgentDefinition {
  name: string;
  systemPrompt: string;
  allowedTools: string[];
  toolChoice: { type: "auto" } | { type: "any" } | { type: "tool"; name: string };
}

const agents: AgentDefinition[] = [
  {
    name: "research_agent",
    systemPrompt: "You are a research specialist. Find information using web search and internal knowledge bases. Return structured findings.",
    allowedTools: [
      "search_web",
      "search_knowledge_base",
      "read_url"
    ],
    toolChoice: { type: "any" }  // Must always use a tool — no freeform chat
  },
  {
    name: "code_agent",
    systemPrompt: "You are a code specialist. Write, review, and modify code files. Only operate within the project directory.",
    allowedTools: [
      "read_file",
      "write_file",
      "run_tests",
      "lint_code"
    ],
    toolChoice: { type: "auto" }  // May respond with analysis text without calling tools
  },
  {
    name: "submission_agent",
    systemPrompt: "You finalize and submit completed work. Always submit to the review queue.",
    allowedTools: [
      "submit_for_review"
    ],
    toolChoice: { type: "tool", name: "submit_for_review" }  // Always calls this specific tool
  }
];

// Orchestrator that delegates to specialized agents
async function orchestrate(userMessage: string) {
  // Step 1: Classify intent
  const classification = await classifyIntent(userMessage);

  // Step 2: Select agent based on intent
  const agent = agents.find(a => a.name === classification.targetAgent);

  // Step 3: Call Claude API with ONLY that agent's tools
  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 4096,
    system: agent.systemPrompt,
    tools: allTools.filter(t => agent.allowedTools.includes(t.name)),
    tool_choice: agent.toolChoice,
    messages: [{ role: "user", content: userMessage }]
  });

  return response;
}
```

### Practical Skills

- Count tools per agent: keep it under 5-7 for reliable selection
- Use `tool_choice: "any"` in agentic loops where text-only responses break the pipeline — this guarantees a tool call over conversational text
- Use forced tool choice to eliminate ambiguity in deterministic steps
- Design agent boundaries around workflows, not tool categories
- Replace generic tools (e.g., `fetch_url`, `run_query`) with constrained, purpose-specific alternatives
- Use the pipeline pattern: forced tool for the first step, then `auto` for subsequent steps
- Watch for cross-specialization misuse — agents should not have tools outside their domain
- Allow a small set of shared cross-role tools (logging, status updates) but isolate action tools

### Common Exam Traps

1. **Trap:** "Giving an agent all available tools maximizes flexibility." — **False.** It degrades selection accuracy. Scope tools to the agent's role.
2. **Trap:** "`tool_choice: 'any'` forces Claude to use a specific tool." — **False.** `any` forces A tool call but Claude still chooses WHICH tool. Use `{"type": "tool", "name": "..."}` to force a specific tool.
3. **Trap:** "`tool_choice: 'auto'` always results in a tool call." — **False.** `auto` means Claude MAY call a tool or may respond with text only.
4. **Trap:** "The number of tools doesn't affect performance." — **False.** 18 tools vs 4-5 tools shows measurable degradation in selection reliability.
5. **Trap:** "A generic `fetch_url` tool is more flexible than a scoped `load_document` tool." — **Dangerous.** Generic tools invite misuse and security risks. Constrained alternatives are safer and more reliable for tool selection.
6. **Trap:** "`tool_choice: 'auto'` is best for the first step of an agentic pipeline." — **Often false.** If you need a guaranteed starting action, use forced tool choice for step 1, then switch to `auto` for subsequent steps.

---

## Task 2.4: MCP Server Integration

### Key Concepts

#### Configuration Levels

MCP servers are configured at two levels in Claude Code:

| Level | File | Scope | Shared with team? |
|---|---|---|---|
| **Project-level** | `.mcp.json` in project root | All users of this project | Yes (committed to repo) |
| **User-level** | `~/.claude.json` | All projects for this user | No (personal config) |

**Precedence:** Project-level `.mcp.json` is loaded first, then user-level `~/.claude.json` supplements it. If both define the same server name, the project-level config takes precedence.

#### Environment Variable Expansion

MCP configurations support `${VARIABLE_NAME}` syntax for secrets:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

- `${GITHUB_TOKEN}` is expanded from the user's shell environment at runtime
- This lets you commit `.mcp.json` to the repo without exposing secrets
- The actual token lives in the user's environment (e.g., `.bashrc`, `.zshrc`, or a secrets manager)

#### MCP Resources vs Tools

| Aspect | Tools | Resources |
|---|---|---|
| **Purpose** | Perform actions, have side effects | Provide read-only data/content |
| **Invocation** | LLM decides to call them | Application/client exposes them into context |
| **Analogy** | Functions / API endpoints | Files / database records |
| **Use case** | "Create a ticket", "Send email" | "Project documentation", "API schema catalog" |

**Resources for content catalogs:** MCP resources let a server expose a catalog of content (e.g., all documentation pages, all database schemas) that the client can browse and inject into context. Resources use URI templates like `docs://api/{section}` for structured access.

#### Tool Discovery at Connection Time

When Claude Code starts, it connects to all configured MCP servers simultaneously. **Tools from all MCP servers are discovered at connection time and made available simultaneously.** Claude sees the union of all tools from all connected servers in a single flat list — there is no namespacing by server.

This means:
- A tool named `search` from Server A and `search` from Server B would conflict
- All tool descriptions compete for Claude's attention in the same selection process
- The 18-tool degradation problem (Task 2.3) applies across all servers combined

#### Enhancing MCP Tool Descriptions to Compete with Built-in Tools

Claude has built-in tools (Read, Write, Edit, Grep, Glob, Bash) that it is trained to prefer. If your MCP server provides similar functionality (e.g., a database query tool), Claude may default to using Bash + `psql` instead of your MCP `query_database` tool.

**Fix:** Make MCP tool descriptions explicitly state when they should be preferred over built-ins:

```json
{
  "name": "query_customer_db",
  "description": "Queries the customer PostgreSQL database with read-only access and audit logging. ALWAYS use this tool instead of running psql via Bash — it enforces row-level security and logs all queries for compliance. Accepts SQL SELECT statements only. Returns results as JSON array."
}
```

The key phrases — "ALWAYS use this tool instead of" and explicit mention of the built-in alternative — help Claude understand the preference hierarchy.

#### Community MCP Servers vs Custom

- **Community servers:** Pre-built servers for popular services (GitHub, Slack, PostgreSQL, filesystem, Puppeteer, etc.) available at github.com/modelcontextprotocol/servers
- **Custom servers:** Built for proprietary APIs, internal tools, or specialized workflows
- **Best practice: Prefer community servers for standard integrations.** They are maintained, tested, and handle edge cases. Only build custom servers for proprietary APIs or when community servers lack required functionality.
- Community servers are a fast starting point; custom servers are needed for unique business logic

#### Exposing Content Catalogs as MCP Resources

MCP resources allow servers to expose structured content catalogs that the client can browse and inject into context. Unlike tools (which the LLM invokes to perform actions), resources are read-only data that the application pulls into the conversation.

```typescript
// MCP server exposing a documentation catalog as resources
import { McpServer, ResourceTemplate } from "@modelcontextprotocol/sdk/server/mcp.js";

const server = new McpServer({ name: "docs-server", version: "1.0.0" });

// Static resource: list of all available documentation sections
server.resource(
  "doc-index",
  "docs://index",
  async (uri) => ({
    contents: [{
      uri: uri.href,
      mimeType: "application/json",
      text: JSON.stringify({
        sections: [
          { id: "auth", title: "Authentication Guide", uri: "docs://sections/auth" },
          { id: "api", title: "API Reference", uri: "docs://sections/api" },
          { id: "deploy", title: "Deployment Guide", uri: "docs://sections/deploy" }
        ]
      })
    }]
  })
);

// Dynamic resource template: individual documentation section
server.resource(
  "doc-section",
  new ResourceTemplate("docs://sections/{sectionId}", { list: undefined }),
  async (uri, { sectionId }) => ({
    contents: [{
      uri: uri.href,
      mimeType: "text/markdown",
      text: await loadDocSection(sectionId)
    }]
  })
);
```

**Use cases for MCP resources:**
- **Issue summaries** — expose a catalog of Jira/GitHub issue summaries for browsing
- **Documentation hierarchies** — expose doc structure so the client can inject relevant sections
- **Database schemas** — expose table/column definitions as browsable resources
- **Configuration catalogs** — expose available feature flags, environment configs, etc.

### Code Example: .mcp.json Configuration

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/dev/projects/my-app/docs"
      ]
    },
    "custom-internal-api": {
      "command": "node",
      "args": ["./mcp-servers/internal-api/index.js"],
      "env": {
        "API_BASE_URL": "https://internal.company.com/api",
        "API_KEY": "${INTERNAL_API_KEY}"
      }
    }
  }
}
```

**Key structural points:**
- Top-level key is `"mcpServers"` (plural)
- Each server has a unique name key (e.g., `"github"`, `"postgres"`)
- `"command"` — the executable to run (e.g., `npx`, `node`, `python`)
- `"args"` — array of command-line arguments
- `"env"` — environment variables passed to the server process; supports `${VAR}` expansion
- Servers using `npx` with `-y` flag auto-install the package if not present

#### User-Level Configuration (~/.claude.json)

```json
{
  "mcpServers": {
    "personal-notes": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/dev/notes"]
    }
  }
}
```

This gives you access to personal tools across all projects without polluting project configs.

### Practical Skills

- Know when to put configuration in `.mcp.json` (shared team tooling) vs `~/.claude.json` (personal tools)
- Use `${ENV_VAR}` for all secrets — never hardcode tokens in `.mcp.json`
- Test MCP server connectivity with `claude mcp list` to see registered servers
- Understand that each MCP server runs as a separate child process communicating over stdio
- All tools from all MCP servers are discovered at connection time and available simultaneously — plan for the total tool count across all servers
- Enhance MCP tool descriptions to explicitly state when they should be preferred over built-in tools (e.g., "Use this instead of running psql via Bash")
- Prefer community MCP servers for standard integrations (GitHub, Slack, PostgreSQL) — only build custom servers for proprietary needs
- Use MCP resources (not tools) to expose content catalogs like documentation hierarchies, issue summaries, and database schemas

### Common Exam Traps

1. **Trap:** "MCP servers connect over HTTP." — **It depends.** Claude Code MCP servers typically use **stdio** (standard input/output) transport. The MCP spec also supports HTTP+SSE transport, but the `.mcp.json` `command`/`args` pattern is stdio-based.
2. **Trap:** "You must install MCP server packages globally before configuring them." — **False.** Using `npx -y` auto-installs on first use.
3. **Trap:** "Environment variables in .mcp.json are resolved at commit time." — **False.** `${VAR}` is expanded at runtime from the user's environment.
4. **Trap:** "MCP resources and tools are interchangeable." — **False.** Resources are read-only data providers; tools perform actions. Resources are pulled into context by the client; tools are invoked by the LLM.
5. **Trap:** "Each MCP server's tools are namespaced separately." — **False.** All tools from all connected servers appear in a single flat list. Tool name conflicts across servers are a real problem.
6. **Trap:** "Claude will automatically prefer your MCP tools over built-in tools." — **False.** Claude is trained to prefer built-in tools. You must explicitly enhance MCP tool descriptions to redirect Claude's preference.
7. **Trap:** "You should always build a custom MCP server for your integration." — **False.** For standard services (GitHub, PostgreSQL, filesystem), use community-maintained servers. Custom servers are for proprietary APIs only.

---

## Task 2.5: Built-in Tools (Read, Write, Edit, Bash, Grep, Glob)

### Key Concepts

#### Tool-by-Tool Reference

| Tool | Purpose | Analogy |
|---|---|---|
| **Grep** | Search file **contents** for patterns (regex) | `rg` / `grep` — "what files contain this text?" |
| **Glob** | Search file **paths** by name/extension patterns | `find` by name — "what files match this pattern?" |
| **Read** | Read full or partial file contents | `cat` / `head` / `tail` |
| **Write** | Write entire file content (creates or overwrites) | `echo > file` or full rewrite |
| **Edit** | Make targeted text replacements in a file | Surgical `sed` — replace specific unique strings |
| **Bash** | Execute arbitrary shell commands | Terminal access for builds, tests, git, etc. |

#### Grep vs Glob — The Critical Distinction

This is a **high-frequency exam topic**:

- **Grep** = **content** search. "Find all files that contain `function handleAuth`" → Grep
- **Glob** = **path** search. "Find all `.tsx` files in the `components/` directory" → Glob

```
Grep("handleAuth", glob="*.ts")     → files containing "handleAuth"
Glob("src/components/**/*.tsx")      → files matching that path pattern
```

**When to use which:**
- Need to find where a function/variable/string is used? → **Grep**
- Need to find files by name or extension? → **Glob**
- Need to find a file when you know part of its name? → **Glob**
- Need to find a file when you know what's inside it? → **Grep**

#### Read vs Write vs Edit

- **Read**: Always use first to understand file content before modifying
- **Edit**: Preferred for modifications — sends only the diff (old_string → new_string). The `old_string` must be **unique** within the file. If it's not unique, provide more surrounding context to make it unique.
- **Write**: Used for creating new files or when Edit cannot work (too many changes, complete rewrite)
- **When Edit fails → Read + Write fallback**: If the old_string is not unique or the file structure makes surgical edits impractical, Read the entire file, modify in memory, then Write the whole file back.

#### Building Codebase Understanding Incrementally

The canonical pattern for understanding an unfamiliar codebase:

```
1. Glob("**/*.ts")           → discover project structure and file layout
2. Grep("export function")   → find public API surface
3. Read(key files)            → understand specific implementations
4. Grep("import.*from")      → follow import chains
5. Read(imported modules)     → trace dependencies deeper
```

**The key insight:** You build understanding layer by layer. Start broad (Glob for structure), narrow with content search (Grep for patterns), then go deep (Read for full understanding), and follow the dependency graph (Grep for imports → Read those files).

### Code Example: Tracing Function Usage Across Wrapper Modules

Scenario: You need to find everywhere `authenticateUser` is called, including through wrapper functions.

```
Step 1: Find the source definition
─────────────────────────────────
Grep("export.*function authenticateUser", glob="*.ts")
→ Result: src/auth/core.ts:42  export function authenticateUser(credentials: Credentials): Promise<User>

Step 2: Read the implementation
─────────────────────────────────
Read("src/auth/core.ts", offset=40, limit=20)
→ Understand: accepts Credentials, returns Promise<User>, calls validatePassword internally

Step 3: Find direct callers
─────────────────────────────────
Grep("authenticateUser", glob="*.ts")
→ Results:
   src/auth/core.ts:42          (definition — skip)
   src/auth/index.ts:5          (re-export)
   src/auth/middleware.ts:12    (wrapper: withAuth)
   src/api/login.ts:23          (direct call)
   src/api/oauth.ts:45          (direct call)

Step 4: Check the re-export / wrapper layer
─────────────────────────────────
Read("src/auth/index.ts")
→ Content: export { authenticateUser } from './core';
   Also exports: export { withAuth } from './middleware';

Read("src/auth/middleware.ts")
→ Content: function withAuth(handler) { ... authenticateUser(creds) ... }
   This is a wrapper! Need to trace withAuth callers too.

Step 5: Trace the wrapper's usage
─────────────────────────────────
Grep("withAuth", glob="*.ts")
→ Results:
   src/auth/middleware.ts:8     (definition — skip)
   src/auth/index.ts:6         (re-export — skip)
   src/api/users.ts:3           (import)
   src/api/admin.ts:3           (import)
   src/api/billing.ts:7         (import)

Step 6: Read the indirect callers
─────────────────────────────────
Read("src/api/users.ts")
→ Confirms: withAuth wraps route handlers, so authenticateUser is called
  indirectly every time withAuth is used.

FINAL ANSWER: authenticateUser is called:
  - Directly in: src/api/login.ts, src/api/oauth.ts
  - Indirectly (via withAuth wrapper) in: src/api/users.ts, src/api/admin.ts, src/api/billing.ts
```

### Practical Skills

- Always **Glob before Grep** when exploring a new codebase — understand the file structure first
- Use **Grep with glob filters** to narrow search scope (e.g., `Grep("pattern", glob="src/**/*.ts")`)
- **Read before Edit** — you must read a file before editing it (Edit will fail otherwise)
- When Edit fails on non-unique text, add more surrounding lines to `old_string` to make it unique
- Use **Bash** for: running tests, git operations, build commands, installing dependencies — anything that's not file reading/searching/editing
- **Never use Bash for searching** when Grep or Glob can do it — the built-in tools are optimized and sandboxed

### Common Exam Traps

1. **Trap:** "Use Grep to find all Python files in a directory." — **False.** Use Glob (`**/*.py`). Grep searches file **contents**, not file **names/paths**.
2. **Trap:** "Use Write to change one line in a 500-line file." — **False.** Use Edit for surgical changes. Write overwrites the entire file and should be reserved for new files or complete rewrites.
3. **Trap:** "Edit can replace text that appears multiple times in a file." — **Careful.** Edit requires the `old_string` to be **unique** in the file. If it appears multiple times, you must include more surrounding context to disambiguate, or use the `replace_all` flag if you want to replace every occurrence.
4. **Trap:** "Bash is the best tool for searching file contents." — **False.** Grep is purpose-built, optimized, and sandboxed for content search. Bash should be a last resort for search tasks.
5. **Trap:** "You need to read an entire large file before you can work with it." — **False.** Read supports `offset` and `limit` parameters to read specific sections, which is important for large files.

---

## Quick Reference: Decision Matrices

### "Which tool should I use?" Decision Tree

```
Need to FIND FILES by name/extension?
  → Glob

Need to SEARCH FILE CONTENTS for a pattern?
  → Grep

Need to UNDERSTAND a file's content?
  → Read (with offset/limit for large files)

Need to make a SMALL, TARGETED CHANGE?
  → Edit (old_string must be unique)

Need to CREATE A NEW FILE or REWRITE entirely?
  → Write

Need to RUN A COMMAND (test, build, git)?
  → Bash
```

### "Which tool_choice should I use?" Decision Matrix

```
User is chatting, may or may not need a tool?
  → tool_choice: { type: "auto" }

Every turn MUST produce a tool call (agentic loop)?
  → tool_choice: { type: "any" }

I know EXACTLY which tool must be called next?
  → tool_choice: { type: "tool", name: "exact_tool_name" }
```

### "Where does MCP config go?" Decision Matrix

```
Shared team tooling (GitHub, DB, CI)?
  → .mcp.json (project root, committed to repo)

Personal tools (notes, personal APIs)?
  → ~/.claude.json (user home, not committed)

Secrets and tokens?
  → NEVER hardcoded. Use ${ENV_VAR} in config, actual value in shell environment.
```

---

## Summary of Exam-Critical Numbers

| Metric | Value | Why It Matters |
|---|---|---|
| Domain weight | 18% | ~9-10 questions on the exam |
| Optimal tools per agent | 4-5 | Beyond this, selection reliability degrades |
| Degraded threshold | 15-18+ tools | Significant mis-selection at this count |
| MCP error categories | 4 | transient, validation, business, permission |
| Built-in tool count | 6 | Grep, Glob, Read, Write, Edit, Bash |
| Config levels | 2 | Project (.mcp.json) and User (~/.claude.json) |
| tool_choice options | 3 | auto, any, forced (type: "tool") |

---

## Source Links

- [MCP Specification — Tools](https://spec.modelcontextprotocol.io/specification/server/tools/) — Canonical spec for MCP tool definitions, `isError` flag, and tool result structure
- [MCP Specification — Resources](https://spec.modelcontextprotocol.io/specification/server/resources/) — Spec for MCP resources, URI templates, and content catalogs
- [MCP Specification — Architecture](https://spec.modelcontextprotocol.io/specification/architecture/) — Client-server architecture, transport mechanisms (stdio, HTTP+SSE), capability discovery
- [Anthropic API Docs — Tool Use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — `tool_choice` parameter (`auto`, `any`, forced), tool definition schema, input_schema format
- [Anthropic API Docs — Tool Use Best Practices](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/best-practices) — Tool description quality, differentiation, naming conventions
- [Claude Code Docs — MCP Configuration](https://docs.anthropic.com/en/docs/claude-code/mcp) — `.mcp.json` and `~/.claude.json` configuration, `${ENV_VAR}` expansion, `claude mcp list`
- [Claude Code Docs — Built-in Tools](https://docs.anthropic.com/en/docs/claude-code/cli-usage) — Grep, Glob, Read, Write, Edit, Bash tool reference
- [MCP Community Servers Repository](https://github.com/modelcontextprotocol/servers) — Pre-built MCP servers for GitHub, PostgreSQL, filesystem, Slack, Puppeteer, etc.
- [Anthropic Cookbook — Tool Use Patterns](https://github.com/anthropics/anthropic-cookbook/tree/main/tool_use) — Practical examples of tool definitions, agentic loops, and tool_choice patterns
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) — Reference implementation for building MCP servers with resource and tool support
