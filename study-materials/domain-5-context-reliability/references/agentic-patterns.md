# Agentic Patterns and Best Practices
Source: https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/agentic-coding
Also: https://docs.anthropic.com/en/docs/agents
Fetched: 2026-03-21 (from training knowledge — WebFetch was denied)

## Key Concepts for Exam

### Agentic System Patterns (from Anthropic's Research)

Anthropic identifies key patterns for building agentic systems with Claude:

#### 1. Augmented LLM (Foundation)
The base building block — an LLM enhanced with:
- **Retrieval** (RAG): Access to external knowledge
- **Tools**: Ability to take actions and get results
- **Memory**: Persistent state across interactions

#### 2. Prompt Chaining
Sequential pipeline where output of one LLM call feeds into the next:
```
Step 1: Generate → Step 2: Validate → Step 3: Refine
```
- Each step has a focused, specific prompt
- Gate/check between steps can include programmatic validation
- Trade-off: Higher latency, but better quality and debuggability

#### 3. Routing
Classify input first, then route to specialized handlers:
```
Input → Classifier → Route A (simple query)
                   → Route B (complex analysis)
                   → Route C (code generation)
```
- Use Claude to classify, then route to different prompts/models/tools
- Allows optimizing each route independently

#### 4. Parallelization
Run multiple LLM calls simultaneously:
- **Sectioning**: Break task into independent subtasks, run in parallel
- **Voting**: Run same prompt multiple times, aggregate results for reliability

#### 5. Orchestrator-Workers
A central orchestrator LLM delegates to worker LLMs:
```
Orchestrator → Worker 1 (research)
             → Worker 2 (analysis)
             → Worker 3 (writing)
             → Synthesize results
```

#### 6. Evaluator-Optimizer
Iterative loop where one LLM generates and another evaluates:
```
Generator → Output → Evaluator → Feedback → Generator (retry)
```
Continue until evaluator approves or max iterations reached.

#### 7. Autonomous Agent (Full Agent Loop)
The most complex pattern — LLM operates in a loop with tools:
```
while not done:
    observe → think → act → observe results
```
- Claude decides which tools to use and when to stop
- Requires careful guardrails and limits

### Agentic Coding Best Practices

#### System Prompt Design for Agents
```xml
<identity>
You are an expert software engineer assistant.
</identity>

<tools>
You have access to the following tools:
- read_file: Read contents of a file
- write_file: Write contents to a file
- run_command: Execute a shell command
- search_code: Search codebase for patterns
</tools>

<rules>
1. Always read a file before editing it
2. Run tests after making changes
3. Never modify files outside the project directory
4. Explain your reasoning before taking actions
5. If uncertain, ask for clarification rather than guessing
</rules>

<workflow>
1. Understand the task
2. Explore relevant code
3. Plan your approach
4. Implement changes
5. Test and verify
6. Report results
</workflow>
```

#### Tool Use in Agentic Loops

```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        "name": "read_file",
        "description": "Read the contents of a file at the given path",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "File path to read"}
            },
            "required": ["path"]
        }
    },
    {
        "name": "write_file",
        "description": "Write content to a file",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "File path to write"},
                "content": {"type": "string", "description": "Content to write"}
            },
            "required": ["path", "content"]
        }
    }
]

def execute_tool(name: str, input: dict) -> str:
    """Execute a tool and return the result."""
    if name == "read_file":
        with open(input["path"]) as f:
            return f.read()
    elif name == "write_file":
        with open(input["path"], "w") as f:
            f.write(input["content"])
        return "File written successfully"
    return "Unknown tool"

def agent_loop(task: str, max_iterations: int = 10):
    """Run an agentic loop with tool use."""
    messages = [{"role": "user", "content": task}]

    for i in range(max_iterations):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            system="You are a coding agent. Use tools to accomplish the task. When done, respond with your final summary.",
            tools=tools,
            messages=messages
        )

        # Check if Claude is done (no tool use)
        if response.stop_reason == "end_turn":
            final_text = next(
                (b.text for b in response.content if b.type == "text"), ""
            )
            return final_text

        # Process tool calls
        messages.append({"role": "assistant", "content": response.content})

        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result
                })

        messages.append({"role": "user", "content": tool_results})

    return "Max iterations reached"
```

#### Error Handling in Agentic Systems

```python
def execute_tool_safely(name: str, input: dict) -> dict:
    """Execute tool with error handling."""
    try:
        result = execute_tool(name, input)
        return {
            "type": "tool_result",
            "tool_use_id": "...",
            "content": result
        }
    except FileNotFoundError:
        return {
            "type": "tool_result",
            "tool_use_id": "...",
            "content": f"Error: File not found: {input.get('path', 'unknown')}",
            "is_error": True  # Signals to Claude that the tool call failed
        }
    except Exception as e:
        return {
            "type": "tool_result",
            "tool_use_id": "...",
            "content": f"Error: {str(e)}",
            "is_error": True
        }
```

**Key: `is_error: True`** — This field in tool_result tells Claude the tool call failed, so it can retry or adjust its approach.

### Guardrails for Agents

1. **Iteration limits**: Always set `max_iterations` to prevent infinite loops
2. **Token budgets**: Track cumulative token usage across the loop
3. **Tool restrictions**: Limit which tools are available based on context
4. **Confirmation gates**: Require human approval for destructive actions
5. **Sandboxing**: Run tool executions in isolated environments
6. **Output validation**: Validate tool inputs before execution

### Context Management in Agentic Loops

As the conversation grows in an agent loop:
- **Summarize periodically**: Compress earlier conversation turns into summaries
- **Sliding window**: Keep only the most recent N turns plus the original task
- **Tool result truncation**: Truncate long tool results to essential information
- **Separate scratchpad**: Use a separate context for intermediate work

```python
def compress_messages(messages, max_turns=20):
    """Compress conversation history to stay within context limits."""
    if len(messages) <= max_turns:
        return messages

    # Keep first message (original task) and recent turns
    original_task = messages[0]
    recent = messages[-max_turns:]

    # Summarize middle section
    summary_prompt = "Summarize the key actions and findings from this conversation so far."
    # ... create summary ...

    return [original_task, summary_message] + recent
```

## API Reference / Configuration

### Tool Result Schema
```json
{
    "type": "tool_result",
    "tool_use_id": "toolu_01ABC123",
    "content": "Result text or structured content",
    "is_error": false
}
```

The `content` field can be:
- A string
- An array of content blocks (text, image)

```json
{
    "type": "tool_result",
    "tool_use_id": "toolu_01ABC123",
    "content": [
        {"type": "text", "text": "File contents here..."},
        {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": "..."}}
    ]
}
```

### Multi-Tool Calls in Single Turn
Claude can request multiple tool calls in a single response. The response `content` array may contain multiple `tool_use` blocks. You must return results for ALL of them:

```python
# Response with multiple tool calls
# content: [tool_use_1, tool_use_2, tool_use_3]

# You must return ALL results in one message:
messages.append({
    "role": "user",
    "content": [
        {"type": "tool_result", "tool_use_id": "id_1", "content": "result 1"},
        {"type": "tool_result", "tool_use_id": "id_2", "content": "result 2"},
        {"type": "tool_result", "tool_use_id": "id_3", "content": "result 3"}
    ]
})
```

## Code Examples

### TypeScript: Agent Loop
```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

async function agentLoop(task: string, maxIterations = 10): Promise<string> {
    const messages: Anthropic.Messages.MessageParam[] = [
        { role: "user", content: task }
    ];

    for (let i = 0; i < maxIterations; i++) {
        const response = await client.messages.create({
            model: "claude-sonnet-4-20250514",
            max_tokens: 4096,
            tools: [/* tool definitions */],
            messages
        });

        if (response.stop_reason === "end_turn") {
            const textBlock = response.content.find(b => b.type === "text");
            return textBlock?.type === "text" ? textBlock.text : "Done";
        }

        messages.push({ role: "assistant", content: response.content });

        const toolResults = response.content
            .filter(b => b.type === "tool_use")
            .map(block => {
                if (block.type !== "tool_use") throw new Error("unreachable");
                const result = executeTool(block.name, block.input);
                return {
                    type: "tool_result" as const,
                    tool_use_id: block.id,
                    content: result
                };
            });

        messages.push({ role: "user", content: toolResults });
    }

    return "Max iterations reached";
}
```

## Important Details

- **Agent loop pattern**: observe → think → act → observe results. This is the core loop. stop_reason "end_turn" means the agent is done; "tool_use" means it wants to continue.
- **is_error in tool_result**: Setting `is_error: true` tells Claude the tool failed. Claude will typically retry with a different approach.
- **Multiple tool calls**: Claude can request multiple tools in one turn. You MUST return results for all of them.
- **Prompt chaining vs full agent**: For predictable workflows, prompt chaining is more reliable. Full agents are for open-ended tasks.
- **Cost control**: Agent loops can be expensive. Track token usage and set hard limits.
- **Anthropic's recommendation**: Use the simplest pattern that works. Start with prompt chaining, escalate to full agents only when needed.
- **Agentic coding specifically**: Claude Code and similar tools use the full agent loop with file system tools. Key practices include always reading before editing, running tests after changes, and explaining reasoning.
- **No multi-turn tool use in batches**: Reiteration — the Batches API does NOT support the agent loop. Each batch request is single-turn.
