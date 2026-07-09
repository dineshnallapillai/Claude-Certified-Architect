# Domain 1 | Topic 1.7: API Fundamentals — Streaming & Models

## 1. Core Concepts

### API Headers & Authentication

Every request to the Claude Messages API requires specific headers:

| Header | Value | Purpose |
|--------|-------|---------|
| `x-api-key` | Your API key | Authentication |
| `anthropic-version` | `2023-06-01` | API version (required) |
| `content-type` | `application/json` | Request body format |
| `anthropic-beta` | Comma-separated flags | Opt-in to beta features |

**Beta features header examples:**
```
anthropic-beta: interleaved-thinking-2025-05-14
anthropic-beta: mcp-client-2025-11-20,files-api-2025-04-14
```

### SDK Initialization Patterns

**Python:**
```python
import anthropic

# Default — reads ANTHROPIC_API_KEY from environment
client = anthropic.Anthropic()

# Explicit key
client = anthropic.Anthropic(api_key="sk-ant-...")

# Async client
async_client = anthropic.AsyncAnthropic()
```

**TypeScript:**
```typescript
import Anthropic from "@anthropic-ai/sdk";

// Default — reads ANTHROPIC_API_KEY from environment
const client = new Anthropic();

// Explicit key
const client = new Anthropic({ apiKey: "sk-ant-..." });
```

### The Messages API Endpoint

All interactions use `POST /v1/messages` with this structure:

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    system="You are a helpful assistant.",
    messages=[{"role": "user", "content": "Hello"}]
)
```

---

## 2. Design Patterns

### Streaming Implementation

Two approaches to streaming:

**Approach 1: Raw SSE (Server-Sent Events)**

Set `"stream": true` in the request body. The response is a stream of SSE events:

```bash
curl https://api.anthropic.com/v1/messages \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 1024,
    "stream": true,
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

**Approach 2: SDK Streaming Helpers**

Python:
```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a story"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
    
    # Get final accumulated message
    final_message = stream.get_final_message()
```

TypeScript:
```typescript
const stream = client.messages.stream({
    model: "claude-sonnet-4-6",
    max_tokens: 1024,
    messages: [{ role: "user", content: "Write a story" }],
});

for await (const event of stream) {
    if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
        process.stdout.write(event.delta.text);
    }
}

const finalMessage = await stream.finalMessage();
```

### SSE Event Types (Complete Sequence)

| Event | Description | Key Fields |
|-------|-------------|------------|
| `message_start` | Opening event with message metadata | `message.id`, `message.model`, `message.usage` |
| `content_block_start` | New content block begins | `index`, `content_block.type` (text/tool_use/thinking) |
| `content_block_delta` | Incremental content | `delta.type` (text_delta/input_json_delta/thinking_delta) |
| `content_block_stop` | Content block complete | `index` |
| `message_delta` | Message-level updates | `delta.stop_reason`, `delta.stop_sequence`, `usage.output_tokens` |
| `message_stop` | Final event — message complete | — |
| `ping` | Keep-alive heartbeat | — |

**Full SSE event sequence example:**

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_01X...","type":"message","role":"assistant","model":"claude-sonnet-4-6","usage":{"input_tokens":25,"output_tokens":1}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: ping
data: {"type":"ping"}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"!"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn","stop_sequence":null},"usage":{"output_tokens":12}}

event: message_stop
data: {"type":"message_stop"}
```

### Tool Use in Streams

When Claude calls a tool during streaming, the `content_block_start` has `type: "tool_use"` and the input accumulates via `input_json_delta` events:

```
event: content_block_start
data: {"type":"content_block_start","index":1,"content_block":{"type":"tool_use","id":"toolu_01...","name":"get_weather","input":{}}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"{\"location\":"}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":" \"Paris\"}"}}

event: content_block_stop
data: {"type":"content_block_stop","index":1}
```

**Key point:** JSON input accumulates across multiple `input_json_delta` events. Concatenate all partial strings, then parse the final JSON.

### Error Events Mid-Stream

If the API becomes overloaded during streaming, an `overloaded_error` can occur:
- The stream may terminate early
- `message_delta` with `stop_reason` may not be sent
- Client should handle incomplete streams gracefully

---

## 3. Implementation Details

### Extended Thinking

Extended thinking allows Claude to reason through complex problems before responding.

**Enabling (older models — Sonnet 4.5, Opus 4.5):**
```python
response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000
    },
    messages=[{"role": "user", "content": "Solve this complex problem..."}]
)
```

**Budget constraints:**
- Minimum `budget_tokens`: 1024
- `budget_tokens` must be strictly less than `max_tokens`
- Thinking tokens are billed as output tokens

**Enabling (newer models — Opus 4.6+, Sonnet 4.6):**
```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16000,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},  # low | medium | high | xhigh | max
    messages=[{"role": "user", "content": "Solve this complex problem..."}]
)
```

**Key differences between approaches:**

| Aspect | `budget_tokens` (older) | Adaptive (newer) |
|--------|-------------------------|-------------------|
| Configuration | Fixed token budget | Claude decides dynamically |
| Supported models | Sonnet 4.5, Opus 4.5 and earlier | Opus 4.6+, Sonnet 4.6 |
| Control mechanism | Hard token ceiling | `effort` parameter |
| Status | Deprecated on 4.6 models | Recommended approach |

**Thinking blocks in response:**
```python
for block in response.content:
    if block.type == "thinking":
        print(f"Thinking: {block.thinking}")
    elif block.type == "text":
        print(f"Response: {block.text}")
```

**Streaming thinking:**
```
event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"thinking","thinking":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"thinking_delta","thinking":"Let me analyze..."}}
```

**Restrictions:**
- Not compatible with `temperature` != 1 on older models
- On Opus 4.7+ and Sonnet 5, `temperature`/`top_p`/`top_k` are fully removed (400 error)
- Redacted thinking: model may redact portions for safety reasons (returns `type: "redacted_thinking"`)

### Model Selection Guide

**Current Model Lineup (use aliases — no date suffixes needed):**

| Model | Alias | Context | Max Output | Input $/MTok | Output $/MTok | Best For |
|-------|-------|---------|------------|-------------|--------------|----------|
| Claude Fable 5 | `claude-fable-5` | 1M | 128K | $10.00 | $50.00 | Most demanding reasoning |
| Claude Opus 4.8 | `claude-opus-4-8` | 1M | 128K | $5.00 | $25.00 | Complex reasoning, agentic |
| Claude Sonnet 5 | `claude-sonnet-5` | 1M | 128K | $3.00 | $15.00 | Balanced intelligence + speed |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | 1M | 128K | $3.00 | $15.00 | Balanced (previous gen) |
| Claude Haiku 4.5 | `claude-haiku-4-5` | 200K | 64K | $1.00 | $5.00 | Fastest, cheapest, high throughput |

**When to use each model:**

| Factor | Use Haiku | Use Sonnet | Use Opus |
|--------|-----------|-----------|----------|
| Task complexity | Simple classification, extraction | General coding, analysis, writing | Complex multi-step reasoning |
| Latency needs | Sub-second response critical | Moderate latency acceptable | Latency not a primary concern |
| Cost constraints | High volume, low margin | Balanced budget | Quality justifies premium |
| Context window | Standard documents | Large codebases | Massive documents/analysis |

**Multi-model architecture pattern:**
```python
def route_request(query, complexity_score):
    if complexity_score < 0.3:
        return "claude-haiku-4-5"    # Triage/classification
    elif complexity_score < 0.7:
        return "claude-sonnet-4-6"     # Standard processing
    else:
        return "claude-opus-4-8"       # Complex reasoning
```

### Token Counting & Usage

**Response `usage` object:**
```json
{
    "usage": {
        "input_tokens": 523,
        "output_tokens": 142,
        "cache_creation_input_tokens": 2048,
        "cache_read_input_tokens": 0
    }
}
```

| Field | Description |
|-------|-------------|
| `input_tokens` | Tokens in your prompt (uncached portion) |
| `output_tokens` | Tokens Claude generated (includes thinking tokens) |
| `cache_creation_input_tokens` | Tokens written to cache this request |
| `cache_read_input_tokens` | Tokens served from cache (90% discount) |

**Token counting API (pre-flight estimation):**
```python
count = client.messages.count_tokens(
    model="claude-sonnet-4-6",
    messages=[{"role": "user", "content": "Hello, world!"}],
    system="You are helpful."
)
print(count.input_tokens)  # e.g., 15
```

**Approximate token ratios:**
- 1 token ~ 4 characters in English
- 1 token ~ 0.75 words
- Code typically uses more tokens per character than prose

**Tracking costs programmatically:**
```python
def calculate_cost(usage, model):
    pricing = {
        "claude-opus-4-8": {"input": 5.0, "output": 25.0},
        "claude-sonnet-4-6": {"input": 3.0, "output": 15.0},
        "claude-haiku-4-5": {"input": 1.0, "output": 5.0},
    }
    rates = pricing[model]
    input_cost = (usage.input_tokens / 1_000_000) * rates["input"]
    output_cost = (usage.output_tokens / 1_000_000) * rates["output"]
    return input_cost + output_cost
```

### Multi-Modal Inputs

**Image support — base64:**
```python
import base64

with open("image.png", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": "image/png",
                    "data": image_data
                }
            },
            {"type": "text", "text": "What's in this image?"}
        ]
    }]
)
```

**Image support — URL:**
```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "image",
                "source": {
                    "type": "url",
                    "url": "https://example.com/photo.jpg"
                }
            },
            {"type": "text", "text": "Describe this image"}
        ]
    }]
)
```

**Supported image formats:** JPEG, PNG, GIF, WebP

**Image sizing:**
- Images are automatically resized to fit within model limits
- Token cost is proportional to image dimensions
- Smaller images = fewer tokens = lower cost

**PDF support:**
```python
import base64

with open("document.pdf", "rb") as f:
    pdf_data = base64.standard_b64encode(f.read()).decode("utf-8")

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "document",
                "source": {
                    "type": "base64",
                    "media_type": "application/pdf",
                    "data": pdf_data
                }
            },
            {"type": "text", "text": "Summarize this document"}
        ]
    }]
)
```

---

## 4. Security & Reliability

### Authentication Best Practices

- **Never hardcode API keys** — use environment variables
- **Use `ANTHROPIC_API_KEY`** environment variable (SDK auto-reads it)
- **Rotate keys** regularly via the Console
- **Scope keys** to specific workspaces for least privilege
- **OAuth profiles** via `ant auth login` for developer workflows (no static key needed)

### Error Handling in Streams

```python
import anthropic

try:
    with client.messages.stream(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": "Hello"}]
    ) as stream:
        for text in stream.text_stream:
            print(text, end="", flush=True)
except anthropic.APIConnectionError:
    print("Connection lost — retry")
except anthropic.RateLimitError as e:
    retry_after = e.response.headers.get("retry-after", "60")
    print(f"Rate limited. Retry after {retry_after}s")
except anthropic.APIStatusError as e:
    if e.status_code >= 500:
        print("Server error — retry with backoff")
    else:
        print(f"Client error: {e.message}")
```

### Rate Limiting & Retries

- SDK auto-retries 429 and 5xx errors (default: `max_retries=2`)
- Exponential backoff with jitter
- Check `retry-after` header for 429 responses
- Per-organization limits: RPM (requests/min), TPM (tokens/min), TPD (tokens/day)

---

## 5. Exam Scenarios

### Scenario 1: Streaming Event Ordering

**Question:** A developer is building a streaming chat UI. They receive a `content_block_delta` event with `index: 1` and `delta.type: "input_json_delta"`. What is happening?

**Answer:** Claude is streaming a tool call's input parameters. The `index: 1` means this is the second content block (likely preceded by a text block at index 0). The `input_json_delta` type means partial JSON is being streamed for a `tool_use` block. The developer must accumulate all `input_json_delta` strings for this index and parse the complete JSON only after receiving `content_block_stop` for that index.

---

### Scenario 2: Extended Thinking Budget

**Question:** A developer sets `thinking: {"type": "enabled", "budget_tokens": 20000}` and `max_tokens: 16000`. What happens?

**Answer:** The request will fail with a 400 error. `budget_tokens` must be strictly less than `max_tokens`. The developer should either increase `max_tokens` above 20000 (e.g., to 25000) or reduce `budget_tokens` below 16000. On newer models (4.6+), they should instead use `thinking: {"type": "adaptive"}` with the `effort` parameter.

---

### Scenario 3: Model Selection for High-Volume Classification

**Question:** An application needs to classify 100,000 customer support tickets per day into 5 categories. Accuracy needs to be >95% but latency must be under 500ms. Which model and what approach?

**Answer:** Use Claude Haiku 4.5 (`claude-haiku-4-5`) for the best latency/cost ratio on simple classification. Enhance accuracy with:
1. A well-crafted system prompt with few-shot examples
2. Structured outputs (`output_config.format`) to guarantee valid categories
3. The Message Batches API for the 100K daily volume (50% cost reduction)
4. Prompt caching for the repeated system prompt

---

### Scenario 4: Missing Headers

**Question:** A developer's raw HTTP request returns `401 authentication_error`. They have a valid API key set in the request body. What's wrong?

**Answer:** The API key must be in the `x-api-key` header, not the request body. Required headers are:
- `x-api-key: sk-ant-...`
- `anthropic-version: 2023-06-01`  
- `content-type: application/json`

Missing `anthropic-version` returns 400; missing `x-api-key` returns 401.

---

### Scenario 5: Multi-Modal Token Optimization

**Question:** An application sends high-resolution screenshots (4K) to Claude for analysis but is hitting token limits rapidly. How should the developer optimize?

**Answer:** Images are automatically resized but still consume tokens proportional to their dimensions. Optimizations:
1. **Resize client-side** before sending — reduce to the minimum resolution needed (e.g., 1024px wide for most analysis)
2. **Crop to relevant regions** — send only the portion Claude needs to analyze
3. **Use URL source** when possible to avoid base64 encoding overhead
4. **Be specific in the text prompt** about what to look for (reduces output tokens)
5. **Use `count_tokens`** to pre-check image token cost before expensive operations

---

## 6. Anti-Patterns

| Anti-Pattern | Why It's Wrong | Correct Approach |
|-------------|----------------|------------------|
| Hardcoding API keys in source code | Security risk, key rotation impossible | Use environment variables |
| Ignoring `stop_reason` in responses | Missing tool calls, truncated output | Always check `stop_reason` before reading content |
| Setting `budget_tokens` >= `max_tokens` | Returns 400 error | Ensure `budget_tokens` < `max_tokens` |
| String-matching raw tool input JSON from streams | Escaping differs across models | Always parse with `json.loads()` after accumulating |
| Using `tiktoken` for Claude token counts | Wrong tokenizer — undercounts by 15-20% | Use `client.messages.count_tokens()` |
| Sending full 4K images when 720p suffices | Unnecessary token spend | Resize images to minimum needed resolution |
| Not handling `overloaded_error` in streams | Silent data loss | Implement stream error handling with retry |
| Using Opus for simple classification | Wasted cost and latency | Use Haiku for simple, high-volume tasks |

---

## 7. Quick Reference

### Required Headers

```
x-api-key: <your-api-key>
anthropic-version: 2023-06-01
content-type: application/json
```

### Stream Event Sequence

```
message_start → [ping]* → content_block_start → content_block_delta* 
→ content_block_stop → [more blocks...] → message_delta → message_stop
```

### Delta Types by Content Block

| Content Block Type | Delta Type | Field |
|-------------------|------------|-------|
| `text` | `text_delta` | `delta.text` |
| `tool_use` | `input_json_delta` | `delta.partial_json` |
| `thinking` | `thinking_delta` | `delta.thinking` |

### Stop Reasons

| Value | Meaning | Action |
|-------|---------|--------|
| `end_turn` | Natural completion | Done |
| `max_tokens` | Output limit hit | Increase `max_tokens` or continue |
| `tool_use` | Tool call requested | Execute tool, send result back |
| `stop_sequence` | Custom stop hit | Handle as needed |
| `pause_turn` | Server tool limit hit | Re-send to continue |
| `refusal` | Safety decline | Check `stop_details` |

### Token Counting Cheat Sheet

```python
# Pre-flight count
count = client.messages.count_tokens(model=MODEL, messages=MSGS)

# Post-response usage
response.usage.input_tokens         # prompt tokens (uncached)
response.usage.output_tokens        # generation tokens
response.usage.cache_read_input_tokens   # cached (cheap)
response.usage.cache_creation_input_tokens  # written to cache

# Cost formula
cost = (input_tokens * input_rate + output_tokens * output_rate) / 1_000_000
```

### Multi-Modal Content Block Types

```python
# Image (base64)
{"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": "..."}}

# Image (URL)
{"type": "image", "source": {"type": "url", "url": "https://..."}}

# PDF document
{"type": "document", "source": {"type": "base64", "media_type": "application/pdf", "data": "..."}}

# Text
{"type": "text", "text": "Describe this image"}
```

### Supported Media Types

| Format | media_type | Use Case |
|--------|-----------|----------|
| JPEG | `image/jpeg` | Photos, screenshots |
| PNG | `image/png` | Diagrams, UI captures |
| GIF | `image/gif` | Simple animations (first frame) |
| WebP | `image/webp` | Modern web images |
| PDF | `application/pdf` | Documents, reports |
