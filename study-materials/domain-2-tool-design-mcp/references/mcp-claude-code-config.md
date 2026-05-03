# MCP in Claude Code (.mcp.json Configuration)
Source: https://docs.anthropic.com/en/docs/claude-code/mcp
Note: WebFetch was denied for this URL. Content below is based on known documentation.
Fetched: 2026-03-21

## Key Concepts for Exam

- Claude Code uses `.mcp.json` files for MCP server configuration
- Two scopes: project-level (`.mcp.json` in project root) and user-level (`~/.claude/.mcp.json`)
- Supports environment variable expansion in configuration
- Server configurations specify command, args, and env
- Supports both stdio and HTTP transports

## Configuration File Format

### .mcp.json Structure

The `.mcp.json` file contains an `mcpServers` object where each key is a server name and the value is the server configuration.

```json
{
  "mcpServers": {
    "server-name": {
      "command": "command-to-run",
      "args": ["arg1", "arg2"],
      "env": {
        "ENV_VAR": "value"
      }
    }
  }
}
```

### Configuration Fields

- `command`: The executable command to launch the MCP server
- `args`: Array of command-line arguments passed to the server
- `env`: Object of environment variables to set for the server process
- `type`: Transport type (default: "stdio"; can be "sse" for Server-Sent Events / HTTP)
- `url`: URL for HTTP/SSE transport servers

### File Locations

1. **Project-level**: `.mcp.json` in the project root directory
   - Shared with team via version control
   - Scoped to the specific project

2. **User-level**: `~/.claude/.mcp.json`
   - Personal configuration
   - Available across all projects

## Code Examples

### Basic stdio Server Configuration

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/directory"]
    }
  }
}
```

### Server with Environment Variables

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

### Environment Variable Expansion

The `${VAR_NAME}` syntax expands environment variables from the user's shell environment. This is important for keeping secrets out of configuration files that may be committed to version control.

```json
{
  "mcpServers": {
    "database": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    }
  }
}
```

### Multiple Servers

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"]
    },
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
    }
  }
}
```

### HTTP/SSE Transport Server

```json
{
  "mcpServers": {
    "remote-server": {
      "type": "sse",
      "url": "https://example.com/mcp/sse",
      "env": {
        "API_KEY": "${API_KEY}"
      }
    }
  }
}
```

### Python-based Server

```json
{
  "mcpServers": {
    "my-python-server": {
      "command": "python",
      "args": ["-m", "my_mcp_server"],
      "env": {
        "PYTHONPATH": "/path/to/server"
      }
    }
  }
}
```

## Managing MCP Servers via CLI

Claude Code also supports managing MCP servers via the command line:

```bash
# Add a server
claude mcp add <name> <command> [args...]

# Add with environment variables
claude mcp add <name> -e KEY=VALUE <command> [args...]

# List configured servers
claude mcp list

# Remove a server
claude mcp remove <name>

# Specify scope
claude mcp add --scope project <name> <command> [args...]
claude mcp add --scope user <name> <command> [args...]
```

## Specification Details

### How Claude Code Connects to MCP Servers

1. On startup, Claude Code reads `.mcp.json` files (both project and user level)
2. For each configured server, Claude Code spawns the server process (stdio) or connects to the URL (HTTP)
3. Claude Code performs the MCP initialization handshake (capability negotiation)
4. Discovered tools become available to Claude during the session
5. Tools from MCP servers appear alongside built-in tools

### Permission Model

- Project-level `.mcp.json` servers require user approval on first use
- User-level servers are trusted by default
- Individual tool invocations may still require confirmation based on Claude Code's permission settings

## Important for Exam (Task Statements 2.1-2.5)

### 2.1 - Configuration Format
- `.mcp.json` uses `mcpServers` as the top-level key
- Each server has `command`, `args`, `env` fields
- Environment variable expansion via `${VAR_NAME}` syntax

### 2.2 - Scoping
- Project-level (`.mcp.json` in project root) for team sharing
- User-level (`~/.claude/.mcp.json`) for personal config
- Project servers need user approval; user servers are trusted

### 2.3 - Transport Types
- Default is stdio (command + args)
- HTTP/SSE supported via `type` and `url` fields

### 2.4 - Security
- Use `${ENV_VAR}` expansion to keep secrets out of committed files
- Never hardcode tokens/keys in `.mcp.json`
- Project-level configs require approval before use

### 2.5 - CLI Management
- `claude mcp add/list/remove` commands for managing servers
- `--scope` flag for project vs user level
- `-e` flag for environment variables
