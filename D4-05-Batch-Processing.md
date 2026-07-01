# Domain 4 | Topic 4.5: Batch Processing Patterns

## 1. Core Concepts

### What is Batch Processing with Claude?

Processing multiple items through Claude systematically — either via the Messages Batches API (async, 50% cheaper) or through programmatic looping.

### Anthropic Batches API

```python
# Create a batch of requests (up to 100,000)
batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": "item_001",
            "params": {
                "model": "claude-sonnet-4-6",
                "max_tokens": 1024,
                "messages": [{"role": "user", "content": "Summarize: ..."}]
            }
        },
        {
            "custom_id": "item_002",
            "params": {
                "model": "claude-sonnet-4-6",
                "max_tokens": 1024,
                "messages": [{"role": "user", "content": "Summarize: ..."}]
            }
        }
    ]
)
# Returns within 24 hours, 50% cost reduction
```

### Batch vs Real-Time Processing

| Aspect | Batch API | Real-Time Loop |
|--------|-----------|----------------|
| Latency | Up to 24h | Immediate |
| Cost | 50% cheaper | Full price |
| Use case | Non-urgent bulk work | Interactive, urgent |
| Max items | 100,000 per batch | Unlimited (rate limited) |
| Error handling | Per-item results | Handle inline |
| Complexity | Simple (fire and forget) | Need orchestration |

---

## 2. Design Patterns

### Pattern 1: Batch Classification

```python
import anthropic

client = anthropic.Anthropic()

# Prepare batch requests
tickets = load_tickets()  # List of support tickets
requests = []

for i, ticket in enumerate(tickets):
    requests.append({
        "custom_id": f"ticket_{ticket.id}",
        "params": {
            "model": "claude-haiku-4-5",
            "max_tokens": 200,
            "messages": [{"role": "user", "content": f"""
Classify this support ticket:
Title: {ticket.title}
Body: {ticket.body}

Respond with JSON: {{"category": "bug|feature|question", "priority": "high|medium|low"}}
"""}]
        }
    })

# Submit batch
batch = client.messages.batches.create(requests=requests)
print(f"Batch submitted: {batch.id}")

# Poll for completion
import time
while True:
    status = client.messages.batches.retrieve(batch.id)
    if status.processing_status == "ended":
        break
    time.sleep(60)

# Retrieve results
results = client.messages.batches.results(batch.id)
for result in results:
    print(f"{result.custom_id}: {result.result.message.content[0].text}")
```

### Pattern 2: Parallel Real-Time Processing

```python
import asyncio
from anthropic import AsyncAnthropic

client = AsyncAnthropic()

async def process_item(item, semaphore):
    """Process one item with concurrency control."""
    async with semaphore:
        response = await client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            messages=[{"role": "user", "content": f"Process: {item}"}]
        )
        return {"id": item.id, "result": response.content[0].text}

async def batch_process_realtime(items, max_concurrent=10):
    """Process items in parallel with rate limiting."""
    semaphore = asyncio.Semaphore(max_concurrent)
    tasks = [process_item(item, semaphore) for item in items]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    successes = [r for r in results if not isinstance(r, Exception)]
    failures = [r for r in results if isinstance(r, Exception)]
    
    return successes, failures
```

### Pattern 3: Chunked Processing with Progress

```python
def process_in_chunks(items, chunk_size=50):
    """Process large datasets in manageable chunks."""
    results = []
    total = len(items)
    
    for i in range(0, total, chunk_size):
        chunk = items[i:i + chunk_size]
        print(f"Processing chunk {i//chunk_size + 1}/{(total + chunk_size - 1)//chunk_size}")
        
        chunk_results = process_chunk(chunk)
        results.extend(chunk_results)
        
        # Save intermediate results (resumable)
        save_checkpoint(results, i + chunk_size)
        
        # Respect rate limits
        time.sleep(1)
    
    return results
```

### Pattern 4: Map-Reduce with Claude

```python
async def map_reduce(documents, question):
    """Process many documents and synthesize."""
    
    # MAP: Process each document independently
    summaries = []
    for doc in documents:
        summary = await client.messages.create(
            model="claude-haiku-4-5",
            max_tokens=500,
            messages=[{"role": "user", "content": f"Summarize this relevant to '{question}':\n{doc}"}]
        )
        summaries.append(summary.content[0].text)
    
    # REDUCE: Synthesize all summaries
    final = await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2000,
        messages=[{"role": "user", "content": f"""
Based on these summaries, answer: {question}

Summaries:
{chr(10).join(f'- {s}' for s in summaries)}
"""}]
    )
    
    return final.content[0].text
```

---

## 3. Implementation Details

### Batches API Lifecycle

```
create → processing → ended
                        ├── results available
                        └── some may have errored (per-item)
```

### Batch Result Structure

```python
# Each result in the batch:
{
    "custom_id": "item_001",
    "result": {
        "type": "succeeded",  # or "errored", "expired"
        "message": {
            "content": [{"type": "text", "text": "..."}],
            "stop_reason": "end_turn"
        }
    }
}
```

### Cost Comparison

```
100 items × 1000 input tokens × 500 output tokens

Real-time (Sonnet):
  Input: 100 × 1000 × $3/M = $0.30
  Output: 100 × 500 × $15/M = $0.75
  Total: $1.05

Batch API (Sonnet, 50% off):
  Input: 100 × 1000 × $1.50/M = $0.15
  Output: 100 × 500 × $7.50/M = $0.375
  Total: $0.525 (50% savings)
```

### Error Handling in Batches

```python
results = client.messages.batches.results(batch.id)

succeeded = []
failed = []

for result in results:
    if result.result.type == "succeeded":
        succeeded.append(result)
    elif result.result.type == "errored":
        failed.append({
            "id": result.custom_id,
            "error": result.result.error
        })
    elif result.result.type == "expired":
        failed.append({"id": result.custom_id, "error": "expired"})

# Retry failed items
if failed:
    retry_batch = create_retry_batch(failed)
```

---

## 4. Security & Reliability

### Batch Processing Risks

| Risk | Mitigation |
|------|-----------|
| PII in batch items | Audit batch content, encrypt at rest |
| Partial batch failure | Idempotent processing, retry failed items |
| Cost runaway | Set budget limits, monitor batch sizes |
| Results contain sensitive data | Secure result storage, access controls |
| Stale results (24h delay) | Check freshness, re-process if needed |

### Reliability Patterns

1. **Checkpointing** — save progress to resume after failures
2. **Idempotent processing** — safe to re-run same items
3. **Dead letter queue** — collect failed items for investigation
4. **Result validation** — validate batch results before using
5. **Monitoring** — track success/failure rates per batch

---

## 5. Exam Scenarios

### Scenario 1
**Q:** You need to classify 50,000 support tickets. It's not time-sensitive (can wait 24h). What is the MOST cost-effective approach?

**A:** Use the **Messages Batches API**:
- 50% cost reduction (within 24h SLA)
- Up to 100,000 items per batch
- Use claude-haiku for classification (cheapest model sufficient for this task)
- Total cost: ~50,000 × minimal tokens × Haiku pricing × 0.5 = very affordable

### Scenario 2
**Q:** A batch of 1000 items completes, but 50 items have `"type": "errored"`. What's the correct handling pattern?

**A:**
1. Separate succeeded from errored results
2. Process succeeded results normally
3. For errored items: create a new retry batch with just those 50 items
4. If retry also fails: log to dead letter queue for human investigation
5. Never block the 950 successful results waiting for 50 retries

### Scenario 3
**Q:** You need real-time processing of 200 items but don't want to overwhelm the API. What pattern controls throughput?

**A:** Use **async processing with a semaphore** for concurrency control:
```python
semaphore = asyncio.Semaphore(10)  # Max 10 concurrent requests
```
- Processes all 200 items as fast as rate limits allow
- Never more than 10 in-flight requests
- Handles backpressure naturally
- Catches and retries individual failures

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Sequential processing of 10K items | Takes hours | Batch API or parallel async |
| No checkpointing | Lose all progress on failure | Save progress per chunk |
| Batch API for urgent tasks | 24h wait unacceptable | Real-time parallel processing |
| No error handling per item | One failure blocks all | Individual item error handling |
| Same model for all batch tasks | Cost inefficient | Use cheapest sufficient model (Haiku for simple tasks) |
| No deduplication | Process same item twice | Track processed IDs |

---

## 7. Quick Reference / Cheat Sheet

```
BATCH API
━━━━━━━━━
• Up to 100,000 requests per batch
• 50% cost reduction
• Results within 24 hours
• Per-item success/failure tracking
• Statuses: processing → ended

WHEN TO USE WHAT
━━━━━━━━━━━━━━━━
Batch API: Non-urgent, cost-sensitive, high volume
Parallel async: Urgent, moderate volume, need results now
Sequential: Low volume, order matters, simple

COST OPTIMIZATION
━━━━━━━━━━━━━━━━━
1. Batch API (50% off) for non-urgent
2. Haiku for simple tasks (classification, extraction)
3. Sonnet for moderate tasks (analysis, generation)
4. Opus only when reasoning quality critical

RELIABILITY CHECKLIST
━━━━━━━━━━━━━━━━━━━━
✓ Checkpoint progress (resumable)
✓ Handle per-item failures
✓ Retry failed items separately
✓ Validate results before using
✓ Monitor success rates
✓ Idempotent processing

CONCURRENCY CONTROL
━━━━━━━━━━━━━━━━━━━
• Semaphore for max concurrent requests
• Rate limiting to stay within API limits
• Chunking for very large datasets
• Backoff on rate limit errors
```
