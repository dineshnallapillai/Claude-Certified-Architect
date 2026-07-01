# Domain 5 | Topic 5.6: Caching & Token Optimization

## 1. Core Concepts

### Why Token Optimization Matters

In agentic systems, token usage compounds rapidly:
```
Single call: system_prompt + tools + messages + response
Agentic loop: (system_prompt + tools + growing_messages + response) × N iterations

Example:
5K system + 3K tools + 2K messages + 1K response = 11K per call
20 iterations with growing context: ~300K total tokens
With optimization: ~100K tokens (67% reduction)
```

### Anthropic Prompt Caching

**Prompt caching** stores frequently-repeated content server-side, reducing cost on subsequent calls.

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    system=[
        {
            "type": "text",
            "text": "Your long system prompt here...",
            "cache_control": {"type": "ephemeral"}  # ← Enable caching
        }
    ],
    messages=[...]
)
```

### Caching Economics

| Aspect | Without Caching | With Caching |
|--------|----------------|--------------|
| First call | Full price | Full price + 25% cache write |
| Subsequent calls (within 5 min) | Full price | 90% discount on cached portion |
| Agentic loop (20 calls) | 20 × full price | 1 × (full + write) + 19 × 10% = massive savings |

### What Can Be Cached

| Content | Cacheable? | Method |
|---------|-----------|--------|
| System prompt | Yes | `cache_control: {"type": "ephemeral"}` |
| Tool definitions | Yes (via system) | Include in cached system block |
| Few-shot examples | Yes | Include in cached block |
| Long documents | Yes | Place in early messages with cache control |
| User messages | Partial | First part of conversation can be cached |
| Dynamic content | No | Changes break cache |

---

## 2. Design Patterns

### Pattern 1: Cached System Prompt for Agents

```python
# In agentic loops, system prompt repeats every call — cache it
system_blocks = [
    {
        "type": "text",
        "text": LONG_SYSTEM_PROMPT,  # 5000+ tokens of instructions
        "cache_control": {"type": "ephemeral"}
    }
]

# Every iteration uses cached system prompt
for iteration in range(20):
    response = client.messages.create(
        model="claude-sonnet-4-6",
        system=system_blocks,  # Cached after first call
        tools=tools,
        messages=messages
    )
    # Process response...
```

### Pattern 2: Document Caching for RAG

```python
# Cache long documents that are referenced multiple times
messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": f"Here is the document to analyze:\n\n{long_document}",
                "cache_control": {"type": "ephemeral"}  # Cache the document
            },
            {
                "type": "text",
                "text": "What are the key security risks in this code?"
            }
        ]
    }
]
```

### Pattern 3: Token-Efficient Tool Results

```python
def optimize_tool_result(result, max_tokens=2000):
    """Keep tool results small to minimize context growth."""
    
    tokens = estimate_tokens(result)
    
    if tokens <= max_tokens:
        return result
    
    # Strategy 1: Truncate with indicator
    if isinstance(result, str) and len(result) > 8000:
        return result[:8000] + f"\n\n[... truncated {tokens - max_tokens} tokens. Use tool again for specific sections.]"
    
    # Strategy 2: Summarize structured data
    if isinstance(result, list) and len(result) > 20:
        return {
            "total_items": len(result),
            "showing_first_10": result[:10],
            "note": "Use pagination or filter for remaining items"
        }
    
    return result
```

### Pattern 4: Message Compaction for Token Savings

```python
def compact_messages_for_tokens(messages, budget_tokens=50000):
    """Keep total message context within budget."""
    
    current_tokens = sum(estimate_tokens(m) for m in messages)
    
    if current_tokens <= budget_tokens:
        return messages
    
    # Strategies (apply in order):
    
    # 1. Truncate large tool results (keep first 500 chars)
    for msg in messages[:-5]:  # Don't touch recent messages
        if isinstance(msg.get("content"), list):
            for block in msg["content"]:
                if block.get("type") == "tool_result" and len(str(block.get("content", ""))) > 500:
                    block["content"] = str(block["content"])[:500] + "...[truncated]"
    
    # 2. If still over budget, summarize old messages
    if sum(estimate_tokens(m) for m in messages) > budget_tokens:
        messages = summarize_old_messages(messages)
    
    return messages
```

### Pattern 5: Model Selection for Cost

```python
def select_model_for_task(task_type, budget_priority):
    """Use cheapest model that can handle the task."""
    
    MODEL_COSTS = {
        "claude-opus-4-8": {"input": 5, "output": 25},       # $/M tokens
        "claude-sonnet-4-6": {"input": 3, "output": 15},
        "claude-haiku-4-5": {"input": 0.80, "output": 4},
    }
    
    TASK_MODEL_MAP = {
        "classification": "claude-haiku-4-5",
        "extraction": "claude-haiku-4-5",
        "summarization": "claude-haiku-4-5",
        "code_generation": "claude-sonnet-4-6",
        "code_review": "claude-sonnet-4-6",
        "complex_reasoning": "claude-opus-4-8",
        "architecture_design": "claude-opus-4-8",
    }
    
    return TASK_MODEL_MAP.get(task_type, "claude-sonnet-4-6")
```

---

## 3. Implementation Details

### Caching Requirements

| Requirement | Detail |
|-------------|--------|
| Minimum cacheable size | Varies by model: 1024 (Sonnet 4.5/4.1/4/3.7), 2048 (Fable 5/Sonnet 4.6/Haiku 3.5/3), 4096 (Opus 4.8/4.7/4.6/4.5, Haiku 4.5) |
| Cache TTL | 5 minutes (default) or 1 hour (`"ttl": "1h"`) |
| Write premium | 25% (5-min TTL) or 100% (1-hour TTL) |
| Cache key | Prefix match — any byte change invalidates everything after it |
| Max breakpoints | 4 per request |
| Dynamic content | Breaks cache — keep dynamic parts AFTER last cached block |

### Cache-Friendly Message Structure

```python
# GOOD: Static content first (cached), dynamic last (not cached)
messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": LARGE_REFERENCE_DOC,  # Static - cacheable
                "cache_control": {"type": "ephemeral"}
            },
            {
                "type": "text",
                "text": f"Now answer: {user_question}"  # Dynamic - not cached
            }
        ]
    }
]

# BAD: Dynamic content before static (breaks caching)
messages = [
    {"role": "user", "content": f"Context: today is {date}. {LARGE_DOC}. Question: ..."}
]
```

### Token Counting Utilities

```python
# Anthropic's token counting
import anthropic

client = anthropic.Anthropic()

# Count tokens for a message
token_count = client.count_tokens(
    model="claude-sonnet-4-6",
    messages=[{"role": "user", "content": "Hello!"}]
)

# Rough estimation (when API not available)
def estimate_tokens(text):
    """Approximate: 1 token ≈ 4 characters for English."""
    if isinstance(text, str):
        return len(text) // 4
    return len(json.dumps(text)) // 4
```

### Cost Calculation Formula

```
COST PER AGENTIC SESSION:
━━━━━━━━━━━━━━━━━━━━━━━━

Without caching:
  N × (system_tokens × input_rate + response_tokens × output_rate)
  + N × cumulative_message_tokens × input_rate

With caching:
  1 × (system_tokens × input_rate × 1.25)  [cache write]
  + (N-1) × (system_tokens × input_rate × 0.1)  [cache reads]
  + N × (message_tokens × input_rate + response_tokens × output_rate)

Savings increase with:
- Larger system prompts (more to cache)
- More iterations (more cache hits)
- Longer tool definitions (cached with system)
```

### Extended Thinking Token Costs

When using extended thinking (thinking budget), be aware that **thinking tokens are billed at the same rate as output tokens**. This can significantly impact costs for complex reasoning tasks.

```
EXTENDED THINKING COST IMPACT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Model: Claude Sonnet 4 ($15/M output tokens)

Without thinking:  500 output tokens = $0.0075
With thinking:     10,000 thinking + 500 output = $0.1575

Thinking tokens can be 10-50x the visible output!
Budget thinking tokens as part of your output cost estimate.
```

Key considerations:
- Set `budget_tokens` to cap thinking costs (e.g., 5,000-10,000 tokens for most tasks)
- Reserve extended thinking for high-value decisions (architecture, complex debugging)
- Simple tasks (classification, extraction) do not benefit from thinking and waste tokens
- Monitor thinking token usage via the `usage` field in API responses

---

## 4. Security & Reliability

### Caching Security

| Concern | Detail |
|---------|--------|
| Cache isolation | Caches are per-API-key, not shared across users |
| Sensitive data in cache | Cached content has same security as API calls |
| Cache poisoning | Not possible (content-addressed, exact match) |
| TTL expiration | 5 minutes — stale data auto-expires |

### Cost Monitoring

```python
class CostTracker:
    """Track and alert on token spending."""
    
    def __init__(self, daily_budget_usd=50):
        self.daily_budget = daily_budget_usd
        self.today_spent = 0
    
    def record(self, input_tokens, output_tokens, cached_input=0, model="sonnet"):
        rates = MODEL_RATES[model]
        cost = (
            (input_tokens - cached_input) * rates["input"] / 1_000_000 +
            cached_input * rates["cached_input"] / 1_000_000 +
            output_tokens * rates["output"] / 1_000_000
        )
        self.today_spent += cost
        
        if self.today_spent > self.daily_budget * 0.8:
            alert(f"⚠️ Token spend at {self.today_spent/self.daily_budget*100:.0f}% of daily budget")
        
        return cost
```

---

## 5. Exam Scenarios

### Scenario 1
**Q:** An agentic system makes 30 API calls per task. System prompt is 4000 tokens, tools are 3000 tokens. Without caching, what's the repeated cost for just system+tools? With caching?

**A:**
- Without: 30 × 7000 = 210,000 input tokens just for system+tools
- With caching:
  - Call 1: 7000 tokens × 1.25 (cache write) = 8,750 effective tokens
  - Calls 2-30: 7000 × 29 × 0.1 (cache read) = 20,300 effective tokens
  - Total: 29,050 vs 210,000 = **86% reduction**

### Scenario 2
**Q:** A tool returns a 20,000 token result. The next 10 API calls all carry this in context. What's the optimization?

**A:** Multiple strategies:
1. **Truncate the result** to relevant portions (~2000 tokens)
2. **Summarize immediately** — extract key findings, discard raw data
3. **Cache the result** in early message position with `cache_control`
4. **External storage** — save to file, include only a reference

Best: summarize immediately (reduces from 200K accumulated tokens to 20K).

### Scenario 3
**Q:** You're choosing between Opus, Sonnet, and Haiku for different parts of a multi-agent system. A sub-agent classifies support tickets. Which model and why?

**A:** **Haiku** because:
- Classification is a simple, well-defined task
- Haiku input: $0.80/M vs Sonnet $3/M vs Opus $15/M (18x cheaper than Opus)
- At 1000 tickets/day: Haiku ≈ $0.80/day vs Opus ≈ $15/day
- Quality difference for classification is minimal
- Speed: Haiku is fastest (lowest latency)

Reserve Sonnet for code generation and Opus for complex architecture decisions.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| No prompt caching in agentic loops | Paying full price 20+ times | Always cache system prompt |
| Keeping raw 50K token tool results | Context bloat, cost explosion | Summarize/truncate immediately |
| Using Opus for classification | 18x more expensive, no quality gain | Use Haiku for simple tasks |
| Dynamic content at start of messages | Breaks cache (content changes) | Static first (cached), dynamic last |
| No cost monitoring | Budget overruns undetected | Track spending, set alerts |
| Ignoring cache minimum (1024–4096 tokens depending on model) | Content too small to cache | Batch small static content together |

---

## 7. Quick Reference / Cheat Sheet

```
PROMPT CACHING
━━━━━━━━━━━━━━
• Add: cache_control: {"type": "ephemeral"}
• Min size: 1024 (Sonnet 4.5/4.1/4/3.7), 2048 (Fable 5/Sonnet 4.6/Haiku 3.5/3), 4096 (Opus/Haiku 4.5)
• TTL: 5 minutes (default) or 1 hour (with `"ttl": "1h"`)
• Write: +25% first time
• Read: -90% subsequent times
• Best for: system prompts, tools, static docs

MODEL COST TIERS ($/M tokens input/output)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Haiku:  $0.80 / $4    (simple tasks)
Sonnet: $3 / $15      (general tasks)
Opus:   $15 / $75     (complex reasoning)

TOKEN OPTIMIZATION STRATEGIES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Cache system prompt (biggest win in loops)
2. Truncate/summarize large tool results
3. Use cheapest sufficient model per task
4. Proactive context compaction
5. Static content first (cache-friendly)
6. Monitor and budget spending

COST FORMULA (per call)
━━━━━━━━━━━━━━━━━━━━━━━
input_tokens × $input_rate
+ cached_tokens × ($input_rate × 0.1)
+ output_tokens × $output_rate

SAVINGS IN AGENTIC LOOPS
━━━━━━━━━━━━━━━━━━━━━━━━
20 iterations, 5K cached system:
Without cache: 100K input tokens at full price
With cache: 5K full + 95K at 10% = ~14.5K effective
Savings: ~85% on system prompt costs
```
