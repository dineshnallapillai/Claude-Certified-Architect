# Domain 1 | Topic 1.6: Error Handling in Agentic Systems

## 1. Core Concepts

### Why Error Handling is Critical in Agentic Systems

Unlike traditional software where errors are deterministic, agentic systems face:
- **Non-deterministic failures** — same input may succeed or fail depending on model reasoning
- **Cascading errors** — one bad tool result corrupts subsequent reasoning
- **Silent degradation** — agent continues with wrong assumptions
- **Compounding costs** — retries in loops multiply token usage

### Error Categories in Agentic Systems

```
┌─────────────────────────────────────────────────────────┐
│  ERROR TAXONOMY                                         │
│                                                         │
│  1. API Errors (deterministic)                          │
│     ├── Rate limiting (429)                             │
│     ├── Invalid request (400)                           │
│     ├── Context too long (400, invalid_request_error)   │
│     ├── Overloaded (529, overloaded_error)              │
│     └── Server error (500/503)                          │
│                                                         │
│  2. Tool Execution Errors                               │
│     ├── Tool not found                                  │
│     ├── Invalid tool input (schema violation)           │
│     ├── Tool timeout                                    │
│     └── Tool returned error                             │
│                                                         │
│  3. Reasoning Errors (non-deterministic)                │
│     ├── Hallucinated tool names                         │
│     ├── Wrong tool selection                            │
│     ├── Invalid parameters (type-correct but wrong)     │
│     └── Infinite loops (agent can't make progress)      │
│                                                         │
│  4. Orchestration Errors                                │
│     ├── Sub-agent timeout                               │
│     ├── Conflicting sub-agent results                   │
│     ├── Context overflow mid-task                       │
│     └── Circular delegation                             │
└─────────────────────────────────────────────────────────┘
```

### Error Handling Strategies

| Strategy | When to Use | Mechanism |
|----------|------------|-----------|
| **Retry** | Transient failures (rate limits, timeouts) | Exponential backoff |
| **Fallback** | Primary path fails, alternative exists | Switch to backup tool/agent |
| **Graceful degradation** | Partial failure, partial result is useful | Return what you have |
| **Escalation** | Agent can't resolve, human needed | Surface error with context |
| **Circuit breaker** | Repeated failures indicate systemic issue | Stop trying, report |
| **Correction** | Claude made a reasoning error | Feed error back, ask to retry |

---

## 2. Design Patterns

### Pattern 1: Structured Error Feedback

```python
def execute_tool_safely(tool_name, tool_input):
    try:
        result = tools[tool_name](tool_input)
        return {
            "type": "tool_result",
            "tool_use_id": tool_id,
            "content": json.dumps({"status": "success", "data": result})
        }
    except ValidationError as e:
        return {
            "type": "tool_result",
            "tool_use_id": tool_id,
            "content": json.dumps({
                "status": "error",
                "error_type": "validation",
                "message": f"Invalid input: {e.message}",
                "field": e.field,
                "expected": e.expected_type,
                "received": e.received_value,
                "suggestion": "Please provide a valid value for this field"
            }),
            "is_error": True  # Signals to Claude this is an error
        }
    except TimeoutError:
        return {
            "type": "tool_result",
            "tool_use_id": tool_id,
            "content": json.dumps({
                "status": "error",
                "error_type": "timeout",
                "message": "Tool execution timed out after 30s",
                "suggestion": "Try a simpler query or break into smaller parts"
            }),
            "is_error": True
        }
```

### Pattern 2: Retry with Exponential Backoff

```python
import time

def call_api_with_retry(messages, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = client.messages.create(
                model="claude-sonnet-4-6",
                messages=messages,
                max_tokens=4096
            )
            return response
        except anthropic.RateLimitError:
            wait_time = 2 ** attempt  # 1s, 2s, 4s
            time.sleep(wait_time)
        except anthropic.APIStatusError as e:
            if e.status_code == 529:
                # Anthropic overloaded_error — retry with backoff
                time.sleep(2 ** attempt)
            elif e.status_code >= 500:
                time.sleep(2 ** attempt)  # Retry server errors
            else:
                raise  # Don't retry client errors (4xx)
    
    raise MaxRetriesExceeded(f"Failed after {max_retries} attempts")
```

### Pattern 3: Circuit Breaker

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, reset_timeout=60):
        self.failures = 0
        self.threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.state = "closed"  # closed = normal, open = blocking
        self.last_failure_time = None
    
    def call(self, func, *args):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.reset_timeout:
                self.state = "half-open"  # Allow one test call
            else:
                raise CircuitOpen("Service unavailable, circuit breaker open")
        
        try:
            result = func(*args)
            self.failures = 0
            self.state = "closed"
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure_time = time.time()
            if self.failures >= self.threshold:
                self.state = "open"
            raise
```

### Pattern 4: Agent Self-Correction Loop

```python
def agentic_loop_with_correction(messages, tools, max_iterations=10):
    error_count = 0
    MAX_CONSECUTIVE_ERRORS = 3
    
    for i in range(max_iterations):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            tools=tools,
            messages=messages
        )
        
        messages.append({"role": "assistant", "content": response.content})
        
        if response.stop_reason != "tool_use":
            return response  # Success: agent produced final answer
        
        # Execute tools, tracking errors
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool_safely(block.name, block.input)
                if result.get("is_error"):
                    error_count += 1
                else:
                    error_count = 0  # Reset on success
                tool_results.append(result)
        
        # Circuit breaker: too many consecutive errors
        if error_count >= MAX_CONSECUTIVE_ERRORS:
            messages.append({"role": "user", "content": [{
                "type": "text",
                "text": f"You've encountered {error_count} consecutive errors. Please reassess your approach or ask for help."
            }]})
            error_count = 0  # Give agent a chance to self-correct
        else:
            messages.append({"role": "user", "content": tool_results})
    
    return "Agent exceeded maximum iterations"
```

### Pattern 5: Graceful Degradation

```python
def research_with_fallback(query):
    # Try primary source
    try:
        result = web_search(query)
        if result.results:
            return {"source": "web", "data": result}
    except Exception:
        pass
    
    # Fallback to cached/local data
    try:
        result = local_knowledge_base.search(query)
        if result.results:
            return {"source": "local_cache", "data": result, 
                    "warning": "Results may be outdated"}
    except Exception:
        pass
    
    # Final fallback: acknowledge limitation
    return {"source": "none", "data": None,
            "message": "Unable to find information. Proceeding with general knowledge only."}
```

---

## 3. Implementation Details

### The `is_error` Flag in Tool Results

```python
# When is_error is True, Claude knows the tool failed
# and should try a different approach
{
    "type": "tool_result",
    "tool_use_id": "toolu_abc123",
    "content": "File not found: /path/to/file.txt",
    "is_error": True  # Claude sees this as a failure signal
}
```

- `is_error: True` → Claude knows to try something different
- `is_error: False` (default) → Claude treats result as successful data
- Even without `is_error`, structured error JSON in content helps Claude reason

### Error Budget Tracking

```python
class ErrorBudget:
    """Track errors per session to prevent runaway costs."""
    
    def __init__(self, max_errors=10, max_tokens=100000):
        self.errors = 0
        self.max_errors = max_errors
        self.tokens_used = 0
        self.max_tokens = max_tokens
    
    def record_error(self):
        self.errors += 1
        if self.errors >= self.max_errors:
            raise BudgetExhausted("Too many errors in session")
    
    def record_tokens(self, count):
        self.tokens_used += count
        if self.tokens_used >= self.max_tokens:
            raise BudgetExhausted("Token budget exhausted")
    
    @property
    def health(self):
        error_pct = self.errors / self.max_errors
        token_pct = self.tokens_used / self.max_tokens
        return min(1.0 - error_pct, 1.0 - token_pct)
```

### Handling Context Overflow Errors

```python
def handle_context_overflow(messages, response_error):
    """When conversation exceeds context limit."""
    
    # Strategy 1: Summarize old messages
    summary = summarize_conversation(messages[:-5])  # Keep last 5
    messages = [
        {"role": "user", "content": f"[Previous context summary: {summary}]"},
        *messages[-5:]
    ]
    
    # Strategy 2: If still too large, trim tool results
    for msg in messages:
        if isinstance(msg["content"], list):
            for block in msg["content"]:
                if block.get("type") == "tool_result":
                    # Truncate large tool results
                    if len(block["content"]) > 2000:
                        block["content"] = block["content"][:2000] + "...[truncated]"
    
    return messages
```

---

## 4. Security & Reliability

### Error Information Leakage

| Risk | Example | Mitigation |
|------|---------|-----------|
| Stack traces to user | Python traceback with file paths | Catch and sanitize |
| Internal URLs in errors | Database connection strings | Redact before returning |
| API keys in error messages | "Invalid key: sk-abc123..." | Never include keys |
| System info disclosure | OS version, library versions | Generic error messages |

### Denial-of-Service via Error Loops

An attacker could craft inputs that cause repeated tool failures, consuming the error budget:

```python
# Defense: rate limit errors per tool
tool_error_rates = defaultdict(lambda: {"count": 0, "window_start": time.time()})

def check_tool_health(tool_name):
    rate = tool_error_rates[tool_name]
    if time.time() - rate["window_start"] > 60:
        rate["count"] = 0
        rate["window_start"] = time.time()
    
    if rate["count"] > 5:
        raise ToolDisabled(f"{tool_name} temporarily disabled due to errors")
```

### Reliability Principles

1. **Fail fast, fail loud** — don't silently swallow errors
2. **Idempotent operations** — safe to retry without side effects
3. **Explicit error boundaries** — know where errors are caught
4. **Preserve partial work** — save progress before risky operations
5. **Timeout everything** — no operation should block indefinitely
6. **Log for debugging** — error context is invaluable for diagnosis

---

## 5. Exam Scenarios

### Scenario 1
**Q:** An agent is in a loop where it keeps calling a tool with slightly different parameters, getting errors each time. After 5 attempts, costs are growing. What is the BEST intervention?

**A:** Implement a **consecutive error counter with a self-correction prompt:**
- After 3 consecutive errors to the same tool, inject a message: "You've failed 3 times with this tool. Reassess your approach — consider using a different tool or ask the user for clarification."
- Set a hard cap (e.g., 5 total errors) before escalating to the user

**Why not just retry forever?** Repeating failed approaches wastes tokens. The agent needs to be told to step back and reason differently.

### Scenario 2
**Q:** A sub-agent in a multi-agent system times out. The orchestrator has partial results from 4 other agents. What should it do?

**A:** **Graceful degradation:**
1. Log the timeout with the failed agent's task details
2. Assess whether the partial results are useful without the failed agent's contribution
3. If yes: synthesize available results with a note about the missing piece
4. If no: retry the failed agent with a shorter task or escalate

**Never:** block indefinitely waiting, or discard all results because one failed.

### Scenario 3
**Q:** Claude returns a tool call for a tool name that doesn't exist in your tool definitions. What is this error and how should you handle it?

**A:** This is a **hallucinated tool name** (reasoning error). Handle by:
```python
{
    "type": "tool_result",
    "tool_use_id": block.id,
    "content": f"Error: Tool '{block.name}' does not exist. Available tools are: {list(tools.keys())}. Please use one of the available tools.",
    "is_error": True
}
```
Feed this back so Claude self-corrects. If it happens repeatedly, the tool descriptions may be confusing and need improvement.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Swallowing errors silently | Agent proceeds with bad state | Always surface errors to the agent |
| Unlimited retries | Exponential cost explosion | Bounded retries with backoff |
| Same retry strategy for all errors | 400 errors won't fix themselves with retry | Distinguish transient vs. permanent errors |
| Generic error messages | Claude can't reason about fix | Structured errors with type, message, suggestion |
| No error budget | Runaway agent loops forever | Track errors, enforce limits |
| Exposing raw exceptions | Security risk + confusing for Claude | Sanitized, structured error format |
| Retrying on 400 (client error) | Will never succeed, wasting tokens | Only retry 429/529/5xx (transient) |

---

## 7. Quick Reference / Cheat Sheet

```
ERROR CATEGORIES
━━━━━━━━━━━━━━━━
• API errors: rate limits(429), bad request(400), context too long(400),
  overloaded(529), server(5xx)
• Tool errors: not found, timeout, invalid input, execution failure
• Reasoning errors: hallucination, wrong tool, invalid params, loops
• Orchestration errors: sub-agent timeout, conflicts, context overflow

RETRY RULES
━━━━━━━━━━━
• 429 (Rate limit): YES, with exponential backoff + jitter
• 529 (Overloaded): YES, with backoff (Anthropic-specific)
• 400 (Bad request): NO, fix the request
• 400 (Context too long): NO, reduce context (summarize/truncate)
• 500/503 (Server): YES, with backoff (max 3 retries)
• Tool timeout: YES (once), then fallback
• Reasoning error: Feed error back (max 3 self-corrections)

ERROR RESULT FORMAT
━━━━━━━━━━━━━━━━━━━
{
  "status": "error",
  "error_type": "validation|timeout|not_found|permission|internal",
  "message": "Human-readable explanation",
  "suggestion": "What to try instead"
}
+ "is_error": true in tool_result wrapper

RELIABILITY CHECKLIST
━━━━━━━━━━━━━━━━━━━━━
✓ Max iterations on all loops
✓ Timeout on all tool calls
✓ Error budget per session
✓ Circuit breaker for flaky tools
✓ Structured error responses
✓ Escalation path when stuck
✓ Partial results > total failure

ESCALATION TRIGGERS
━━━━━━━━━━━━━━━━━━━
• 3+ consecutive errors (same tool)
• Error budget exhausted
• Conflicting results can't be resolved
• Security-sensitive failure
• User input needed to proceed
```
