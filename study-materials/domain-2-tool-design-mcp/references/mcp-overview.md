# MCP Overview and Architecture
Source: https://modelcontextprotocol.io/docs/concepts/architecture
Also: https://modelcontextprotocol.io/introduction
Fetched: 2026-03-21

## Key Concepts for Exam

- MCP = Model Context Protocol, open-source standard for connecting AI apps to external systems
- Client-server architecture: Host -> Client(s) -> Server(s)
- Two layers: Data layer (JSON-RPC 2.0) and Transport layer (stdio or Streamable HTTP)
- Three server primitives: Tools, Resources, Prompts
- Client primitives: Sampling, Elicitation, Logging
- Capability negotiation during initialization handshake
- Stateful protocol with lifecycle management

## Specification Details

### What is MCP?

MCP (Model Context Protocol) is an open-source standard for connecting AI applications to external systems. It provides a standardized way to connect AI applications to data sources, tools, and workflows. Think of it like a USB-C port for AI applications.

### Participants

MCP follows a client-server architecture:

- **MCP Host**: The AI application that coordinates and manages one or multiple MCP clients (e.g., Claude Code, Claude Desktop, VS Code)
- **MCP Client**: A component that maintains a connection to an MCP server and obtains context from it for the host to use
- **MCP Server**: A program that provides context to MCP clients

Key relationship: One host creates one MCP client per MCP server connection.

Example: VS Code (host) connects to Sentry MCP server via one client, and to filesystem server via another client.

Local MCP servers (STDIO transport) typically serve a single client.
Remote MCP servers (Streamable HTTP transport) typically serve many clients.

### Two Layers

#### Data Layer (Inner Layer)
Implements JSON-RPC 2.0 based exchange protocol:
- **Lifecycle management**: Connection initialization, capability negotiation, termination
- **Server features**: Tools, Resources, Prompts
- **Client features**: Sampling, Elicitation, Logging
- **Utility features**: Notifications, progress tracking

#### Transport Layer (Outer Layer)
Manages communication channels:
- **Stdio transport**: Standard input/output for local process communication. No network overhead.
- **Streamable HTTP transport**: HTTP POST for client-to-server, optional SSE for streaming. Supports standard HTTP auth (bearer tokens, API keys, custom headers). OAuth recommended.

### Core Primitives

#### Server Primitives (what servers expose to clients)

1. **Tools**: Executable functions that AI apps can invoke (e.g., file operations, API calls, database queries)
   - Discovery: `tools/list`
   - Execution: `tools/call`

2. **Resources**: Data sources providing contextual information (e.g., file contents, database records, API responses)
   - Discovery: `resources/list`
   - Retrieval: `resources/read`

3. **Prompts**: Reusable templates for structuring LLM interactions (e.g., system prompts, few-shot examples)
   - Discovery: `prompts/list`
   - Retrieval: `prompts/get`

#### Client Primitives (what clients expose to servers)

1. **Sampling**: Allows servers to request LLM completions from the client's AI application (`sampling/complete`). Useful for server authors who want model access without bundling an LLM SDK.

2. **Elicitation**: Allows servers to request additional information from users (`elicitation/request`). For getting user input or confirmation.

3. **Logging**: Enables servers to send log messages to clients for debugging/monitoring.

#### Utility Primitives
- **Tasks (Experimental)**: Durable execution wrappers for deferred result retrieval and status tracking.

### Lifecycle Management

MCP is a stateful protocol. Initialization sequence:

1. Client sends `initialize` request with:
   - `protocolVersion` (e.g., "2025-06-18")
   - `capabilities` (what the client supports)
   - `clientInfo` (name, version)

2. Server responds with:
   - `protocolVersion`
   - `capabilities` (what the server supports)
   - `serverInfo` (name, version)

3. Client sends `notifications/initialized` notification

## Code Examples

### Initialization Handshake

**Initialize Request:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "elicitation": {}
    },
    "clientInfo": {
      "name": "example-client",
      "version": "1.0.0"
    }
  }
}
```

**Initialize Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "tools": {
        "listChanged": true
      },
      "resources": {}
    },
    "serverInfo": {
      "name": "example-server",
      "version": "1.0.0"
    }
  }
}
```

**Initialized Notification:**
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

### Tool Discovery
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list"
}
```

Response includes tools array with name, title, description, inputSchema for each tool.

### Tool Execution
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "weather_current",
    "arguments": {
      "location": "San Francisco",
      "units": "imperial"
    }
  }
}
```

### Notification Example
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

No `id` field = no response expected (JSON-RPC 2.0 notification semantics).

### Pseudo-code Patterns

**Initialization:**
```python
async with stdio_client(server_config) as (read, write):
    async with ClientSession(read, write) as session:
        init_response = await session.initialize()
        if init_response.capabilities.tools:
            app.register_mcp_server(session, supports_tools=True)
        app.set_server_ready(session)
```

**Tool Discovery:**
```python
available_tools = []
for session in app.mcp_server_sessions():
    tools_response = await session.list_tools()
    available_tools.extend(tools_response.tools)
conversation.register_available_tools(available_tools)
```

**Tool Execution:**
```python
async def handle_tool_call(conversation, tool_name, arguments):
    session = app.find_mcp_session_for_tool(tool_name)
    result = await session.call_tool(tool_name, arguments)
    conversation.add_tool_result(result.content)
```

**Notification Handling:**
```python
async def handle_tools_changed_notification(session):
    tools_response = await session.list_tools()
    app.update_available_tools(session, tools_response.tools)
    if app.conversation.is_active():
        app.conversation.notify_llm_of_new_capabilities()
```

## Important for Exam (Task Statements 2.1-2.5)

### 2.1 - Understanding MCP Architecture
- Host-Client-Server relationship (1 host, N clients, N servers)
- JSON-RPC 2.0 is the underlying protocol
- Stateful protocol with capability negotiation
- Two transports: stdio (local) and Streamable HTTP (remote)

### 2.2 - Protocol Flow
- Initialize -> capabilities exchange -> initialized notification
- Discovery (list) -> Selection -> Execution (call) -> Result
- Notifications are fire-and-forget (no id, no response)

### 2.3 - Primitives Distinction
- Tools = model-controlled, executable functions
- Resources = application-driven, contextual data
- Prompts = reusable interaction templates
- Each has list/get or list/call pattern

### 2.4 - Client Primitives
- Sampling lets servers use the host's LLM without bundling their own
- Elicitation lets servers ask users for input
- These are OPTIONAL capabilities declared during init

### 2.5 - Ecosystem
- MCP is an open standard with broad support
- Clients: Claude, ChatGPT, VS Code, Cursor, etc.
- Servers can be local (stdio) or remote (HTTP)
- MCP Inspector is a development tool for testing servers
