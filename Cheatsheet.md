# Claude Certified Architect — Foundations (CCAR-F) Last-Minute Cheat Sheet

> **Exam:** 60 MCQs | 120 min | Pass: 720/1000 | 5 Domains

---

# 1. CLI Commands & Flags

| Flag | Remember | Trap |
|------|----------|------|
| `claude` | Interactive mode (default) | Requires TTY; fails in CI without `-p` |
| `claude -p` / `--print` | Non-interactive; reads stdin, prints to stdout, exits | No session persistence by default |
| `--output-format text` | Plain text output (default for `-p`) | No metadata, no session_id returned |
| `--output-format json` | JSON result envelope with session_id, cost, tokens | NOT model output schema — it's the CLI wrapper |
| `--output-format stream-json` | Streaming JSON events (real-time) | For CI pipelines that need progressive output |
| `--input-format text` | Default stdin interpretation | — |
| `--input-format stream-json` | Programmatic multi-turn input | For building custom UIs or automation |
| `--json-schema <schema>` | Constrains MODEL output to match schema | Different from `--output-format json` |
| `--resume <id>` / `-r` | Resume exact session by ID | Fails if session doesn't exist |
| `--resume` (no arg) | Interactive session picker | Not available in `-p` mode |
| `--continue` / `-c` | Continue most recent relevant session | Non-deterministic if multiple sessions exist |
| `--session-id <uuid>` | Use specific session ID | Creates new if doesn't exist |
| `--fork-session` | Branch into new session ID when resuming | Original session unchanged |
| `--name` / `-n` | Display name for session | Cosmetic only, not for selection |
| `--model <model>` | Model selection | Accepts aliases: opus, sonnet, haiku, fable |
| `--effort <level>` | Reasoning effort: low/medium/high/xhigh/max | Affects quality AND cost |
| `--fallback-model` | Automatic fallback if primary fails | Chain of fallbacks |
| `--allowedTools` | Tool allowlist for session | Restricts available tools |
| `--disallowedTools` | Tool denylist for session | Takes precedence over allow |
| `--permission-mode <mode>` | acceptEdits/auto/bypassPermissions/manual/dontAsk/plan | Controls approval behavior |
| `--dangerously-skip-permissions` | Bypass ALL permission checks | **DANGEROUS** — no safety net |
| `--max-budget-usd` | Maximum spend limit for session | Hard cap; session stops |
| `--system-prompt` | REPLACE system prompt entirely | Overrides CLAUDE.md + defaults |
| `--append-system-prompt` | APPEND to default system prompt | CLAUDE.md + defaults preserved |
| `--mcp-config <path>` | Load MCP server configuration | JSON file with server definitions |
| `--strict-mcp-config` | Only use explicitly specified MCP servers | Ignores project/user MCP configs |
| `--bg` / `--background` | Start as background agent | Non-blocking; runs asynchronously |
| `--bare` | Skip hooks, LSP, plugins, auto-memory | Minimal execution environment |
| `--safe-mode` | Disable all customizations | For troubleshooting |
| `--worktree` / `-w` | Create git worktree for session | Isolated file changes |
| `--add-dir <dirs>` | Additional directories for tool access | Expands filesystem scope |
| `--autocompact` | Auto-compact context when approaching limit | Prevents context overflow |
| `--agent <agent>` | Select agent type for session | Must be defined agent |
| `--agents <json>` | Define custom agents inline | JSON agent definitions |
| `--max-turns` | Maximum conversation turns | Limits agentic loop iterations |
| `--verbose` | Verbose output | Debugging |
| `--debug` / `-d` | Debug mode with optional filter | Low-level diagnostics |

**Key subcommands:**

| Subcommand | Purpose |
|-----------|---------|
| `claude mcp add/remove/list` | Manage MCP servers |
| `claude doctor` | Health check / diagnostics |
| `claude auth login/logout/status` | Authentication management |
| `claude agents` | Background agent management |
| `claude update` | Check for / install updates |

---

# 2. Critical Distinctions

## `--resume` vs `--continue`

| | `--resume <id>` | `--continue` |
|---|---|---|
| Selection | Exact session by ID | Most recent relevant |
| Deterministic | YES | NO — risky with multiple sessions |
| CI-safe | YES | NO |
| Fails if | Session doesn't exist | No recent session found |

**Memory rule → If exam says "CI pipeline needs exact previous context" → `--resume <id>`**

## Interactive vs `-p`

| | Interactive | `-p` (print/non-interactive) |
|---|---|---|
| TTY required | YES | NO |
| Stdin | User typing | Pipe/redirect |
| Stdout | TUI interface | Clean text/JSON output |
| Session persistence | YES (default) | NO (default) |
| Use case | Developer workflow | CI/CD, scripts, automation |

**Memory rule → If exam says "headless" or "CI" or "script" → `-p`**

## Output formats

| Format | Returns | Use case |
|--------|---------|----------|
| `text` | Raw model text only | Simple scripts |
| `json` | `{session_id, cost, tokens, result, ...}` envelope | Capture metadata, chain steps |
| `stream-json` | Progressive JSON events | Real-time UIs, streaming pipelines |

**Memory rule → `json` = CLI metadata envelope; `--json-schema` = model output constraint**

## `--output-format json` vs `--json-schema`

| | `--output-format json` | `--json-schema` |
|---|---|---|
| What it controls | CLI output wrapper | Model response structure |
| Constrains model | NO | YES |
| Returns | Envelope: session_id, cost, result | Model output matching your schema |
| Purpose | Pipeline metadata | Structured extraction/generation |

**Memory rule → "format json" wraps the outside; "json-schema" shapes the inside**

## `--system-prompt` vs `--append-system-prompt`

| | `--system-prompt` | `--append-system-prompt` |
|---|---|---|
| Effect | REPLACES everything | APPENDS to existing |
| CLAUDE.md | Lost/overridden | Preserved |
| Default behavior | Gone | Preserved |
| Use case | Full control (testing) | Add CI-specific instructions |

**Memory rule → If exam says "preserve project instructions AND add CI context" → `--append-system-prompt`**

## Permission modes

| Mode | Behavior |
|------|----------|
| `manual` | Ask for every tool use |
| `acceptEdits` | Auto-approve reads; ask for writes |
| `auto` | Auto-approve most safe operations |
| `dontAsk` | Never ask; block if not allowed |
| `plan` | Read-only exploration; understand before acting; no writes/edits |
| `bypassPermissions` | Allow everything without asking |

**Memory rule → CI headless = `--dangerously-skip-permissions` or `--permission-mode bypassPermissions`**

## Tool vs Skill vs Subagent

| | Tool | Skill | Subagent |
|---|---|---|---|
| What | Atomic executable capability | Reusable workflow instructions | Specialized separate context |
| Execution | Single action | Instructions Claude follows | Isolated Claude session |
| Context | Main conversation | Main or forked (`context: fork`) | Always isolated |
| Defined by | Code/API/MCP server | SKILL.md file | Agent definition |
| Reusable | Yes (always available) | Yes (invoked by name) | Yes (delegated to) |

**Memory rule → Tool=does one thing; Skill=knows how to do a workflow; Subagent=thinks separately**

## CLAUDE.md vs Skill

| | CLAUDE.md | Skill |
|---|---|---|
| Loading | Always (automatic) | On-demand (invoked) |
| Scope | Persistent project rules | Task-specific workflow |
| Context isolation | No (main context) | Optional (`context: fork`) |
| Tool restriction | No | Yes (`allowed-tools`) |

**Memory rule → CLAUDE.md = "always active rules"; Skill = "on-demand specialized procedure"**

## CLAUDE.md vs Hook

| | CLAUDE.md | Hook |
|---|---|---|
| Enforcement | Probabilistic (~95%) | Deterministic (100%) |
| Mechanism | Instructions Claude interprets | Shell commands that execute |
| Can block | No (only guide) | YES (PreToolUse exit non-zero) |
| For | Style, conventions, context | Security gates, compliance |

**Memory rule → CLAUDE.md = "please do X"; Hook = "X WILL happen / be blocked"**

## Hook vs Permission rule

| | Hook | Permission rule |
|---|---|---|
| Granularity | Custom logic (regex, path checks) | Tool + pattern matching |
| Complexity | Arbitrary shell scripts | Simple allow/deny lists |
| Blocking | Custom conditions | Always allow or always block |
| Configuration | settings.json hooks section | settings.json permissions |

**Memory rule → Permission = simple allow/deny gate; Hook = programmable enforcement logic**

## MCP vs Built-in Tool

| | MCP Tool | Built-in Tool |
|---|---|---|
| Execution | External server process | Claude Code runtime |
| Setup | Server config + transport | Available by default |
| Latency | Higher (IPC/network) | Lower (in-process) |
| Examples | Database, Slack, JIRA | Read, Write, Edit, Bash, Grep, Glob |

**Memory rule → Built-in = always there, fast; MCP = external system integration**

## MCP vs API

| | MCP | Claude API |
|---|---|---|
| Purpose | Tool/context integration protocol | Send messages to Claude |
| Direction | Connects Claude TO external systems | Your app talks TO Claude |
| Protocol | JSON-RPC over stdio/HTTP | REST API (HTTP) |
| Who calls whom | Claude calls MCP tools | You call Claude |

**Memory rule → API = talk TO Claude; MCP = give Claude tools to reach OUT**

## Configuration levels

| | Project (.claude/settings.json) | Local (.claude/settings.local.json) | User (~/.claude/settings.json) |
|---|---|---|---|
| Shared | YES (git-tracked) | NO (gitignored) | NO (personal) |
| Scope | This project, all developers | This project, this machine | All projects, this user |
| Override | Overrides user | Overrides project + user | Base level |

**Memory rule → Local > Project > User (higher = more specific = higher priority)**

---

# 3. Sessions & Context

| Fact | Remember |
|------|----------|
| Session ID | UUID assigned to each conversation |
| What's retained | Full message history, tool calls, tool results, system prompt |
| `--resume <id>` | Exact session restoration — deterministic |
| `--continue` | Most recent session — non-deterministic |
| `--fork-session` | Creates new branch from resumed session |
| `--no-session-persistence` | Session cannot be resumed later |
| Non-interactive (`-p`) | No persistence by default; must capture session_id from JSON output |
| Context compaction | Automatic summarization when approaching context limit |
| Concurrent CI jobs | Each MUST capture its own session_id |
| Shared runners | `--continue` is DANGEROUS — may pick wrong session |
| Resume concurrently | Same session_id from multiple processes = undefined behavior |
| Deterministic selection | ALWAYS use `--resume <id>`, NEVER rely on `--continue` in CI |

**Critical CI pattern:**
```bash
# Step 1: Run analysis, capture session
RESULT=$(claude -p "Analyze this code" --output-format json)
SESSION_ID=$(echo "$RESULT" | jq -r '.session_id')

# Step 2: Resume exact session
claude -p --resume "$SESSION_ID" "Now generate tests"
```

---

# 4. CI/CD Memory Sheet

## Patterns to recognize instantly:

```bash
# Stdin piping (non-interactive)
git diff | claude -p "Review this diff"
```
→ stdin + `-p` = non-interactive analysis

```bash
# Capture session for chaining
RESULT=$(claude -p "Analyze" --output-format json)
SESSION_ID=$(echo "$RESULT" | jq -r '.session_id')
```
→ JSON envelope → extract session_id

```bash
# Resume in later step
claude -p --resume "$SESSION_ID" "Generate tests"
```
→ Exact session continuation

```bash
# Structured output extraction
claude -p "Extract entities" --json-schema '{"type":"object","properties":{"entities":{"type":"array"}}}'
```
→ Model output constrained to schema

```bash
# Budget-limited CI run
claude -p --max-budget-usd 1.00 "Full code review"
```
→ Hard cost cap; stops if exceeded

## CI Critical Facts

| Topic | Fact |
|-------|------|
| Headless | Always use `-p`; no TTY available |
| Stdin/stdout | Pipe data in; capture stdout for results |
| JSON output | `--output-format json` for metadata |
| Structured output | `--json-schema` for model response shape |
| Exit behavior | Exits after response (no loop) |
| Max turns | `--max-turns` limits iterations |
| Budget | `--max-budget-usd` hard cap |
| Permissions | `--dangerously-skip-permissions` for trusted CI |
| Secrets | NEVER pass in prompts; use environment variables |
| Concurrency | Each job = own session_id |
| Isolation | `--worktree` for parallel file modifications |
| Determinism | `--resume <id>` only; never `--continue` |
| System prompt | `--append-system-prompt` to add CI context |

---

# 5. Configuration Precedence

```
highest precedence
↓
CLI flags (--model, --permission-mode, etc.)
↓
Enterprise/managed policy (if applicable)
↓
.claude/settings.local.json (project-local, not committed)
↓
.claude/settings.json (project-level, git-tracked)
↓
~/.claude/settings.json (user-level global)
↓
Built-in defaults
↓
lowest precedence
```

**Key rules:**
- Higher specificity wins
- CLI always overrides everything
- Local overrides project (same directory)
- Enterprise/managed can restrict (cannot be overridden by lower levels)
- `deny` rules are absolute — cannot be overridden by `allow` at a lower level

---

# 6. CLAUDE.md — Critical Facts

| Fact | Detail |
|------|--------|
| Purpose | Persistent instructions loaded into every conversation |
| Loading | Automatic — no user action needed |
| Hierarchy | Additive — all levels combined, NOT overriding |
| User global | `~/.claude/CLAUDE.md` — all projects |
| Project root | `./CLAUDE.md` — shared via git |
| Subdirectory | `./src/CLAUDE.md` — loaded when working in that dir |
| Project local | `./.claude/CLAUDE.md` — not committed (personal) |
| Activation | Subdirectory CLAUDE.md loads when Claude accesses any file under that path |
| Survives compaction | YES — persists through context summarization |
| Enforcement | Probabilistic (~95%) — Claude interprets, may deviate |
| Cannot enforce | Deterministic blocking (use Hooks for that) |
| `@import` syntax | Import other markdown files into CLAUDE.md |
| Relationship to Skills | CLAUDE.md = always active; Skills = on-demand |
| Relationship to Hooks | CLAUDE.md = guidance; Hooks = enforcement |
| Relationship to config | Config = behavior settings; CLAUDE.md = instructions |

---

# 7. Tools / Skills / Subagents

| Concept | What It Is | Best Use | Don't Confuse With |
|---------|-----------|----------|-------------------|
| **Tool** | Atomic executable capability | Single actions (read file, run command, search) | Skill (tools DO; skills KNOW) |
| **Skill** | Reusable workflow instructions (SKILL.md) | Complex repeatable procedures | Subagent (skills = instructions; subagents = separate context) |
| **Subagent** | Specialized isolated Claude context | Delegated reasoning with different tools/permissions | Skill (subagents think separately; skills run in main/forked context) |

**Key facts:**

| Property | Tool | Skill | Subagent |
|----------|------|-------|----------|
| Context isolation | No | Optional (`context: fork`) | Always |
| Reuse mechanism | Always available | `/name` invocation | Delegation by orchestrator |
| Tool access | N/A (IS a tool) | Restricted via `allowed-tools` | Restricted via `allowedTools` |
| Filesystem | N/A | `.claude/skills/<name>/SKILL.md` | `.claude/agents/<name>.md` |
| Specialization | Fixed capability | Procedural knowledge | Reasoning + tools |
| Permission inheritance | Session permissions | Restricted subset possible | Independent permissions |

**Architecture decision:**
- Need to DO something → Tool
- Need to KNOW HOW to do a workflow → Skill
- Need to THINK SEPARATELY with isolation → Subagent

---

# 8. Agentic Loops & stop_reason (D1)

## The Core Agentic Loop

```
while True:
    response = messages.create(model, system, tools, messages)
    
    if response.stop_reason == "end_turn":
        break  # Done
    elif response.stop_reason == "tool_use":
        execute tools → append tool_results → continue
    elif response.stop_reason == "pause_turn":
        re-send same messages (no changes) → continue
    elif response.stop_reason == "max_tokens":
        handle truncation (may continue or stop)
    elif response.stop_reason == "refusal":
        check stop_details.category → do not retry
```

## stop_reason Values

| Value | Meaning | Action |
|-------|---------|--------|
| `"end_turn"` | Claude finished naturally | Exit loop — task complete |
| `"tool_use"` | Claude wants to use tool(s) | Execute tools, send results, continue |
| `"max_tokens"` | Response truncated | May need to continue or increase max_tokens |
| `"stop_sequence"` | Hit custom stop sequence | Handle based on use case |
| `"pause_turn"` | Server-side tool hit iteration limit | Re-send same messages to continue (do NOT add "Continue" text) |
| `"refusal"` | Safety classifiers declined (Opus 5, Fable 5) | Check `stop_details.category`; do not retry |

## Critical API Facts

| Fact | Detail |
|------|--------|
| Tool results go in | `"user"` role message |
| Each tool_result must reference | `tool_use_id` (matching the request) |
| Multiple tool_use blocks | Parallel tool calls in one response |
| `tool_use_id` format | `"toolu_..."` (generated by API) |
| Max iterations | ALWAYS set a cap (prevent infinite loops) |
| Messages array | Append, never replace — full history |
| System prompt | Stable across loop iterations (cache it!) |
| Extended thinking | Compatible with agentic loops |
| Streaming | Works inside agentic loops |

## tool_choice Parameter

| Value | Behavior |
|-------|----------|
| `{"type": "auto"}` | Claude decides whether to use tools (DEFAULT) |
| `{"type": "any"}` | Claude MUST use at least one tool |
| `{"type": "tool", "name": "X"}` | Claude MUST use specific tool X |
| `{"type": "none"}` | Claude cannot use any tools |
| `disable_parallel_tool_use: true` | Max one tool call per response |

**Memory rule → auto=Claude decides; any=must use something; tool=must use specific one; none=no tools**

---

# 9. Multi-Agent Architecture Patterns (D1)

## Patterns

| Pattern | Structure | Best For |
|---------|-----------|----------|
| **Hub-and-Spoke** | Central orchestrator → specialist workers | Most common; clear control |
| **Pipeline** | Agent A → B → C (sequential) | Ordered processing stages |
| **Parallel fan-out** | Split → N workers → merge | Independent sub-tasks |
| **Hierarchical** | Orchestrator(Opus) → Managers(Sonnet) → Workers(Haiku) | Complex, layered tasks |
| **Peer-to-peer** | Agents communicate directly | Rare; hard to control |

## Model Selection for Agents

| Model | Use For |
|-------|---------|
| Opus | Complex reasoning, planning, architecture, judgment |
| Sonnet | Code, analysis, general tasks, review |
| Haiku | Classification, validation, simple lookups, routing |

## Delegation Rules

| Guideline | Value |
|-----------|-------|
| Max delegation depth | 2-3 levels |
| Right-sized task | 3-10 tool calls |
| Too small to delegate | < 1 tool call (just do it) |
| Too large | > 20 tool calls (decompose further) |
| Every sub-agent needs | Self-contained prompt, clear success criteria, timeout |
| Context passing | Minimal — only what's needed |
| Results | Always validate sub-agent outputs |

## Error Handling

| Error Type | Retry? | Strategy |
|-----------|--------|----------|
| 429 (Rate limit) | YES | Exponential backoff + jitter |
| 400 (Bad request) | NO | Fix the request |
| 500/503 (Server) | YES | Backoff, max 3 retries |
| Tool timeout | YES (once) | Then fallback |
| Reasoning loop | Feed error back | Max 3 self-corrections |
| 3+ consecutive failures | ESCALATE | To parent or human |

## Escalation Triggers
- 3+ consecutive errors on same tool
- Security-sensitive failure
- Conflicting results unresolvable
- Outside agent's defined capabilities
- Customer explicitly requests human

---

# 10. MCP Critical Facts (D2)

| Fact | Detail |
|------|--------|
| MCP = | Model Context Protocol |
| Architecture | Host (Claude Code) → Client (protocol layer) → Server (tools/resources) |
| Purpose | Standardized protocol for connecting AI to external systems |
| Transports | **stdio** (local, stdin/stdout) and **HTTP+SSE** (remote, web) |

## MCP Capabilities

| Capability | What It Does |
|-----------|-------------|
| **Tools** | Actions Claude can CALL (execute operations) |
| **Resources** | Data Claude can READ (browse available content) |
| **Prompts** | Templates for common interactions |
| **Sampling** | Server requests LLM completion (reverse direction) |

**Memory rule → Tools DO; Resources SEE; Prompts TEMPLATE; Sampling = server asks Claude**

## Configuration Locations

| Location | Scope | Shared? |
|----------|-------|---------|
| `.mcp.json` (project root) | Project | YES (git) |
| `~/.claude.json` mcpServers | User | NO (personal) |
| `--mcp-config <path>` | Session (CLI) | N/A |
| `--strict-mcp-config` | Ignore all others | N/A |

## MCP vs Native Tool vs API

| | MCP | Native/Built-in Tool | Claude API |
|---|---|---|---|
| Direction | Claude → external system | Claude → local runtime | App → Claude |
| Protocol | JSON-RPC (stdio or HTTP) | Internal | REST HTTP |
| Latency | Higher | Lowest | Variable |
| Setup | Server config | None | API key |
| Discovery | Dynamic (tool list from server) | Static (always available) | N/A |

## Security/Trust

- Never hardcode credentials in MCP config
- Use `${ENV_VAR}` expansion for secrets
- MCP servers are external — treat as untrusted boundary
- Audit server code before use
- Read-only default; escalate permissions explicitly
- Version-pin server dependencies

---

# 11. Tool Design & JSON Schema (D2)

## Tool Description Formula

```
"[WHAT it does]. Use when [TRIGGER]. Do NOT use for [DISAMBIGUATION]. Returns: [OUTPUT]."
```

## JSON Schema Key Constraints

| Type | Constraints |
|------|------------|
| `string` | minLength, maxLength, pattern, enum, format |
| `integer`/`number` | minimum, maximum, exclusiveMinimum/Maximum |
| `array` | items, minItems, maxItems, uniqueItems |
| `object` | properties, required, additionalProperties |

## Critical Schema Facts

| Fact | Detail |
|------|--------|
| `required` | Array of property names that MUST be present |
| `enum` | Fixed set of allowed values (use for stable/known sets) |
| `additionalProperties: false` | No extra fields allowed (strict mode) |
| `nullable` | Use `{"type": ["string", "null"]}` |
| Best complexity | ≤5 properties per tool (simple); 6-10 (good with descriptions) |
| Too complex | 15+ properties → split into multiple tools |
| `strict: true` | On tool definition → guarantees schema conformance |

## Built-in Tool Selection (Claude Code)

| Need | Use |
|------|-----|
| Find files by NAME/pattern | **Glob** |
| Find files by CONTENT | **Grep** |
| Read file contents | **Read** |
| Small targeted edit | **Edit** |
| New file or full rewrite | **Write** |
| Run shell commands | **Bash** |

**Critical distinction:** Grep = "which files CONTAIN X"; Glob = "which files are NAMED X"

## Structured Error Responses

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_...",
  "is_error": true,
  "content": "Error message with suggestion"
}
```

- `is_error: true` → signals Claude this was a failure
- Always include: what happened + what to try instead
- NEVER expose: stack traces, file paths, connection strings, API keys

## Parallel Tool Use

- Multiple `tool_use` blocks in one assistant message = parallel calls
- Each matched by unique `tool_use_id`
- All results sent back in single user message
- Disable with: `disable_parallel_tool_use: true` in tool_choice

---

# 12. Prompt Engineering & Structured Output (D4)

## Explicit Criteria Formula

```
[WHAT] must [VERB] [MEASURABLE CONDITION]
```

**Priority order:** Security → Correctness → Reliability → Performance → Readability → Style

## Criteria Types

| Type | Meaning |
|------|---------|
| MUST | Blocking requirement |
| SHOULD | Advisory (preferred but not blocking) |
| MUST NOT | Blocking prohibition |
| FORMAT | Structural requirement |
| DONE WHEN | Completion criteria |

## Few-Shot Prompting

**Structure:**
```xml
<examples>
  <example>
    <input>...</input>
    <output>...</output>
  </example>
</examples>
<task>Now handle: [actual input]</task>
```

**Quantity rules:**
- Format conformance: 1-2 examples
- Classification: 1 per category + ambiguous edge case
- Complex reasoning: 3-5 examples
- Nuanced judgment: 5-10+

**Key principles:**
- Show don't tell
- Include boundary/edge cases
- Simple → complex ordering
- Identical format across examples
- Separate examples from user content (security)
- Put examples in system prompt (enables caching)

## Validation-Retry Loop Pattern

```
Generate → Validate → Pass? → Done
                   → Fail? → Format errors → Retry (max N)
```

**Validation order (cheapest first):** syntax → schema → type → lint → unit test → integration

**Retry rules:**
- Max 3-5 attempts
- Same error 2× = change strategy
- Errors increasing = revert
- Security error = stop immediately

## Multi-Instance Review

| Pattern | How | Benefit |
|---------|-----|---------|
| Generator + Reviewer | Full context → code; Fresh reviewer → only sees output | +30-40% bugs found |
| Multi-pass | Pass 1: per-file; Pass 2: cross-file integration | +25% coverage |
| Confidence routing | 0.9-1.0 auto; 0.7-0.9 flag; <0.7 suppress | Reduces noise |

## tool_choice in Practice

| Scenario | Use |
|----------|-----|
| "Let Claude decide" | `"auto"` (default) |
| "Force tool use before responding" | `"any"` |
| "Must use this exact tool" | `{"type": "tool", "name": "X"}` |
| "Text only, no tools" | `"none"` |
| "One tool at a time" | `disable_parallel_tool_use: true` |

---

# 13. Message Batches API (D4)

| Fact | Detail |
|------|--------|
| Cost savings | **50% off** standard API prices |
| Max batch size | 100,000 requests OR 256 MB (whichever first) |
| Processing time | Most complete within 1 hour |
| Max processing window | **24 hours** (items expire after) |
| Results available | 29 days after creation |
| Supported models | ALL active models |
| Streaming | NOT supported (`stream: true` invalid) |
| `max_tokens: 0` | NOT supported in batches (works for regular API cache pre-warming) |
| Each request | Independent (can mix types in one batch) |

## Batch Lifecycle

```
created → in_progress → ended
                      → expired (if 24h exceeded)
                      → canceled
```

## Result Types Per Item

| Type | Meaning |
|------|---------|
| `succeeded` | Normal completion |
| `errored` | Request-level error |
| `canceled` | Batch was canceled |
| `expired` | Didn't complete in 24h |

## When to Use

| Approach | Use When |
|----------|----------|
| Batch API | Non-urgent + cost-sensitive + high-volume |
| Parallel async | Urgent + moderate volume |
| Sequential | Low-volume + ordering matters |

---

# 14. Context Management & Caching (D5)

## Prompt Caching

| Fact | Detail |
|------|--------|
| Format | `"cache_control": {"type": "ephemeral"}` |
| Default TTL | 5 minutes |
| Extended TTL | 1 hour (`"ttl": "1h"`) |
| Cache write cost | +25% (5m) or +100% (1h) of base |
| Cache read cost | **-90%** (0.1× base) |
| Max breakpoints | 4 per request |
| Min tokens (Opus 5, Fable 5, Mythos 5) | 512 |
| Min tokens (Opus 4.8, Sonnet 5, Sonnet 4.6) | 1,024 |
| Min tokens (Opus 4.7) | 2,048 |
| Min tokens (Opus 4.6, Opus 4.5, Haiku 4.5) | 4,096 |
| Below minimum | Silently skipped (no error) |
| Cacheable | System, tools, messages (text, images, documents) |
| Cache key | Hash of prefix up to breakpoint |
| Lookback window | 20 blocks maximum |
| Below minimum | Silently skipped (no error returned) |

**Placement rule:** Put `cache_control` on the LAST block whose prefix is IDENTICAL across requests.

**Invalidation triggers:** Changing tool_choice, images, thinking config, or any content before breakpoint.

## Usage Tracking

```json
{
  "cache_read_input_tokens": 100000,
  "cache_creation_input_tokens": 5000,
  "input_tokens": 50
}
```

`total = cache_read + cache_creation + input_tokens`

## Compaction vs Context Editing (Distinct Features!)

| Feature | What It Does | Beta Header |
|---------|-------------|-------------|
| **Compaction** | Summarizes older context into shorter form | `compact-2026-01-12` |
| **Context Editing** | Removes specific blocks (stale tool results, thinking) — NO summary | `context-management-2025-06-27` |

Context editing tools:
- `clear_tool_uses_20250919` — removes old tool_use/tool_result blocks
- `clear_thinking_20251015` — removes old thinking blocks

**Memory rule → Compaction = summarize; Context editing = delete specific blocks (no summary)**

## output_config (API-Level Structured Output)

```json
{
  "output_config": {
    "format": {
      "type": "json_schema",
      "json_schema": { ... },
      "name": "my_output"
    }
  }
}
```

- API parameter: `output_config.format` (replaces deprecated `output_format`)
- CLI equivalent: `--json-schema`
- Forces model to produce valid JSON matching your schema
- Distinct from `tool_choice` (which forces tool use, not output format)

## Progressive Summarization

| Fact | Detail |
|------|--------|
| Compaction trigger (API) | ~150K tokens (configurable; beta `compact-2026-01-12`) |
| Compaction trigger (manual) | `/compact` command anytime |
| Summarize first | Oldest messages |
| Keep intact | Recent 5-10 messages |
| Never compacted | System prompt, tools, CLAUDE.md |
| `/compact` | Manual compaction command |
| Scratchpad files | External state persistence |

## Context Degradation Signals

- Contradicts earlier information
- Re-asks previously answered questions
- Violates stated constraints
- Hallucinating details about prior context
- Producing generic patterns instead of specifics

## Attention Priority (best → worst)

```
System prompt > Last few messages > First few messages > Middle
```

## Context Budget Rule of Thumb

| Component | Allocation |
|-----------|-----------|
| System prompt | ~2.5% |
| Tools | ~5% |
| State/instructions | ~2.5% |
| Working space | ~75% |
| Safety margin | ~15% |

## Information Provenance Trust Hierarchy

```
Direct observation > User statement > Verified tool result > Logical inference > External source > Prior memory > General knowledge
```

---

# 15. Hooks Critical Facts

| Fact | Detail |
|------|--------|
| What | Deterministic shell commands triggered by lifecycle events |
| Enforcement | 100% — always executes (unlike CLAUDE.md ~95%) |
| Runs | Synchronously (Claude waits) |

## Hook Events

| Event | Trigger | Can Block? | Receives |
|-------|---------|-----------|----------|
| `PreToolUse` | Before tool executes | **YES** (exit non-zero) | Tool name, input |
| `PostToolUse` | After tool executes | No | Tool name, input, output |
| `Notification` | Claude sends notification | No | Notification content |
| `Stop` | Claude finishes response | No | Final response |

## Hook Mechanics

| Behavior | Detail |
|----------|--------|
| Exit 0 | Pass (tool proceeds) |
| Exit non-zero | BLOCK (PreToolUse only) |
| Stdout | Fed back to Claude as context |
| Matcher | Regex pattern matching tool names |
| Timeout | Configurable per hook |
| Env vars | `$TOOL_NAME`, `$TOOL_INPUT`, `$FILE_PATH`, `$TOOL_OUTPUT` |

## Hook vs CLAUDE.md

```
CLAUDE.md says "never delete production files"
→ Claude usually complies (~95%) but CAN deviate

Hook blocks `Bash(rm *)` via PreToolUse
→ Physically impossible to bypass (100%)
```

**Memory rule → CLAUDE.md = instruction (probabilistic); Hook = enforcement (deterministic)**

---

# 16. Permissions & Security

## Permission Flow

```
Claude wants to use tool
  ├─ In "allow" list? → Execute silently
  ├─ In "deny" list? → Block always (never ask)
  └─ Neither? → Prompt user for approval
```

## Permission Patterns

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(npm test *)",
      "Write(src/**)",
      "Edit(src/**)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Write(.env*)"
    ]
  }
}
```

## Precedence

- `deny` ALWAYS wins over `allow`
- Local > Project > User (same key)
- CLI flags override all settings
- Enterprise/managed can restrict (cannot be overridden below)

## Security Facts

| Topic | Remember |
|-------|----------|
| `--dangerously-skip-permissions` | **DANGEROUS** — no approval for anything |
| Secrets | NEVER in prompts; use env vars |
| Prompt injection | MCP servers are external trust boundary |
| Least privilege | Minimum tools needed per agent/task |
| Filesystem | Use path patterns in allow/deny |
| Headless/CI | Must pre-configure permissions (no user to approve) |
| Enterprise | Managed policies override local settings |
| `--safe-mode` | Disables all customizations for troubleshooting |
| `--bare` | Skips hooks, plugins, auto-memory |

---

# 17. Defaults I Might Forget

| Behavior | Default |
|----------|---------|
| tool_choice | `"auto"` (Claude decides) |
| Parallel tool use | Enabled |
| Output format (`-p`) | `text` |
| Permission mode | Ask user (manual-ish, not auto) |
| Session persistence (interactive) | ON |
| Session persistence (`-p`) | OFF |
| CLAUDE.md loading | Automatic (always) |
| Context auto-compact | Enabled when approaching limit |
| Cache TTL | 5 minutes |
| Batch processing window | 24 hours max |
| Batch cost | 50% of standard |
| Cache read discount | 90% off |
| Cache write premium | +25% (5m) / +100% (1h) |
| Max cache breakpoints | 4 per request |
| stop_reason for normal completion | `"end_turn"` |
| stop_reason when tools needed | `"tool_use"` |
| Hook PreToolUse exit 0 | Allow (proceed) |
| Hook PreToolUse exit non-zero | Block (tool not executed) |
| CLAUDE.md enforcement | ~95% (probabilistic) |
| Hook enforcement | 100% (deterministic) |
| Settings override direction | Local > Project > User |
| `deny` vs `allow` conflict | `deny` wins |
| Subdirectory CLAUDE.md | Loads only when working in that dir |
| MCP transport (local) | stdio |
| MCP transport (remote) | HTTP+SSE |
| Batch results expiry | 29 days |

---

# 18. Important Locations

| Item | Path |
|------|------|
| User CLAUDE.md | `~/.claude/CLAUDE.md` |
| Project CLAUDE.md | `./CLAUDE.md` (root) |
| Local CLAUDE.md | `./.claude/CLAUDE.md` |
| Subdirectory CLAUDE.md | `./<subdir>/CLAUDE.md` |
| User settings | `~/.claude/settings.json` |
| Project settings (shared) | `.claude/settings.json` |
| Project settings (local) | `.claude/settings.local.json` |
| Skills (project) | `.claude/skills/<name>/SKILL.md` |
| Skills (personal) | `~/.claude/skills/<name>/SKILL.md` |
| Agents | `.claude/agents/<name>.md` |
| MCP config (project) | `.mcp.json` (project root) |
| MCP config (user) | `~/.claude.json` (mcpServers section) |
| Slash commands | `.claude/commands/<name>.md` |
| Rules (path-specific) | `.claude/rules/` |
| Hooks | In `settings.json` hooks section |
| Sessions/history | `~/.claude/` (internal) |

---

# 19. Architecture Decision Rules

| Requirement | Correct Choice |
|------------|---------------|
| Reusable procedural instructions | **Skill** |
| External system/tool integration | **MCP** |
| Specialized delegated reasoning | **Subagent** |
| Deterministic lifecycle enforcement | **Hook** |
| Project-wide persistent instructions | **CLAUDE.md** |
| Atomic executable capability | **Tool** |
| Non-interactive CI execution | `claude -p` |
| Exact previous conversation | `--resume <session-id>` |
| Path-specific rules | Subdirectory CLAUDE.md or `.claude/rules/` |
| Prevent dangerous commands | Hook (PreToolUse + exit non-zero) |
| Cost reduction for bulk processing | **Message Batches API** (50% off) |
| Reduce token costs on repeated prompts | **Prompt caching** (90% read savings) |
| Complex reasoning, planning | **Opus** model |
| General code tasks | **Sonnet** model |
| Classification, routing, validation | **Haiku** model |
| Multiple independent sub-tasks | Parallel fan-out pattern |
| Ordered processing stages | Pipeline pattern |
| Central coordination | Hub-and-Spoke |
| Security gate (block actions) | Hook PreToolUse (exit non-zero) |
| Audit trail (log actions) | Hook PostToolUse (stdout) |
| Force tool use | `tool_choice: "any"` |
| Force specific tool | `tool_choice: {"type":"tool","name":"X"}` |
| Structured model output | `--json-schema` (CLI) or JSON schema in tool |
| Team-shared conventions | Project CLAUDE.md (git-tracked) |
| Personal preferences | User CLAUDE.md or local settings |
| Isolated file changes in parallel | `--worktree` |
| Skip all customizations for debug | `--safe-mode` |
| Add CI-specific instructions | `--append-system-prompt` |
| Understand before changing (explore first) | Plan mode (`--permission-mode plan`) |
| Summarize old context to free space | Compaction (beta `compact-2026-01-12`) |
| Remove stale tool results/thinking | Context editing (`context-management-2025-06-27`) |
| Constrain model response at API level | `output_config.format` with json_schema |

---

# 20. Easy Things to Get Wrong

| Trap | Correct Understanding |
|------|----------------------|
| `--continue` = exact session | NO — it's "most recent relevant" (non-deterministic) |
| `--output-format json` = structured model output | NO — it's the CLI metadata envelope |
| `--json-schema` = output format | NO — it constrains the MODEL's response content |
| CLAUDE.md = deterministic enforcement | NO — ~95% compliance; use Hooks for 100% |
| Skill = Subagent | NO — Skill = instructions; Subagent = separate thinking context |
| MCP = model | NO — MCP is the protocol connecting Claude to external tools |
| Tool = Skill | NO — Tool does ONE thing; Skill knows HOW to do a workflow |
| `--no-session-persistence` then `--resume` | FAILS — session was never saved |
| Shared runner + `--continue` | DANGEROUS — may pick another job's session |
| Hook PostToolUse can block | NO — tool already executed; only PreToolUse blocks |
| `deny` can be overridden by `allow` | NO — deny ALWAYS wins |
| `--system-prompt` preserves CLAUDE.md | NO — it REPLACES everything |
| `--append-system-prompt` replaces | NO — it ADDS to existing |
| Prompt caching = free | NO — writes cost +25%/+100%; reads save 90% |
| Batch API = instant | NO — up to 24h (most within 1 hour) |
| `tool_choice: "any"` = specific tool | NO — "any" = must use SOME tool; use `"tool"` for specific |
| `is_error: true` stops execution | NO — it signals failure TO Claude; Claude decides next step |
| MCP resources = tools | NO — Resources READ; Tools DO |
| All CLAUDE.md files override each other | NO — they're ADDITIVE (all combined) |
| Subdirectory CLAUDE.md always loads | NO — only when Claude works in that directory |
| Auto-compact removes system prompt | NO — system prompt is NEVER compacted |
| Cache breakpoints unlimited | NO — max 4 per request |
| `--bare` = `--safe-mode` | NO — bare skips hooks/plugins/memory; safe-mode disables ALL customizations |
| `--fork-session` = `--resume` | NO — fork creates NEW session branched from old one |
| exit 0 in PreToolUse = block | NO — exit 0 = ALLOW; non-zero = block |
| `disable_parallel_tool_use` = no tools | NO — still uses tools, just one at a time |
| Prompt caching min tokens always 1024 | NO — varies by model (512 to 4096) |
| Batch results available forever | NO — 29 days, then gone |
| Only 4 stop_reason values | NO — also `pause_turn` and `refusal` |
| `pause_turn` = add "Continue" message | NO — re-send SAME messages unchanged |
| `refusal` = retry with different prompt | NO — safety refusal; do NOT retry |
| Compaction = context editing | NO — compaction summarizes; context editing (clear_tool_uses/clear_thinking) removes specific blocks |
| `--json-schema` = API output_config | CLI-level flag; API uses `output_config.format` parameter |

---

# 21. One-Line Memory Bank

**CLI & Sessions:**
- `-p` → non-interactive execution; always for CI.
- `--resume <id>` → exact session; deterministic.
- `--continue` → most recent session; non-deterministic; dangerous in CI.
- `--fork-session` → branch into new session from old one.
- `--output-format json` → CLI result envelope with session_id.
- `--json-schema` → constrains MODEL output to schema.
- `--system-prompt` → REPLACES all default instructions.
- `--append-system-prompt` → ADDS to existing instructions.
- `--dangerously-skip-permissions` → bypasses ALL safety checks.
- `--bare` → minimal mode (no hooks, plugins, memory).
- `--safe-mode` → disables all customizations.
- `--worktree` → git worktree isolation for parallel work.
- `--max-budget-usd` → hard cost cap; session stops.
- `--effort` → reasoning depth: low/medium/high/xhigh/max.
- `--strict-mcp-config` → ignores all MCP configs except specified.

**Architecture Concepts:**
- Tool → atomic capability; does one thing.
- Skill → reusable workflow knowledge; SKILL.md.
- Subagent → specialized separate thinking context; isolated.
- MCP → protocol connecting Claude to external systems.
- Hook → deterministic lifecycle enforcement; shell commands.
- CLAUDE.md → persistent instructions; always loaded; additive.

**Agentic Patterns:**
- `stop_reason: "end_turn"` → done; exit loop.
- `stop_reason: "tool_use"` → execute tools; continue loop.
- `stop_reason: "pause_turn"` → re-send same messages unchanged; continue.
- `stop_reason: "refusal"` → safety declined; check stop_details.category; do NOT retry.
- `stop_reason: "max_tokens"` → truncated; handle.
- `tool_choice: "auto"` → Claude decides (default).
- `tool_choice: "any"` → must use some tool.
- `tool_choice: {"type":"tool","name":"X"}` → must use X.
- `tool_choice: "none"` → no tools allowed.
- `disable_parallel_tool_use: true` → one tool per turn.
- Hub-and-Spoke → central orchestrator + specialist workers.
- Pipeline → sequential stage processing.
- Parallel fan-out → independent sub-tasks.
- Max delegation depth → 2-3 levels.
- Right-sized task → 3-10 tool calls.

**Tool Design:**
- `is_error: true` → signals failure to Claude.
- tool_use_id → matches request to result.
- Tool description → WHAT + WHEN + NOT FOR + RETURNS.
- Grep → finds CONTENT; Glob → finds NAMES.
- `strict: true` → guarantees schema conformance.
- `additionalProperties: false` → no extra fields.
- Enum → for stable/known/finite value sets only.

**Prompt Engineering:**
- Few-shot in system prompt → enables caching.
- Validation order → cheapest first (syntax before integration).
- Retry max → 3-5; same error 2× = change strategy.
- Confidence routing → 0.9+ auto; 0.7-0.9 flag; <0.7 suppress.
- Multi-instance → generator + fresh reviewer = +30-40% bugs.

**Context & Caching:**
- Cache read → 90% savings (0.1× base cost).
- Cache write → +25% (5m) or +100% (1h).
- Cache TTL default → 5 minutes.
- Cache min tokens → varies by model (512-4096).
- Max breakpoints → 4 per request.
- Compact at → 75-80% context usage.
- Never compacted → system prompt, tools, CLAUDE.md.
- Attention priority → system prompt > recent > first > middle.
- Provenance → direct observation > user statement > tool result > inference.

**Batch API:**
- 50% cost reduction.
- Up to 100K requests or 256 MB per batch.
- Results within 1 hour (most); 24h max.
- Results expire → 29 days.
- No streaming in batches.
- All active models supported.

**Permissions & Security:**
- deny ALWAYS beats allow.
- Local > Project > User (settings precedence).
- CLI flags override everything.
- Hook PreToolUse exit 0 = allow; non-zero = block.
- Hook PostToolUse = cannot block (already ran).
- CLAUDE.md compliance ≈ 95% (not guaranteed).
- Secrets → environment variables, NEVER in prompts.

**Configuration:**
- CLAUDE.md hierarchy = additive (all levels combined).
- Subdirectory CLAUDE.md = loads only when working there.
- Settings hierarchy = override (higher wins).
- `@import` = include other files in CLAUDE.md.
- Skill frontmatter: `context: fork`, `allowed-tools`, `argument-hint`.

---

# 22. 10-Minute Before Exam Cheat Sheet

## CLI — The Essentials

```
-p                    = non-interactive (CI/scripts)
--output-format json  = CLI envelope (session_id, cost)
--json-schema         = MODEL output constraint
--resume <id>         = exact session (DETERMINISTIC)
--continue            = most recent (NON-DETERMINISTIC, CI-DANGEROUS)
--system-prompt       = REPLACES everything
--append-system-prompt = ADDS to existing
--dangerously-skip-permissions = NO SAFETY NET
--fork-session        = branch conversation
--effort              = reasoning depth
--max-budget-usd      = hard cost cap
--bare                = no hooks/plugins/memory
--safe-mode           = no customizations
```

## The Agentic Loop

```
1. Call messages.create with tools
2. Check stop_reason:
   - "end_turn" → DONE
   - "tool_use" → execute → send tool_result → goto 1
   - "pause_turn" → re-send SAME messages unchanged → goto 1
   - "refusal" → check stop_details.category → STOP (don't retry)
   - "max_tokens" → handle truncation
3. ALWAYS set max iterations
```

## tool_choice

```
auto = Claude decides (DEFAULT)
any  = MUST use a tool
tool = MUST use specific named tool
none = cannot use tools
```

## Session Rules for CI

```
ALWAYS: capture session_id from JSON output
ALWAYS: use --resume <id> for continuation
NEVER:  use --continue on shared runners
NEVER:  rely on session persistence with -p (off by default)
```

## Configuration Precedence

```
CLI > Enterprise/Managed > Local > Project > User > Defaults
deny ALWAYS wins over allow
CLAUDE.md levels are ADDITIVE (not overriding)
Settings levels are OVERRIDING (higher wins)
```

## Tools / Skills / Subagents

```
Tool     = does ONE thing (atomic capability)
Skill    = knows HOW (reusable instructions, SKILL.md)
Subagent = thinks SEPARATELY (isolated context)
```

## MCP

```
Protocol for Claude → external systems
Transports: stdio (local) | HTTP+SSE (remote)
Capabilities: Tools (DO) | Resources (READ) | Prompts (TEMPLATE)
Config: .mcp.json (project) | ~/.claude.json (user) | --mcp-config (CLI)
```

## Hooks

```
PreToolUse:  exit 0 = allow | exit non-zero = BLOCK
PostToolUse: cannot block (already ran); stdout → Claude
CLAUDE.md = guidance (~95%) | Hook = enforcement (100%)
Events: PreToolUse, PostToolUse, Notification, Stop
```

## CLAUDE.md

```
Always loaded. Additive hierarchy. Survives compaction.
User (~/.claude/) + Project root + Subdirectory + Local (.claude/)
Cannot enforce deterministically — that's what Hooks do.
Subdirectory = loads ONLY when working in that dir.
```

## Prompt Caching

```
cache_control: {"type": "ephemeral"}
Read = 90% savings | Write = +25% (5m) / +100% (1h)
TTL: 5min (default) | 1h (extended)
Max 4 breakpoints | Min tokens varies by model (512-4096)
Place on LAST STABLE block (not changing content)
```

## Batch API

```
50% cost savings | 100K requests or 256MB max
Most complete <1h | Expires at 24h | Results last 29 days
No streaming | All active models supported
```

## Permissions

```
allow → silent execute | deny → always block | neither → ask user
deny > allow (deny always wins)
Local > Project > User (higher specificity wins)
CI: pre-configure permissions OR --dangerously-skip-permissions
```

## Architecture Decisions

```
CI execution          → claude -p
Exact session         → --resume <id>
External integration  → MCP
Workflow reuse        → Skill
Delegated thinking    → Subagent
Enforce action        → Hook (PreToolUse)
Persistent rules      → CLAUDE.md
Bulk processing       → Batch API
Cost optimization     → Prompt caching
Complex reasoning     → Opus
General tasks         → Sonnet
Routing/validation    → Haiku
```

## Top Traps

```
--output-format json ≠ --json-schema
--continue ≠ deterministic session selection
CLAUDE.md ≠ deterministic enforcement
Skill ≠ Subagent
Tool ≠ Skill
MCP Resource ≠ MCP Tool
--system-prompt REPLACES; --append-system-prompt ADDS
deny ALWAYS beats allow
PostToolUse CANNOT block
exit 0 = ALLOW (not block)
Cache min tokens VARIES by model
Batch results expire after 29 days
CLAUDE.md is ADDITIVE; settings are OVERRIDING
```

---

# 23. Critical Things Missing From My Original Request

| Topic | Key Fact |
|-------|----------|
| **stop_reason values** | Core of agentic loop — `end_turn`, `tool_use`, `max_tokens`, `stop_sequence` |
| **tool_choice parameter** | auto/any/tool/none + `disable_parallel_tool_use` |
| **tool_use_id** | Required in every tool_result to match request |
| **is_error field** | `true` signals failure to Claude in tool_result |
| **strict: true** | Guarantees tool call schema conformance |
| **Message Batches API** | 50% off, 100K max, 24h window, 29-day results |
| **Prompt caching** | 90% read savings, 4 breakpoints max, model-specific minimums |
| **Cache TTL options** | 5m (default) vs 1h (extended, 2× write cost) |
| **Multi-agent patterns** | Hub-and-Spoke, Pipeline, Parallel, Hierarchical |
| **Model selection per task** | Opus=reasoning, Sonnet=code, Haiku=routing |
| **Delegation sizing** | 3-10 tool calls = right size |
| **Error handling** | 429 retry with backoff; 400 fix; 3+ failures escalate |
| **Few-shot prompting** | XML structure, system prompt placement, quantity rules |
| **Validation-retry loops** | Max 3-5, cheapest validation first, same error 2× = change strategy |
| **Confidence routing** | 0.9+ auto-accept, 0.7-0.9 flag, <0.7 suppress |
| **Progressive summarization** | Compact at 75-80%, oldest first, system prompt never compacted |
| **Context degradation signals** | Contradictions, re-asking, constraint violations |
| **Attention distribution** | System prompt > recent > first > middle |
| **Information provenance** | Trust hierarchy for sources |
| **`--effort` flag** | low/medium/high/xhigh/max — controls reasoning depth |
| **`--fork-session`** | Branch from existing session into new one |
| **`--bare` vs `--safe-mode`** | bare = no hooks/plugins; safe-mode = no customizations at all |
| **`--worktree`** | Git worktree for isolated parallel file changes |
| **Skill frontmatter** | `context: fork`, `allowed-tools`, `argument-hint` |
| **`.claude/rules/`** | Path-specific rules with glob patterns |
| **`@import` in CLAUDE.md** | Include other markdown files |
| **Slash commands** | `.claude/commands/<name>.md` — simple prompt templates |
| **Server tools** | web_search, web_fetch, code_execution run on Anthropic infrastructure |
| **Client tools** | Your code executes; send back tool_result |
| **disable_parallel_tool_use** | In tool_choice; limits to one tool per turn |
| **Tool use token overhead** | 286-804 tokens depending on model + tool_choice |
| **Escalation patterns** | L1 self-correct → L2 change approach → L3 parent → L4 human |
| **Handoff patterns** | Prerequisite gates, structured handoff summaries |
| **CLAUDE.md additive** | All levels COMBINED (not overriding like settings) |
| **pause_turn** | Server-side tool hit iteration limit; re-send same messages to continue |
| **refusal** | Safety classifiers declined; check stop_details.category (Opus 5, Fable 5) |
| **Compaction beta** | `compact-2026-01-12`; server-side summarization with `context_management.edits` |
| **Context editing tools** | `clear_tool_uses_20250919`, `clear_thinking_20251015` — distinct from compaction |
| **output_config.format** | API-level structured output config (replaces deprecated output_format param) |
| **pause_turn handling** | Do NOT add "Continue" text; just re-send the same messages array unchanged |
| **refusal → stop_details** | Contains `.category` field explaining why content was refused |
