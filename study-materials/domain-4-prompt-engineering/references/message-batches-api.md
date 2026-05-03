# Message Batches API Reference
Source: https://docs.anthropic.com/en/docs/build-with-claude/batch-processing
Fetched: 2026-03-21

## Key Concepts for Exam

### What is the Message Batches API?
- Asynchronous processing of large volumes of Messages API requests
- **50% cost reduction** compared to standard API prices
- Increased throughput
- Most batches finish in less than 1 hour
- Maximum processing window: 24 hours (batches expire if not completed)
- NOT eligible for Zero Data Retention (ZDR)

### When to Use
- Large-scale evaluations (thousands of test cases)
- Content moderation (large volumes of user-generated content)
- Data analysis (insights/summaries for large datasets)
- Bulk content generation (product descriptions, article summaries)
- Any task that does NOT require immediate responses

### Batch Limitations
- **Max requests**: 100,000 Message requests per batch
- **Max size**: 256 MB per batch (whichever limit is reached first)
- **Processing time**: Most complete within 1 hour; maximum 24 hours before expiration
- **Results availability**: 29 days after batch **creation** (not after processing ends)
- **Workspace scoped**: Batches belong to the Workspace of the API key that created them
- **Rate limits**: Apply to both HTTP requests AND number of requests within a batch waiting to be processed
- **Spend limits**: May go slightly over Workspace's configured spend limit due to high throughput
- **Cannot modify**: Once submitted, a batch cannot be modified (cancel and resubmit instead)
- **Streaming**: NOT supported for batch requests

### What Can Be Batched
Any Messages API request, including:
- Vision
- Tool use
- System messages
- Multi-turn conversations
- Any beta features
- Requests can be MIXED within a single batch (different types)

### Supported Models
All active models support the Message Batches API.

## API Reference

### Batch Request Structure
Each request in a batch has:
- `custom_id` (string): Unique identifier for the request (user-defined)
- `params` (object): Standard Messages API parameters (model, max_tokens, messages, etc.)

### Create a Batch
**Endpoint:** `POST /v1/messages/batches`

### Batch Response Object
```json
{
  "id": "msgbatch_01HkcTjaV5uDC8jWR4ZsDV8d",
  "type": "message_batch",
  "processing_status": "in_progress",
  "request_counts": {
    "processing": 2,
    "succeeded": 0,
    "errored": 0,
    "canceled": 0,
    "expired": 0
  },
  "ended_at": null,
  "created_at": "2024-09-24T18:37:24.100435Z",
  "expires_at": "2024-09-25T18:37:24.100435Z",
  "cancel_initiated_at": null,
  "results_url": null
}
```

### processing_status Values
- `in_progress` - Batch is being processed
- `canceling` - Cancellation has been initiated
- `ended` - All requests finished processing, results ready

### Result Types (4 types)
| Result Type | Description | Billed? |
|-------------|-------------|---------|
| `succeeded` | Request was successful, includes message result | Yes |
| `errored` | Request encountered an error (invalid request or server error) | No |
| `canceled` | User canceled batch before this request was processed | No |
| `expired` | Batch reached 24-hour expiration before this request was processed | No |

### Results Format
Results are in `.jsonl` format (one JSON object per line):
```jsonl
{"custom_id":"my-second-request","result":{"type":"succeeded","message":{"id":"msg_014VwiXbi91y3JMjcpyGBHX5","type":"message","role":"assistant","model":"claude-opus-4-6","content":[{"type":"text","text":"Hello again!..."}],"stop_reason":"end_turn","stop_sequence":null,"usage":{"input_tokens":11,"output_tokens":36}}}}
{"custom_id":"my-first-request","result":{"type":"succeeded","message":{"id":"msg_01FqfsLoHwgeFbguDgpz48m7","type":"message","role":"assistant","model":"claude-opus-4-6","content":[{"type":"text","text":"Hello!..."}],"stop_reason":"end_turn","stop_sequence":null,"usage":{"input_tokens":10,"output_tokens":34}}}}
```

**IMPORTANT**: Batch results may NOT match input order. Always use `custom_id` to match results with requests.

### Error Handling in Results
- For `errored` results: check `result.error` for standard error shape
- `invalid_request` errors: Request body must be fixed before re-sending
- Other errors: Request can be retried directly

### Cancellation
- Cancel endpoint: `POST /v1/messages/batches/{id}/cancel`
- Status goes to `canceling` immediately, then `ended` when finalized
- May contain partial results for requests processed before cancellation

### Canceling Response
```json
{
  "id": "msgbatch_013Zva2CMHLNnXjNJJKqJ2EF",
  "type": "message_batch",
  "processing_status": "canceling",
  "request_counts": {
    "processing": 2,
    "succeeded": 0,
    "errored": 0,
    "canceled": 0,
    "expired": 0
  },
  "cancel_initiated_at": "2024-09-24T18:39:03.114875Z",
  "results_url": null
}
```

## Pricing

**All usage charged at 50% of standard API prices.**

| Model | Batch Input | Batch Output |
|-------|-------------|--------------|
| Claude Opus 4.6 | $2.50 / MTok | $12.50 / MTok |
| Claude Opus 4.5 | $2.50 / MTok | $12.50 / MTok |
| Claude Opus 4.1 | $7.50 / MTok | $37.50 / MTok |
| Claude Opus 4 | $7.50 / MTok | $37.50 / MTok |
| Claude Sonnet 4.6 | $1.50 / MTok | $7.50 / MTok |
| Claude Sonnet 4.5 | $1.50 / MTok | $7.50 / MTok |
| Claude Sonnet 4 | $1.50 / MTok | $7.50 / MTok |
| Claude Haiku 4.5 | $0.50 / MTok | $2.50 / MTok |
| Claude Haiku 3.5 | $0.40 / MTok | $2 / MTok |
| Claude Haiku 3 | $0.125 / MTok | $0.625 / MTok |

## Code Examples

### Python - Create a Batch
```python
import anthropic
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

client = anthropic.Anthropic()

message_batch = client.messages.batches.create(
    requests=[
        Request(
            custom_id="my-first-request",
            params=MessageCreateParamsNonStreaming(
                model="claude-opus-4-6",
                max_tokens=1024,
                messages=[{"role": "user", "content": "Hello, world"}],
            ),
        ),
        Request(
            custom_id="my-second-request",
            params=MessageCreateParamsNonStreaming(
                model="claude-opus-4-6",
                max_tokens=1024,
                messages=[{"role": "user", "content": "Hi again, friend"}],
            ),
        ),
    ]
)
```

### TypeScript - Create a Batch
```typescript
import Anthropic from "@anthropic-ai/sdk";

const anthropic = new Anthropic();

const messageBatch = await anthropic.messages.batches.create({
  requests: [
    {
      custom_id: "my-first-request",
      params: {
        model: "claude-opus-4-6",
        max_tokens: 1024,
        messages: [{ role: "user", content: "Hello, world" }]
      }
    },
    {
      custom_id: "my-second-request",
      params: {
        model: "claude-opus-4-6",
        max_tokens: 1024,
        messages: [{ role: "user", content: "Hi again, friend" }]
      }
    }
  ]
});
```

### Python - Poll for Completion
```python
import anthropic
import time

client = anthropic.Anthropic()
MESSAGE_BATCH_ID = "msgbatch_01HkcTjaV5uDC8jWR4ZsDV8d"

message_batch = None
while True:
    message_batch = client.messages.batches.retrieve(MESSAGE_BATCH_ID)
    if message_batch.processing_status == "ended":
        break
    print(f"Batch {MESSAGE_BATCH_ID} is still processing...")
    time.sleep(60)
print(message_batch)
```

### TypeScript - Poll for Completion
```typescript
import Anthropic from "@anthropic-ai/sdk";

const anthropic = new Anthropic();
const messageBatchId = "msgbatch_01HkcTjaV5uDC8jWR4ZsDV8d";

let messageBatch;
while (true) {
  messageBatch = await anthropic.messages.batches.retrieve(messageBatchId);
  if (messageBatch.processing_status === "ended") {
    break;
  }
  console.log(`Batch ${messageBatchId} is still processing... waiting`);
  await new Promise((resolve) => setTimeout(resolve, 60_000));
}
console.log(messageBatch);
```

### Python - Stream Results (Memory-Efficient)
```python
import anthropic

client = anthropic.Anthropic()

# Stream results file in memory-efficient chunks, processing one at a time
for result in client.messages.batches.results("msgbatch_01HkcTjaV5uDC8jWR4ZsDV8d"):
    match result.result.type:
        case "succeeded":
            print(f"Success! {result.custom_id}")
        case "errored":
            if result.result.error.type == "invalid_request":
                # Request body must be fixed before re-sending request
                print(f"Validation error {result.custom_id}")
            else:
                # Request can be retried directly
                print(f"Server error {result.custom_id}")
        case "expired":
            print(f"Request expired {result.custom_id}")
```

### TypeScript - Stream Results
```typescript
import Anthropic from "@anthropic-ai/sdk";

const anthropic = new Anthropic();

for await (const result of await anthropic.messages.batches.results(
  "msgbatch_01HkcTjaV5uDC8jWR4ZsDV8d"
)) {
  switch (result.result.type) {
    case "succeeded":
      console.log(`Success! ${result.custom_id}`);
      break;
    case "errored":
      if (result.result.error.type === "invalid_request_error") {
        console.log(`Validation error: ${result.custom_id}`);
      } else {
        console.log(`Server error: ${result.custom_id}`);
      }
      break;
    case "expired":
      console.log(`Request expired: ${result.custom_id}`);
      break;
  }
}
```

### Python - List Batches (with auto-pagination)
```python
import anthropic

client = anthropic.Anthropic()

# Automatically fetches more pages as needed.
for message_batch in client.messages.batches.list(limit=20):
    print(message_batch)
```

### Python - Cancel a Batch
```python
import anthropic

client = anthropic.Anthropic()

message_batch = client.messages.batches.cancel("msgbatch_01HkcTjaV5uDC8jWR4ZsDV8d")
print(message_batch)
```

### Prompt Caching with Batches
```python
import anthropic
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

client = anthropic.Anthropic()

message_batch = client.messages.batches.create(
    requests=[
        Request(
            custom_id="my-first-request",
            params=MessageCreateParamsNonStreaming(
                model="claude-opus-4-6",
                max_tokens=1024,
                system=[
                    {
                        "type": "text",
                        "text": "You are an AI assistant tasked with analyzing literary works.",
                    },
                    {
                        "type": "text",
                        "text": "<the entire contents of Pride and Prejudice>",
                        "cache_control": {"type": "ephemeral"},
                    },
                ],
                messages=[
                    {"role": "user", "content": "Analyze the major themes in Pride and Prejudice."}
                ],
            ),
        ),
        Request(
            custom_id="my-second-request",
            params=MessageCreateParamsNonStreaming(
                model="claude-opus-4-6",
                max_tokens=1024,
                system=[
                    {
                        "type": "text",
                        "text": "You are an AI assistant tasked with analyzing literary works.",
                    },
                    {
                        "type": "text",
                        "text": "<the entire contents of Pride and Prejudice>",
                        "cache_control": {"type": "ephemeral"},
                    },
                ],
                messages=[
                    {"role": "user", "content": "Write a summary of Pride and Prejudice."}
                ],
            ),
        ),
    ]
)
```

## Important Details for Exam

### Validation is Asynchronous
- `params` validation for each request happens asynchronously
- Validation errors returned when ENTIRE batch processing has ended
- Best practice: Test request shape with Messages API first before batching

### Prompt Caching with Batches
- Prompt caching IS supported with batches
- **Pricing discounts STACK** (50% batch + caching discounts)
- Cache hits are **best-effort** (30% to 98% hit rates depending on traffic patterns)
- To maximize cache hits:
  1. Include identical `cache_control` blocks in every request
  2. Maintain steady stream of requests (cache entries expire after 5 minutes)
  3. Structure requests to share as much cached content as possible
  4. Consider using 1-hour cache duration for better hit rates with batches

### Privacy and Data Separation
- Batches isolated within Workspace they were created in
- Only accessible by API keys from that same Workspace
- Each request processed independently (no data leakage between requests)
- Results available for 29 days after creation
- Console download can be disabled at organization or workspace level

### Best Practices
1. Monitor batch processing status regularly
2. Implement retry logic for failed requests
3. Use meaningful `custom_id` values (order is NOT guaranteed)
4. Break very large datasets into multiple batches
5. Dry run a single request with Messages API first

### Troubleshooting
- 413 `request_too_large` error: Total batch exceeds 256 MB
- Check supported models for all requests
- Each request must have UNIQUE `custom_id`
- Results available for 29 days after `created_at` (NOT `ended_at`)
- Failure of one request does NOT affect other requests in the batch

### Rate Limits
- HTTP requests-based rate limits apply
- Limits on number of requests within batch waiting to be processed
- Batches API usage does NOT affect Messages API rate limits (separate)

### FAQ Key Points
- Batches cannot be modified after submission
- Streaming is NOT supported for batch requests
- All Messages API features work in batches (including beta features)
- 50% discount applies to input tokens, output tokens, AND special tokens
- Batches may take up to 24 hours; it IS possible for a batch to expire and not complete
- Each request within a batch is processed independently with no data leakage
