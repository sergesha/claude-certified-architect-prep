# Claude API Tool Use Reference
Source: https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview
Fetched: 2026-03-21

## Key Concepts for Exam

### Two Types of Tools
1. **Client tools**: Execute on YOUR systems
   - User-defined custom tools you create and implement
   - Anthropic-defined tools (computer use, text editor) that require client implementation
2. **Server tools**: Execute on Anthropic's servers
   - Web search (`web_search_20250305`), web fetch (`web_fetch_tool`)
   - Must be specified in API request but don't require your implementation
   - Anthropic-defined tools use versioned types for compatibility

### Tool Use Workflow (Client Tools)
1. Provide Claude with tools (names, descriptions, input schemas) + user prompt
2. Claude decides to use a tool -> API response has `stop_reason: "tool_use"`
3. Extract tool name + input, execute on your system, return results in `tool_result` content block within a `user` message
4. Claude uses tool result to formulate final response
- Steps 3-4 are OPTIONAL - sometimes just the tool call (step 2) is all you need

### Server Tools Workflow
1. Provide server tools + user prompt
2. Claude executes tool on Anthropic's servers; results automatically incorporated
3. Server runs a sampling loop (default limit: 10 iterations)
4. If limit reached: returns `stop_reason="pause_turn"` (may include `server_tool_use` block without corresponding `server_tool_result`)
5. When you receive `pause_turn`, send response back to let Claude finish processing

### MCP Tools Integration
- MCP tool definitions use `inputSchema` -> rename to `input_schema` for Claude's format
- Convert via `list_tools()` on MCP server, then map to Claude format
- MCP connector available to connect directly to remote MCP servers without implementing a client

## API Reference

### Tool Definition Schema
```json
{
  "name": "get_weather",
  "description": "Get the current weather in a given location",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "The city and state, e.g. San Francisco, CA"
      },
      "unit": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "description": "The unit of temperature"
      }
    },
    "required": ["location"]
  }
}
```

### Strict Tool Use (Structured Outputs)
- Add `strict: true` to tool definitions for guaranteed schema validation
- Ensures Claude's tool calls ALWAYS match your schema exactly
- Eliminates type mismatches or missing fields
- Perfect for production agents where invalid tool parameters would cause failures

### tool_choice Options
- `auto` - Claude decides whether to use tools (default)
- `none` - Claude won't use any tools
- `any` - Claude MUST use one of the provided tools
- `tool` - Claude MUST use a specific named tool

### stop_reason Values
- `"tool_use"` - Claude wants to call a client tool
- `"stop_sequence"` - normal completion after tool result processing
- `"end_turn"` - normal end of response
- `"pause_turn"` - server-side sampling loop hit iteration limit (default 10)

### Response Format (Tool Use)
```json
{
  "id": "msg_01Aq9w938a90dw8q",
  "model": "claude-opus-4-6",
  "stop_reason": "tool_use",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "I'll check the current weather in San Francisco for you."
    },
    {
      "type": "tool_use",
      "id": "toolu_01A09q90qw90lq917835lq9",
      "name": "get_weather",
      "input": { "location": "San Francisco, CA", "unit": "celsius" }
    }
  ]
}
```

### Tool Result Format
Tool results go in a `user` message with `tool_result` content block:
```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
      "content": "15 degrees"
    }
  ]
}
```

## Code Examples

### Python - Basic Tool Use
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
                    }
                },
                "required": ["location"],
            },
        }
    ],
    messages=[{"role": "user", "content": "What's the weather like in San Francisco?"}],
)
```

### TypeScript - Basic Tool Use
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
          }
        },
        required: ["location"]
      }
    }
  ],
  messages: [{ role: "user", content: "What's the weather like in San Francisco?" }]
});
```

### Python - Converting MCP Tools to Claude Format
```python
from mcp import ClientSession

async def get_claude_tools(mcp_session: ClientSession):
    """Convert MCP tools to Claude's tool format."""
    mcp_tools = await mcp_session.list_tools()

    claude_tools = []
    for tool in mcp_tools.tools:
        claude_tools.append(
            {
                "name": tool.name,
                "description": tool.description or "",
                "input_schema": tool.inputSchema,  # Rename inputSchema to input_schema
            }
        )
    return claude_tools
```

### Python - Full Tool Result Loop
```python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    tools=[...],
    messages=[
        {"role": "user", "content": "What's the weather like in San Francisco?"},
        {
            "role": "assistant",
            "content": [
                {"type": "text", "text": "I'll check the current weather."},
                {
                    "type": "tool_use",
                    "id": "toolu_01A09q90qw90lq917835lq9",
                    "name": "get_weather",
                    "input": {"location": "San Francisco, CA", "unit": "celsius"},
                },
            ],
        },
        {
            "role": "user",
            "content": [
                {
                    "type": "tool_result",
                    "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
                    "content": "15 degrees",
                }
            ],
        },
    ],
)
```

## Important Details for Exam

### Parallel Tool Use
- Claude can call multiple tools in parallel within a single response
- All `tool_use` blocks are included in a single assistant message
- ALL corresponding `tool_result` blocks MUST be provided in a single subsequent user message
- Tool results must be formatted correctly to avoid API errors

### Missing Information Behavior
- **Claude Opus**: Much more likely to recognize missing required parameters and ASK for them
- **Claude Sonnet**: May ask (especially when prompted to think first) but may also GUESS/INFER values
- **Claude Haiku**: More likely to call unnecessary tools or infer missing parameters

### Sequential Tool Chaining
- Claude calls one tool at a time when outputs are dependencies
- If prompted to call all at once, Claude may GUESS parameters for downstream tools
- Example: get_location -> get_weather (location result feeds into weather call)

### Chain of Thought for Better Tool Use
Prompt for Sonnet/Haiku to improve tool assessment:
> "Answer the user's request using relevant tools (if they are available). Before calling a tool, do some analysis. First, think about which of the provided tools is the relevant tool to answer the user's request. Second, go through each of the required parameters of the relevant tool and determine if the user has directly provided or given enough information to infer a value..."

Key instruction: "If one of the values for a required parameter is missing, DO NOT invoke the function and instead, ask the user to provide the missing parameters."

### Pricing Details
- Tool use is priced based on total input tokens (including `tools` parameter) + output tokens
- Server-side tools may incur additional usage-based pricing
- Additional tokens come from: `tools` parameter, `tool_use` blocks, `tool_result` blocks
- Automatic system prompt added when tools provided

### Tool Use System Prompt Token Counts
| Model | auto/none | any/tool |
|-------|-----------|----------|
| Claude Opus 4.x / Sonnet 4.x / Haiku 4.5 | 346 tokens | 313 tokens |
| Claude Haiku 3.5 | 264 tokens | 340 tokens |
| Claude Opus 3 | 530 tokens | 281 tokens |

### Empty Tool Schema
Tools with no parameters use empty properties:
```json
{
  "name": "get_location",
  "description": "Get the current user location based on their IP address.",
  "input_schema": {
    "type": "object",
    "properties": {}
  }
}
```
