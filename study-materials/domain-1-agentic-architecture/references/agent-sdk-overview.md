# Claude Agent SDK Documentation
Source: https://docs.anthropic.com/en/docs/agents
Fetched: 2026-03-21 (from training knowledge — WebFetch was denied)

## Key Concepts for Exam

### What is the Claude Agent SDK?

The Claude Agent SDK (also referred to as the Anthropic Agent SDK) provides a framework for building agentic applications with Claude. It offers:
- A structured way to define agents with tools, system prompts, and guardrails
- Built-in agent loop management (the observe-think-act cycle)
- Tool registration and execution
- Multi-agent orchestration (handoffs between agents)
- Guardrails and safety controls

### Agent Architecture Components

1. **Agent**: The core unit — has a system prompt, tools, model config
2. **Tools**: Functions the agent can call (Python functions, API calls, etc.)
3. **Guardrails**: Input/output validators that run before/after each turn
4. **Handoffs**: Transfer control between specialized agents
5. **Runner**: Orchestrates the agent loop execution

### Building Blocks of Agents (from Anthropic's docs)

Anthropic describes agents as having three core components:
1. **Model** — The LLM (Claude) that drives reasoning
2. **Tools** — Interfaces to external systems and actions
3. **Orchestration** — The loop/framework managing the interaction

### Tool Types
- **Function tools**: Wrap Python/TypeScript functions
- **Computer use tools**: Control a computer GUI
- **Text editor tools**: Read/write files
- **Bash tools**: Execute shell commands
- **MCP tools**: Connect to Model Context Protocol servers
- **Custom tools**: Any user-defined tool

### Agent SDK Patterns

#### Single Agent
```python
from anthropic_agent import Agent, Runner

agent = Agent(
    name="Research Assistant",
    instructions="You are a helpful research assistant. Use your tools to find and analyze information.",
    tools=[search_tool, summarize_tool],
    model="claude-sonnet-4-20250514"
)

result = Runner.run_sync(agent, "Find recent papers on transformer architectures")
print(result.final_output)
```

#### Multi-Agent with Handoffs
```python
from anthropic_agent import Agent, Runner

# Specialized agents
triage_agent = Agent(
    name="Triage",
    instructions="Determine the type of request and hand off to the appropriate specialist.",
    handoffs=[code_agent, research_agent, writing_agent]
)

code_agent = Agent(
    name="Code Expert",
    instructions="You are a coding expert. Help with programming tasks.",
    tools=[run_code, read_file, write_file]
)

research_agent = Agent(
    name="Researcher",
    instructions="You are a research expert. Find and synthesize information.",
    tools=[web_search, summarize]
)

writing_agent = Agent(
    name="Writer",
    instructions="You are a writing expert. Help draft and edit content.",
    tools=[grammar_check]
)

# Run starting from triage
result = Runner.run_sync(triage_agent, "Write a Python function to sort a list")
# Triage hands off to code_agent automatically
```

#### Guardrails
```python
from anthropic_agent import Agent, GuardrailFunctionOutput, input_guardrail, output_guardrail

@input_guardrail
async def check_appropriate(ctx, agent, input):
    """Reject inappropriate inputs."""
    # Use a fast model to check input
    result = await classify_input(input)
    return GuardrailFunctionOutput(
        output_info=result,
        tripwire_triggered=result.is_inappropriate
    )

@output_guardrail
async def check_safe_output(ctx, agent, output):
    """Validate output safety."""
    result = await classify_output(output)
    return GuardrailFunctionOutput(
        output_info=result,
        tripwire_triggered=result.contains_pii
    )

agent = Agent(
    name="Safe Agent",
    instructions="You are a helpful assistant.",
    input_guardrails=[check_appropriate],
    output_guardrails=[check_safe_output]
)
```

### Model Context Protocol (MCP) Integration

Agents can connect to MCP servers for additional capabilities:

```python
from anthropic_agent import Agent
from anthropic_agent.mcp import MCPServerStdio

# Connect to an MCP server
mcp_server = MCPServerStdio(
    command="npx",
    args=["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
)

agent = Agent(
    name="File Agent",
    instructions="You can read and write files using MCP tools.",
    mcp_servers=[mcp_server]
)
```

### Agent Orchestration Patterns

#### 1. Sequential (Pipeline)
```
Agent A → Agent B → Agent C → Result
```
Each agent processes and passes to the next.

#### 2. Hierarchical (Orchestrator-Worker)
```
Orchestrator Agent
├── Worker Agent 1
├── Worker Agent 2
└── Worker Agent 3
```
Orchestrator delegates, workers report back.

#### 3. Handoff (Transfer)
```
Agent A ──handoff──→ Agent B ──handoff──→ Agent C
```
Control transfers between specialized agents based on context.

## API Reference / Configuration

### Agent Configuration
```python
Agent(
    name: str,                          # Agent name
    instructions: str | Callable,       # System prompt (can be dynamic)
    model: str = "claude-sonnet-4-20250514",  # Model to use
    tools: list[Tool] = [],             # Available tools
    handoffs: list[Agent] = [],         # Agents this agent can hand off to
    input_guardrails: list = [],        # Input validators
    output_guardrails: list = [],       # Output validators
    model_settings: ModelSettings = None  # Temperature, max_tokens, etc.
)
```

### ModelSettings
```python
ModelSettings(
    temperature: float = None,
    max_tokens: int = None,
    top_p: float = None,
    tool_choice: str | dict = None,     # "auto", "any", {"type": "tool", "name": "..."}
)
```

### Runner
```python
# Synchronous execution
result = Runner.run_sync(agent, input_text)

# Async execution
result = await Runner.run(agent, input_text)

# Streaming
async for event in Runner.run_streamed(agent, input_text):
    # Process streaming events
    pass
```

### RunResult
```python
result.final_output      # The final text response
result.last_agent        # Which agent produced the final output
result.new_items         # All items generated during the run
result.input_guardrail_results
result.output_guardrail_results
```

### Tool Definition
```python
from anthropic_agent import function_tool

@function_tool
def get_weather(location: str, unit: str = "celsius") -> str:
    """Get the current weather for a location.

    Args:
        location: City name or coordinates
        unit: Temperature unit (celsius or fahrenheit)
    """
    # Implementation
    return f"72°F in {location}"
```

The decorator automatically:
- Extracts the function signature as the tool's input_schema
- Uses the docstring as the tool description
- Handles type conversion and validation

## Code Examples

### Python: Complete Agent with Tools and Error Handling
```python
from anthropic_agent import Agent, Runner, function_tool, RunConfig

@function_tool
def read_file(path: str) -> str:
    """Read the contents of a file."""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        return f"Error: File '{path}' not found"

@function_tool
def list_directory(path: str = ".") -> str:
    """List files in a directory."""
    import os
    try:
        files = os.listdir(path)
        return "\n".join(files)
    except FileNotFoundError:
        return f"Error: Directory '{path}' not found"

agent = Agent(
    name="Code Explorer",
    instructions="""You are a code exploration agent.
    Always list directory contents before reading specific files.
    Explain what you find clearly.""",
    tools=[read_file, list_directory],
    model="claude-sonnet-4-20250514"
)

# Run with configuration
config = RunConfig(
    max_turns=25,           # Maximum conversation turns
    tracing_disabled=False  # Enable tracing for debugging
)

result = Runner.run_sync(agent, "Explore the src/ directory and summarize the code structure", run_config=config)
print(result.final_output)
```

### TypeScript: Agent with Handoffs
```typescript
import { Agent, Runner } from "@anthropic-ai/agent";

const billingAgent = new Agent({
    name: "Billing Specialist",
    instructions: "You handle billing inquiries. Be precise about amounts and dates.",
    tools: [lookupInvoice, processRefund]
});

const technicalAgent = new Agent({
    name: "Technical Support",
    instructions: "You handle technical issues. Walk through troubleshooting steps.",
    tools: [checkSystemStatus, runDiagnostics]
});

const triageAgent = new Agent({
    name: "Customer Service Triage",
    instructions: `You are the first point of contact.
    - For billing questions, hand off to Billing Specialist
    - For technical issues, hand off to Technical Support
    - For general questions, answer directly`,
    handoffs: [billingAgent, technicalAgent]
});

const result = await Runner.run(triageAgent, "I was charged twice for my subscription");
// triageAgent will hand off to billingAgent
console.log(result.finalOutput);
```

## Important Details

- **Agent loop termination**: The loop ends when Claude produces a response with `stop_reason: "end_turn"` (no tool calls). Or when max_turns is reached.
- **Handoffs are one-way**: When Agent A hands off to Agent B, Agent A is no longer in control. Agent B takes over completely.
- **Dynamic instructions**: The `instructions` parameter can be a callable that generates the system prompt based on context — useful for including dynamic state.
- **Guardrail tripwires**: When a guardrail's `tripwire_triggered` is True, the entire run is halted with an error. This is a hard stop for safety.
- **MCP integration**: MCP servers provide tools dynamically. The agent discovers available tools at runtime from the MCP server.
- **Tracing**: The SDK includes built-in tracing for debugging agent behavior. Each tool call, model call, and handoff is logged.
- **Max turns**: Always set `max_turns` in RunConfig to prevent runaway agents. A "turn" is one model call + tool execution cycle.
- **Token efficiency**: In long agent runs, context grows with each turn. Monitor token usage. Consider summarization strategies for long runs.
- **The SDK abstracts the loop**: You don't need to manually implement the while loop with tool result handling. The Runner does it for you.
- **function_tool decorator**: Automatically extracts JSON Schema from Python type hints and docstrings. Use type hints for best results.
- **Multiple tool calls per turn**: Claude can call multiple tools in a single turn. The SDK handles parallel tool execution automatically.
- **Error propagation**: If a tool raises an exception, the SDK catches it and sends the error as a tool_result with `is_error: true`, allowing Claude to recover.
