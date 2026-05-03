# Prompt Engineering Guide
Source: https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview
Fetched: 2026-03-21 (from training knowledge — WebFetch was denied)

## Key Concepts for Exam

### Core Prompt Engineering Techniques

1. **Be clear and direct** — Give Claude clear instructions. Avoid ambiguity. State what you want explicitly.

2. **Use examples (few-shot prompting)** — Provide examples of desired input/output pairs to guide Claude's responses.

3. **Give Claude a role (system prompts)** — Use system prompts to set context, persona, or behavioral guidelines.

4. **Use XML tags** — Structure prompts with XML tags like `<instructions>`, `<example>`, `<document>` for clarity.

5. **Chain of thought (CoT)** — Ask Claude to think step by step before answering. Use `<thinking>` tags.

6. **Prefill Claude's response** — Start the assistant turn with specific text to guide output format/direction.

7. **Chain prompts** — Break complex tasks into sequential sub-tasks.

8. **Extended thinking** — Use the `thinking` parameter for complex reasoning (budget tokens for internal reasoning).

### Few-Shot Prompting Patterns

Few-shot prompting provides examples of desired behavior. Key patterns:

```xml
<examples>
  <example>
    <input>What is the capital of France?</input>
    <output>The capital of France is Paris.</output>
  </example>
  <example>
    <input>What is the capital of Japan?</input>
    <output>The capital of Japan is Tokyo.</output>
  </example>
</examples>
```

**Best practices for few-shot examples:**
- Include 3-5 diverse examples covering edge cases
- Make examples representative of real use cases
- Use consistent formatting across examples
- Include both typical and boundary cases
- Place examples before the actual task

### System Prompts

```python
import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system="You are a senior software engineer. Respond with concise, technical answers.",
    messages=[
        {"role": "user", "content": "Explain dependency injection."}
    ]
)
```

### XML Tag Usage

XML tags help Claude parse complex prompts:

```xml
<instructions>
Analyze the following document and extract key findings.
</instructions>

<document>
{{DOCUMENT_CONTENT}}
</document>

<output_format>
Return a JSON object with keys: "findings", "confidence", "summary"
</output_format>
```

### Prefilling Responses

Start the assistant message to constrain output format:

```python
message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Extract the name and age from: John is 30 years old."},
        {"role": "assistant", "content": "{"}  # Forces JSON output
    ]
)
```

### Chain of Thought Prompting

```xml
<instructions>
Think step by step before answering. Show your reasoning in <thinking> tags,
then provide your final answer.
</instructions>
```

**Extended Thinking (API parameter):**
```python
message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000  # tokens allocated for internal reasoning
    },
    messages=[{"role": "user", "content": "Solve this complex problem..."}]
)
```

### Tool Use and tool_choice

**tool_choice options:**
- `{"type": "auto"}` — Claude decides whether to use tools (default)
- `{"type": "any"}` — Claude MUST use one of the provided tools
- `{"type": "tool", "name": "tool_name"}` — Claude MUST use the specific named tool
- `{"type": "none"}` — Claude will not use any tools (added later)

```python
message = client.messages.create(
    model="claude-sonnet-4-20250514",
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
                        "description": "The city and state, e.g. San Francisco, CA"
                    }
                },
                "required": ["location"]
            }
        }
    ],
    tool_choice={"type": "auto"},
    messages=[{"role": "user", "content": "What's the weather in SF?"}]
)
```

**Force tool use for structured output:**
```python
tool_choice={"type": "tool", "name": "extract_info"}
```

## API Reference / Configuration

### Messages API Parameters
- `model` (required): Model identifier string
- `max_tokens` (required): Maximum tokens in response
- `messages` (required): Array of message objects with `role` and `content`
- `system` (optional): System prompt string or array of content blocks
- `temperature` (optional): 0.0 to 1.0, controls randomness
- `top_p` (optional): Nucleus sampling parameter
- `top_k` (optional): Top-k sampling parameter
- `tools` (optional): Array of tool definitions
- `tool_choice` (optional): How Claude should use tools
- `stop_sequences` (optional): Custom stop sequences
- `stream` (optional): Boolean for streaming responses
- `metadata` (optional): Object with `user_id` for abuse tracking
- `thinking` (optional): Extended thinking configuration

### Response Format
```json
{
  "id": "msg_...",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Response text here"
    }
  ],
  "model": "claude-sonnet-4-20250514",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 25,
    "output_tokens": 150
  }
}
```

### Stop Reasons
- `"end_turn"` — Natural end of response
- `"max_tokens"` — Hit max_tokens limit
- `"stop_sequence"` — Hit a custom stop sequence
- `"tool_use"` — Claude wants to call a tool

## Code Examples

### Python SDK
```python
import anthropic

client = anthropic.Anthropic()  # Uses ANTHROPIC_API_KEY env var

# Basic message
message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Hello, Claude"}
    ]
)
print(message.content[0].text)
```

### TypeScript SDK
```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic(); // Uses ANTHROPIC_API_KEY env var

const message = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    messages: [
        { role: "user", content: "Hello, Claude" }
    ]
});
console.log(message.content[0].text);
```

### Streaming
```python
with client.messages.stream(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Tell me a story"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

### Multi-turn Conversation
```python
messages = [
    {"role": "user", "content": "What is machine learning?"},
    {"role": "assistant", "content": "Machine learning is..."},
    {"role": "user", "content": "Can you give an example?"}
]

message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=messages
)
```

## Important Details

- **Temperature**: Lower (0.0) = more deterministic; higher (1.0) = more creative. Use 0 for factual/analytical tasks.
- **Max tokens**: Must be set explicitly. Claude will stop at this limit even mid-sentence.
- **System prompt**: Not part of the messages array — it's a separate parameter.
- **Prefilling**: The assistant prefill content is NOT counted in the output but IS counted in input tokens.
- **XML tags**: Claude is specifically trained to understand XML-structured prompts. Prefer XML over other delimiters.
- **Order matters**: Put the most important instructions at the beginning and end of prompts (primacy and recency effects).
- **Chaining prompts**: Better accuracy for complex tasks than trying to do everything in one prompt.
- **Extended thinking**: Cannot be used with `temperature` > 1 or with prefilled assistant messages. The `budget_tokens` must be less than `max_tokens`.
- **tool_choice "any"**: Forces tool use but Claude picks which tool. Useful when you definitely want structured output.
- **tool_choice with specific name**: Forces a specific tool call — great for guaranteed structured extraction.
