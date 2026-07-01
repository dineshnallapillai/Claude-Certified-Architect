# Critical Exam Traps & Frequently Confused Topics

> These are the topics where candidates most often lose marks. Each section contrasts the WRONG assumption with the CORRECT answer.

---

## 1. Permissions: Deny Rules vs Priority

**TRAP:** "Local settings have highest priority, so local allow overrides project deny"
**CORRECT:** Deny rules are CUMULATIVE and ABSOLUTE. A deny at ANY level blocks, regardless of allow at any other level. Priority (local > project > global) only applies to allow/general settings merge — NOT to deny rules. Deny at any level = permanently blocked.

---

## 2. Hooks: How They Receive Data

**TRAP:** "Hooks use environment variables like $TOOL_NAME"
**CORRECT:** Hooks receive data via **STDIN as JSON**. The script must parse JSON from stdin containing fields like `tool_name`, `tool_input`, `session_id`, `transcript_path`.

---

## 3. Plan Mode Toggle

**TRAP:** "Type `/plan` to enter plan mode"
**CORRECT:** Plan mode is toggled with **Shift+Tab** in the interactive TUI. There is no `/plan` slash command. In headless mode, use the `--plan` CLI flag.

---

## 4. HTTP Error Codes

**TRAP:** "HTTP 413 means context too long"
**CORRECT:**
- 400 = Invalid request INCLUDING context too long
- 401 = Invalid API key
- 429 = Rate limited (retry with backoff)
- 529 = Overloaded (Anthropic-specific, retry with exponential backoff + jitter)
There is NO 413 response from the Claude API.

---

## 5. MCP Transport

**TRAP:** "HTTP+SSE is the current transport for remote MCP servers"
**CORRECT:** HTTP+SSE is LEGACY/deprecated. **Streamable HTTP** is the current recommended transport for remote servers. stdio remains recommended for local servers.

---

## 6. Extended Thinking

**TRAP:** "Extended thinking works with any temperature setting" / "Use `budget_tokens` on all models"
**CORRECT:**
- **Opus 4.7/4.8, Fable 5:** Use `thinking: {"type": "adaptive"}` ONLY. `budget_tokens` returns 400. `temperature`/`top_p`/`top_k` are removed entirely (sending them = 400).
- **Opus 4.6, Sonnet 4.6:** Use `thinking: {"type": "adaptive"}` (recommended). `budget_tokens` is deprecated but still functional as transitional escape hatch.
- **Older models (Sonnet 4.5, Opus 4.5 and earlier):** `thinking: {"type": "enabled", "budget_tokens": N}` — temperature must be exactly 1.

Additional key facts:
- Minimum budget (older models only): 1024 tokens, must be < `max_tokens`
- Use `output_config: {"effort": "low"|"medium"|"high"|"xhigh"|"max"}` to control depth
- Billed at OUTPUT token rate
- Redacted thinking blocks can appear (safety classifier) — handle gracefully

---

## 7. tool_choice Values

**TRAP:** "`tool_choice: any` means Claude picks the best tool"
**CORRECT:**

| Value | Behavior |
|-------|----------|
| `auto` (default) | Claude decides whether to use any tool at all |
| `any` | FORCES Claude to call A tool (but Claude picks which) |
| `{"type": "tool", "name": "X"}` | FORCES a specific named tool |

Important: `any` forces tool use — if no tool is relevant, Claude will force-fit one (bad results). Use `auto` unless you specifically need guaranteed tool use.

Additional: `disable_parallel_tool_use: true` can be added to any tool_choice config to force one tool call per response.

---

## 8. Sub-Agent Context Isolation

**TRAP:** "Sub-agents inherit the parent's full context/CLAUDE.md"
**CORRECT:** Sub-agents have ISOLATED context. They do NOT automatically inherit:
- Parent's conversation history
- Parent's CLAUDE.md content
- Parent's system prompt

You must EXPLICITLY pass only the relevant context to each sub-agent. This is a security feature (data minimization) and a cost feature (fewer tokens).

---

## 9. Batch API Facts

**TRAP:** "Batch results come back in order" / "Expired means the item timed out"
**CORRECT:**
- Results are NON-DETERMINISTIC order (use `custom_id` to match)
- Up to 100,000 items per batch
- 50% cost discount vs real-time
- 24-hour SLA
- Statuses: `succeeded` / `errored` / `expired` / `canceled`
- `expired` = never processed within the 24h window (resubmit)
- `errored` = attempted but failed (malformed request)

---

## 10. Prompt Caching Minimums

**TRAP:** "Any content can be cached with `cache_control`"
**CORRECT:**

| Model | Minimum Cacheable Size |
|-------|----------------------|
| Opus 4.8/4.7/4.6/4.5, Haiku 4.5 | 4096 tokens |
| Fable 5, Sonnet 4.6, Haiku 3.5/3 | 2048 tokens |
| Sonnet 4.5/4.1/4/3.7 | 1024 tokens |

Content below these thresholds will NEVER be cached even with `cache_control` set.

Additional caching facts:
- TTL: 5 minutes (default) or 1 hour (`"cache_control": {"type": "ephemeral", "ttl": "1h"}`)
- Read discount: 90% off input cost
- Write premium: 25% (5-min TTL) or 100% (1-hour TTL)
- Maximum 4 cache breakpoints per request
- Static content FIRST, dynamic content LAST (prefix matching)
- Caching is a PREFIX match — any byte change invalidates everything after it

---

## 11. Token Economics

**TRAP:** "1 token = 1 character" or "1 token = 1 word"
**CORRECT:**
- ~4 characters per token (English)
- ~0.75 words per token
- 2,000 characters ≈ 500 tokens (NOT 2,000)
- Tool definitions consume input tokens on EVERY call
- In agentic loops, ALL message history (including tool results) sent with EVERY API call
- A 30K token tool result x 10 iterations = 300K input tokens billed

---

## 12. CLAUDE.md Hierarchy

**TRAP:** "Subdirectory CLAUDE.md overrides root CLAUDE.md" or "Only one loads"
**CORRECT:** CLAUDE.md files are ADDITIVE. All applicable files load:
```
~/.claude/CLAUDE.md           (user-level, always)
project/CLAUDE.md             (project root, always)
project/src/CLAUDE.md         (only when working in src/)
```
- `#import ./path/file.md` for modular includes
- Survives context compaction (treated as system prompt)
- Subdirectory files ACTIVATE when Claude operates on files in that subtree

---

## 13. PostToolUse vs PreToolUse Hooks

**TRAP:** "PostToolUse can block/prevent tool execution"
**CORRECT:**

| Hook | Timing | Can Block? | Use For |
|------|--------|-----------|---------|
| PreToolUse | Before execution | YES (exit 2 = block) | Security gates, validation |
| PostToolUse | After execution | NO (tool already ran) | Formatting, logging, post-processing |
| Stop | After agent finishes | N/A | Linting, cleanup, notifications |

Only PreToolUse can PREVENT execution. PostToolUse sees the result but cannot undo it.

---

## 14. Streaming Events

**TRAP:** Confusing the event sequence or when to parse tool JSON
**CORRECT sequence:**
```
message_start → content_block_start → content_block_delta (repeated) → content_block_stop → message_delta → message_stop
```

Critical rule: For tool_use blocks, accumulate `input_json_delta` strings across all deltas. ONLY parse the complete JSON after `content_block_stop`. Attempting to parse partial JSON will fail.

---

## 15. API Authentication

**TRAP:** "Use `Authorization: Bearer sk-ant-...`" (OpenAI pattern)
**CORRECT:** Anthropic uses `x-api-key` header:
```
x-api-key: sk-ant-...
anthropic-version: 2023-06-01
content-type: application/json
```
NOT the `Authorization: Bearer` pattern. This is one of the most common API integration errors.

---

## 16. tool_result Message Role

**TRAP:** "tool_result goes in a system or assistant message"
**CORRECT:** Tool results are sent in the **user** role, each with a matching `tool_use_id`:
```
user message → assistant (with tool_use) → user (with tool_result) → assistant continues
```
If Claude makes 3 parallel tool calls, return ONE user message with 3 tool_result blocks (not 3 separate messages).

---

## 17. Context Compaction Survival

**What SURVIVES:**
1. System prompt / CLAUDE.md (always preserved)
2. Recent messages (recency bias)
3. Tool results from recent actions
4. Key decisions/conclusions

**What is LOST:**
- Early conversation details (summarized)
- Intermediate reasoning (dropped)
- Verbatim tool outputs from many turns ago
- Specific Q&A from early in conversation

**Key insight:** Critical information that must ALWAYS apply should be in system prompt or CLAUDE.md, NOT buried in message history.

---

## 18. `is_error: true` Semantics

**TRAP:** "Including 'error' in the text content is enough"
**CORRECT:** Without `is_error: true`, error text in content is treated as SUCCESSFUL DATA. Claude may interpret "Error: User not found" as the user's status field value. Always set `is_error: true` for failures — it explicitly signals to Claude's reasoning that the operation failed.

---

## 19. Settings Files and Git

| File | Git Status | Scope |
|------|-----------|-------|
| `.claude/settings.json` | Committed | Team-wide |
| `.claude/settings.local.json` | NOT committed (.gitignore) | Personal |
| `~/.claude/settings.json` | User's home | All projects |
| `CLAUDE.md` | Committed | Team-wide instructions |

**TRAP:** Putting secrets in `.claude/settings.json` (committed!). Use env var references: `"${MY_SECRET}"`.

---

## 20. Count Tokens API vs Estimation

**TRAP:** "Just divide characters by 4"
**CORRECT:** Use the `/v1/messages/count_tokens` endpoint for EXACT pre-flight counts. It accepts the same messages/model/tools params as a real request but returns only the token count — no response generated, no output tokens billed. Essential for capacity planning in batch pipelines.
