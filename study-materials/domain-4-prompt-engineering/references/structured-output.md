# Structured Output / JSON Mode
Source: https://docs.anthropic.com/en/docs/build-with-claude/tool-use (structured output via tools)
Also: https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/prefill-claudes-response
Fetched: 2026-03-21 (from training knowledge — WebFetch/WebSearch were denied)

## Key Concepts for Exam

### Two Approaches to Structured Output

**1. Tool Use (Recommended for reliable structured output)**
- Define a tool with a JSON Schema `input_schema`
- Force Claude to use it with `tool_choice: {"type": "tool", "name": "tool_name"}`
- Claude returns structured JSON matching the schema
- **Most reliable method** — schema-validated output

**2. Text-Based JSON (via prompting/prefilling)**
- Instruct Claude to output JSON in the text response
- Prefill the assistant response with `{` or `[` to force JSON
- Less reliable — no schema validation, may include extra text
- Simpler to implement for quick prototyping

### Comparison: Tool Use vs Text-Based JSON

| Feature | Tool Use | Text-Based JSON |
|---------|----------|-----------------|
| Schema validation | Yes (JSON Schema) | No |
| Reliability | High | Medium |
| Nested structures | Excellent | Good |
| Extra text risk | None (pure JSON) | May include prose |
| Setup complexity | More boilerplate | Minimal |
| Streaming parsing | Tool input streaming | Text streaming |
| Use case | Production systems | Prototyping |

### Tool Use for Structured Output (Primary Method)

The key insight: you can define a "tool" that doesn't actually call anything external. You just use the tool definition as a structured output schema, and extract the JSON from the tool call.

```python
import anthropic

client = anthropic.Anthropic()

# Define a tool purely for structured extraction
tools = [
    {
        "name": "extract_entities",
        "description": "Extract named entities from the text",
        "input_schema": {
            "type": "object",
            "properties": {
                "people": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "properties": {
                            "name": {"type": "string"},
                            "role": {"type": "string"},
                            "organization": {"type": "string"}
                        },
                        "required": ["name"]
                    },
                    "description": "List of people mentioned"
                },
                "organizations": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "List of organizations mentioned"
                },
                "locations": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "List of locations mentioned"
                }
            },
            "required": ["people", "organizations", "locations"]
        }
    }
]

message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "tool", "name": "extract_entities"},  # Force this specific tool
    messages=[
        {
            "role": "user",
            "content": "John Smith, CEO of Acme Corp in New York, met with Jane Doe from GlobalTech in London."
        }
    ]
)

# Extract the structured data from the tool call
tool_input = message.content[0].input  # This is the structured JSON
print(tool_input)
# {
#   "people": [
#     {"name": "John Smith", "role": "CEO", "organization": "Acme Corp"},
#     {"name": "Jane Doe", "role": null, "organization": "GlobalTech"}
#   ],
#   "organizations": ["Acme Corp", "GlobalTech"],
#   "locations": ["New York", "London"]
# }
```

### Text-Based JSON (Prefilling Method)

```python
message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": """Extract entities from this text as JSON with keys "people", "orgs", "locations":

John Smith, CEO of Acme Corp in New York, met with Jane Doe from GlobalTech in London."""
        },
        {
            "role": "assistant",
            "content": "{"  # Prefill to force JSON
        }
    ]
)

# Note: response will start from where the prefill left off
# You need to prepend the "{" back to parse
import json
result = json.loads("{" + message.content[0].text)
```

## API Reference / Configuration

### Tool Definition Schema
```json
{
    "name": "string (required) — tool name, [a-zA-Z0-9_-]+",
    "description": "string (optional but recommended) — what the tool does",
    "input_schema": {
        "type": "object",
        "properties": {
            "param_name": {
                "type": "string|number|integer|boolean|array|object",
                "description": "Parameter description",
                "enum": ["optional", "list", "of", "allowed", "values"]
            }
        },
        "required": ["list_of_required_params"]
    }
}
```

### tool_choice Options (Critical for Exam)
```python
# Auto: Claude decides whether to use tools (default)
tool_choice={"type": "auto"}

# Any: Claude MUST use a tool, but picks which one
tool_choice={"type": "any"}

# Specific tool: Claude MUST use this exact tool
tool_choice={"type": "tool", "name": "extract_entities"}

# None: Claude will not use any tools
tool_choice={"type": "none"}
```

### Response Content Block Types

When Claude uses a tool, the response contains `tool_use` content blocks:

```json
{
    "content": [
        {
            "type": "text",
            "text": "I'll extract the entities."
        },
        {
            "type": "tool_use",
            "id": "toolu_01ABC123",
            "name": "extract_entities",
            "input": {
                "people": [{"name": "John Smith", "role": "CEO"}],
                "organizations": ["Acme Corp"],
                "locations": ["New York"]
            }
        }
    ],
    "stop_reason": "tool_use"
}
```

**Important**: When `tool_choice` is set to a specific tool, the response may NOT include a `text` block — it may only have the `tool_use` block.

### Tool Result (for multi-turn tool use)
```python
# After receiving a tool_use response, send back the result:
messages = [
    {"role": "user", "content": "What's the weather in SF?"},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "toolu_01ABC123", "name": "get_weather", "input": {"location": "San Francisco"}}
    ]},
    {"role": "user", "content": [
        {
            "type": "tool_result",
            "tool_use_id": "toolu_01ABC123",
            "content": "72°F, sunny"
        }
    ]}
]
```

## Code Examples

### Python: Structured Extraction with Tool Use
```python
import anthropic
import json

client = anthropic.Anthropic()

def extract_structured(text: str, schema: dict, schema_name: str) -> dict:
    """Generic structured extraction using tool_use."""
    message = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        tools=[{
            "name": schema_name,
            "description": f"Extract structured data matching the {schema_name} schema",
            "input_schema": schema
        }],
        tool_choice={"type": "tool", "name": schema_name},
        messages=[{"role": "user", "content": text}]
    )

    # Find the tool_use block
    for block in message.content:
        if block.type == "tool_use":
            return block.input
    return {}

# Example: Extract product info
product_schema = {
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "price": {"type": "number"},
        "currency": {"type": "string", "enum": ["USD", "EUR", "GBP"]},
        "features": {"type": "array", "items": {"type": "string"}},
        "in_stock": {"type": "boolean"}
    },
    "required": ["name", "price", "currency"]
}

result = extract_structured(
    "The UltraWidget Pro costs $49.99 and features wireless charging, water resistance, and a 3-year warranty. Currently available.",
    product_schema,
    "extract_product"
)
print(json.dumps(result, indent=2))
```

### TypeScript: Structured Output
```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

interface ExtractedEntity {
    people: Array<{ name: string; role?: string }>;
    organizations: string[];
    locations: string[];
}

async function extractEntities(text: string): Promise<ExtractedEntity> {
    const message = await client.messages.create({
        model: "claude-sonnet-4-20250514",
        max_tokens: 1024,
        tools: [{
            name: "extract_entities",
            description: "Extract named entities from text",
            input_schema: {
                type: "object",
                properties: {
                    people: {
                        type: "array",
                        items: {
                            type: "object",
                            properties: {
                                name: { type: "string" },
                                role: { type: "string" }
                            },
                            required: ["name"]
                        }
                    },
                    organizations: { type: "array", items: { type: "string" } },
                    locations: { type: "array", items: { type: "string" } }
                },
                required: ["people", "organizations", "locations"]
            }
        }],
        tool_choice: { type: "tool" as const, name: "extract_entities" },
        messages: [{ role: "user", content: text }]
    });

    const toolBlock = message.content.find(b => b.type === "tool_use");
    if (toolBlock && toolBlock.type === "tool_use") {
        return toolBlock.input as ExtractedEntity;
    }
    throw new Error("No tool use block in response");
}
```

### Python: Classification with Enum Constraints
```python
# Use enum in schema to constrain output to specific values
classification_tool = {
    "name": "classify_sentiment",
    "description": "Classify the sentiment of the text",
    "input_schema": {
        "type": "object",
        "properties": {
            "sentiment": {
                "type": "string",
                "enum": ["positive", "negative", "neutral", "mixed"],
                "description": "The overall sentiment"
            },
            "confidence": {
                "type": "number",
                "description": "Confidence score from 0.0 to 1.0"
            },
            "key_phrases": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Key phrases that indicate the sentiment"
            }
        },
        "required": ["sentiment", "confidence"]
    }
}

message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[classification_tool],
    tool_choice={"type": "tool", "name": "classify_sentiment"},
    messages=[{"role": "user", "content": "This product is amazing but shipping was terrible."}]
)

# result.input = {"sentiment": "mixed", "confidence": 0.85, "key_phrases": ["amazing", "terrible"]}
```

## Important Details

- **Tool use is the recommended approach** for production structured output. It provides schema validation.
- **tool_choice with specific name** is critical for guaranteed structured output — without it, Claude may choose not to use the tool.
- **`tool_choice: {"type": "any"}`** forces tool use but lets Claude pick which tool. Good when you have multiple extraction schemas.
- **stop_reason**: When Claude uses a tool, `stop_reason` is `"tool_use"`, not `"end_turn"`.
- **Tool input is valid JSON**: The `input` field of a `tool_use` block is always valid JSON matching the schema (when schema is well-defined).
- **Descriptions matter**: Tool and parameter descriptions significantly affect extraction quality. Be specific.
- **Required fields**: Use the `required` array in the schema to ensure critical fields are always present.
- **Nested objects**: Tool schemas support arbitrary nesting — objects within objects, arrays of objects, etc.
- **Prefilling limitation**: When prefilling with `{`, you must reconstruct the full JSON by prepending `{` to the response. The response may also include trailing text after the JSON.
- **No native JSON mode**: Unlike some other APIs, Anthropic does not have a dedicated `response_format: {"type": "json_object"}` parameter. Tool use IS the JSON mode.
- **Batch + Tool use**: You can use tools in batch requests, but only single-turn (no tool result follow-up). Use forced tool_choice for extraction.
- **Token cost**: Tool definitions count toward input tokens. Keep schemas focused to minimize costs.
