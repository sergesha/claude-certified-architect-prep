# Domain 4: Prompt Engineering & Structured Output (20%)

This domain covers the practical techniques for getting precise, structured, and reliable output from Claude at scale. It spans prompt design, few-shot examples, schema-enforced output, validation loops, batch processing, and multi-pass review architectures.

---

## Task 4.1: Explicit Criteria for Precision

### Key Concepts

**Explicit criteria dramatically outperform vague instructions.** When Claude is told to "check comments are accurate," it generates inconsistent, noisy results. When told to "flag when a docstring's claimed return type contradicts the function signature," it produces precise, actionable findings.

**Why vague modifiers fail:**
- Instructions like "be conservative," "only report high-confidence issues," or "be careful" do NOT measurably improve precision. Claude lacks a reliable internal calibration mechanism for these qualitative hedges.
- The model has no grounded definition of "conservative" in a domain-specific review context. It may suppress some findings, but not systematically the false positives.

**High false positive rates undermine trust across ALL categories**, including accurate ones. If a tool reports 80% false positives in style issues, reviewers start ignoring security findings too. This is a key architectural concern, not just a prompt concern.

**Strategies for precision:**

1. **Specify what TO report and what to SKIP:**
   - Report: actual bugs, security vulnerabilities, correctness issues, API contract violations
   - Skip: minor style preferences, subjective naming opinions, whitespace formatting

2. **Temporarily disable high false-positive categories** rather than trying to tune them with vague instructions. Removing a noisy category entirely is better than leaving it on with "be careful."

3. **Define explicit severity levels with concrete code examples per level:**

```
CRITICAL: Code will crash or lose data at runtime.
  Example: `user_list[index]` where index can exceed list length
  Example: SQL query built with string concatenation from user input

HIGH: Functional bug that produces wrong results silently.
  Example: `if (a = b)` instead of `if (a == b)` in a conditional
  Example: Off-by-one error in pagination that skips records

LOW: Code smell that increases maintenance burden but works correctly.
  Example: Function exceeds 200 lines with deeply nested logic
  Example: Magic numbers without named constants
```

### Practical Skills

- Writing review criteria that serve as a "rubric" for Claude, not just a wish list
- Iterating on criteria by examining false positives from prior runs
- Recognizing that precision is a prompt design problem, not a temperature/model parameter problem

### Code Example (Python)

```python
import anthropic

client = anthropic.Anthropic()

# BAD: Vague instruction
vague_prompt = """Review this code. Be conservative and only report
high-confidence issues. Check that comments are accurate."""

# GOOD: Explicit criteria with examples
precise_prompt = """Review the following code. Report ONLY issues matching
these criteria:

REPORT (with severity):
- CRITICAL: Null pointer dereference, SQL injection, unhandled exceptions
  that crash the process, data loss scenarios
- HIGH: Logic errors that produce silently wrong results (e.g., wrong
  comparison operator, off-by-one in loops affecting output)
- MEDIUM: Resource leaks (unclosed connections/files), race conditions

DO NOT REPORT:
- Naming conventions or style preferences
- Missing comments or documentation
- Import ordering
- Minor formatting issues

For each finding, cite the exact line and explain WHY it is a bug,
not just WHAT the code does."""

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=2048,
    messages=[{"role": "user", "content": precise_prompt + "\n\n```python\n" + code + "\n```"}]
)
```

### Common Exam Traps

- **Trap:** Believing "be conservative" or "only report high-confidence findings" will reduce false positives. It will NOT reliably do so.
- **Trap:** Thinking you can fix precision by adjusting `temperature`. Precision is a prompt design problem.
- **Trap:** Keeping noisy categories active because "they sometimes catch real issues." The net effect on reviewer trust is negative.
- **Trap:** Writing criteria that describe what the code IS rather than what constitutes a FINDING.

---

## Task 4.2: Few-Shot Prompting

### Key Concepts

**Few-shot examples are the single most effective technique for getting consistent, formatted output.** They outperform lengthy instructions about format because they SHOW rather than TELL.

**Key principles:**

1. **Demonstrate ambiguous-case handling.** The highest-value examples are NOT the obvious cases -- they are the ambiguous ones where Claude might go either way. Show your reasoning for borderline decisions.

2. **Enable generalization to novel patterns.** Good few-shot examples teach a PATTERN of reasoning, not a lookup table. Claude should handle new cases it has never seen by applying the reasoning demonstrated in examples, not by pattern-matching to pre-specified cases.

3. **Reduce hallucination in extraction tasks.** When extracting structured data from text, few-shot examples that show "field not found -> null" behavior teach Claude to leave fields empty rather than fabricate values.

4. **2-4 targeted examples is the sweet spot.** More examples have diminishing returns and consume context. Choose examples that cover:
   - A clear positive case
   - A clear negative case
   - 1-2 ambiguous/edge cases with reasoning

5. **Examples should distinguish acceptable patterns from genuine issues.** For code review: show a case where a pattern LOOKS like a bug but is actually fine, and a case where a similar pattern IS a bug.

### Practical Skills

- Selecting few-shot examples that maximize coverage of edge cases
- Writing examples that show reasoning chains, not just input-output pairs
- Knowing when few-shot is more effective than detailed instructions (format consistency, ambiguous classification)

### Code Example (Python)

```python
import anthropic

client = anthropic.Anthropic()

few_shot_prompt = """You are extracting structured product data from descriptions.
Return JSON with fields: name, price, currency, in_stock, category.

Here are examples showing how to handle various cases:

EXAMPLE 1 (straightforward):
Input: "The UltraWidget Pro is available now for $49.99. Ships within 24 hours."
Output: {"name": "UltraWidget Pro", "price": 49.99, "currency": "USD",
         "in_stock": true, "category": "electronics"}
Reasoning: Price explicitly stated, "available now" + shipping time = in stock.

EXAMPLE 2 (ambiguous availability):
Input: "Check out the MegaGadget - starting at EUR 29. Join the waitlist!"
Output: {"name": "MegaGadget", "price": 29.00, "currency": "EUR",
         "in_stock": false, "category": "electronics"}
Reasoning: "Starting at" = base price. "Join the waitlist" = NOT in stock,
despite product being described positively.

EXAMPLE 3 (missing data - DO NOT fabricate):
Input: "Our premium leather wallet is back in stock."
Output: {"name": "Premium Leather Wallet", "price": null, "currency": null,
         "in_stock": true, "category": "accessories"}
Reasoning: No price mentioned anywhere. Set price and currency to null
rather than guessing. "Back in stock" = in_stock is true.

EXAMPLE 4 (tricky pattern):
Input: "The SmartLamp was $79.99, now FREE with any purchase over $200."
Output: {"name": "SmartLamp", "price": 0, "currency": "USD",
         "in_stock": true, "category": "electronics"}
Reasoning: Current price is free (0), not the original $79.99.
The conditional ("with any purchase") doesn't change the stated price.

Now extract from this description:
"""

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=512,
    messages=[{"role": "user", "content": few_shot_prompt + user_input}]
)
```

### Code Example (TypeScript)

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

// Few-shot for code review: distinguishing real bugs from acceptable patterns
const reviewPrompt = `Classify each finding as REAL_BUG or FALSE_POSITIVE.

EXAMPLE 1:
Code: \`if (user.role = "admin") { grant_access(); }\`
Classification: REAL_BUG
Reasoning: Assignment operator = used instead of comparison == in conditional.
This always assigns "admin" to user.role and evaluates truthy.

EXAMPLE 2:
Code: \`const items = data?.items ?? [];\`
Classification: FALSE_POSITIVE
Reasoning: This is defensive programming using optional chaining and nullish
coalescing. Not a bug -- it correctly handles null/undefined data.

EXAMPLE 3:
Code: \`for (let i = 0; i <= arr.length; i++) { process(arr[i]); }\`
Classification: REAL_BUG
Reasoning: Off-by-one: <= should be <. When i equals arr.length,
arr[i] is undefined, causing potential runtime error in process().

EXAMPLE 4:
Code: \`catch (e) { logger.error(e); throw e; }\`
Classification: FALSE_POSITIVE
Reasoning: Catch-log-rethrow is a valid pattern for adding logging at
module boundaries while preserving the error for upstream handlers.

Now classify:
Code: \`${codeSnippet}\``;

const response = await client.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 512,
  messages: [{ role: "user", content: reviewPrompt }],
});
```

### Demonstrating Ambiguous-Case Handling: Tool Selection and Test Coverage

Few-shot examples are especially valuable for teaching Claude how to handle **tool selection** in ambiguous scenarios and how to identify **test coverage gaps**:

```python
import anthropic

client = anthropic.Anthropic()

# Few-shot for ambiguous tool selection decisions
tool_selection_prompt = """You are an AI assistant with access to tools: search_web, query_database, read_file.
Decide which tool to use for each user request.

EXAMPLE 1 (ambiguous - could be web or database):
Request: "What is the current price of ACME stock?"
Tool: query_database
Reasoning: "Current price" suggests real-time data our database tracks.
If the database query returns nothing, THEN fall back to search_web.

EXAMPLE 2 (ambiguous - could be file or database):
Request: "Show me the Q3 revenue numbers"
Tool: read_file
Reasoning: "Q3 revenue" is likely in a quarterly report file, not a transactional
database. Check for Q3_report.xlsx or similar before querying the database.

EXAMPLE 3 (test coverage gap identification):
Request: "Review this test file for coverage gaps"
Tool: read_file
Reasoning: Must read the test file AND the source file it tests. A test file
that only tests happy paths but skips error handling, edge cases, or boundary
conditions has coverage gaps even if line coverage is 100%.

Now decide for this request:
"""
```

### Handling Varied Document Structures and Reducing Hallucination

Few-shot examples are critical for extraction from **varied document structures** (inline citations vs bibliographies, informal measurements, different formatting conventions):

```python
# Few-shot for handling varied document structures
varied_docs_prompt = """Extract measurements and citations from scientific text.
Handle different document structures consistently.

EXAMPLE 1 (inline citation with formal measurement):
Text: "The sample showed 45.2 mg/L concentration (Johnson et al., 2023)."
Output: {"measurement": "45.2", "unit": "mg/L", "parameter": "concentration",
         "citation": "Johnson et al., 2023", "citation_type": "inline"}

EXAMPLE 2 (informal measurement, no citation):
Text: "Roughly half the participants reported improvement after about 3 weeks."
Output: {"measurement": "~50", "unit": "percent", "parameter": "improvement_rate",
         "citation": null, "citation_type": null,
         "measurement_note": "Informal: 'roughly half' interpreted as ~50%"}
Reasoning: "Roughly half" is an informal measurement. Capture the approximation
with a note rather than fabricating a precise number.

EXAMPLE 3 (bibliography-style reference, varied format):
Text: "Results showed 12.5 cm growth [7]."
Reference [7]: "Smith, J. Growth Patterns. Nature, 2022;401:23-29."
Output: {"measurement": "12.5", "unit": "cm", "parameter": "growth",
         "citation": "Smith, J. Growth Patterns. Nature, 2022;401:23-29",
         "citation_type": "bibliography"}
Reasoning: "[7]" is a numbered reference. Resolve to full citation from
bibliography. Do NOT leave as just "[7]".

EXAMPLE 4 (ambiguous unit, missing data):
Text: "The tank holds about 50 gallons, maybe more."
Output: {"measurement": "~50", "unit": "gallons", "parameter": "capacity",
         "citation": null, "citation_type": null,
         "measurement_note": "Approximate: 'about 50, maybe more'"}
Reasoning: Informal estimate. Do NOT fabricate a precise value or a citation.

Now extract from:
"""
```

```typescript
// TypeScript: Few-shot for format demonstration (location/issue/severity/fix)
const formatDemoPrompt = `Report code issues in this exact format:

EXAMPLE 1:
Location: src/auth.ts:42
Issue: SQL injection via string concatenation in login query
Severity: CRITICAL
Fix: Use parameterized query: db.query("SELECT * FROM users WHERE id = $1", [userId])

EXAMPLE 2:
Location: src/utils.ts:108
Issue: Array index accessed without bounds check
Severity: HIGH
Fix: Add guard: if (index >= 0 && index < items.length) before access

EXAMPLE 3:
Location: src/config.ts:15
Issue: Hardcoded timeout value of 5000ms
Severity: LOW
Fix: Extract to configuration constant: const DEFAULT_TIMEOUT = 5000

Now review this code:
${codeToReview}`;
```

### Common Exam Traps

- **Trap:** Thinking more examples = better results. Beyond 4-5, you get diminishing returns and waste context window.
- **Trap:** Only showing "happy path" examples. The ambiguous/edge cases are where few-shot provides the most value.
- **Trap:** Providing examples without reasoning. Input-output pairs alone teach FORMAT but not JUDGMENT.
- **Trap:** Confusing few-shot with retrieval. Few-shot teaches reasoning patterns; it is not a substitute for providing the actual data to process.
- **Trap:** Assuming few-shot only helps with format. It also teaches Claude how to handle informal measurements without fabricating precision, and how to navigate varied document structures (inline citations vs bibliographies).

---

## Task 4.3: Structured Output via Tool Use and JSON Schemas

### Key Concepts

**`tool_use` with JSON schemas is the most reliable method for getting schema-compliant structured output from Claude.** It is more reliable than asking for JSON in a system prompt because the API enforces the schema at the response layer.

**`tool_choice` parameter controls tool calling behavior:**

| Value | Behavior | When to use |
|-------|----------|-------------|
| `"auto"` | Claude decides whether to call a tool or respond with text | When tool use is optional; Claude may answer directly |
| `{"type": "any"}` | Claude MUST call at least one tool (from available tools) | When you need structured output but any tool is acceptable |
| `{"type": "tool", "name": "specific_tool"}` | Claude MUST call the specified tool | When you need a specific schema; most predictable for extraction |

**Critical distinction: Strict schemas eliminate SYNTAX errors but NOT SEMANTIC errors.**
- Schema enforcement guarantees valid JSON matching the structure (correct types, required fields present, enum values valid).
- It does NOT guarantee the VALUES are correct. A hallucinated company name in a valid string field passes schema validation.

**Schema design best practices:**

1. **`required` vs optional fields:** Mark fields `required` only when the source data reliably contains them. Optional fields let Claude omit data it cannot find rather than fabricating values.

2. **`enum` + `"other"` + detail string pattern:**
```json
{
  "category": {"type": "string", "enum": ["bug", "feature", "refactor", "other"]},
  "category_detail": {"type": "string", "description": "If category is 'other', describe here"}
}
```
This prevents Claude from forcing data into ill-fitting categories while still providing structure.

3. **Nullable fields:** Use `{"type": ["string", "null"]}` or `"nullable": true` (depending on API version) for fields that may legitimately have no value. This explicitly signals "absence of data" vs "empty string."

4. **Force a specific tool first** when you need ordered operations (e.g., `extract_metadata` must run before `enrich_data`). Use `tool_choice: {"type": "tool", "name": "extract_metadata"}` for the first call.

### Practical Skills

- Designing tool schemas that capture the full output space (including "not found" / "other")
- Choosing the right `tool_choice` mode for extraction vs conversational use cases
- Understanding that schema compliance != semantic correctness
- Ordering multi-step tool calls by forcing the first tool

### Code Example (Python) -- Extraction Tool Definition

```python
import anthropic
import json

client = anthropic.Anthropic()

# Define an extraction tool with a comprehensive JSON schema
extraction_tool = {
    "name": "extract_invoice_data",
    "description": "Extract structured invoice data from the provided document text.",
    "input_schema": {
        "type": "object",
        "required": ["vendor_name", "invoice_number", "line_items", "total_amount"],
        "properties": {
            "vendor_name": {
                "type": "string",
                "description": "The name of the vendor/supplier on the invoice"
            },
            "invoice_number": {
                "type": "string",
                "description": "The invoice number or ID"
            },
            "invoice_date": {
                "type": ["string", "null"],
                "description": "Invoice date in ISO 8601 format, or null if not found"
            },
            "due_date": {
                "type": ["string", "null"],
                "description": "Payment due date in ISO 8601 format, or null if not found"
            },
            "line_items": {
                "type": "array",
                "items": {
                    "type": "object",
                    "required": ["description", "amount"],
                    "properties": {
                        "description": {"type": "string"},
                        "quantity": {"type": ["number", "null"]},
                        "unit_price": {"type": ["number", "null"]},
                        "amount": {"type": "number"}
                    }
                }
            },
            "total_amount": {
                "type": "number",
                "description": "The total invoice amount"
            },
            "currency": {
                "type": "string",
                "enum": ["USD", "EUR", "GBP", "CAD", "AUD", "other"],
                "description": "Currency code"
            },
            "currency_detail": {
                "type": ["string", "null"],
                "description": "If currency is 'other', specify the actual currency here"
            },
            "confidence_notes": {
                "type": ["string", "null"],
                "description": "Any uncertainty about extracted values"
            }
        }
    }
}

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[extraction_tool],
    tool_choice={"type": "tool", "name": "extract_invoice_data"},  # Force this tool
    messages=[{
        "role": "user",
        "content": f"Extract invoice data from this document:\n\n{document_text}"
    }]
)

# The response will contain a tool_use content block
for block in response.content:
    if block.type == "tool_use":
        invoice_data = block.input  # Already a dict matching the schema
        print(json.dumps(invoice_data, indent=2))
```

### Code Example (TypeScript)

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const tools: Anthropic.Tool[] = [
  {
    name: "classify_support_ticket",
    description: "Classify a support ticket into category and priority.",
    input_schema: {
      type: "object" as const,
      required: ["category", "priority", "summary", "requires_escalation"],
      properties: {
        category: {
          type: "string",
          enum: ["billing", "technical", "account", "feature_request", "other"],
        },
        category_detail: {
          type: ["string", "null"],
          description: "Details if category is 'other'",
        },
        priority: {
          type: "string",
          enum: ["critical", "high", "medium", "low"],
        },
        summary: {
          type: "string",
          description: "One-sentence summary of the issue",
        },
        requires_escalation: {
          type: "boolean",
          description: "True if this needs immediate human attention",
        },
      },
    },
  },
];

const response = await client.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 512,
  tools: tools,
  tool_choice: { type: "any" }, // Must call a tool, but we only have one
  messages: [
    {
      role: "user",
      content: `Classify this support ticket:\n\n${ticketText}`,
    },
  ],
});

// Extract the tool call result
const toolBlock = response.content.find((b) => b.type === "tool_use");
if (toolBlock && toolBlock.type === "tool_use") {
  const classification = toolBlock.input;
  console.log(classification);
}
```

### Using `tool_choice: "any"` for Unknown Document Types

When processing documents of **unknown type** (could be invoice, receipt, contract, etc.), use `tool_choice: {"type": "any"}` with multiple tool definitions so Claude picks the right schema:

```python
import anthropic

client = anthropic.Anthropic()

# Multiple extraction tools for different document types
tools = [
    {
        "name": "extract_invoice",
        "description": "Extract data from an invoice document.",
        "input_schema": {
            "type": "object",
            "required": ["vendor", "total", "line_items"],
            "properties": {
                "vendor": {"type": "string"},
                "total": {"type": "number"},
                "line_items": {"type": "array", "items": {"type": "object"}}
            }
        }
    },
    {
        "name": "extract_contract",
        "description": "Extract data from a contract document.",
        "input_schema": {
            "type": "object",
            "required": ["parties", "effective_date", "terms"],
            "properties": {
                "parties": {"type": "array", "items": {"type": "string"}},
                "effective_date": {"type": ["string", "null"]},
                "terms": {"type": "array", "items": {"type": "string"}}
            }
        }
    },
    {
        "name": "extract_receipt",
        "description": "Extract data from a receipt.",
        "input_schema": {
            "type": "object",
            "required": ["store_name", "total", "items"],
            "properties": {
                "store_name": {"type": "string"},
                "total": {"type": "number"},
                "items": {"type": "array", "items": {"type": "object"}}
            }
        }
    }
]

# "any" forces Claude to pick the BEST tool for the document type
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "any"},  # Must call a tool, but Claude chooses which
    messages=[{
        "role": "user",
        "content": f"Extract structured data from this document:\n\n{unknown_document}"
    }]
)

# Determine which tool Claude selected
for block in response.content:
    if block.type == "tool_use":
        print(f"Document type detected: {block.name}")
        print(f"Extracted data: {block.input}")
```

### Enum with "unclear" + "other" + Detail String Pattern

Beyond just `"other"`, include `"unclear"` for cases where Claude genuinely cannot determine the category. This prevents forcing a classification when the data is ambiguous:

```python
severity_schema = {
    "type": "object",
    "required": ["severity", "confidence_level"],
    "properties": {
        "severity": {
            "type": "string",
            "enum": ["critical", "high", "medium", "low", "unclear", "other"],
            "description": "Issue severity. Use 'unclear' when insufficient context to judge, 'other' when none of the defined levels apply"
        },
        "severity_detail": {
            "type": ["string", "null"],
            "description": "Required explanation when severity is 'unclear' or 'other'"
        },
        "confidence_level": {
            "type": "string",
            "enum": ["high", "medium", "low"],
            "description": "How confident the model is in the severity assessment"
        }
    }
}
```

### Format Normalization Rules in Prompts Alongside Schemas

JSON schemas enforce structure but NOT format consistency of values. Add **format normalization rules** directly in the prompt to ensure semantic consistency:

```python
normalization_prompt = """Extract data from the document below. Follow these
FORMAT NORMALIZATION RULES for field values:

- Dates: Always ISO 8601 format (YYYY-MM-DD). Convert "March 5th, 2024" → "2024-03-05"
- Phone numbers: E.164 format with country code. Convert "(555) 123-4567" → "+15551234567"
- Currency amounts: Numeric only, no symbols. Convert "$1,234.56" → 1234.56
- Names: Title Case. Convert "john doe" or "JOHN DOE" → "John Doe"
- Addresses: Expand abbreviations. Convert "123 Main St." → "123 Main Street"
- Country: ISO 3166-1 alpha-2 code. Convert "United States" → "US"

If a value cannot be normalized (e.g., ambiguous date "05/06/2024"), set the
field to null and explain in the confidence_notes field.
"""

# Use this prompt alongside the schema to get both structural AND format compliance
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[extraction_tool],
    tool_choice={"type": "tool", "name": "extract_invoice_data"},
    messages=[{
        "role": "user",
        "content": normalization_prompt + f"\n\nDocument:\n{document_text}"
    }]
)
```

### Common Exam Traps

- **Trap:** Believing schema enforcement prevents hallucination. It prevents STRUCTURAL errors (wrong types, missing fields) but NOT incorrect values.
- **Trap:** Using `tool_choice: "auto"` when you NEED structured output. With `"auto"`, Claude may respond with plain text instead of calling the tool.
- **Trap:** Making all fields `required` when some data may genuinely be absent. This forces Claude to fabricate values.
- **Trap:** Forgetting the `enum` + `"other"` + detail pattern. A closed enum without an escape hatch forces bad classifications.
- **Trap:** Confusing `{"type": "any"}` with `{"type": "auto"}`. The value `"auto"` is a string shorthand, while `"any"` and specific tool forcing use the object form `{"type": "any"}`.
- **Trap:** Relying solely on schemas for format consistency. Schemas enforce types (string, number) but not value formats (date format, phone format). Use prompt-level normalization rules alongside schemas.
- **Trap:** Using only `"other"` without `"unclear"`. When Claude cannot determine a category due to insufficient context, `"unclear"` is semantically different from `"other"` (which means "a known category not in the list").

---

## Task 4.4: Validation, Retry, and Feedback Loops

### Key Concepts

**Retry-with-error-feedback** is the pattern of appending specific validation errors back into the conversation and asking Claude to correct its output. This is highly effective for FORMAT errors (malformed JSON, missing required fields, wrong types) but NOT for missing information.

**When retries work vs. when they do NOT:**

| Scenario | Retry effective? | Why |
|----------|-----------------|-----|
| JSON syntax error | Yes | Claude can fix structural mistakes |
| Missing required field (data exists in source) | Yes | Claude overlooked it, can re-extract |
| Missing required field (data NOT in source) | **No** | No amount of retrying creates absent information |
| Value out of expected range | Yes | Claude can re-read and correct |
| Hallucinated value | **Marginal** | Claude may produce a different hallucination |

**Feedback loops with `detected_pattern` fields:** Track which findings users dismiss. Feed this back as context:

```
Previous findings dismissed by reviewers:
- "Unused import" in test files (dismissed 12 times)
- "Missing null check" on builder pattern chains (dismissed 8 times)

Adjust: skip these patterns in future reviews.
```

**Self-correction via computed cross-checks:** Design schemas with built-in consistency checks:
- `calculated_total` (sum of line items) vs `stated_total` (from document)
- `conflict_detected` boolean when these differ
- This lets downstream code flag discrepancies without a separate validation pass

### Practical Skills

- Implementing a validation-retry loop with a maximum retry count
- Designing prompts that include the SPECIFIC error, not just "try again"
- Knowing when to retry vs. when to flag for human review
- Building self-checking fields into extraction schemas

### Code Example (Python) -- Validation-Retry Loop

```python
import anthropic
import json
from typing import Optional

client = anthropic.Anthropic()

def extract_with_validation(
    document: str,
    max_retries: int = 3
) -> Optional[dict]:
    """Extract structured data with validation and retry on failure."""

    tools = [{
        "name": "extract_financial_data",
        "description": "Extract financial data from the document.",
        "input_schema": {
            "type": "object",
            "required": ["revenue", "expenses", "net_income",
                        "calculated_net", "amounts_consistent"],
            "properties": {
                "revenue": {"type": "number"},
                "expenses": {"type": "number"},
                "net_income": {
                    "type": "number",
                    "description": "Net income as stated in the document"
                },
                "calculated_net": {
                    "type": "number",
                    "description": "Revenue minus expenses (self-check)"
                },
                "amounts_consistent": {
                    "type": "boolean",
                    "description": "True if calculated_net matches net_income"
                },
                "period": {"type": ["string", "null"]},
                "currency": {"type": "string"}
            }
        }
    }]

    messages = [{
        "role": "user",
        "content": f"Extract financial data from this report:\n\n{document}"
    }]

    for attempt in range(max_retries):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            tools=tools,
            tool_choice={"type": "tool", "name": "extract_financial_data"},
            messages=messages
        )

        # Extract the tool call
        tool_block = next(
            (b for b in response.content if b.type == "tool_use"), None
        )
        if not tool_block:
            continue

        result = tool_block.input

        # Validate the extraction
        errors = validate_extraction(result)

        if not errors:
            return result  # Success

        # Append the assistant's response and error feedback for retry
        messages.append({"role": "assistant", "content": response.content})
        messages.append({
            "role": "user",
            "content": (
                f"The extraction had validation errors. Please fix them:\n\n"
                f"ERRORS:\n" + "\n".join(f"- {e}" for e in errors) + "\n\n"
                f"Please call the tool again with corrected values."
            )
        })

    return None  # Exhausted retries -- flag for human review


def validate_extraction(data: dict) -> list[str]:
    """Return a list of specific validation error messages."""
    errors = []

    # Check self-consistency
    if data.get("revenue") is not None and data.get("expenses") is not None:
        expected_net = data["revenue"] - data["expenses"]
        if abs(data.get("calculated_net", 0) - expected_net) > 0.01:
            errors.append(
                f"calculated_net ({data.get('calculated_net')}) should be "
                f"revenue ({data['revenue']}) - expenses ({data['expenses']}) "
                f"= {expected_net}"
            )

    # Check stated vs calculated consistency flag
    if data.get("net_income") is not None and data.get("calculated_net") is not None:
        actually_consistent = abs(data["net_income"] - data["calculated_net"]) < 0.01
        if data.get("amounts_consistent") != actually_consistent:
            errors.append(
                f"amounts_consistent is {data.get('amounts_consistent')} but "
                f"net_income ({data['net_income']}) vs calculated_net "
                f"({data['calculated_net']}) suggests it should be "
                f"{actually_consistent}"
            )

    # Check reasonable ranges
    if data.get("revenue", 0) < 0:
        errors.append("Revenue should not be negative. Re-read the document.")

    return errors
```

### Code Example (TypeScript) -- Retry Loop

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

interface ExtractionResult {
  data: Record<string, unknown> | null;
  attempts: number;
  lastErrors: string[];
}

async function extractWithRetry(
  document: string,
  tools: Anthropic.Tool[],
  toolName: string,
  validate: (data: Record<string, unknown>) => string[],
  maxRetries = 3
): Promise<ExtractionResult> {
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: `Extract data from:\n\n${document}` },
  ];

  let lastErrors: string[] = [];

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    const response = await client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 1024,
      tools,
      tool_choice: { type: "tool", name: toolName },
      messages,
    });

    const toolBlock = response.content.find((b) => b.type === "tool_use");
    if (!toolBlock || toolBlock.type !== "tool_use") continue;

    const data = toolBlock.input as Record<string, unknown>;
    const errors = validate(data);

    if (errors.length === 0) {
      return { data, attempts: attempt, lastErrors: [] };
    }

    // Feed errors back for retry
    messages.push({ role: "assistant", content: response.content });
    messages.push({
      role: "user",
      content: `Validation errors found:\n${errors.map((e) => `- ${e}`).join("\n")}\n\nPlease correct and call the tool again.`,
    });

    lastErrors = errors;
  }

  return { data: null, attempts: maxRetries, lastErrors };
}
```

### `detected_pattern` Field for False Positive Analysis

Include a `detected_pattern` field in review tool schemas to track which code constructs trigger findings. This enables systematic false positive analysis:

```python
review_tool_with_pattern = {
    "name": "report_code_finding",
    "description": "Report a code review finding with pattern tracking.",
    "input_schema": {
        "type": "object",
        "required": ["finding", "severity", "detected_pattern", "confidence"],
        "properties": {
            "finding": {"type": "string", "description": "Description of the issue"},
            "severity": {"type": "string", "enum": ["critical", "high", "medium", "low"]},
            "detected_pattern": {
                "type": "string",
                "description": "The specific code construct that triggered this finding, e.g., 'unchecked_array_index', 'string_sql_concat', 'catch_and_ignore', 'unused_import'",
            },
            "confidence": {"type": "string", "enum": ["high", "medium", "low"]},
            "conflict_detected": {
                "type": "boolean",
                "description": "True if the finding conflicts with other evidence in the code (e.g., a null check exists nearby)"
            },
            "line_number": {"type": "integer"},
            "file_path": {"type": "string"}
        }
    }
}

# After collecting findings across runs, analyze false positive rates by pattern:
# findings_log = [{"detected_pattern": "unused_import", "was_false_positive": True}, ...]
# pattern_fp_rates = compute_fp_rates_by_pattern(findings_log)
# Disable patterns with >60% false positive rate in subsequent prompts
```

### Follow-Up Requests: Original Doc + Failed Extraction + Validation Errors

The most effective retry pattern sends all three pieces of context together:

```python
def retry_with_full_context(
    original_doc: str,
    failed_extraction: dict,
    validation_errors: list[str],
    tools: list,
    tool_name: str
) -> dict:
    """Retry with original document, failed extraction, AND specific errors."""
    client = anthropic.Anthropic()

    retry_prompt = f"""The previous extraction from this document had errors.

ORIGINAL DOCUMENT:
{original_doc}

PREVIOUS (INCORRECT) EXTRACTION:
{json.dumps(failed_extraction, indent=2)}

SPECIFIC VALIDATION ERRORS:
{chr(10).join(f'- {e}' for e in validation_errors)}

Please re-read the original document carefully and call the tool again
with corrected values. Focus specifically on fixing the errors listed above.
"""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        tools=tools,
        tool_choice={"type": "tool", "name": tool_name},
        messages=[{"role": "user", "content": retry_prompt}]
    )

    tool_block = next(b for b in response.content if b.type == "tool_use")
    return tool_block.input
```

### Identifying When Retries Are Ineffective

```python
def should_retry_or_escalate(
    extraction: dict,
    validation_errors: list[str],
    source_document: str
) -> str:
    """Determine whether to retry, escalate, or accept with caveats."""

    for error in validation_errors:
        # Format/structural errors -> retry is effective
        if "wrong type" in error or "missing field" in error:
            return "retry"

        # Value inconsistency (e.g., total != sum of line items) -> retry
        if "calculated" in error or "inconsistent" in error:
            return "retry"

        # Data simply not present in source -> retry is INEFFECTIVE
        if "not found in document" in error or "no mention of" in error:
            return "escalate_to_human"  # Don't waste tokens retrying

        # Hallucination detected -> retry has marginal value
        if "fabricated" in error or "not supported by text" in error:
            return "escalate_to_human"  # May hallucinate differently

    return "retry"  # Default: try once more
```

### `conflict_detected` Booleans in Extraction Schemas

Design schemas with built-in conflict detection fields so downstream code can flag discrepancies automatically:

```typescript
const invoiceExtractionTool: Anthropic.Tool = {
  name: "extract_invoice_with_checks",
  description: "Extract invoice data with built-in conflict detection.",
  input_schema: {
    type: "object" as const,
    required: ["stated_total", "calculated_total", "totals_conflict_detected",
               "line_items", "tax_amount"],
    properties: {
      line_items: {
        type: "array",
        items: {
          type: "object",
          required: ["description", "amount"],
          properties: {
            description: { type: "string" },
            amount: { type: "number" },
          },
        },
      },
      stated_total: {
        type: "number",
        description: "Total amount as printed on the invoice",
      },
      calculated_total: {
        type: "number",
        description: "Sum of all line_items amounts + tax_amount (self-check)",
      },
      tax_amount: { type: ["number", "null"] },
      totals_conflict_detected: {
        type: "boolean",
        description: "True if stated_total != calculated_total. Indicates possible OCR error, missing line items, or discount not captured.",
      },
      currency_conflict_detected: {
        type: "boolean",
        description: "True if multiple currencies appear in the document.",
      },
      date_conflict_detected: {
        type: "boolean",
        description: "True if due_date is before invoice_date.",
      },
    },
  },
};
```

### Common Exam Traps

- **Trap:** Retrying when the source document simply lacks the required information. Retries only help when Claude made a fixable mistake, not when the data is absent.
- **Trap:** Sending a generic "try again" instead of SPECIFIC validation errors. Vague retry prompts lead to the same mistakes.
- **Trap:** Not setting a max retry limit. Infinite loops waste tokens and money.
- **Trap:** Assuming self-correction (e.g., `amounts_consistent` boolean) replaces external validation. It is a SIGNAL, not a guarantee. Always validate programmatically.
- **Trap:** Not including `detected_pattern` in review schemas. Without it, you cannot systematically analyze which code constructs produce false positives and iteratively improve prompts.
- **Trap:** Retrying hallucinated values expecting a correct answer. Claude may produce a *different* hallucination. Escalate to human review instead.

---

## Task 4.5: Batch Processing Strategies

### Key Concepts

**Message Batches API:**
- **50% cost reduction** compared to standard API calls
- Processing window: **up to 24 hours** (no latency SLA -- results may arrive in minutes or hours)
- Ideal for workloads where latency is not critical

**Appropriate use cases:**
- Overnight report generation
- Weekly code audits across an entire repository
- Nightly test case generation
- Bulk document classification / extraction
- Historical data re-processing

**Inappropriate use cases:**
- Pre-merge CI checks (blocking developer workflow)
- Real-time user-facing features
- Any workflow where someone is waiting for the result

**Key limitations:**
- **No multi-turn tool calling within batch requests.** Each batch request is a single message exchange. You cannot have Claude call a tool, receive results, and call another tool within one batch request.
- Each request in the batch is independent -- no shared state between them.

**`custom_id` for correlation:**
- Every request in a batch must have a unique `custom_id` (string)
- This is how you match results back to the original input
- Use it for resubmitting only failed documents

**Batch submission frequency and SLA constraints:**
- If you need results by 8 AM and the batch window is 24 hours, submit by 8 AM the previous day
- In practice, most batches complete much faster, but you cannot rely on this
- For "results by Monday morning" scenarios, submit Friday evening

### Practical Skills

- Structuring batch requests with meaningful `custom_id` values
- Calculating submission timing to meet delivery SLAs
- Implementing resubmission of only failed items
- Knowing which workloads belong in batch vs. real-time

### Code Example (Python) -- Batch Submission and Result Retrieval

```python
import anthropic
import json
import time

client = anthropic.Anthropic()

# Step 1: Prepare batch requests
documents = [
    {"id": "doc_001", "text": "First document content..."},
    {"id": "doc_002", "text": "Second document content..."},
    {"id": "doc_003", "text": "Third document content..."},
]

requests = []
for doc in documents:
    requests.append({
        "custom_id": doc["id"],  # Used to correlate results
        "params": {
            "model": "claude-sonnet-4-20250514",
            "max_tokens": 1024,
            "messages": [
                {
                    "role": "user",
                    "content": f"Summarize this document:\n\n{doc['text']}"
                }
            ]
        }
    })

# Step 2: Create the batch
batch = client.messages.batches.create(requests=requests)
print(f"Batch created: {batch.id}")
print(f"Status: {batch.processing_status}")  # "in_progress"

# Step 3: Poll for completion (in production, use webhooks or check periodically)
while True:
    batch = client.messages.batches.retrieve(batch.id)
    if batch.processing_status == "ended":
        break
    time.sleep(60)  # Check every minute

# Step 4: Retrieve results
results = {}
failed_ids = []

for result in client.messages.batches.results(batch.id):
    custom_id = result.custom_id

    if result.result.type == "succeeded":
        message = result.result.message
        results[custom_id] = message.content[0].text
    elif result.result.type == "errored":
        failed_ids.append(custom_id)
        print(f"Failed: {custom_id} - {result.result.error}")
    elif result.result.type == "expired":
        failed_ids.append(custom_id)
        print(f"Expired: {custom_id}")

# Step 5: Resubmit only failed documents
if failed_ids:
    retry_requests = [r for r in requests if r["custom_id"] in failed_ids]
    retry_batch = client.messages.batches.create(requests=retry_requests)
    print(f"Retry batch created: {retry_batch.id} with {len(retry_requests)} items")
```

### Code Example (TypeScript) -- Batch with Tool Use

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

// Batch extraction with tool use (single-turn only)
const batchRequests = documents.map((doc) => ({
  custom_id: doc.id,
  params: {
    model: "claude-sonnet-4-20250514" as const,
    max_tokens: 1024,
    tools: [
      {
        name: "extract_metadata",
        description: "Extract document metadata",
        input_schema: {
          type: "object" as const,
          required: ["title", "author", "date"],
          properties: {
            title: { type: "string" },
            author: { type: ["string", "null"] },
            date: { type: ["string", "null"] },
            keywords: {
              type: "array",
              items: { type: "string" },
            },
          },
        },
      },
    ],
    tool_choice: {
      type: "tool" as const,
      name: "extract_metadata",
    },
    messages: [
      {
        role: "user" as const,
        content: `Extract metadata:\n\n${doc.text}`,
      },
    ],
  },
}));

const batch = await client.messages.batches.create({
  requests: batchRequests,
});

console.log(`Batch ${batch.id} submitted with ${batchRequests.length} requests`);
// No latency SLA -- poll or check back later
```

### Calculating Batch Frequency for SLA Constraints

When your SLA requires results within a specific window, calculate batch submission timing carefully:

```python
# Example: SLA requires audit results every 30 hours
# Batch processing window: up to 24 hours
# Buffer for processing results: 2 hours
# Available window for submission: 30 - 24 - 2 = 4 hours

# If results must be ready by Monday 8 AM:
# Latest submission: Sunday 8 AM (24h before)
# Recommended submission: Saturday 8 PM (with 12h buffer)

# For a 4-hour processing window within a 30-hour SLA:
BATCH_SLA_HOURS = 30
MAX_BATCH_PROCESSING = 24
RESULT_PROCESSING_BUFFER = 2
SUBMISSION_WINDOW = BATCH_SLA_HOURS - MAX_BATCH_PROCESSING - RESULT_PROCESSING_BUFFER
# SUBMISSION_WINDOW = 4 hours -- you must submit within this window

# For recurring batches:
# Submit every (SLA - MAX_BATCH_PROCESSING) hours to ensure continuous coverage
SUBMISSION_INTERVAL = BATCH_SLA_HOURS - MAX_BATCH_PROCESSING  # 6 hours
```

### Prompt Refinement on Sample Before Full Batch

**Always test prompts on a small sample before submitting a full batch.** A bad prompt on 10,000 documents wastes significant money and up to 24 hours:

```python
import anthropic
import json
import random

client = anthropic.Anthropic()

def refine_prompt_then_batch(
    documents: list[dict],
    prompt_template: str,
    tools: list,
    tool_name: str,
    sample_size: int = 10
) -> str:
    """Test prompt on sample via synchronous API, then submit batch."""

    # Step 1: Random sample for synchronous testing
    sample = random.sample(documents, min(sample_size, len(documents)))

    # Step 2: Run synchronous calls on sample
    sample_results = []
    for doc in sample:
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            tools=tools,
            tool_choice={"type": "tool", "name": tool_name},
            messages=[{
                "role": "user",
                "content": prompt_template.format(document=doc["text"])
            }]
        )
        tool_block = next(
            (b for b in response.content if b.type == "tool_use"), None
        )
        if tool_block:
            sample_results.append({
                "doc_id": doc["id"],
                "result": tool_block.input
            })

    # Step 3: Review sample results manually or programmatically
    # Check: Are null fields appropriate? Are enums used correctly?
    # Are format normalization rules followed?
    print(f"Sample results ({len(sample_results)}/{sample_size}):")
    for r in sample_results:
        print(json.dumps(r, indent=2))

    # Step 4: Only after sample looks good, submit full batch
    batch_requests = []
    for doc in documents:
        batch_requests.append({
            "custom_id": doc["id"],
            "params": {
                "model": "claude-sonnet-4-20250514",
                "max_tokens": 1024,
                "tools": tools,
                "tool_choice": {"type": "tool", "name": tool_name},
                "messages": [{
                    "role": "user",
                    "content": prompt_template.format(document=doc["text"])
                }]
            }
        })

    batch = client.messages.batches.create(requests=batch_requests)
    return batch.id
```

### Handling Failures by `custom_id` and Chunking Oversized Batches

```python
def submit_with_chunking_and_retry(
    all_requests: list[dict],
    max_batch_size: int = 10000
) -> list[str]:
    """Chunk oversized request lists and handle failures per custom_id."""
    client = anthropic.Anthropic()
    batch_ids = []

    # Chunk into manageable batch sizes
    for i in range(0, len(all_requests), max_batch_size):
        chunk = all_requests[i:i + max_batch_size]
        batch = client.messages.batches.create(requests=chunk)
        batch_ids.append(batch.id)
        print(f"Submitted batch {batch.id} with {len(chunk)} requests")

    return batch_ids


def collect_and_retry_failures(
    batch_id: str,
    original_requests: list[dict]
) -> tuple[dict, list[dict]]:
    """Collect results and prepare retry list for failures only."""
    client = anthropic.Anthropic()
    results = {}
    failed_requests = []

    # Build lookup for original requests
    request_by_id = {r["custom_id"]: r for r in original_requests}

    for result in client.messages.batches.results(batch_id):
        cid = result.custom_id
        if result.result.type == "succeeded":
            results[cid] = result.result.message
        elif result.result.type in ("errored", "expired"):
            # Resubmit ONLY this failed request
            if cid in request_by_id:
                failed_requests.append(request_by_id[cid])

    if failed_requests:
        print(f"Resubmitting {len(failed_requests)} failed requests")
        retry_batch = client.messages.batches.create(requests=failed_requests)
        print(f"Retry batch: {retry_batch.id}")

    return results, failed_requests
```

### Common Exam Traps

- **Trap:** Using Message Batches for pre-merge CI checks or any blocking workflow. The 24-hour window makes this unacceptable for anything time-sensitive.
- **Trap:** Expecting multi-turn tool calling in batch mode. Each batch request is a single API call -- no tool result round-trips.
- **Trap:** Forgetting `custom_id` is how you correlate results. Without meaningful IDs, you cannot match outputs to inputs or resubmit failures selectively.
- **Trap:** Assuming batch results arrive in submission order. They do not -- process by `custom_id`.
- **Trap:** Calculating batch timing incorrectly. If the SLA is "results by 9 AM" and the max window is 24 hours, you must submit by 9 AM the previous day at the latest.
- **Trap:** Submitting a full batch without testing the prompt on a sample first. A flawed prompt wastes up to 24 hours and the full batch cost. Always run synchronous tests on 5-10 documents first.
- **Trap:** Submitting all documents in a single batch when the set is very large. Chunk into manageable sizes and track per-chunk failures for targeted resubmission.

---

## Task 4.6: Multi-Instance and Multi-Pass Review

### Key Concepts

**Self-review limitations:** When Claude generates content and then reviews it, it retains the reasoning context from generation. This makes it biased toward confirming its own output. The model tends to rationalize its original choices rather than critically evaluating them.

**Independent review instances are more effective than self-review OR extended thinking:**
- Use a SEPARATE API call (separate `messages.create()`) with a fresh conversation context
- The reviewer instance sees only the output, not the generation reasoning
- This is analogous to code review by a different developer vs. self-review
- **Extended thinking** (longer chain-of-thought within a single call) does NOT solve the self-confirmation problem: the model still retains its reasoning context and is biased toward confirming its own conclusions. Independent instances are strictly better for catching errors because they have zero shared reasoning state

**Multi-pass architecture patterns:**

1. **Per-file local analysis + cross-file integration:**
   - Pass 1: Analyze each file independently (parallelizable)
   - Pass 2: Feed all per-file findings into a cross-file integration pass
   - Pass 2 can detect: inconsistent API usage across files, missing error handling at integration points, architectural violations

2. **Generate-then-review:**
   - Instance 1: Generate the output (code, analysis, extraction)
   - Instance 2: Review the output for errors, with different instructions
   - Optionally Instance 3: Adjudicate disagreements

3. **Confidence-based routing:**
   - Claude reports `self_confidence: "high" | "medium" | "low"` alongside findings
   - High confidence: auto-accept
   - Medium confidence: lightweight human review
   - Low confidence: detailed human review
   - This is a ROUTING mechanism, not a replacement for validation

### Practical Skills

- Designing multi-pass pipelines where each pass has a focused role
- Using separate API calls for generation and review to avoid self-confirmation bias
- Implementing confidence-based routing to optimize human review time
- Parallelizing per-file analysis and aggregating into cross-file passes

### Code Example (Python) -- Multi-Pass Review Architecture

```python
import anthropic
import json
from concurrent.futures import ThreadPoolExecutor

client = anthropic.Anthropic()

def analyze_file(file_path: str, file_content: str) -> dict:
    """Pass 1: Per-file local analysis (parallelizable)."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        tools=[{
            "name": "file_analysis",
            "description": "Report findings for a single file.",
            "input_schema": {
                "type": "object",
                "required": ["file_path", "findings", "exports_used",
                            "imports_from"],
                "properties": {
                    "file_path": {"type": "string"},
                    "findings": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "required": ["line", "severity", "description",
                                        "confidence"],
                            "properties": {
                                "line": {"type": "integer"},
                                "severity": {
                                    "type": "string",
                                    "enum": ["critical", "high", "medium", "low"]
                                },
                                "description": {"type": "string"},
                                "confidence": {
                                    "type": "string",
                                    "enum": ["high", "medium", "low"]
                                }
                            }
                        }
                    },
                    "exports_used": {
                        "type": "array",
                        "items": {"type": "string"},
                        "description": "Functions/classes this file exports"
                    },
                    "imports_from": {
                        "type": "array",
                        "items": {"type": "string"},
                        "description": "Other project files this file imports from"
                    }
                }
            }
        }],
        tool_choice={"type": "tool", "name": "file_analysis"},
        messages=[{
            "role": "user",
            "content": f"Analyze this file for bugs and security issues.\n\n"
                      f"File: {file_path}\n```\n{file_content}\n```"
        }]
    )

    tool_block = next(b for b in response.content if b.type == "tool_use")
    return tool_block.input


def cross_file_integration(file_analyses: list[dict]) -> dict:
    """Pass 2: Cross-file integration analysis."""
    summary = json.dumps(file_analyses, indent=2)

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": f"""Review these per-file analysis results and identify
cross-file issues that individual analyses would miss:

- Inconsistent error handling patterns across modules
- Broken import/export dependencies
- API contract mismatches between caller and callee
- Security issues at module boundaries

Per-file analyses:
{summary}"""
        }]
    )
    return {"cross_file_findings": response.content[0].text}


def independent_review(original_output: str, original_prompt: str) -> dict:
    """Separate instance reviews output WITHOUT access to generation reasoning."""
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        tools=[{
            "name": "review_result",
            "description": "Review the output for errors.",
            "input_schema": {
                "type": "object",
                "required": ["is_acceptable", "issues_found", "confidence"],
                "properties": {
                    "is_acceptable": {"type": "boolean"},
                    "issues_found": {
                        "type": "array",
                        "items": {"type": "string"}
                    },
                    "confidence": {
                        "type": "string",
                        "enum": ["high", "medium", "low"]
                    }
                }
            }
        }],
        tool_choice={"type": "tool", "name": "review_result"},
        messages=[{
            "role": "user",
            "content": f"Review this output for correctness and completeness.\n\n"
                      f"TASK: {original_prompt}\n\n"
                      f"OUTPUT TO REVIEW:\n{original_output}"
        }]
    )

    tool_block = next(b for b in response.content if b.type == "tool_use")
    return tool_block.input


# Orchestration: parallel per-file analysis, then integration
def run_multi_pass_review(files: dict[str, str]):
    """Full multi-pass pipeline."""

    # Pass 1: Parallel per-file analysis
    with ThreadPoolExecutor(max_workers=5) as executor:
        futures = {
            executor.submit(analyze_file, path, content): path
            for path, content in files.items()
        }
        file_analyses = []
        for future in futures:
            file_analyses.append(future.result())

    # Pass 2: Cross-file integration
    integration = cross_file_integration(file_analyses)

    # Route by confidence
    for analysis in file_analyses:
        for finding in analysis["findings"]:
            if finding["confidence"] == "low":
                # Queue for human review
                queue_for_human_review(finding)
            elif finding["confidence"] == "medium":
                # Independent review instance
                review = independent_review(
                    json.dumps(finding),
                    f"Verify this finding in {analysis['file_path']}"
                )
                if review["is_acceptable"]:
                    accept_finding(finding)
                else:
                    queue_for_human_review(finding)
            else:
                # High confidence -- auto-accept
                accept_finding(finding)

    return {
        "file_analyses": file_analyses,
        "integration": integration
    }
```

### Code Example (TypeScript) -- Generate-then-Review Pattern

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

async function generateAndReview(task: string): Promise<{
  output: string;
  review: { approved: boolean; issues: string[] };
}> {
  // Instance 1: Generate
  const generation = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 2048,
    messages: [{ role: "user", content: task }],
  });

  const output = generation.content
    .filter((b) => b.type === "text")
    .map((b) => {
      if (b.type === "text") return b.text;
      return "";
    })
    .join("");

  // Instance 2: Independent review (fresh context -- no generation reasoning)
  const reviewResponse = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    tools: [
      {
        name: "submit_review",
        description: "Submit review of the generated output",
        input_schema: {
          type: "object" as const,
          required: ["approved", "issues", "confidence"],
          properties: {
            approved: { type: "boolean" },
            issues: { type: "array", items: { type: "string" } },
            confidence: {
              type: "string",
              enum: ["high", "medium", "low"],
            },
          },
        },
      },
    ],
    tool_choice: { type: "tool", name: "submit_review" },
    messages: [
      {
        role: "user",
        content: `You are reviewing output generated by another AI instance.
Check for: factual errors, logical inconsistencies, missing information,
hallucinated details.

ORIGINAL TASK: ${task}

OUTPUT TO REVIEW:
${output}`,
      },
    ],
  });

  const reviewBlock = reviewResponse.content.find(
    (b) => b.type === "tool_use"
  );
  const review =
    reviewBlock && reviewBlock.type === "tool_use"
      ? (reviewBlock.input as { approved: boolean; issues: string[] })
      : { approved: false, issues: ["Review failed"] };

  return { output, review };
}
```

### Verification Passes with Self-Reported Confidence for Routing

A dedicated **verification pass** re-examines findings and assigns confidence, enabling automated routing decisions:

```python
import anthropic
import json

client = anthropic.Anthropic()

def verification_pass(
    original_findings: list[dict],
    source_code: str
) -> list[dict]:
    """Third pass: verify findings and assign confidence for routing."""
    verification_tool = {
        "name": "verify_findings",
        "description": "Verify each finding and assign routing confidence.",
        "input_schema": {
            "type": "object",
            "required": ["verified_findings"],
            "properties": {
                "verified_findings": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "required": ["original_finding", "verification_status",
                                    "confidence", "routing_decision"],
                        "properties": {
                            "original_finding": {"type": "string"},
                            "verification_status": {
                                "type": "string",
                                "enum": ["confirmed", "uncertain", "likely_false_positive"]
                            },
                            "confidence": {
                                "type": "string",
                                "enum": ["high", "medium", "low"]
                            },
                            "routing_decision": {
                                "type": "string",
                                "enum": ["auto_accept", "lightweight_review", "detailed_review", "auto_reject"],
                                "description": "high+confirmed=auto_accept, medium=lightweight_review, low=detailed_review, likely_fp=auto_reject"
                            },
                            "verification_reasoning": {"type": "string"}
                        }
                    }
                }
            }
        }
    }

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        tools=[verification_tool],
        tool_choice={"type": "tool", "name": "verify_findings"},
        messages=[{
            "role": "user",
            "content": f"""You are an independent verification instance. You did NOT
generate these findings. Review each finding against the source code and assess
whether it is a genuine issue.

SOURCE CODE:
```
{source_code}
```

FINDINGS TO VERIFY:
{json.dumps(original_findings, indent=2)}

For each finding, verify it against the code and assign:
- verification_status: confirmed, uncertain, or likely_false_positive
- confidence: high, medium, or low
- routing_decision based on the matrix:
  confirmed + high confidence -> auto_accept
  confirmed + medium confidence -> lightweight_review
  uncertain OR low confidence -> detailed_review
  likely_false_positive -> auto_reject
"""
        }]
    )

    tool_block = next(b for b in response.content if b.type == "tool_use")
    return tool_block.input["verified_findings"]


# Usage in pipeline:
# Pass 1: Generate findings (instance 1)
# Pass 2: Independent review (instance 2)
# Pass 3: Verification with confidence routing (instance 3)
# Only "detailed_review" items reach human reviewers
```

```typescript
// TypeScript: Routing based on verification pass results
interface VerifiedFinding {
  original_finding: string;
  verification_status: "confirmed" | "uncertain" | "likely_false_positive";
  confidence: "high" | "medium" | "low";
  routing_decision: "auto_accept" | "lightweight_review" | "detailed_review" | "auto_reject";
}

function routeFindings(verified: VerifiedFinding[]) {
  const autoAccepted = verified.filter(f => f.routing_decision === "auto_accept");
  const needsLightReview = verified.filter(f => f.routing_decision === "lightweight_review");
  const needsDetailedReview = verified.filter(f => f.routing_decision === "detailed_review");
  const autoRejected = verified.filter(f => f.routing_decision === "auto_reject");

  console.log(`Auto-accepted: ${autoAccepted.length}`);
  console.log(`Needs lightweight review: ${needsLightReview.length}`);
  console.log(`Needs detailed review: ${needsDetailedReview.length}`);
  console.log(`Auto-rejected (likely FP): ${autoRejected.length}`);

  // Only detailed_review items consume human reviewer time
  return { autoAccepted, needsLightReview, needsDetailedReview, autoRejected };
}
```

### Common Exam Traps

- **Trap:** Believing self-review (same conversation) is as effective as independent review (separate API call). It is NOT -- the model retains its generation reasoning and is biased toward confirming it.
- **Trap:** Thinking extended thinking (longer chain-of-thought) solves the self-confirmation bias. It does NOT -- the model still reasons from the same context. Independent instances are strictly better.
- **Trap:** Running cross-file analysis as a single monolithic pass. The correct pattern is per-file local analysis FIRST (parallelizable), then a cross-file integration pass.
- **Trap:** Treating self-reported confidence as ground truth. It is a ROUTING signal for deciding the level of human review needed, not a reliability guarantee.
- **Trap:** Using multi-pass review for simple, well-defined extraction tasks. Multi-pass adds latency and cost -- reserve it for complex analysis where errors are costly.
- **Trap:** Skipping the verification pass. Without it, you cannot route findings by confidence and all findings require the same level of human review.

---

## Quick Reference: Key Decision Matrix

| Need | Best approach |
|------|--------------|
| Consistent output format | Few-shot examples (Task 4.2) |
| Schema-compliant structured data | tool_use with JSON schema (Task 4.3) |
| Reduce false positives | Explicit criteria + disable noisy categories (Task 4.1) |
| Fix format errors in output | Retry with specific error feedback (Task 4.4) |
| Bulk processing, cost savings | Message Batches API (Task 4.5) |
| Catch errors in generated content | Independent review instance (Task 4.6) |
| Ensure semantic correctness | Cross-check fields + programmatic validation (Task 4.4) |
| Handle missing source data | Nullable fields + "not found" examples (Task 4.2 + 4.3) |

## Key API Reference Summary

```python
# tool_choice variants
tool_choice = {"type": "auto"}                         # Claude decides; may return text
tool_choice = {"type": "any"}                          # Must call some tool
tool_choice = {"type": "tool", "name": "my_tool"}     # Must call specific tool

# Batch API
batch = client.messages.batches.create(requests=[...]) # Submit
batch = client.messages.batches.retrieve(batch_id)     # Check status
results = client.messages.batches.results(batch_id)    # Get results

# Nullable field in schema
{"type": ["string", "null"]}  # Field can be string or null

# Enum with escape hatch
{"type": "string", "enum": ["option_a", "option_b", "other"]}
```

## Exam Preparation Checklist

- [ ] Can you explain why "be conservative" does NOT improve precision?
- [ ] Can you design few-shot examples for an ambiguous classification task?
- [ ] Can you write a tool definition with JSON schema including nullable fields and enum+other?
- [ ] Do you know the three `tool_choice` modes and when to use each?
- [ ] Can you explain why schema enforcement prevents syntax but not semantic errors?
- [ ] Can you implement a validation-retry loop with specific error feedback?
- [ ] Do you know when retries are ineffective (missing source data)?
- [ ] Can you calculate batch submission timing given a delivery SLA?
- [ ] Do you know why multi-turn tool calling is NOT available in batch mode?
- [ ] Can you explain why independent review instances beat self-review?
- [ ] Can you design a multi-pass pipeline (per-file local + cross-file integration)?
- [ ] Do you know that self-reported confidence is a routing signal, not a guarantee?
- [ ] Can you explain why extended thinking does NOT solve self-confirmation bias?
- [ ] Can you design a verification pass with confidence-based routing?
- [ ] Do you know how to calculate batch submission timing for a given SLA (e.g., 4-hour window for 30-hour SLA)?
- [ ] Can you explain why prompt refinement on a sample should precede full batch submission?
- [ ] Do you know the `detected_pattern` approach for systematic false positive analysis?
- [ ] Can you use `tool_choice: "any"` for unknown document types with multiple tool schemas?
- [ ] Do you understand the difference between `"unclear"` and `"other"` in enum values?
- [ ] Can you write format normalization rules alongside JSON schemas?

## Source Links

- [Anthropic Tool Use Documentation](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview)
- [Anthropic Tool Choice Configuration](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview#controlling-tool-use)
- [Anthropic Structured Output with Tool Use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/forced-tool-use)
- [Anthropic Message Batches API](https://docs.anthropic.com/en/docs/build-with-claude/message-batches)
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [Anthropic Few-Shot Prompting](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/use-examples)
- [Anthropic JSON Mode / Structured Output](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/structured-output)
- [Anthropic Extended Thinking](https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking)
- [Anthropic API Reference - Messages](https://docs.anthropic.com/en/api/messages)
