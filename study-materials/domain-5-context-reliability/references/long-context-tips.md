# Long Context Window Tips
Source: https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/long-context-tips
Fetched: 2026-03-21 (from training knowledge — WebFetch was denied)

## Key Concepts for Exam

### Context Window Sizes
- **Claude 3.5 Sonnet / Claude 3.5 Haiku**: 200K tokens
- **Claude 3 Opus**: 200K tokens
- **Claude 4 / Claude Sonnet 4**: 200K tokens (check latest docs for updates)
- 200K tokens ≈ ~150,000 words ≈ ~500 pages of text

### The "Lost in the Middle" Effect

**Critical exam concept**: When Claude processes very long contexts, information placed in the **middle** of the context is more likely to be overlooked than information at the **beginning** or **end**.

This is a well-documented phenomenon in large language models:
- Information at the **start** of the context: HIGH recall (primacy effect)
- Information in the **middle** of the context: LOWER recall
- Information at the **end** of the context: HIGH recall (recency effect)

### Position-Aware Ordering Strategies

**Place the most important information at the top or bottom of your prompt:**

```xml
<important_context>
<!-- MOST CRITICAL information goes here (beginning) -->
{{CRITICAL_DOCUMENTS}}
</important_context>

<supporting_context>
<!-- Less critical information can go in the middle -->
{{SUPPORTING_DOCUMENTS}}
</supporting_context>

<query>
<!-- The actual question/task goes at the END -->
{{USER_QUERY}}
</query>
```

**Best practices:**
1. Put the user's question/task AFTER the documents, not before
2. Place the most relevant documents closest to the query (at the end)
3. Use XML tags to clearly delineate document boundaries
4. Add document identifiers/labels so Claude can cite sources

### Document Placement Strategy

**Recommended ordering for long context:**
```
[System prompt with instructions]
[Less relevant documents]
[More relevant documents]
[Most relevant documents]
[User's actual question]
```

This leverages recency bias — documents closest to the question get more attention.

### Structured Document Tags

```xml
<documents>
  <document index="1">
    <source>annual_report_2024.pdf</source>
    <content>
    {{DOCUMENT_1_CONTENT}}
    </content>
  </document>
  <document index="2">
    <source>quarterly_earnings.pdf</source>
    <content>
    {{DOCUMENT_2_CONTENT}}
    </content>
  </document>
</documents>

Based on the documents above, answer the following question:
{{QUESTION}}
```

### Chunking and Retrieval Strategies

For contexts approaching or exceeding the window:
1. **Pre-filter**: Use embeddings/search to select only relevant chunks
2. **Summarize**: Compress less-relevant sections into summaries
3. **Hierarchical**: Provide summaries first, then full text of key sections
4. **RAG (Retrieval-Augmented Generation)**: Retrieve only the most relevant passages

### Needle-in-a-Haystack Performance

Claude demonstrates strong "needle in a haystack" retrieval across its full 200K context window, but performance considerations:
- Near-perfect recall for information at beginning and end
- Slightly reduced recall for information buried in the middle of very long contexts
- Adding explicit instructions like "search the entire document carefully" can improve recall
- Asking Claude to quote the relevant passage before answering improves accuracy

## API Reference / Configuration

### Token Counting
```python
# Count tokens before sending (useful for context management)
count = client.messages.count_tokens(
    model="claude-sonnet-4-20250514",
    messages=[{"role": "user", "content": long_text}]
)
print(f"Input tokens: {count.input_tokens}")
```

### Handling Long Contexts
```python
# Use system prompt to set expectations
message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=4096,
    system="""You will be given a collection of documents.
    Read ALL documents carefully before answering.
    Always cite the specific document source when referencing information.
    If the answer is not found in the documents, say so explicitly.""",
    messages=[
        {
            "role": "user",
            "content": f"""<documents>
{formatted_documents}
</documents>

Question: {user_question}

Search through ALL the provided documents carefully and answer the question.
Quote the relevant passage(s) that support your answer."""
        }
    ]
)
```

## Code Examples

### Python: Managing Long Context with Citation
```python
import anthropic

client = anthropic.Anthropic()

def query_documents(documents: list[dict], question: str) -> str:
    """Query across multiple documents with citation support."""

    # Format documents with indices for citation
    formatted_docs = ""
    for i, doc in enumerate(documents):
        formatted_docs += f"""<document index="{i+1}">
<source>{doc['source']}</source>
<content>
{doc['content']}
</content>
</document>
"""

    message = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        messages=[
            {
                "role": "user",
                "content": f"""<documents>
{formatted_docs}
</documents>

Based on the above documents, please answer: {question}

In your answer, cite the document source for each claim."""
            }
        ]
    )
    return message.content[0].text
```

### Python: Context Window Management
```python
def manage_context_window(messages, max_context_tokens=180000):
    """Trim older messages to fit within context window."""

    # Count tokens
    total_tokens = client.messages.count_tokens(
        model="claude-sonnet-4-20250514",
        messages=messages
    ).input_tokens

    # Trim from the beginning (oldest messages) if over limit
    while total_tokens > max_context_tokens and len(messages) > 1:
        messages = messages[1:]  # Remove oldest message
        total_tokens = client.messages.count_tokens(
            model="claude-sonnet-4-20250514",
            messages=messages
        ).input_tokens

    return messages
```

### TypeScript: Long Context Query
```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

interface Document {
    source: string;
    content: string;
}

async function queryDocuments(documents: Document[], question: string): Promise<string> {
    const formattedDocs = documents.map((doc, i) =>
        `<document index="${i + 1}">
<source>${doc.source}</source>
<content>${doc.content}</content>
</document>`
    ).join("\n");

    const message = await client.messages.create({
        model: "claude-sonnet-4-20250514",
        max_tokens: 4096,
        messages: [{
            role: "user",
            content: `<documents>\n${formattedDocs}\n</documents>\n\nBased on the above documents, answer: ${question}\nCite your sources.`
        }]
    });

    return message.content[0].type === "text" ? message.content[0].text : "";
}
```

## Important Details

- **Lost in the middle**: The single most important concept for exam. Information in the middle of long contexts has lower recall.
- **Put query AFTER documents**: Claude attends more strongly to content near the query. Always place the question/task after the reference material.
- **Explicit retrieval instructions**: Telling Claude to "carefully search all documents" and "quote the relevant passage" significantly improves recall accuracy.
- **XML document tags with indices**: Using `<document index="N">` tags with source labels enables Claude to cite specific documents.
- **Token counting**: Use `client.messages.count_tokens()` to check context usage before sending requests. Avoids hitting limits.
- **Context window != output window**: The 200K limit is for input. Output is limited by `max_tokens` parameter (up to model-specific limits, commonly 4096 or 8192 default).
- **Prompt caching**: For repeated queries over the same long context, use prompt caching to reduce costs and latency. Cache the document context, vary only the query.
- **Diminishing returns**: More context is not always better. Irrelevant context can actually hurt performance. Pre-filter when possible.
