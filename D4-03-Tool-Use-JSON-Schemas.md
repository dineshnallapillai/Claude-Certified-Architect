# Domain 4 | Topic 4.3: Tool Use with JSON Schemas

## 1. Core Concepts

### How Tool Use Works End-to-End

```
1. DEFINE: You specify tools with JSON Schema input definitions
2. SEND: Include tool definitions in API call
3. DECIDE: Claude decides which tool (if any) to use
4. REQUEST: Claude outputs a tool_use content block
5. EXECUTE: Your code runs the tool
6. RETURN: You send tool_result back to Claude
7. RESPOND: Claude uses result to formulate response
```

### Tool Definition Structure

```json
{
  "name": "tool_name",           // snake_case identifier
  "description": "When/how...",  // Usage guidance
  "input_schema": {              // JSON Schema for parameters
    "type": "object",
    "properties": {...},
    "required": [...]
  }
}
```

### Message Flow with Tools

```python
# Request with tools
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=[
        {
            "name": "get_weather",
            "description": "Get current weather for a location",
            "input_schema": {
                "type": "object",
                "properties": {
                    "location": {"type": "string", "description": "City, State"},
                    "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
                },
                "required": ["location"]
            }
        }
    ],
    messages=[{"role": "user", "content": "What's the weather in NYC?"}]
)

# Response contains tool_use block:
# content: [
#   {"type": "text", "text": "Let me check the weather..."},
#   {"type": "tool_use", "id": "toolu_abc", "name": "get_weather", 
#    "input": {"location": "New York, NY", "unit": "fahrenheit"}}
# ]
```

### Tool Result Format

```python
# Feed result back:
messages.append({"role": "assistant", "content": response.content})
messages.append({
    "role": "user",
    "content": [{
        "type": "tool_result",
        "tool_use_id": "toolu_abc",  # Must match the tool_use id
        "content": "72°F, Partly Cloudy, Humidity 45%"
    }]
})

# Claude then produces final response using the result
```

---

## 2. Design Patterns

### Pattern 1: Forced Tool Use

```python
# Force Claude to use a specific tool (no choice)
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=[...],
    tool_choice={"type": "tool", "name": "get_weather"},  # MUST use this tool
    messages=[...]
)

# Or force ANY tool use (Claude picks which):
tool_choice={"type": "any"}

# Or let Claude decide (default):
tool_choice={"type": "auto"}
```

### Pattern 2: Multiple Tools in One Response

```python
# Claude can use multiple tools at once
# Response might contain:
content = [
    {"type": "text", "text": "I'll look up both..."},
    {"type": "tool_use", "id": "t1", "name": "get_weather", "input": {"location": "NYC"}},
    {"type": "tool_use", "id": "t2", "name": "get_weather", "input": {"location": "LA"}}
]

# Return all results in one message:
tool_results = [
    {"type": "tool_result", "tool_use_id": "t1", "content": "72°F Sunny"},
    {"type": "tool_result", "tool_use_id": "t2", "content": "85°F Clear"}
]
messages.append({"role": "user", "content": tool_results})
```

> **Note:** You can set `disable_parallel_tool_use: true` within the `tool_choice` object to prevent Claude from making multiple tool calls in a single response. This forces sequential, one-at-a-time tool use — useful when tool calls have ordering dependencies or when your system cannot handle concurrent tool executions.
>
> ```python
> tool_choice={"type": "auto", "disable_parallel_tool_use": True}
> ```

### Pattern 3: Tool with Complex Nested Schema

```json
{
  "name": "create_ticket",
  "description": "Create a Jira ticket with structured metadata",
  "input_schema": {
    "type": "object",
    "properties": {
      "title": {"type": "string", "maxLength": 200},
      "description": {"type": "string"},
      "type": {"type": "string", "enum": ["bug", "feature", "task"]},
      "priority": {"type": "string", "enum": ["critical", "high", "medium", "low"]},
      "assignee": {"type": "string", "description": "Username of assignee"},
      "labels": {
        "type": "array",
        "items": {"type": "string"},
        "maxItems": 5
      },
      "metadata": {
        "type": "object",
        "properties": {
          "sprint": {"type": "string"},
          "story_points": {"type": "integer", "minimum": 1, "maximum": 21},
          "affected_services": {
            "type": "array",
            "items": {"type": "string"}
          }
        }
      }
    },
    "required": ["title", "type", "priority"]
  }
}
```

### Pattern 4: Tool Use in Streaming

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    messages=messages
) as stream:
    for event in stream:
        if event.type == "content_block_start":
            if event.content_block.type == "tool_use":
                print(f"Starting tool: {event.content_block.name}")
        elif event.type == "content_block_delta":
            if event.delta.type == "input_json_delta":
                # Streaming tool input JSON
                print(event.delta.partial_json)
    
    final = stream.get_final_message()
```

---

## 3. Implementation Details

### Tool Use Response Structure

```json
{
  "id": "msg_123",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "I'll search for that information."
    },
    {
      "type": "tool_use",
      "id": "toolu_abc123",
      "name": "search_database",
      "input": {
        "query": "recent orders",
        "limit": 10
      }
    }
  ],
  "stop_reason": "tool_use"
}
```

### Key Fields

| Field | Purpose |
|-------|---------|
| `stop_reason: "tool_use"` | Signals Claude wants to use a tool |
| `content[].type: "tool_use"` | The tool call block |
| `content[].id` | Unique ID to match with tool_result |
| `content[].name` | Which tool to execute |
| `content[].input` | Parameters (validated against schema) |

### Error in Tool Results

```python
# Successful result
{"type": "tool_result", "tool_use_id": "t1", "content": "Success data here"}

# Error result (Claude knows it failed)
{"type": "tool_result", "tool_use_id": "t1", "content": "Error: not found", "is_error": True}

# Rich content in results (images, etc.)
{"type": "tool_result", "tool_use_id": "t1", "content": [
    {"type": "text", "text": "Here's what I found:"},
    {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": "..."}}
]}
```

### Tool_choice Options

| Option | Behavior |
|--------|----------|
| `{"type": "auto"}` | Claude decides whether/which tool (default) |
| `{"type": "any"}` | Claude MUST use a tool (picks which) |
| `{"type": "tool", "name": "X"}` | Claude MUST use tool X specifically |

---

## 4. Security & Reliability

### Tool Use Security Checklist

- Validate tool inputs server-side (don't trust schema alone)
- Sanitize tool results before returning to Claude
- Set `is_error: true` for failures (prevents misinterpretation)
- Never include secrets in tool results
- Rate limit tool executions
- Log all tool calls for audit

### Common Pitfalls

| Pitfall | Impact | Fix |
|---------|--------|-----|
| Missing `tool_use_id` in result | API error | Always match IDs |
| Wrong message ordering | Broken conversation | tool_result must follow assistant tool_use |
| Forgetting to handle multiple tool calls | Only first tool executed | Loop through all tool_use blocks |
| Not handling `max_tokens` stop | Incomplete tool call | Check stop_reason, handle truncation |

---

## 5. Exam Scenarios

### Scenario 1
**Q:** Claude's response has `stop_reason: "tool_use"` with two tool_use blocks. Your code only executes the first. What happens?

**A:** The second tool call is lost. You must:
1. Loop through ALL `content` blocks
2. Execute every block with `type == "tool_use"`
3. Return ALL results in the next message

Missing a tool result causes Claude to lose context about what it requested.

### Scenario 2
**Q:** You want Claude to always extract structured data without sometimes responding conversationally. How do you ensure tool use?

**A:** Use `tool_choice: {"type": "tool", "name": "extract_data"}` to force Claude to use the extraction tool. This prevents Claude from deciding to respond with text instead.

### Scenario 3
**Q:** Claude calls a tool with input that doesn't match your schema (e.g., string where integer expected). What is the CORRECT handling?

**A:** Return a structured error as the tool result with `is_error: true`:
```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_abc",
  "content": "{\"error\": \"validation\", \"message\": \"'limit' must be integer, got string '10'\", \"suggestion\": \"Provide limit as a number, not string\"}",
  "is_error": true
}
```
Claude will self-correct and retry with proper input.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Not handling all tool_use blocks | Lost calls | Loop through entire content array |
| Ignoring `stop_reason` | Miss tool requests | Check stop_reason every response |
| Returning raw exceptions as results | Confusing, insecure | Structured error format |
| Missing `tool_use_id` matching | API validation fails | Always propagate the ID |
| Using `tool_choice: "any"` always | Claude can't say "no tool needed" | Use "auto" unless forcing is needed |
| Huge tool results (100KB+) | Context overflow | Truncate/summarize large results |

---

## 7. Quick Reference / Cheat Sheet

```
API FLOW
━━━━━━━━
1. messages.create(tools=[...], messages=[...])
2. Check response.stop_reason
3. If "tool_use": execute tools, send results
4. If "end_turn": done, show response

MESSAGE ORDERING
━━━━━━━━━━━━━━━━
user → assistant(tool_use) → user(tool_result) → assistant(response)
                                                 └─ or more tool_use

TOOL_CHOICE OPTIONS
━━━━━━━━━━━━━━━━━━━
"auto" → Claude decides (default)
"any"  → Must use some tool
{"type":"tool","name":"X"} → Must use tool X

TOOL RESULT FORMAT
━━━━━━━━━━━━━━━━━━
{
  "type": "tool_result",
  "tool_use_id": "<matching_id>",  ← REQUIRED
  "content": "<string or array>",
  "is_error": true/false           ← signals failure
}

STOP_REASON MEANINGS
━━━━━━━━━━━━━━━━━━━━
"end_turn"  → Done, show response
"tool_use"  → Execute tools, continue loop
"max_tokens"→ Truncated, may need continuation
```
