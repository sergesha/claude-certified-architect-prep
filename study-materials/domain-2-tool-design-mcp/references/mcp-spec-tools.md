# MCP Tools Specification
Source: https://modelcontextprotocol.io/docs/concepts/tools
Fetched: 2026-03-21

## Key Concepts for Exam

- Tools are **model-controlled** -- the LLM discovers and invokes tools automatically
- This contrasts with Resources (application-controlled) and Prompts (user-controlled)
- Tools allow LLMs to perform actions, compute results, and interact with external systems
- Servers expose tools via `tools/list`; clients invoke them via `tools/call`
- Tools are analogous to POST endpoints in REST -- they perform actions and may have side effects
- There SHOULD always be a human in the loop with ability to deny tool invocations

### Tool Definition Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique identifier for the tool |
| `title` | No | Human-readable display name |
| `description` | No | Human-readable description of what the tool does |
| `inputSchema` | Yes | JSON Schema object defining expected parameters |
| `outputSchema` | No | JSON Schema object defining expected structured output |
| `annotations` | No | Optional hints about tool behavior (see below) |

### Tool Annotations

Annotations provide hints about tool behavior without affecting functionality:

- `title` -- human-readable title
- `readOnlyHint` (default: false) -- tool does not modify its environment
- `destructiveHint` (default: true) -- tool may perform destructive updates (only meaningful when readOnlyHint is false)
- `idempotentHint` (default: false) -- calling repeatedly with same args has no additional effect (only meaningful when readOnlyHint is false)
- `openWorldHint` (default: true) -- tool interacts with external entities

**IMPORTANT**: Clients MUST consider tool annotations to be untrusted unless they come from trusted servers.

### Two Error Types (CRITICAL FOR EXAM)

1. **Protocol errors** -- standard JSON-RPC error responses (malformed requests, unknown tools, etc.). Uses `error` field.
2. **Tool execution errors** -- indicated by `isError: true` in the tool result. The tool ran but encountered an error. Uses `result` field with `isError: true`. Content still contains error information for the LLM to see and potentially recover from.

Key distinction: Protocol errors use JSON-RPC `error` field (tool not found, invalid args). Tool execution errors use `result` with `isError: true` (the tool was found and called, but the operation failed).

### Tool Result Content Types

Tool results can include multiple content items of these types:
- `text` -- plain text or structured text output
- `image` -- base64-encoded image data with MIME type
- `audio` -- base64-encoded audio data with MIME type
- `resource_link` -- links to resources (NOT guaranteed to appear in `resources/list`)
- `resource` -- embedded resources with inline content

### Structured Content

- The `structuredContent` field returns machine-readable data alongside human-readable `content`
- Validated against the tool's `outputSchema` if one is defined
- Servers MUST provide structured results that conform to the schema
- Clients SHOULD validate structured results against the schema
- For backwards compatibility, a tool that returns structured content SHOULD also return the serialized JSON in a TextContent block

## Protocol/Configuration Details

### Capabilities Declaration

Servers that support tools MUST declare the `tools` capability:

```json
{
  "capabilities": {
    "tools": {
      "listChanged": true
    }
  }
}
```

`listChanged` indicates whether the server will emit `notifications/tools/list_changed` when the tool list changes.

### Discovery: `tools/list` (supports pagination via cursor)
### Invocation: `tools/call` (with tool name and arguments)
### Notification: `notifications/tools/list_changed` (when tool list changes)

### Security Best Practices

Servers MUST:
- Validate all tool inputs
- Implement proper access controls
- Rate limit tool invocations
- Sanitize tool outputs

Clients SHOULD:
- Prompt for user confirmation on sensitive operations
- Show tool inputs to user before calling server (to avoid data exfiltration)
- Validate tool results before passing to LLM
- Implement timeouts for tool calls
- Log tool usage for audit purposes

## Code/JSON Examples

### Tool Definition Example

```json
{
  "name": "get_weather",
  "title": "Weather Information Provider",
  "description": "Get current weather information for a location",
  "inputSchema": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "City name or zip code"
      }
    },
    "required": ["location"]
  },
  "annotations": {
    "title": "Get Weather",
    "readOnlyHint": true,
    "openWorldHint": true
  }
}
```

### tools/list Request and Response

```json
// Request
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {
    "cursor": "optional-cursor-value"
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "title": "Weather Information Provider",
        "description": "Get current weather information for a location",
        "inputSchema": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "City name or zip code"
            }
          },
          "required": ["location"]
        }
      }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

### tools/call Request and Response

```json
// Request
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "location": "New York"
    }
  }
}

// Response (success)
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Current weather in New York:\nTemperature: 72F\nConditions: Partly cloudy"
      }
    ],
    "isError": false
  }
}
```

### Protocol Error (JSON-RPC error)

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "error": {
    "code": -32602,
    "message": "Unknown tool: invalid_tool_name"
  }
}
```

### Tool Execution Error (isError: true)

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Failed to fetch weather data: API rate limit exceeded"
      }
    ],
    "isError": true
  }
}
```

### List Changed Notification

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

### Tool Result Content Types

#### Text Content
```json
{
  "type": "text",
  "text": "Tool result text"
}
```

#### Image Content
```json
{
  "type": "image",
  "data": "base64-encoded-data",
  "mimeType": "image/png",
  "annotations": {
    "audience": ["user"],
    "priority": 0.9
  }
}
```

#### Audio Content
```json
{
  "type": "audio",
  "data": "base64-encoded-audio-data",
  "mimeType": "audio/wav"
}
```

#### Resource Links
```json
{
  "type": "resource_link",
  "uri": "file:///project/src/main.rs",
  "name": "main.rs",
  "description": "Primary application entry point",
  "mimeType": "text/x-rust",
  "annotations": {
    "audience": ["assistant"],
    "priority": 0.9
  }
}
```

Note: Resource links returned by tools are NOT guaranteed to appear in `resources/list` results.

#### Embedded Resources
```json
{
  "type": "resource",
  "resource": {
    "uri": "file:///project/src/main.rs",
    "mimeType": "text/x-rust",
    "text": "fn main() {\n    println!(\"Hello world!\");\n}",
    "annotations": {
      "audience": ["user", "assistant"],
      "priority": 0.7,
      "lastModified": "2025-05-03T14:30:00Z"
    }
  }
}
```

### Structured Content with Output Schema

Tool definition:
```json
{
  "name": "get_weather_data",
  "title": "Weather Data Retriever",
  "description": "Get current weather data for a location",
  "inputSchema": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "City name or zip code"
      }
    },
    "required": ["location"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "temperature": {
        "type": "number",
        "description": "Temperature in celsius"
      },
      "conditions": {
        "type": "string",
        "description": "Weather conditions description"
      },
      "humidity": {
        "type": "number",
        "description": "Humidity percentage"
      }
    },
    "required": ["temperature", "conditions", "humidity"]
  }
}
```

Response with structured content:
```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"temperature\": 22.5, \"conditions\": \"Partly cloudy\", \"humidity\": 65}"
      }
    ],
    "structuredContent": {
      "temperature": 22.5,
      "conditions": "Partly cloudy",
      "humidity": 65
    }
  }
}
```

## Important for Exam (Mapping to Task Statements)

### Domain 2: Tool Design and MCP Integration

**Task 2.1 -- Tool Design**
- Tools are model-controlled (LLM selects which tool to use)
- Tool `name` must be unique identifier; `description` guides LLM selection
- `inputSchema` uses JSON Schema for parameter validation (required field)
- `outputSchema` enables structured content validation (optional field)
- Annotations provide behavioral metadata but are untrusted by default

**Task 2.2 -- Error Handling**
- Two distinct error types: protocol errors vs tool execution errors
- `isError: true` flag on tool results for execution failures
- Protocol errors use standard JSON-RPC error codes (-32602, -32603)
- Tool execution errors still return a `result` (not `error`) but with `isError: true`

**Task 2.3 -- Tool Descriptions**
- `description` field is critical for LLM tool selection
- Should clearly explain what the tool does and when to use it
- `title` provides human-readable display name
- `inputSchema` descriptions help LLM provide correct arguments

**Task 2.4 -- Resources vs Tools**
- Tools are MODEL-CONTROLLED (LLM invokes them)
- Resources are APPLICATION-DRIVEN (host app determines context inclusion)
- Tools perform actions; Resources provide data/context
- Tools use `tools/call`; Resources use `resources/read`

**Task 2.5 -- MCP Configuration**
- Servers declare capabilities during initialization
- `listChanged: true` enables dynamic tool list updates via notifications
- Human-in-the-loop SHOULD always be maintained

### Primitives Comparison (MEMORIZE THIS)

| Primitive | Controlled By | Analogy | Direction |
|-----------|--------------|---------|-----------|
| Tools | Model (LLM) | POST endpoint | Client -> Server |
| Resources | Application | GET endpoint | Client -> Server |
| Prompts | User | Slash command | Client -> Server |
