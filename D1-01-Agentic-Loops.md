# Domain 1 | Topic 1.1: Agentic Loops & Core API Functionality

## 1. Core Concepts

### What is an Agentic Loop?

An **agentic loop** is a control pattern where Claude repeatedly:
1. Receives input (user message or tool result)
2. Processes and reasons about it
3. Decides on an action (tool call OR final response)
4. Executes the action
5. Loops back to step 1 with the result

```
┌─────────────────────────────────────────┐
│           AGENTIC LOOP                  │
│                                         │
│  User Input → Claude Reasoning          │
│       ↑              ↓                  │
│       │      Decision Point             │
│       │       ↙        ↘               │
│       │  Tool Call    Final Response    │
│       │      ↓                          │
│       │  Execute Tool                   │
│       │      ↓                          │
│       └── Tool Result                   │
└─────────────────────────────────────────┘
```

### Key Characteristics
- **Autonomous decision-making**: Claude decides what to do next without human intervention per step
- **Iterative refinement**: Each loop iteration adds information or makes progress
- **Termination condition**: The loop ends when Claude produces a final text response (no tool calls)
- **Stateful context**: The full message history accumulates across iterations

### Core API Functionality for Agents

| API Feature | Role in Agentic Systems |
|-------------|------------------------|
| `messages.create()` | The fundamental call — each loop iteration is one API call |
| `tool_use` content blocks | Claude signals it wants to use a tool |
| `tool_result` content blocks | You feed results back to Claude |
| `system` prompt | Defines agent personality, constraints, available tools |
| `max_tokens` | Controls output length per iteration |
| `stop_reason` | Determines if loop continues (`tool_use`) or ends (`end_turn`) |

### The Messages API Loop Pattern (Python)

```python
import anthropic

client = anthropic.Anthropic()

def run_agent(user_message, tools, system_prompt):
    messages = [{"role": "user", "content": user_message}]
    
    while True:
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            system=system_prompt,
            tools=tools,
            messages=messages
        )
        
        # Append assistant response to conversation
        messages.append({"role": "assistant", "content": response.content})
        
        # Check if agent wants to use a tool
        if response.stop_reason == "tool_use":
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
        else:
            # Agent produced final response — exit loop
            return response.content
```

### Why the Template Above is Incomplete (Exam Awareness)

The template above demonstrates the **core pattern** but is NOT production-ready. It's missing:

1. **`execute_tool()` function** — YOUR code that actually runs tools (API calls, DB queries, etc.)
2. **Tool definitions** — the `tools` array with name, description, input_schema
3. **Max iteration limit** — `while True` is dangerous; use `for i in range(MAX):`
4. **Error handling** — what if tool execution fails?
5. **The caller** — how to invoke `run_agent()`

### Complete Runnable Example

```python
import anthropic
import json

client = anthropic.Anthropic()  # Requires ANTHROPIC_API_KEY env variable

# 1. Define tools (what Claude can use)
tools = [
    {
        "name": "get_weather",
        "description": "Get current weather for a city. Use when user asks about weather conditions.",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "City name (e.g., 'Tokyo', 'New York')"}
            },
            "required": ["city"]
        }
    }
]

# 2. Implement tool execution (YOUR application logic)
def execute_tool(name, input_params):
    """Run the tool and return a result string."""
    if name == "get_weather":
        # In production: call a real weather API
        city = input_params.get("city", "Unknown")
        return json.dumps({"city": city, "temp": "72°F", "condition": "Sunny"})
    else:
        return json.dumps({"error": f"Unknown tool: {name}", "available": ["get_weather"]})

# 3. The agentic loop (BOUNDED — never use while True in production)
def run_agent(user_message, tools, system_prompt, max_iterations=10):
    messages = [{"role": "user", "content": user_message}]
    
    for iteration in range(max_iterations):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            system=system_prompt,
            tools=tools,
            messages=messages
        )
        
        # Add Claude's response to conversation history
        messages.append({"role": "assistant", "content": response.content})
        
        # Decision point: does Claude want to use a tool?
        if response.stop_reason == "tool_use":
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,  # MUST match the tool_use block's id
                        "content": result
                    })
            # Tool results go in "user" role (NOT assistant, NOT system)
            messages.append({"role": "user", "content": tool_results})
            # Loop continues → next iteration, Claude sees results
        else:
            # stop_reason is "end_turn" → Claude produced final answer
            return response.content
    
    # Safety: if we exceed max_iterations, don't loop forever
    return [{"type": "text", "text": "Agent exceeded maximum iterations. Escalating."}]

# 4. Run the agent
if __name__ == "__main__":
    result = run_agent(
        user_message="What's the weather in Tokyo?",
        tools=tools,
        system_prompt="You are a helpful weather assistant. Use tools when needed."
    )
    for block in result:
        if hasattr(block, 'text'):
            print(block.text)
```

### Execution Flow Walkthrough

```
ITERATION 1:
━━━━━━━━━━━━
  Input messages: [user: "What's the weather in Tokyo?"]
  
  Claude responds:
    content: [
      {type: "text", text: "Let me check the weather..."},
      {type: "tool_use", id: "toolu_abc", name: "get_weather", input: {"city": "Tokyo"}}
    ]
    stop_reason: "tool_use"  ← Claude wants a tool
  
  Your code:
    → Appends assistant response to messages
    → Executes: get_weather({"city": "Tokyo"})
    → Returns: '{"city": "Tokyo", "temp": "72°F", "condition": "Sunny"}'
    → Appends tool_result to messages (role: "user")
    → Loop continues to iteration 2

ITERATION 2:
━━━━━━━━━━━━
  Input messages: [user msg, assistant msg, tool_result msg]
  
  Claude responds:
    content: [
      {type: "text", text: "The weather in Tokyo is 72°F and Sunny!"}
    ]
    stop_reason: "end_turn"  ← Claude is done
  
  Your code:
    → Appends assistant response to messages
    → stop_reason != "tool_use" → EXIT loop
    → Returns response.content to caller
```

### Critical Implementation Rules (Exam Topics)

| Rule | Why It Matters |
|------|---------------|
| Tool results go in `"role": "user"` | API contract — "assistant" can't contain tool_results |
| Each `tool_result` MUST have matching `tool_use_id` | API validates this; mismatches cause errors |
| Process ALL `tool_use` blocks (not just first) | Claude may call multiple tools per turn |
| `execute_tool()` is YOUR code | Claude requests tools; your app runs them |
| `while True` → `for i in range(N)` | Prevents infinite loops and cost explosion |
| Check `stop_reason`, not content type | The definitive signal for loop control |

### Stop Reasons

| `stop_reason` | Meaning | Action |
|---------------|---------|--------|
| `"end_turn"` | Claude finished responding | Exit loop, return response |
| `"tool_use"` | Claude wants to call tool(s) | Execute tools, feed results back |
| `"max_tokens"` | Hit token limit mid-response | May need to continue or handle truncation |
| `"stop_sequence"` | Hit a custom stop sequence | Application-specific handling |

---

## 2. Design Patterns

### Pattern 1: Simple Tool Loop
- Single agent with tools
- Loop until `end_turn`
- Best for: straightforward task automation

### Pattern 2: ReAct (Reason + Act)
- Claude reasons about what to do (in thinking/text), then acts
- Explicit reasoning traces improve reliability
- Best for: complex multi-step tasks requiring planning

### Pattern 3: Plan-Execute-Verify
```
Plan → Execute Step 1 → Verify → Execute Step 2 → Verify → ... → Final
```
- Agent first creates a plan, then executes steps, verifying after each
- Best for: high-stakes tasks where errors compound

### Pattern 4: Bounded Loop with Escalation
```python
MAX_ITERATIONS = 10

for i in range(MAX_ITERATIONS):
    response = client.messages.create(...)
    if response.stop_reason != "tool_use":
        break
else:
    # Escalate: agent exceeded iteration budget
    handle_escalation()
```
- Prevents runaway agents
- Critical for production systems

---

## 3. Implementation Details

### Tool Definition Schema

```python
tools = [
    {
        "name": "read_file",
        "description": "Read the contents of a file at the given path. Use this when you need to examine file contents.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "Absolute path to the file to read"
                }
            },
            "required": ["path"]
        }
    }
]
```

### Streaming in Agentic Loops

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    tools=tools,
    messages=messages
) as stream:
    for event in stream:
        # Handle real-time events
        pass
    response = stream.get_final_message()
```

### Extended Thinking in Agents

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000
    },
    tools=tools,
    messages=messages
)
```
- Thinking blocks appear in `response.content` with `type: "thinking"`
- Improves reasoning quality for complex decisions
- Budget controls cost vs. quality tradeoff

### Tool Choice Parameter

The `tool_choice` parameter controls how Claude interacts with available tools:

```python
# AUTO (default): Claude decides whether to use a tool or respond with text
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    tools=tools,
    tool_choice={"type": "auto"},  # Default behavior
    messages=messages
)

# ANY: Claude MUST use one of the available tools (no text-only response)
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    tools=tools,
    tool_choice={"type": "any"},  # Forces tool use
    messages=messages
)

# TOOL (specific): Claude MUST use the specified tool
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    tools=tools,
    tool_choice={"type": "tool", "name": "get_weather"},  # Forces specific tool
    messages=messages
)
```

| `tool_choice` Type | Behavior | Use Case |
|-------------------|----------|----------|
| `auto` (default) | Claude decides freely | General agentic loops |
| `any` | Must call some tool | Force action, prevent premature stopping |
| `tool` (+ name) | Must call the named tool | Structured extraction, mandatory steps |

**Key considerations:**
- With `any` or `tool`, `stop_reason` will always be `"tool_use"` (Claude cannot produce a final text-only response)
- Use `any` to prevent the agent from stopping prematurely when you know a tool call is required
- Use `tool` for structured output extraction (e.g., forcing Claude to call a `format_response` tool)
- In agentic loops, `auto` is typically used so the agent can naturally terminate

---

## 4. Security & Reliability

### Guardrails for Agentic Systems

| Risk | Mitigation |
|------|-----------|
| Infinite loops | Max iteration count + timeout |
| Prompt injection via tool results | Validate/sanitize tool outputs |
| Unauthorized tool use | Strict tool allowlists per agent role |
| Data exfiltration | Network egress controls, tool scoping |
| Runaway costs | Token budgets, rate limiting |

### Best Practices
- **Always set max iterations** — no unbounded loops in production
- **Validate tool inputs** — Claude may hallucinate invalid parameters
- **Log all iterations** — full audit trail of agent reasoning
- **Implement circuit breakers** — stop if error rate exceeds threshold
- **Principle of least privilege** — give agents only the tools they need

---

## 5. Exam Scenarios

### Scenario 1
**Q:** You're building an agent that processes customer support tickets. The agent needs to: read the ticket, search the knowledge base, draft a reply, and optionally escalate. What is the MOST appropriate loop structure?

**A:** A bounded agentic loop with tools for `read_ticket`, `search_kb`, `draft_reply`, and `escalate`. Use a Plan-Execute-Verify pattern with max iterations (e.g., 10). The agent reads the ticket, searches for relevant articles, drafts a reply, and decides whether to escalate — all within the loop. The loop terminates when the agent produces a final response or escalates.

**Why not other options:**
- Hardcoded sequential steps = brittle, can't handle edge cases
- Unbounded loop = production risk
- Single API call = insufficient for multi-step reasoning

### Scenario 2
**Q:** An agentic system is making 50+ tool calls per user request, driving up costs. What architectural change BEST addresses this?

**A:** Implement a **budget-aware loop** that tracks cumulative tokens and tool calls. Add a planning step at the start so the agent reasons about the minimum tools needed. Consider batching related operations into composite tools (e.g., `search_and_summarize` instead of separate `search` + `read` + `summarize`).

### Scenario 3
**Q:** Your agent's `stop_reason` is `"max_tokens"` mid-response. What should your loop do?

**A:** Append the partial response to messages and make another API call to continue. This is a continuation scenario, not a termination. However, if this happens repeatedly, increase `max_tokens` or investigate if the agent is generating unnecessarily verbose output.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Unbounded loops | Runaway costs, infinite execution | Always set max iterations |
| Ignoring `stop_reason` | Missing tool calls or truncated output | Explicitly handle all stop reasons |
| Giant system prompts | Wasted tokens every iteration | Use tool descriptions for per-tool instructions |
| No error handling in tool execution | Crashes break the loop | Return error messages as tool results |
| Feeding raw exceptions to Claude | Confusing, may leak internals | Structured error format: `{"error": "...", "type": "..."}` |
| Single monolithic agent | Hard to debug, maintain | Decompose into specialized sub-agents |

---

## 7. Quick Reference / Cheat Sheet

```
AGENTIC LOOP ESSENTIALS
━━━━━━━━━━━━━━━━━━━━━━
• Loop = messages.create() in a while loop
• Continue if stop_reason == "tool_use"
• Exit if stop_reason == "end_turn"
• Always set MAX_ITERATIONS
• Tool results go in "user" role message
• Each tool_result must reference tool_use_id
• Streaming works inside agentic loops
• Extended thinking improves complex decisions
• Budget tokens control thinking cost

KEY API PARAMS
━━━━━━━━━━━━━
• model: which Claude model
• max_tokens: per-response limit
• system: agent instructions
• tools: available tool definitions
• messages: full conversation history
• stop_sequences: custom terminators

STOP_REASON VALUES
━━━━━━━━━━━━━━━━━
• "end_turn" → done
• "tool_use" → execute tools, continue
• "max_tokens" → truncated, may continue
• "stop_sequence" → custom stop hit
```
