# Domain 2 | Topic 2.6: Tool Chaining & Composition Patterns

## 1. Core Concepts

### What is Tool Chaining?

**Tool chaining** is when Claude uses the output of one tool as input to another tool in sequence, building up a solution step by step within an agentic loop.

```
Tool A result → informs → Tool B input → informs → Tool C input → Final Answer
```

### Chaining vs. Composition

| Concept | Definition | Example |
|---------|-----------|---------|
| **Chaining** | Sequential tool calls where output flows forward | search → read → summarize |
| **Composition** | Multiple tool results combined for one decision | search A + search B → compare |
| **Nesting** | One tool call informed by another within same turn | N/A (Claude does one at a time) |

### How Claude Chains Tools

```
Turn 1: Claude → tool_use: search_code("auth handler")
Turn 2: [tool_result: found in auth.py:42] → Claude → tool_use: read_file("auth.py")
Turn 3: [tool_result: file contents] → Claude → tool_use: edit_file("auth.py", changes)
Turn 4: [tool_result: success] → Claude → tool_use: run_tests()
Turn 5: [tool_result: all pass] → Claude → "Done! I fixed the auth handler..."
```

Key insight: Claude decides the chain at each step based on accumulated context.

### Parallel Tool Calls

Claude can request **multiple tools in a single turn**:

```json
// Claude's response with multiple tool_use blocks
{
  "content": [
    {"type": "text", "text": "I'll check both files..."},
    {"type": "tool_use", "id": "t1", "name": "read_file", "input": {"path": "a.py"}},
    {"type": "tool_use", "id": "t2", "name": "read_file", "input": {"path": "b.py"}}
  ]
}
```

Both tools execute concurrently, and results are returned together.

---

## 2. Design Patterns

### Pattern 1: Search-Read-Act Chain

```
Common workflow:
1. search_code("login function") → finds auth/login.py:25
2. read_file("auth/login.py") → sees the implementation
3. edit_file("auth/login.py", fix) → applies the change
4. run_tests("auth/") → verifies the fix
5. [Final response] → reports success
```

### Pattern 2: Fan-Out Composition

```
Query requires data from multiple sources:
1. [Parallel] search_users("john") + search_orders("john") + search_tickets("john")
2. [All results returned together]
3. Claude synthesizes: "John has 3 open orders and 1 support ticket about..."
```

### Pattern 3: Iterative Refinement

```
Loop until satisfied:
1. generate_code(spec) → first attempt
2. run_linter(code) → finds issues
3. fix_issues(code, lint_results) → improved code
4. run_tests(code) → some failures
5. fix_tests(code, test_results) → fixed code
6. run_tests(code) → all pass ✓
```

### Pattern 4: Validation Chain

```
Write-then-verify:
1. write_file("config.yaml", content)
2. validate_yaml("config.yaml") → valid
3. apply_config("config.yaml") → applied
4. health_check() → healthy ✓
```

### Pattern 5: Composite Tools (Server-Side Chaining)

```python
# Instead of Claude doing 3 tool calls, combine them server-side:
{
    "name": "search_and_summarize",
    "description": "Search documents and return a summary of top results. Combines search + read + summarize in one call."
}

# Implementation:
def search_and_summarize(query, max_results=5):
    results = search(query)[:max_results]
    contents = [read_document(r.id) for r in results]
    summary = summarize(contents)
    return {"results": results, "summary": summary}
```

Benefits:
- Fewer round trips (lower latency)
- Fewer tokens (less conversation overhead)
- More reliable (no mid-chain failures)

---

## 3. Implementation Details

### Managing Chain State

```python
# The messages array naturally tracks chain state
messages = [
    {"role": "user", "content": "Fix the login bug"},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "t1", "name": "search_code", "input": {"q": "login"}}
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "t1", "content": "Found in auth.py:42"}
    ]},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "t2", "name": "read_file", "input": {"path": "auth.py"}}
    ]},
    # ... chain continues
]
```

### Optimizing Chain Length

| Problem | Solution |
|---------|----------|
| Too many small tools | Create composite tools |
| Redundant reads | Cache results in conversation |
| Unnecessary verification | Trust tools that have strong contracts |
| Sequential when parallel possible | Claude can call multiple tools per turn |

### Token Cost of Long Chains

```
Each chain link adds to context:
- Tool call: ~50-200 tokens (name + input)
- Tool result: variable (could be 100-10000 tokens)
- Claude's reasoning: ~100-500 tokens per turn

A 10-step chain might cost:
10 × (150 + 500 + 300) = ~9,500 tokens just for the chain
+ original system prompt and tools loaded every turn
```

### When to Combine Tools (Composite) vs. Chain

| Use Composite When... | Use Chain When... |
|----------------------|-------------------|
| Steps are always done together | Steps may vary based on results |
| No conditional logic between | Next step depends on previous result |
| Want to reduce latency | Need Claude to reason between steps |
| Steps don't need Claude's judgment | Claude adds value at each step |
| Reducing token cost is priority | Flexibility is priority |

---

## 4. Security & Reliability

### Chain Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|-----------|
| Mid-chain tool failure | Subsequent steps operate on bad data | Validate each step's result |
| Context overflow during chain | Claude loses early chain context | Summarize intermediate results |
| Chain divergence | Claude goes off-track after step 3 | Bounded chains + checkpoints |
| Cascading errors | Error in step 1 → wrong step 2 → wrong step 3 | Fail fast on errors |
| Non-idempotent retry | Retrying a write chain causes duplicates | Idempotent tool design |

### Safe Chaining Practices

```python
# Validate before chaining forward
def execute_chain_step(tool_name, input, previous_result):
    # Don't proceed if previous step failed
    if previous_result and previous_result.get("status") == "error":
        return {"status": "error", "message": "Chain halted: previous step failed"}
    
    result = execute_tool(tool_name, input)
    
    # Validate result before it becomes input for next step
    if not validate_chain_result(tool_name, result):
        return {"status": "error", "message": f"Unexpected result from {tool_name}"}
    
    return result
```

### Data Flow Security in Chains

```
CONCERN: Sensitive data flowing through chain
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Step 1: query_database → returns PII
Step 2: Claude sees PII in context
Step 3: Claude passes PII to another tool

MITIGATION:
- Redact sensitive fields in tool results
- Use IDs/references instead of raw data
- Scope each tool to need-to-know data
```

---

## 5. Exam Scenarios

### Scenario 1
**Q:** Claude is performing a 7-step chain to refactor code: search, read, plan, edit, lint, test, commit. The edit step fails with a validation error. What should happen?

**A:** The chain should:
1. Return the validation error to Claude (with `is_error: true`)
2. Claude examines the error and decides whether to:
   - Fix the edit and retry (most likely)
   - Re-read the file (if context was stale)
   - Report the issue to the user (if it can't resolve)
3. The chain does NOT automatically proceed to lint/test/commit
4. Progress before the error (search, read, plan results) remains in context

Key: errors in a chain halt forward progress but don't destroy prior work.

### Scenario 2
**Q:** A tool chain takes 15 steps to complete a task. Token usage is high because each step adds to context. What optimization MOST reduces cost?

**A:** **Create composite tools** for sub-sequences that always occur together:
- Combine `search` + `read` into `search_and_read`
- Combine `lint` + `test` into `validate_code`
- This reduces 15 steps to ~8 steps

Secondary: Summarize large intermediate results before they accumulate in context.

### Scenario 3
**Q:** Claude calls three tools in parallel: `read_file("a.py")`, `read_file("b.py")`, `read_file("c.py")`. How should results be returned?

**A:** All three results are returned in a single `user` message with three `tool_result` blocks:
```json
{"role": "user", "content": [
    {"type": "tool_result", "tool_use_id": "t1", "content": "...contents of a.py..."},
    {"type": "tool_result", "tool_use_id": "t2", "content": "...contents of b.py..."},
    {"type": "tool_result", "tool_use_id": "t3", "content": "...contents of c.py..."}
]}
```

Each result is matched to its tool call via `tool_use_id`. Claude receives all results at once and can reason about them together.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Always sequential when parallel possible | Unnecessary latency | Let Claude call independent tools in parallel |
| Extremely long chains (20+ steps) | Context overflow, token waste | Composite tools, checkpoints |
| No validation between steps | Bad data cascades | Validate each result |
| Rigid pre-planned chains | Can't adapt to unexpected results | Let Claude decide next step |
| Ignoring partial chain results on failure | Lost work | Preserve and report what succeeded |
| Giant tool results in chain | Context bloat | Summarize/truncate large results |

---

## 7. Quick Reference / Cheat Sheet

```
CHAIN TYPES
━━━━━━━━━━━
• Sequential: A → B → C (each depends on previous)
• Parallel: [A, B, C] → D (independent, then merge)
• Iterative: A → B → [check] → A → B → [pass] → C
• Fan-out/Fan-in: A → [B1,B2,B3] → C (split then merge)

OPTIMIZATION STRATEGIES
━━━━━━━━━━━━━━━━━━━━━━━
1. Composite tools (combine always-together steps)
2. Parallel execution (independent tools same turn)
3. Result summarization (truncate large intermediates)
4. Smart caching (don't re-read unchanged files)
5. Early termination (fail fast on errors)

PARALLEL TOOL CALLS
━━━━━━━━━━━━━━━━━━━
• Multiple tool_use blocks in one assistant message
• All execute concurrently
• All results returned in one user message
• Each result matched by tool_use_id

COMPOSITE TOOL DECISION
━━━━━━━━━━━━━━━━━━━━━━━
Combine when:
✓ Steps always occur together
✓ No Claude reasoning needed between
✓ Reducing latency/tokens matters
Keep separate when:
✓ Conditional logic between steps
✓ Claude's judgment adds value
✓ Steps may be used independently

CHAIN COST FORMULA
━━━━━━━━━━━━━━━━━━
N steps × (tool_input + tool_output + claude_reasoning)
+ system_prompt × N (repeated every turn)
+ accumulated_context (grows each step)
```
