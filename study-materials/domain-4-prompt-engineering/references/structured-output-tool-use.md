# Structured Output via Tool Use
Source: https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview
Fetched: 2026-03-21

## Key Concepts for Exam

### Using Tools for Structured Output
Tool use is a primary mechanism for getting structured JSON output from Claude. By defining a tool with a specific input_schema, you force Claude to return data conforming to that schema.

### Strict Tool Use (Structured Outputs)
- Add `strict: true` to tool definitions for **guaranteed schema validation**
- Ensures Claude's tool calls ALWAYS match your schema exactly
- Eliminates type mismatches or missing fields
- Related feature: [Structured Outputs](/docs/en/build-with-claude/structured-outputs)
- When to use: Production agents where invalid tool parameters would cause failures

### tool_choice for Forcing Structured Output
- `tool_choice: {"type": "tool", "name": "my_tool"}` - Forces Claude to use a specific tool, guaranteeing structured output
- `tool_choice: {"type": "any"}` - Forces Claude to use at least one tool
- `tool_choice: {"type": "auto"}` - Claude decides (default)
- `tool_choice: {"type": "none"}` - Claude won't use tools

### JSON Schema in Tool Definitions
Tool input schemas use standard JSON Schema:
```json
{
  "type": "object",
  "properties": {
    "location": {
      "type": "string",
      "description": "The city and state, e.g. San Francisco, CA"
    },
    "unit": {
      "type": "string",
      "enum": ["celsius", "fahrenheit"],
      "description": "Temperature unit"
    }
  },
  "required": ["location"]
}
```

Supported JSON Schema features:
- `type`: string, number, integer, boolean, array, object
- `enum`: constrained values
- `required`: mandatory fields
- `properties`: object field definitions
- `description`: field-level descriptions (important for Claude to understand intent)

## API Reference

### Tool Definition with Structured Output
```json
{
  "name": "extract_entity",
  "description": "Extract structured entity data from text",
  "input_schema": {
    "type": "object",
    "properties": {
      "name": {"type": "string"},
      "age": {"type": "integer"},
      "email": {"type": "string"}
    },
    "required": ["name", "age", "email"]
  }
}
```

### Response Format when Tool is Used
```json
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01A09q90qw90lq917835lq9",
      "name": "extract_entity",
      "input": {
        "name": "John Doe",
        "age": 30,
        "email": "john@example.com"
      }
    }
  ]
}
```

The `input` field contains the structured JSON conforming to your schema.

## Code Examples

### Python - Structured JSON Extraction via Tool Use
```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    tools=[
        {
            "name": "get_weather",
            "description": "Get the current weather in a given location",
            "input_schema": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "The city and state, e.g. San Francisco, CA",
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "Temperature unit",
                    },
                },
                "required": ["location"],
            },
        }
    ],
    messages=[{"role": "user", "content": "What's the weather like in San Francisco?"}],
)
```

### TypeScript - Structured JSON Extraction
```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const response = await client.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 1024,
  tools: [
    {
      name: "get_weather",
      description: "Get the current weather in a given location",
      input_schema: {
        type: "object",
        properties: {
          location: {
            type: "string",
            description: "The city and state, e.g. San Francisco, CA"
          },
          unit: {
            type: "string",
            enum: ["celsius", "fahrenheit"]
          }
        },
        required: ["location"]
      }
    }
  ],
  messages: [{ role: "user", content: "What's the weather like in San Francisco?" }]
});
```

## Important Details for Exam

### Key Differences: JSON Output vs Strict Tool Use
- **JSON mode**: Request Claude to output JSON in text response (less reliable schema conformance)
- **Tool use**: Define schema as tool input_schema; Claude returns structured data in `tool_use` block
- **Strict tool use** (`strict: true`): Guaranteed schema validation on tool inputs

### Best Practices for Structured Output
1. Use detailed `description` fields in your schema - these guide Claude on what values to produce
2. Use `required` to ensure all needed fields are present
3. Use `enum` to constrain values to valid options
4. Use `tool_choice: {"type": "tool", "name": "..."}` to force tool usage when you always want structured output
5. With `strict: true`, you get schema guarantees eliminating need for client-side validation

### stop_reason Indicates Output Type
- `"tool_use"` -> Response contains structured tool call data
- `"end_turn"` or `"stop_sequence"` -> Response contains text (not structured tool call)

### Extracting Structured Data from Response
When `stop_reason` is `"tool_use"`, iterate through `content` array to find blocks with `type: "tool_use"` and extract the `input` field for your structured data.

### Parallel Structured Output
Claude can return multiple `tool_use` blocks in a single response for parallel tool calls. Each block has its own `id`, `name`, and `input` with structured data.
