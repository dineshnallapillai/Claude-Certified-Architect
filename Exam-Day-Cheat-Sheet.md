# Claude Certified Architect — Exam Day Cheat Sheet

> Last-minute review of the most testable facts. Read this the night before and morning of the exam.
> **Exam:** 60 questions, 120 minutes, 720/1000 to pass, 4 of 6 scenarios, no guessing penalty.

---

## Domain 1: Agentic Architecture (27% weight — highest)

### Agentic Loop
```
User Input → Claude (reason) → tool_use → Execute Tool → tool_result → Claude (reason) → ... → end_turn
```
- **Stop reasons:** `end_turn`, `tool_use`, `max_tokens`, `stop_sequence`, `pause_turn`, `refusal`
- Loop continues while `stop_reason == "tool_use"`
- `tool_result` must include matching `tool_use_id`
- `tool_result` goes in the **user** role (NOT assistant or system)
- Multiple parallel tool results → ONE user message with multiple tool_result blocks
- `stop_reason` is the ONLY reliable loop control signal — NEVER parse text for "I'm done"

### tool_choice Parameter
| Value | Behavior |
|-------|----------|
| `auto` (default) | Claude decides whether to use tools |
| `any` | Forces Claude to use A tool (Claude picks which — but MUST use one) |
| `{"type": "tool", "name": "X"}` | Forces a specific named tool (100% structural compliance) |
| `none` | Claude cannot use tools |
| `disable_parallel_tool_use: true` | Prevents multiple tools in one response (add to any above) |

- `any` ≠ "Claude picks best tool" — it FORCES tool use even if no tool is appropriate
- Forced specific tool = guaranteed JSON output matching schema (use for structured output)

### Multi-Agent Topologies
| Topology | Use When | Communication |
|----------|----------|---------------|
| **Hub-and-spoke** | Independent tasks, parallel execution + aggregation | All routes through coordinator |
| **Pipeline** | Sequential stages, each transforms output for next | Linear, A→B→C |
| **Parallel (voting)** | Same task, different perspectives, consensus | Independent, results merged |
| **Hierarchical** | Complex decomposition, specialist teams | Manager → team leads → workers |

- **Context isolation:** Sub-agents inherit NOTHING — no conversation, no CLAUDE.md, no tools, no state
- Coordinator must EXPLICITLY serialize all needed context into subagent prompt
- **Model tiering:** Haiku=routing/classification, Sonnet=general work, Opus=complex reasoning
- Sub-agents CANNOT communicate directly with each other (hub-and-spoke = coordinator mediates)

### Hooks (Critical - frequently tested)
- Hooks receive data via **STDIN as JSON** (NOT environment variables)
- JSON contains: `tool_name`, `tool_input`, `session_id`, `transcript_path`
- **PreToolUse**: exit 0=allow, exit 2=block, stdout=feedback to Claude
- **PostToolUse**: runs after tool execution, CANNOT block (tool already ran)
- **Stop**: runs after agent finishes — for linting, cleanup, notifications
- **Notification**: when Claude sends notification — custom routing
- Hooks are **synchronous** — agent waits for completion
- Enforcement strength: **Hooks > Settings > CLAUDE.md > Prompt instructions**

### Error Handling
| HTTP Code | Meaning | Action | Trap |
|-----------|---------|--------|------|
| 400 | Invalid request / context too long | Fix request, don't retry | Context overflow = 400, NOT 413 |
| 401 | Invalid API key | Fix credentials | — |
| 403 | Permission denied | Check access | — |
| 413 | Payload too large (raw HTTP size) | Reduce payload | NOT token overflow |
| 429 | Rate limited | Retry with exponential backoff | — |
| 500 | Internal server error | Retry cautiously | — |
| 529 | Overloaded (Anthropic-specific) | Retry with backoff + jitter | Unique to Anthropic |

- Circuit breaker: track failures → open circuit → periodic test requests → close on recovery
- `is_error: true` in tool_result tells Claude the tool failed (triggers self-correction)
- WITHOUT `is_error: true`, error text may be treated as successful data ("Error: not found" → interpreted as content)

### Escalation Design
- Escalate on: policy gaps, exception required, customer requests human, agent cannot progress
- DO NOT escalate based on: sentiment/frustration, self-reported confidence scores
- Explicit criteria with few-shot examples > confidence thresholds > sentiment analysis
- Sentiment ≠ complexity (angry customer may have trivial issue; calm customer may have complex one)

### Task Decomposition Diagnosis
When multi-agent output is incomplete:
```
Check Layer 1 FIRST: Did coordinator decompose correctly?
  → Layer 2: Did each subagent receive correct context?
    → Layer 3: Did subagents execute correctly?
      → Layer 4: Was synthesis/combination correct?
```
Blame decomposition before blaming execution.

---

## Domain 2: Tool Design & MCP (18% weight)

### Tool Definition Anatomy
```json
{
  "name": "verb_noun",
  "description": "WHAT it does + WHEN to use + WHEN NOT to use + what it RETURNS + example inputs",
  "input_schema": {
    "type": "object",
    "properties": {...},
    "required": [...],
    "additionalProperties": false
  }
}
```
- Names: `snake_case`, verb_noun pattern (`get_customer`, `process_refund`)
- Descriptions are the PRIMARY mechanism for tool selection — fix these FIRST when selection fails
- `additionalProperties: false` recommended for all schemas
- Boundary clauses ("Use this INSTEAD OF X when...") prevent tool confusion

### Tool Scoping & Degradation
| Tools per Agent | Routing Accuracy | Action |
|---|---|---|
| 1-5 | ~95%+ (sweet spot) | Ideal for specialist agents |
| 6-10 | ~85% | Acceptable with excellent descriptions |
| 11-15 | ~70% | Split into specialists |
| 16+ | ~60% (degradation cliff) | Architectural problem — restructure |

- Accuracy drops ~8% per tool past 5
- Fix priority: reduce tool count > improve descriptions > add examples > add router

### 80/20 Tool Allocation Pattern
- 85% of cases are simple → give agent a scoped lightweight tool
- 15% of cases are complex → route through coordinator to specialist
- Balance: latency reduction for common case + proper handling for rare case

### Tool Annotations (MCP)
| Annotation | Default | Meaning |
|------------|---------|---------|
| `readOnlyHint` | false | Tool doesn't modify state |
| `destructiveHint` | true | Tool may delete/destroy data |
| `idempotentHint` | false | Safe to call multiple times |
| `openWorldHint` | true | Interacts with external systems |

### MCP Architecture
```
Host (Claude Desktop/Code) → Client (1:1 per server) → Server → [Tools, Resources, Prompts]
```
- Client-Server relationship is 1:1 (one client per server)
- Host can have MULTIPLE clients (connecting to different servers)

### MCP Primitives — Who Controls What (CRITICAL)
| Primitive | Controlled By | Purpose | Example |
|-----------|---------------|---------|---------|
| **Tools** | Model (Claude decides) | Actions Claude can invoke | `get_weather`, `send_email` |
| **Resources** | Application (your code) | Data the app can read | `file://config.json`, database records |
| **Prompts** | User (explicitly selected) | Pre-built templates | `/summarize`, `/translate` |

**Trap:** Confusing control. Tools = model-initiated. Resources = app-initiated. Prompts = user-initiated.

### MCP Transports
| Transport | Use Case | Status |
|-----------|----------|--------|
| **stdio** | Local servers, same machine | Recommended for local |
| **Streamable HTTP** | Remote servers, single endpoint | **Recommended for remote (current)** |
| HTTP+SSE | Remote servers | **Legacy (deprecated)** |

### MCP Initialization Handshake
```
Client → initialize (protocol version, capabilities) → Server
Server → response (server capabilities, protocol version) → Client
Client → notifications/initialized → Server
```
- Three-step handshake BEFORE any tool/resource calls
- Client sends protocol version + capabilities it supports
- Server responds with its capabilities
- Client confirms with initialized notification

### MCP Notifications
- `notifications/progress` — progress reporting (progressToken, progress, total)
- `notifications/message` — log messages (level, logger, data)
- `notifications/tools/list_changed` — dynamic tool availability changed (client should re-fetch)
- `notifications/resources/updated` — resource content changed

### Structured Error Responses
```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_xxx",
  "is_error": true,
  "content": [{"type": "text", "text": "{\"error_code\": \"TIMEOUT\", \"message\": \"Device unreachable\", \"retryable\": true, \"suggested_action\": \"check_connectivity\"}"}]
}
```
- `is_error: true` — REQUIRED for failures (without it, error text = successful data)
- Include: error type, what was attempted, retryable flag, suggested recovery
- Return partial results when available (don't throw away progress)
- NEVER expose stack traces, internal paths, or credentials
- Structured errors enable intelligent coordinator recovery decisions

---

## Domain 3: Claude Code Configuration

### CLAUDE.md Hierarchy (loaded additively)
```
~/.claude/CLAUDE.md          (user-level, personal prefs)
  └── project/CLAUDE.md      (project root, shared standards)
       └── project/src/CLAUDE.md  (subdirectory, activates per file path)
```
- `@path/to/file.md` — modular includes (just `@` + path, no keyword)
- Survives context compaction (treated as system prompt)
- Subdirectory CLAUDE.md activates when Claude operates on files in that subtree

### Plan Mode
- **Toggle:** `Shift+Tab` in interactive TUI (NOT `/plan`)
- **Headless:** `--plan` CLI flag
- Plan mode: can read files, CANNOT write/edit/run modifying commands

### Permissions (CRITICAL)
- **Deny rules are CUMULATIVE** — deny at ANY level blocks, regardless of allow elsewhere
- Local allow CANNOT override project deny (security-first design)
- For hard enforcement beyond allow/deny → use hooks

### Settings Priority (highest to lowest for allow/general settings)
1. Enterprise policy (absolute override)
2. Local settings (`.claude/settings.local.json`) — personal, not committed
3. Project settings (`.claude/settings.json`) — team-shared, git-tracked
4. User settings (`~/.claude/settings.json`) — global personal defaults

**BUT: Deny rules are CUMULATIVE regardless of level — deny anywhere = blocked everywhere**

### CLI Flags (most testable)
| Flag | Purpose |
|------|---------|
| `-p` / `--print` | Single-shot mode, stdout output |
| `--continue` | Resume most recent conversation |
| `--resume <id>` | Resume specific session by ID |
| `--max-budget-usd <amount>` | Cap API spending for the invocation |
| `--output-format json\|text\|stream-json` | Control print mode output |
| `--dangerously-skip-permissions` | Skip ALL permission checks (CI only!) |
| `--model <id>` | Override model for session |
| `--allowedTools` | Allow specific tools without prompting |

### Custom Commands
- Location: `.claude/commands/` directory
- Naming: `commands/db/migrate.md` → `/db:migrate`
- `$ARGUMENTS` placeholder for user input

### Built-in Skills
- `/code-review` — review working diff for bugs
- `/review` — review GitHub PR (remote)
- `/deep-research` — multi-source research report
- `/verify` — run app, confirm change works
- `/simplify` — reuse/efficiency cleanups
- `/security-review` — security audit of changes

---

## Domain 4: Prompt Engineering

### System Prompt Best Practices
- Static content FIRST (cacheable), dynamic content LAST
- Structure: Identity → Capabilities → Constraints → Context → Format → Tools
- Caching: 90% read discount, 25% write premium (5-min TTL), 5-min or 1-hour TTL, min varies by model (1024–4096 tokens)

### Prompt Caching Requirements
| Requirement | Value |
|-------------|-------|
| Minimum cacheable size (Opus 4.8/4.7/4.6/4.5, Haiku 4.5) | 4096 tokens |
| Minimum cacheable size (Fable 5, Sonnet 4.6, Haiku 3.5/3) | 2048 tokens |
| Minimum cacheable size (Sonnet 4.5/4.1/4/3.7) | 1024 tokens |
| TTL | 5 minutes (default) or 1 hour (`"ttl": "1h"`) |
| Read discount | 90% off input price |
| Write premium | 25% (5-min TTL) or 100% (1-hour TTL) |
| Breakpoint | `"cache_control": {"type": "ephemeral"}` |
| Max breakpoints per request | 4 |

### tool_choice + Structured Output
- Force specific tool with `tool_choice` → guaranteed JSON output matching schema
- Better than asking Claude to output JSON in text (no parsing needed)

### Validation-Retry Pattern
```
Generate → Validate (cheapest first) → Feedback with specific error → Retry (max 3)
```
- Escalate specificity: generic feedback → specific line/error → worked example
- Always bound retries (prevent infinite loops)

### Batch API
- Up to 100,000 items per batch
- 50% cost discount vs real-time
- 24-hour SLA (not for latency-sensitive work)
- Results: `succeeded` / `errored` / `expired`

---

## Domain 5: Context Management

### What Survives Compaction (priority order)
1. System prompt / CLAUDE.md (always preserved)
2. Recent messages (recency bias)
3. Tool results from recent actions
4. Decisions and conclusions
5. ~~Early conversation details~~ (summarized/lost)
6. ~~Intermediate reasoning~~ (dropped)

### Lost-in-the-Middle
- Strongest recall: **beginning** (system prompt) and **end** (recent)
- Weakest recall: **middle** of long contexts
- Strategy: critical info at start/end, reference material in middle

### Context Window Limits
| Model | Max Context | Max Output |
|-------|-------------|------------|
| Fable 5, Opus 4.8/4.7/4.6, Sonnet 5, Sonnet 4.6 | 1M tokens | 128K |
| Haiku 4.5 | 200K tokens | 64K |

Stream for any `max_tokens` above ~16K to avoid SDK HTTP timeouts.

### Prompt Caching Structure (cache-friendly)
```
[System prompt - CACHED] → [Tools - CACHED] → [Few-shot examples - CACHED] → [Dynamic user message]
```
Static/repeated content at the TOP, dynamic content at the BOTTOM.

### Token Economics
- 1 token ≈ 4 characters (English), ≈ 0.75 words
- Extended thinking tokens billed at OUTPUT token rate
- Every tool definition consumes input tokens (minimize unused tools)

---

## Streaming & Extended Thinking

### Streaming Event Sequence
```
message_start → content_block_start → content_block_delta (repeated) → content_block_stop → message_delta → message_stop
```
- `content_block_delta` types: `text_delta`, `input_json_delta`, `thinking_delta`
- Tool use: accumulate `input_json_delta` strings, parse complete JSON at `content_block_stop`

### Extended Thinking
- **Newer models (Opus 4.6+, Sonnet 4.6):** `thinking: {"type": "adaptive"}` — Claude decides dynamically
- **Opus 4.7/4.8, Fable 5, Sonnet 5:** `budget_tokens` returns 400 — adaptive only; `temperature`/`top_p`/`top_k` also removed
- **Older models (Sonnet 4.5, Opus 4.5 and earlier):** `thinking: {"type": "enabled", "budget_tokens": N}` (min 1024, must be < `max_tokens`)
- Control depth with `output_config: {"effort": "low"|"medium"|"high"|"xhigh"|"max"}`
- Billing: thinking tokens = output token rate
- Redacted thinking: may appear for safety reasons (opaque to developer)

---

## API Quick Reference

### Required Headers
```
x-api-key: sk-ant-...
anthropic-version: 2023-06-01
content-type: application/json
anthropic-beta: feature-1,feature-2  (optional, for beta features)
```

### Model IDs (use aliases — no date suffixes needed)
| Model | Alias (use this) | Context Window | Max Output | Best For |
|-------|------------------|----------------|------------|----------|
| Fable 5 | `claude-fable-5` | 1M | 128K | Most demanding reasoning |
| Opus 4.8 | `claude-opus-4-8` | 1M | 128K | Complex reasoning, agentic |
| Sonnet 5 | `claude-sonnet-5` | 1M | 128K | Balanced speed + capability |
| Sonnet 4.6 | `claude-sonnet-4-6` | 1M | 128K | Balanced (previous gen) |
| Haiku 4.5 | `claude-haiku-4-5` | 200K | 64K | Fast, cheap, simple tasks |

### Usage Response Object
```json
"usage": {
  "input_tokens": 1234,
  "output_tokens": 567,
  "cache_creation_input_tokens": 800,
  "cache_read_input_tokens": 1000
}
```

### Count Tokens API (Pre-flight estimation)
- Endpoint: `POST /v1/messages/count_tokens`
- Same params as Messages (model, messages, system, tools)
- Returns exact input token count WITHOUT generating a response
- No output tokens billed — use for cost estimation before committing

### Multi-Modal Content Blocks
```json
// Image (base64)
{"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": "..."}}
// Image (URL)
{"type": "image", "source": {"type": "url", "url": "https://..."}}
// PDF document
{"type": "document", "source": {"type": "base64", "media_type": "application/pdf", "data": "..."}}
```

### Batch API Quick Facts
- Max: 100,000 items per batch
- Discount: 50% off real-time pricing
- SLA: 24 hours
- Results: `succeeded` / `errored` / `expired` / `canceled`
- Each item can have DIFFERENT model/params (mixed batch OK)
- Use `custom_id` to match results to requests (order not guaranteed)

---

## Top 10 "Trap" Answers to Watch For

1. Hooks use environment variables → **WRONG** (they use STDIN JSON)
2. `/plan` enters plan mode → **WRONG** (it's `Shift+Tab`)
3. Local allow overrides project deny → **WRONG** (deny is cumulative/absolute)
4. HTTP 413 for context too long → **WRONG** (it's HTTP 400)
5. HTTP+SSE is current MCP transport → **WRONG** (Streamable HTTP is current)
6. Extended thinking works with temperature 0.5 → **WRONG** (on older models must be 1; on Opus 4.7+/Fable 5 `temperature` is removed entirely)
7. Sub-agents inherit parent's full context → **WRONG** (explicit isolation)
8. CLAUDE.md instructions can be overridden by user messages → Mostly **WRONG** (system-level)
9. Batch API guarantees response order → **WRONG** (non-deterministic order)
10. `tool_choice: "any"` means Claude picks the best tool → **WRONG** (it forces ANY tool use, not selection)

---

*Good luck on the exam!*
