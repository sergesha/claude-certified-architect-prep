# MCP Resources Specification
Source: https://modelcontextprotocol.io/docs/concepts/resources
Fetched: 2026-03-21

## Key Concepts for Exam

- Resources are **application-driven** (NOT model-controlled like tools)
- Resources provide contextual data to LLMs (files, database schemas, application info)
- Each resource is uniquely identified by a URI (RFC 3986)
- Resources support subscriptions for real-time change notifications
- Resources can contain text or binary data
- Resource templates use URI templates (RFC 6570) for parameterized resources
- Annotations provide audience, priority, and lastModified metadata
- **KEY DISTINCTION**: Resources = content catalogs for context. Tools = actions/operations.

### Resource Definition Fields

| Field | Required | Description |
|-------|----------|-------------|
| `uri` | Yes | Unique identifier for the resource (RFC 3986) |
| `name` | Yes | The name of the resource |
| `title` | No | Human-readable display name |
| `description` | No | Optional description |
| `mimeType` | No | Optional MIME type |
| `size` | No | Optional size in bytes |

### Resource Content Types

- **Text content**: UTF-8 encoded text (source code, configs, logs, JSON, XML, plain text)
- **Binary content**: Base64-encoded binary data (images, PDFs, etc.) via `blob` field

### Common URI Schemes

- `https://` -- Resources available on the web (use only when client can fetch directly)
- `file://` -- Filesystem-like resources (don't need to map to actual filesystem)
- `git://` -- Git version control integration
- Custom URI schemes -- MUST follow RFC 3986

### Annotations

Resources, resource templates, and content blocks support optional annotations:
- `audience`: Array of intended audiences -- valid values: `"user"` and `"assistant"`
- `priority`: Number from 0.0 to 1.0 (1 = most important/required, 0 = least important/optional)
- `lastModified`: ISO 8601 timestamp (e.g., `"2025-01-12T15:00:58Z"`)

Clients can use annotations to:
- Filter resources based on intended audience
- Prioritize which resources to include in context
- Display modification times or sort by recency

## Protocol/Configuration Details

### Capabilities Declaration

Servers that support resources MUST declare the `resources` capability:

```json
{
  "capabilities": {
    "resources": {
      "subscribe": true,
      "listChanged": true
    }
  }
}
```

Two optional features:
- `subscribe`: client can subscribe to individual resource change notifications
- `listChanged`: server emits notifications when the list of available resources changes

Both are optional -- servers can support neither, either, or both:
```json
{ "capabilities": { "resources": {} } }
{ "capabilities": { "resources": { "subscribe": true } } }
{ "capabilities": { "resources": { "listChanged": true } } }
```

### Protocol Messages

- `resources/list` -- discover available resources (supports pagination via cursor)
- `resources/read` -- read a specific resource by URI
- `resources/templates/list` -- discover resource templates (supports pagination)
- `resources/subscribe` -- subscribe to changes for a specific resource
- `resources/unsubscribe` -- unsubscribe from changes
- `notifications/resources/list_changed` -- the list of available resources changed
- `notifications/resources/updated` -- a specific subscribed resource was updated

### Security Considerations

1. Servers MUST validate all resource URIs
2. Access controls SHOULD be implemented for sensitive resources
3. Binary data MUST be properly encoded
4. Resource permissions SHOULD be checked before operations

### Error Handling

Standard JSON-RPC errors:
- Resource not found: `-32002`
- Internal errors: `-32603`

## Code/JSON Examples

### Resource Definition

```json
{
  "uri": "file:///project/src/main.rs",
  "name": "main.rs",
  "title": "Rust Software Application Main File",
  "description": "Primary application entry point",
  "mimeType": "text/x-rust"
}
```

### resources/list Request and Response

```json
// Request
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "resources/list",
  "params": {
    "cursor": "optional-cursor-value"
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resources": [
      {
        "uri": "file:///project/src/main.rs",
        "name": "main.rs",
        "title": "Rust Software Application Main File",
        "description": "Primary application entry point",
        "mimeType": "text/x-rust"
      }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

### resources/read Request and Response

```json
// Request
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "resources/read",
  "params": {
    "uri": "file:///project/src/main.rs"
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "contents": [
      {
        "uri": "file:///project/src/main.rs",
        "mimeType": "text/x-rust",
        "text": "fn main() {\n    println!(\"Hello world!\");\n}"
      }
    ]
  }
}
```

### Resource Templates

```json
// Request
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "resources/templates/list",
  "params": {
    "cursor": "optional-cursor-value"
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "resourceTemplates": [
      {
        "uriTemplate": "file:///{path}",
        "name": "Project Files",
        "title": "Project Files",
        "description": "Access files in the project directory",
        "mimeType": "application/octet-stream"
      }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

### Resource with Annotations

```json
{
  "uri": "file:///project/README.md",
  "name": "README.md",
  "title": "Project Documentation",
  "mimeType": "text/markdown",
  "annotations": {
    "audience": ["user"],
    "priority": 0.8,
    "lastModified": "2025-01-12T15:00:58Z"
  }
}
```

### Text Content

```json
{
  "uri": "file:///example.txt",
  "mimeType": "text/plain",
  "text": "Resource content"
}
```

### Binary Content

```json
{
  "uri": "file:///example.png",
  "mimeType": "image/png",
  "blob": "base64-encoded-data"
}
```

### Subscriptions

```json
// Subscribe Request
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "resources/subscribe",
  "params": {
    "uri": "file:///project/src/main.rs"
  }
}

// Update Notification
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": {
    "uri": "file:///project/src/main.rs"
  }
}
```

### List Changed Notification

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/list_changed"
}
```

### Error Response

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "error": {
    "code": -32002,
    "message": "Resource not found",
    "data": {
      "uri": "file:///nonexistent.txt"
    }
  }
}
```

## Important for Exam (Mapping to Task Statements)

### Resources vs Tools (CRITICAL DISTINCTION -- MEMORIZE THIS)

| Aspect | Resources | Tools |
|--------|-----------|-------|
| Control model | Application-driven | Model-controlled |
| Purpose | Provide context/data | Perform actions |
| Discovery | `resources/list` | `tools/list` |
| Usage | `resources/read` | `tools/call` |
| Identification | URI (RFC 3986) | Name string |
| Subscriptions | Yes (optional) | No |
| Change notifications | `notifications/resources/updated` | N/A |
| List changes | `notifications/resources/list_changed` | `notifications/tools/list_changed` |
| Content | Text or binary (blob) | Content array (text, image, audio, resource links) |
| Error flag | No isError flag | Has `isError` flag |
| Templates | URI templates (RFC 6570) | N/A |

### Key Points

- Resources are for DATA/CONTEXT; Tools are for ACTIONS
- Resources use URIs as identifiers; Tools use names
- Resources support subscriptions to individual items; Tools do not
- Resource templates enable parameterized access via URI templates (RFC 6570)
- Both support pagination via cursors
- Both support `listChanged` notifications
- Resources = GET endpoints; Tools = POST endpoints
- The host application decides which resources to expose to the LLM (application-driven)
- The LLM decides which tools to call (model-controlled)
