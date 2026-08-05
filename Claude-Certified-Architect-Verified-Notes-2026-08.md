# Claude Certified Architect - Foundations: Verified Study Notes

**Verified:** August 5, 2026  
**Sources:** platform.claude.com, code.claude.com, modelcontextprotocol.io  
**Exam Guide:** v1.0 (effective July 2026)  
**Exam Fee:** $125 | **Validity:** 12 months | **Passing:** 720/1000

---

## Exam Overview

| Detail | Value |
|--------|-------|
| Questions | 60 multiple-choice |
| Time | 120 minutes |
| Passing Score | 720/1000 (scaled) |
| Format | 4 scenarios drawn from bank of 6 |
| Domains | 5 (weights: 27%, 18%, 20%, 20%, 15%) |
| Delivery | Pearson VUE (OnVUE or test center) |
| Attempts | Up to 4 per rolling 12 months (14/30/90-day waits) |

---

## Current Models (August 2026)

| Model ID | Notes |
|----------|-------|
| `claude-fable-5` | Most capable widely released model; 1M context |
| `claude-opus-5` | Latest Opus; 1M context; **thinking ON by default** (unlike 4.8/4.7 where it's off) |
| `claude-opus-4-8` | Previous Opus tier; 1M context, 128k max output; thinking OFF by default (when enabled, only adaptive mode available) |
| `claude-opus-4-7` | Legacy Opus; 1M context |
| `claude-sonnet-5` | Current Sonnet; 1M context |
| `claude-haiku-4-5-20251001` | Fast/cheap; 200k context |
| `claude-sonnet-4-6` | Previous-generation (still active) |

---

# Domain 1: Agentic Architecture & Orchestration (27%)

## 1.1 The Agentic Loop Lifecycle

At its core, an agentic system is a loop: Claude receives a prompt with tools, decides whether to call a tool or respond, and the loop continues until Claude signals it's done.

### Loop Flow

```
User sends prompt with tool definitions
    ↓
Claude responds
    ↓
Check stop_reason
    ↓
┌─────────────────────────────────────────┐
│ "tool_use"      → Execute tool          │
│                 → Return tool_result     │
│                 → Send next request      │
│                 → Loop continues         │
│                                         │
│ "end_turn"      → Claude is done        │
│                 → Present final text     │
│                 → Loop ends             │
│                                         │
│ "stop_sequence" → Custom stop matched   │
│                 → Loop ends             │
│                                         │
│ "pause_turn"    → Server-side tool      │
│                   loop hit its cap      │
│                 → Send response back    │
│                 → Loop continues        │
│                                         │
│ "max_tokens"    → Hit token limit       │
│                 → May need continue     │
│                                         │
│ "refusal"       → Declined on safety    │
│                 → Inspect stop_details  │
│                 → Loop ends             │
│                                         │
│ "model_context_window_exceeded"         │
│                 → Hit context window    │
│                 → Response valid but    │
│                   truncated             │
└─────────────────────────────────────────┘
```

### stop_reason Values (VERIFIED)

| Value | Meaning | Action |
|-------|---------|--------|
| `"tool_use"` | Claude wants to call a tool | Execute it, return result, continue loop |
| `"end_turn"` | Claude has finished reasoning | Present the final text response |
| `"stop_sequence"` | Output matched a configured stop_sequences string | Loop ends; inspect which sequence was hit |
| `"pause_turn"` | Server-side tool loop (e.g., web_search) hit its internal iteration limit | Send the response back to continue |
| `"max_tokens"` | Response hit the max_tokens limit | May need to request continuation |
| `"refusal"` | Claude declined to respond on safety grounds | Loop ends; inspect stop_details, rephrase or route |
| `"model_context_window_exceeded"` | Generation hit context window before max_tokens | Response valid but truncated; trim input or continue |

### stop_details on Refusals (VERIFIED - Opus 4.7+)

A refusal response carries a `stop_details` object:
- `stop_details.type` — always `"refusal"`
- `stop_details.category` — policy category (e.g., "cyber", "bio", "reasoning_extraction", "frontier_llm", or null)
- `stop_details.explanation` — human-readable string (don't parse it)
- `stop_details` is `null` for every other stop reason

### API Response Structure (VERIFIED)

```json
{
  "id": "msg_01Aq9w938a90dw8q",
  "model": "claude-opus-4-8",
  "stop_reason": "tool_use",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "I'll check the current weather."
    },
    {
      "type": "tool_use",
      "id": "toolu_01A09q90qw90lq917835lq9",
      "name": "get_weather",
      "input": { "location": "San Francisco, CA", "unit": "celsius" }
    }
  ]
}
```

The `content` array can contain multiple blocks. Claude may call multiple tools simultaneously by including multiple `tool_use` blocks.

### Returning Tool Results (VERIFIED)

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
      "content": "15 degrees celsius, partly cloudy"
    }
  ]
}
```

For multiple simultaneous tool calls, return multiple `tool_result` blocks in the same message, each matching its `tool_use_id`.

### Error Results (VERIFIED)

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
  "is_error": true,
  "content": "API timeout after 30 seconds"
}
```

Setting `is_error: true` helps Claude understand the tool failed and reason about recovery.

### Anti-Patterns (Wrong Answers on Exam)

1. Parsing natural language for loop termination — Always check `stop_reason`
2. Arbitrary iteration caps as primary stopping — `stop_reason` drives the loop
3. Checking for text content as completion — Responses can contain both text and `tool_use` blocks

---

## 1.2 Multi-Agent Orchestration

### Hub-and-Spoke Architecture

```
┌──────────────┐
│  Coordinator │
│    Agent     │
└──────┬───────┘
       │
┌──────┼──────────────┐
↓      ↓              ↓
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Subagent │  │ Subagent │  │ Subagent │
│  Search  │  │ Analysis │  │ Synthesis│
└──────────┘  └──────────┘  └──────────┘
```

### Coordinator Responsibilities

- Decomposes the task into subtasks
- Delegates to appropriate subagents
- Routes all inter-subagent communication (subagents never talk to each other directly)
- Aggregates results from subagents
- Handles errors from subagent failures
- Evaluates synthesis quality and re-delegates if gaps exist

### Subagent Isolation (CRITICAL EXAM CONCEPT)

Subagents have isolated context. They do NOT inherit the coordinator's conversation history. Everything a subagent needs must be explicitly passed in its prompt.

**Bad:** `"Analyze the customer's issue"`  
**Good:** `"Customer #12345 reported that their order #67890 (placed 2024-01-15, total $149.99) arrived damaged. They are requesting a full refund. Analyze the order history and return policy applicability."`

### Decomposition Risks

Overly narrow decomposition is tested. Example: coordinator asked to research "creative industries" decomposes only into visual arts subtasks, missing music, writing, film, and design.

Fix: broader initial decomposition with explicit coverage checks.

---

## 1.3 Subagent Spawning and Context Passing

### The Task / Agent Tool (VERIFIED)

Subagents are spawned using the **Task** tool in Claude Code and the **Agent** tool in the Claude Agent SDK.

### Claude Agent SDK (VERIFIED naming)

- Python: `pip install claude-agent-sdk`
- TypeScript: `npm install @anthropic-ai/claude-agent-sdk`
- Top-level entry point: `query()` + `ClaudeAgentOptions`

```python
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async for message in query(
    prompt="Use the code-reviewer agent to review this codebase",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Glob", "Grep", "Agent"],
        agents={
            "code-reviewer": AgentDefinition(
                description="Expert code reviewer for Python and TypeScript",
                prompt="Analyze code quality, identify bugs, suggest improvements.",
                tools=["Read", "Glob", "Grep"],
            ),
        },
    ),
):
    print(message)
```

Messages emitted from inside a subagent carry a `parent_tool_use_id` field for attribution.

### Parallel Subagent Execution

To run subagents in parallel, emit multiple Task/Agent tool calls in a single coordinator response (not across separate turns).

### Branching (formerly "fork_session") (CORRECTED)

The correct term is **branch**. Creates independent branches from a shared analysis baseline.

- In Claude Code: `/branch` command
- CLI: `claude --continue --fork-session`
- Use case: Testing divergent approaches from the same starting point without branches influencing each other

---

## 1.4 Workflow Enforcement and Handoff

### The Critical Distinction (HEAVILY TESTED)

| Enforcement Type | When to Use | Guarantee Level |
|-----------------|-------------|-----------------|
| Programmatic (hooks, prerequisites) | Financial/safety consequences | Deterministic — 100% enforced |
| Prompt-based (instructions) | Best-effort acceptable | Probabilistic — non-zero failure rate |

**Wrong (prompt-based):** "Always verify the customer's identity before processing any refund."  
**Correct (programmatic):** A PreToolUse hook on `process_refund` that checks whether `get_customer` has been called and returned a verified customer ID.

### Handoff to Humans

Include a structured summary (human agent typically can't see full transcript):

```
Customer: Jane Doe (#12345)
Issue: Damaged product received
Order: #67890, $149.99, placed 2024-01-15
Actions taken: Verified order, confirmed damage via photos
Recommended action: Full refund + replacement shipment
```

---

## 1.5 Claude Code Hooks (VERIFIED)

### Hook Events — 30 Events Across Seven Groups

| Group | Events |
|-------|--------|
| Setup | SessionStart, SessionEnd, Stop, StopFailure |
| Session & turn | UserPromptSubmit, UserPromptExpansion, Notification, InstructionsLoaded, MessageDisplay, TeammateIdle |
| Tool / agentic loop | PreToolUse, PostToolUse, PostToolUseFailure, PostToolBatch, PermissionRequest, PermissionDenied |
| Agent & task | SubagentStart, SubagentStop, TaskCreated, TaskCompleted |
| File & config | FileChanged, CwdChanged, ConfigChange |
| Compaction | PreCompact, PostCompact |
| Context & worktree | Elicitation, ElicitationResult, WorktreeCreate, WorktreeRemove |

### Hook Types (VERIFIED — 5 types)

| Type | How It Works | Use Case |
|------|-------------|----------|
| `command` | Runs a shell command. Exit 0 = pass, exit 2 = block | File validation, linting gates |
| `http` | POST to a URL endpoint | External audit logging, webhooks |
| `mcp_tool` | Calls a tool on a configured MCP server | Structured validation via reusable service |
| `prompt` | Single LLM call that returns yes/no | Context-dependent approval |
| `agent` | Multi-turn subagent with tool access | Complex compliance checks |

### PreToolUse Hook Configuration (Current JSON Shape)

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "process_refund",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/check-refund-authorization.sh",
            "timeout": 600,
            "statusMessage": "Verifying refund authorization..."
          }
        ]
      }
    ]
  }
}
```

### Decision Control Output

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Customer identity not yet verified",
    "additionalContext": "Call get_customer first to verify identity"
  }
}
```

`permissionDecision` values: `"allow"`, `"deny"`, `"ask"`

### Hook Configuration Locations (VERIFIED)

| Location | Scope | Shared? |
|----------|-------|---------|
| `~/.claude/settings.json` | All projects for this user | No (personal) |
| `.claude/settings.json` | This project | Yes (version control) |
| `.claude/settings.local.json` | This project, this machine | No (gitignored) |
| Managed policy settings | Organization-wide | Yes (admin-managed) |

---

## 1.6 Task Decomposition Strategies

### Prompt Chaining (Fixed Sequential)
Steps predetermined. Each step's output feeds into the next.

### Dynamic Adaptive Decomposition
Subtasks emerge from discoveries (e.g., profiling → discovering DB bottleneck → analyzing queries).

### Multi-Pass Review Architecture
- **Pass 1:** Per-file local analysis (catches local bugs, style, types)
- **Pass 2:** Cross-file integration (catches data flow issues, interface mismatches)

Single-pass on large PRs leads to attention dilution, contradictory findings, and missed integration issues.

---

## 1.7 Session Management (CORRECTED)

### Resuming Sessions
`--resume <session-name>` continues a named prior conversation with full context.

**When to resume:** Ongoing investigation where tool results are still valid  
**When to start fresh:** Tool results are stale, context has drifted

### Branching (CORRECTED from "fork_session")

- `/branch` creates independent branches from a shared analysis baseline
- CLI flag: `--fork-session`
- Each branch can explore a different approach without contaminating the others

### Informing Resumed Sessions

When resuming after file changes, explicitly tell Claude what changed. Don't assume the resumed session will notice changes on its own.

---

# Domain 2: Tool Design & MCP Integration (18%)

## 2.1 Tool Interface Design

### Tool Descriptions Are the Selection Criteria

**Bad:** `"description": "Search for things"`  
**Good:** `"description": "Search the internal knowledge base for support articles, FAQs, and troubleshooting guides. Use when the customer asks a question about product features, billing, or technical issues."`

### Full Tool Definition Schema (VERIFIED)

```json
{
  "name": "get_weather",
  "description": "Get the current weather in a given location. Returns temperature, conditions, humidity.",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "The city and state/country, e.g. 'San Francisco, CA'"
      },
      "unit": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "description": "Temperature unit. Defaults to celsius if not specified."
      }
    },
    "required": ["location"]
  }
}
```

Valid fields: `name` (regex `^[a-zA-Z0-9_-]{1,64}$`), `description`, `input_schema`, `strict` (optional), `input_examples`, `cache_control`, `defer_loading`, `allowed_callers`

### Strict Mode (VERIFIED)

```json
{
  "name": "extract_invoice",
  "strict": true,
  "input_schema": {
    "type": "object",
    "properties": { ... },
    "required": ["vendor_name", "invoice_number", "total_amount", "currency"],
    "additionalProperties": false
  }
}
```

Requirements:
- `additionalProperties: false` must be set on every object
- ALL properties must be in `required`
- Only a subset of JSON Schema supported

---

## 2.2 Structured Error Responses

### Error Categories (VERIFIED)

| Category | Retryable? | Agent Action |
|----------|-----------|--------------|
| Transient | Yes | Retry with backoff |
| Validation | No (without input changes) | Fix input, retry |
| Business | No | Escalate or inform user |
| Permission | No | Escalate |

### Anti-Patterns

1. **Generic error messages:** `"Operation failed"` — hides cause, prevents recovery
2. **Silent suppression:** Returning `{ "results": [] }` when search is down — agent says "found nothing" when truth is "couldn't search"
3. **Workflow termination on single failure:** Should continue with partial results

### Empty vs Failed (KEY EXAM CONCEPT)

| Situation | What Happened | Correct Response |
|-----------|---------------|------------------|
| Search returns `[]` with status 200 | Valid query, no matches | "No results found" |
| Search returns timeout error | Query didn't execute | "I wasn't able to search, let me try again" |

---

## 2.3 Tool Distribution and tool_choice

### Tool Consolidation Principle (CORRECTED)

The documented best practice is: "Consolidate related operations into fewer tools" and "Fewer, more capable tools reduce selection ambiguity."

**Note:** No specific numeric maximum (like "4-5 tools") appears in current documentation. The principle is quality over quantity — well-designed tools beat many narrow ones. For the exam, the concept of distributing tools across specialized subagents when there are too many is still the correct architectural approach.

### tool_choice Parameter (VERIFIED)

| Value | Behavior | Use Case |
|-------|----------|----------|
| `"auto"` (default) | Claude may call a tool OR respond with text | General conversation |
| `"any"` | Claude MUST call a tool, can choose which | Guarantee structured output |
| `{"type": "tool", "name": "specific_tool"}` | Claude MUST call this exact tool | Force a specific operation first |
| `"none"` | Claude cannot call any tool; text only | Force text while keeping tools in context |

### Parallel Tool Use Control (VERIFIED)

By default, Claude may call multiple tools in a single response. To restrict to one tool per turn:

```json
{
  "tool_choice": {"type": "auto", "disable_parallel_tool_use": true}
}
```

---

## 2.4 MCP Server Configuration (CORRECTED)

### What is MCP?

The Model Context Protocol is an open standard for connecting AI models to external data sources and tools.

### Architecture

```
┌─────────────────────────────────────┐
│              Host                    │
│          (Claude Code)              │
│  ┌──────────┐  ┌──────────┐       │
│  │ Client 1 │  │ Client 2 │  ...  │
│  └─────┬────┘  └─────┬────┘       │
└────────┼──────────────┼────────────┘
         │              │
         ↓              ↓
   ┌──────────┐   ┌──────────┐
   │ Server 1 │   │ Server 2 │
   │ (GitHub)  │   │(Postgres)│
   └──────────┘   └──────────┘
```

### Transport Types (CORRECTED)

| Transport | How It Works | When to Use | Status |
|-----------|-------------|-------------|--------|
| stdio | Standard input/output | Local process communication. Most common for local tools. | Current |
| Streamable HTTP | HTTP POST for client→server, optional SSE for streaming | Remote servers, cloud-hosted tools | Current |
| SSE | Server-Sent Events | Legacy remote transport | DEPRECATED |
| WebSocket | Persistent bidirectional connection | Push events, real-time notification | UNVERIFIED for protocol v2026-07-28 |

**Note:** stdio and Streamable HTTP are the confirmed primary transports. SSE is deprecated but still supported. WebSocket is referenced in some Claude Code docs but not independently verifiable from the MCP specification at time of writing.

### Protocol (CORRECTED)

Uses JSON-RPC 2.0 message format.

**Lifecycle:**

Confirmed (all protocol versions):
1. Capability negotiation — Client and server agree on supported features
2. `tools/list` — Client discovers available tools
3. `tools/call` — Client invokes a tool

**Note:** The original spec uses `initialize` → `notifications/initialized` for the negotiation step. Some sources reference `server/discover` as a replacement in protocol v2026-07-28 — this is plausible but unverifiable from primary MCP specification docs. For the exam, know that negotiation happens first, then tool discovery, then tool invocation.

### Server Primitives (VERIFIED)

| Primitive | What It Is | Example |
|-----------|-----------|---------|
| Tools | Executable functions | `search_database`, `create_issue` |
| Resources | Data sources (read-only content catalogs) | File listings, database schemas |
| Prompts | Interaction templates | Pre-built prompt workflows |

### Client Primitives (CORRECTED)

| Primitive | What It Is | Status |
|-----------|-----------|--------|
| Elicitation | Server requests user input from the client | Current |
| Sampling | Server requests an LLM completion from the client | DEPRECATED (protocol 2026-07-28) |

**Correction:** "Roots" is NOT a client primitive. The `roots/list` is a server concept (servers advertise allowed directory roots). "Logging" is also deprecated — use stderr or OpenTelemetry instead.

### Project-Level Configuration (`.mcp.json`) (VERIFIED)

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

- `${GITHUB_TOKEN}` — Environment variable expansion (secret NOT in config)
- `command` + `args` — How to start the MCP server process (stdio transport)
- Tools from ALL configured servers are discovered at connection

### User-Level Configuration (`~/.claude.json`) (VERIFIED)

For personal servers not in project version control.

---

## 2.5 Built-in Tools (CORRECTED — Expanded)

### Core File/Search Tools

| Tool | Purpose | When to Use |
|------|---------|-------------|
| Grep | Search file contents for patterns | Finding function definitions, imports, string patterns |
| Glob | Find files by name/extension | Locating files: `**/*.test.ts`, `src/**/config.*` |
| Read | Load full file contents | Reading a specific file |
| Write | Create new files | Creating files that don't exist |
| Edit | Targeted modifications | Changing specific sections (uses unique text matching) |
| Bash | Run terminal commands | git, npm, running scripts |

### Additional Built-in Tools (40+ Total)

| Category | Tools |
|----------|-------|
| Web | WebFetch, WebSearch |
| Workflow | Agent, Skill, Workflow |
| Task management | TaskCreate, TaskGet, TaskList, TaskUpdate, TaskStop |
| Planning | EnterPlanMode, ExitPlanMode |
| Worktrees | EnterWorktree, ExitWorktree |
| Communication | SendMessage, AskUserQuestion |
| MCP | ToolSearch, ListMcpResources, ReadMcpResource |
| Scheduling | CronCreate, CronDelete, CronList |

### Codebase Exploration Pattern

1. Grep for entry points (main functions, route definitions)
2. Read the entry point files
3. Follow imports — Grep for referenced modules
4. Read those files
5. Build understanding iteratively

Anti-pattern: Reading all files upfront wastes context.

---

# Domain 3: Claude Code Configuration & Workflows (20%)

## 3.1 CLAUDE.md Hierarchy (VERIFIED)

### Loading Order (Broader → More Specific)

1. **Managed Policy** (org-wide, admin-controlled)
   - Linux/WSL: `/etc/claude-code/CLAUDE.md`
   - macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`
   - Windows: `C:\Program Files\ClaudeCode\CLAUDE.md`
2. **User-Level** (personal, NOT version-controlled)
   - `~/.claude/CLAUDE.md`
3. **Project-Level** (shared via version control)
   - `./CLAUDE.md` or `./.claude/CLAUDE.md`
4. **Local Project** (personal, gitignored)
   - `./CLAUDE.local.md`
5. **Directory-Level** (loaded on demand)
   - `./src/api/CLAUDE.md` (loaded when Claude reads files in that dir)

Content loads root-to-current-directory, so project-specific instructions appear after user instructions. Within each directory, `CLAUDE.local.md` appends after `CLAUDE.md`.

### @import Syntax (CORRECTED)

```markdown
# Project Configuration
@docs/coding-standards.md
@docs/api-conventions.md
@~/.claude/my-personal-preferences.md
```

Resolution rules:
- Relative paths resolve relative to the file containing the import
- `@~/` resolves to the user's home directory
- **Maximum import depth: 4 hops** (not 5 — corrected per docs)
- Imported files can themselves contain @imports (up to depth limit)
- `` `@README` `` (backticks) keeps the text literal without importing

### .claude/rules/ Directory (VERIFIED)

```
.claude/
  rules/
    testing.md       — Testing conventions
    api-design.md    — API endpoint patterns
    security.md      — Security requirements
```

- Without frontmatter: Rules load unconditionally at session start
- With frontmatter: Rules load conditionally based on file paths
- Max budget: 1,000 expanded patterns and 4 MiB per rule's paths list
- Symlinks resolved and loaded

### Auto Memory (VERIFIED)

Location: `~/.claude/projects/<project>/memory/MEMORY.md`

- First 200 lines OR 25 KB (whichever comes first) loaded at session start
- Machine-local (not version controlled)
- All worktrees in the same git repo share one memory directory
- Topic files alongside MEMORY.md loaded on demand
- Configurable via `autoMemoryDirectory` in settings
- `autoMemoryEnabled: false` disables it

---

## 3.2 Skills (VERIFIED)

### Skill Locations (Priority: Enterprise > Personal > Project > Plugin)

| Location | Path | Applies To |
|----------|------|------------|
| Enterprise | Managed settings path | All users in org |
| Personal | `~/.claude/skills/<name>/SKILL.md` | All your projects |
| Project | `.claude/skills/<name>/SKILL.md` | This project only |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | Where plugin enabled |

Legacy `.claude/commands/*.md` still works. When a skill and command share a name, the skill wins. Skills follow the open Agent Skills standard.

### Example SKILL.md

```markdown
---
name: deploy
description: Deploy the application to a specified environment
context: fork
agent: Explore
allowed-tools: Bash(git *) Bash(npm run deploy *)
argument-hint: [environment]
---

# Deploy Skill

Deploy the application to $ARGUMENTS.

1. Run the test suite
2. Build the application
3. Push to the deployment target
```

### Key Frontmatter Fields

| Field | Purpose |
|-------|---------|
| `name` | Slash-command name (defaults to directory name) |
| `description` | When to use — Claude uses for auto-load decisions |
| `argument-hint` | Hint shown in autocomplete |
| `context` | `fork` runs in isolated subagent |
| `agent` | Which subagent type (e.g., `Explore`, `Plan`) |
| `allowed-tools` | Tools pre-approved while skill active |
| `model` | Model override |
| `effort` | `low` / `medium` / `high` / `xhigh` / `max` |
| `paths` | Glob patterns — auto-load with matching files |
| `user-invocable` | `false` → hidden from `/` menu |

### Dynamic Context Injection

```markdown
## PR context
- Diff: !`gh pr diff`
- Comments: !`gh pr view --comments`
```

`!`cmd`` runs the command before the skill body is sent to Claude.

---

## 3.3 Path-Specific Rules (VERIFIED)

```yaml
---
paths:
  - "src/api/**/*.ts"
  - "src/api/**/*.test.ts"
---

# API Development Rules
All API endpoints must:
- Use Zod for request validation
- Return standardized error responses
- Include rate limiting middleware
```

### Why Path-Specific Rules Beat Directory CLAUDE.md

- Single source of truth
- Applies everywhere matching files exist (including new directories)
- No need to duplicate CLAUDE.md files across directories

---

## 3.4 Plan Mode vs Direct Execution (VERIFIED)

### Permission Modes (VERIFIED — 6 modes)

| Mode | Behavior |
|------|----------|
| `default` | Standard permission prompts (displayed as "Manual" in UI) |
| `acceptEdits` | Auto-accept file edits; still prompt for other tools |
| `plan` | Exploration-only; designs approach before touching files |
| `auto` | Classifier decides per-tool whether to prompt |
| `dontAsk` | Never prompt; only pre-approved tools |
| `bypassPermissions` | Skip all prompts (dangerous) |

Start with `--permission-mode <mode>` or cycle with **Shift+Tab** in interactive session.

### When to Use Plan Mode

- Multi-file changes
- Architectural decisions
- Multiple valid approaches
- Large scope (45+ files)
- Unclear requirements

### When to Use Direct Execution

- Single-file fix with clear stack trace
- Adding a null check
- Clear instructions ("Add createdAt to User model")
- Fixing a typo

---

## 3.5 Iterative Refinement Techniques

1. **Concrete Examples** — 2-3 input/output pairs beat vague prose
2. **TDD Iteration** — Write tests first, share failures, iterate
3. **Interview Pattern** — Claude asks clarifying questions before implementing
4. **Multiple vs Independent Issues** — Interacting issues: fix together. Independent: fix sequentially.

---

## 3.6 CI/CD Integration (VERIFIED)

### CLI Flags for CI

| Flag | Purpose | Critical? |
|------|---------|-----------|
| `-p` / `--print` | Non-interactive mode | MUST use in CI |
| `--output-format` | `text`, `json`, or `stream-json` | Machine parsing |
| `--input-format` | `stream-json` for pipeline input | Pipeline composition |
| `--json-schema '{...}'` | Validate output against schema (print mode) | Enforce CI output shape |
| `--max-turns N` | Hard cap on agentic turns | Guardrail |
| `--max-budget-usd X` | Stop at API spend limit (print mode) | Cost guardrail |
| `--bare` | Skip auto-discovery (hooks, skills, plugins, MCP, memory, CLAUDE.md) | Fast scripted starts |
| `--no-session-persistence` | Don't save session to disk | Ephemeral CI runs |
| `--fallback-model` | Auto-fall back when primary overloaded | Reliability |
| `--permission-mode` | Start in specific mode | Non-interactive approvals |

### Message Batches API (VERIFIED — 50% savings)

| Feature | Detail |
|---------|--------|
| Cost savings | 50% compared to synchronous API |
| Processing window | Up to 24 hours, no latency SLA |
| Correlation | `custom_id` per request |
| Multi-turn | NOT supported within a single batch request |

**Good for:** Overnight analysis, weekly audits, nightly test generation, bulk processing  
**NOT good for:** Pre-merge checks, real-time interactions, tool calling loops

### Handling Batch Failures

Use `custom_id` to identify and resubmit only failures — don't resubmit the entire batch.

### Session Context Isolation for Reviews

The same session that wrote code is less effective at reviewing it (reasoning context bias). Use independent review instances.

---

# Domain 4: Prompt Engineering & Structured Output (20%)

## 4.1 Explicit Criteria

**Bad (vague):** "Be conservative when flagging issues."  
**Good (explicit):** "Flag a comment as outdated ONLY when the claimed behavior directly contradicts the actual code behavior. Do NOT flag comments that are merely incomplete."

For each judgment:
1. What qualifies — specific conditions
2. What doesn't qualify — boundary cases
3. Examples — concrete input/output pairs

---

## 4.2 Few-Shot Prompting

### When to Use

- Ambiguous scenarios where prose isn't precise enough
- Format/reasoning pattern needs demonstration
- Reducing hallucination in extraction
- Showing edge case handling

### How Many

2-4 targeted examples. Each should include Input, Output, and Reasoning (WHY).

---

## 4.3 Structured Output via tool_use (VERIFIED)

### Method 1: tool_use with tool_choice

Define a tool representing your desired output structure, then force it:

```python
response = client.messages.create(
    model="claude-opus-4-8",
    messages=[...],
    tools=[extract_tool],
    tool_choice={"type": "tool", "name": "extract_invoice_data"}
)
```

### Method 2: output_config.format (VERIFIED — Current Method)

```python
response = client.messages.create(
    model="claude-opus-4-8",
    messages=[...],
    output_config={
        "format": {
            "type": "json_schema",
            "schema": {
                "type": "object",
                "properties": {
                    "vendor_name": { "type": "string" },
                    "total_amount": { "type": "number" }
                },
                "required": ["vendor_name", "total_amount"],
                "additionalProperties": false
            }
        }
    }
)
```

**SDK helper:** `client.messages.parse()` wraps this — pass a Pydantic model or Zod schema and it validates the response automatically.

### Schema Compliance vs Semantic Correctness (KEY EXAM CONCEPT)

Structured output guarantees:
- Correct types, required fields present, valid enums

Does NOT guarantee:
- Semantic correctness (e.g., line items summing to total)
- You still need validation logic for semantic checks

### JSON Schema in Strict Mode — Supported vs Not

| Supported | NOT Supported |
|-----------|---------------|
| type, properties, required | pattern (regex), patternProperties |
| enum, const | if/then/else |
| items (arrays) | Complex array constraints |
| anyOf, allOf, $ref/$defs | additionalProperties set to anything other than false |
| format (date-time, time, date, duration, email, hostname, uri, ipv4, ipv6, uuid) | minimum, maximum, multipleOf (numeric constraints) |
| — | minLength, maxLength (string constraints) |
| — | Recursive schemas |

**Implication:** Format validation IS supported in strict mode. However, numeric range validation (`minimum`, `maximum`), string length validation (`minLength`, `maxLength`), and regex patterns must be handled in your application validation code, not in the schema.

---

## 4.4 Validation-Retry Loops

```python
max_retries = 3
for iteration in range(max_retries):
    response = extract_data(document, prompt)
    errors = validate(response)
    if not errors:
        return response
    prompt += f"\nYour previous extraction had these errors:\n{json.dumps(errors)}\nCorrect these issues."
raise ExtractionError(f"Failed after {max_retries} attempts")
```

Retries don't work when information is absent from the source. Use nullable fields and accept null for missing data.

---

## 4.5 Batch Processing

Always refine prompts on a small sample before batch-processing large volumes.

---

## 4.6 Multi-Pass Review

### The Self-Review Problem

Same-session review retains reasoning context bias. Solution: independent review instances.

### Architecture for Large PRs

- **Pass 1:** Per-file local analysis (separate prompt per file)
- **Pass 2:** Cross-file integration (different prompt focused on interactions)

---

# Domain 5: Context Management & Reliability (15%)

## 5.1 Conversation Context Management

### Progressive Summarization Risks

What gets lost in summarization:
- Exact numerical values (`$149.99` → "approximately $150")
- Specific dates, order numbers, customer IDs
- Exact error messages and stack traces

### Solution: Case Facts Block

```
CASE FACTS (do not summarize):
- Customer ID: #12345
- Order: #67890
- Order date: 2024-01-15
- Total: $149.99
- Issue: Damaged product on arrival
```

### "Lost in the Middle" Effect

Models process beginning and end of long inputs more reliably than the middle. Place key findings at beginning/end.

### What Survives /compact

| Survives | May Be Lost |
|----------|-------------|
| CLAUDE.md (re-read from disk) | Earlier conversation details |
| System prompts | Specific tool results |
| Recent messages | Nuanced reasoning from earlier |
| Auto-memory (MEMORY.md) | Exact quotes/numbers from early context |

---

## 5.2 Escalation Patterns

### When to Escalate Immediately

- Customer explicitly requests a human
- Policy gap or exception
- No progress after reasonable attempts

### When NOT to Escalate (EXAM TRAPS)

- **High sentiment/emotion** — Sentiment does NOT equal complexity
- **Self-reported low confidence** — Confidence scores are poorly calibrated
- **Long conversation** — Length doesn't indicate need for escalation

### Multiple Customer Matches

**Wrong:** Pick the most likely match using heuristics  
**Right:** Ask customer for additional identifying information

---

## 5.3 Error Propagation

### Structured Error Context

```json
{
  "failure_type": "timeout",
  "service": "knowledge_base_search",
  "attempted_query": "return policy for electronics over $500",
  "is_retryable": true,
  "alternative_approaches": ["Try simpler terms", "Check FAQ directly"]
}
```

### Subagent Error Recovery

Subagents implement local recovery for transient failures (retry with backoff). Only propagate errors they cannot resolve. When propagating, include what was attempted, partial results, the failure, and whether it's retryable.

---

## 5.4 Large Codebase Context Management

### Mitigation Strategies

1. **Scratchpad files** — Persist findings to files that survive compression
2. **Subagent delegation** — Isolate verbose exploration; return concise summaries
3. **Proactive /compact** — Better to compact with structured summary than let auto-compression lose info
4. **Crash recovery** — Each agent exports state; coordinator maintains manifest

---

## 5.5 Human Review Workflows

### Stratified Random Sampling

- High-risk: sample 20%
- Medium-risk: sample 10%
- Low-risk: sample 5%

### Per-Document-Type Reporting (KEY CONCEPT)

Aggregate metrics mask per-type failures:

```
Overall accuracy: 97% ← Looks great!
  Standard invoices: 99.5%
  Handwritten notes: 45% ← Terrible
  Foreign language: 62% ← Bad
```

Always break down by type AND field.

---

## 5.6 Information Provenance

### Claim-Source Mappings

```json
{
  "claim": "Revenue increased 15% YoY in Q4 2024",
  "evidence": "Q4 earnings report, page 3",
  "source_url": "...",
  "publication_date": "2025-01-15",
  "source_type": "primary"
}
```

### Handling Conflicting Data

Annotate the conflict rather than silently picking one. Include both values, sources, and resolution reasoning.

### Temporal Data

Include `data_as_of`, `source_published`, and `retrieved_on` dates.

---

# The 10 Critical Anti-Patterns (Wrong Answers)

1. Parsing natural language for loop termination instead of checking `stop_reason`
2. Arbitrary iteration caps as primary stopping mechanism
3. Prompt-based enforcement for critical business rules (should use hooks)
4. Self-reported confidence scores for escalation decisions
5. Sentiment-based escalation (sentiment does not equal complexity)
6. Generic error messages hiding diagnostic context
7. Silently suppressing errors (returning empty results as success)
8. Too many tools per agent (consolidation principle — fewer, well-designed tools)
9. Same-session self-review (retains reasoning context bias)
10. Aggregate accuracy metrics masking per-document-type failures

---

# Key Decision Frameworks

| Decision | Option A | Option B |
|----------|----------|----------|
| Enforcement | Programmatic hooks — financial/safety | Prompt-based — best-effort |
| Execution mode | Plan mode — multi-file, architectural | Direct — single-file, obvious |
| tool_choice | `"any"` — guarantee tool call | `"auto"` — model decides |
| API type | Synchronous — blocking workflows | Batch — latency-tolerant, 50% savings |
| Error handling | Structured context — category, retry, partial | Never generic or silent |
| Escalation | Resolve first — within capability | Immediate — explicit customer request |
| Code review | Multi-pass — large PRs | Single-pass — small, focused |
| Context passing | Always explicit in subagent prompts | Never rely on auto inheritance |

---

# Corrections Applied vs Original Notes

| Topic | Old (Incorrect) | New (Verified) |
|-------|----------------|----------------|
| @import max depth | 5 hops | **4 hops** |
| fork_session | Primary term | **"Branch"** is the concept; `--fork-session` is the CLI flag |
| MCP lifecycle | initialize → notifications/initialized → tools/list → tools/call | **Negotiation → tools/list → tools/call** (exact method unverifiable for v2026-07-28) |
| MCP transports | 2 (stdio, Streamable HTTP) | **3 confirmed** (stdio, Streamable HTTP, SSE deprecated) + WebSocket (unverified) |
| Client primitives | Sampling, Elicitation, Roots | **Elicitation** only current; Sampling deprecated; Roots is NOT a client primitive |
| Tool maximum | "4-5 tools" as documented rule | **Not documented** — principle is "consolidate, fewer is better" |
| Built-in tools | 6 listed | **40+** total; 6 are the core file/search tools |
| disable_parallel_tool_use | Not mentioned | **Valid** on tool_choice object |
| Model recommendations | claude-opus-4-8 default | **claude-fable-5** most capable widely released; **claude-opus-5** latest Opus |

---

---

# The 6 Exam Scenarios

You will face 4 of these 6 scenarios (randomly selected). Each scenario provides context for multiple questions.

## Scenario 1: Customer Support Resolution Agent

**Focus:** Agent SDK, MCP tools, escalation logic

Key concepts tested:
- Agentic loop with `stop_reason` checking
- Tool design for customer lookup, order search, refund processing
- PreToolUse hooks for financial enforcement (identity verification before refund)
- Escalation patterns (when to escalate vs resolve)
- Structured handoff to human agents
- Error handling when customer lookup returns multiple matches

**Common trap:** Using prompt-based enforcement ("always verify identity first") instead of programmatic hooks for refund authorization.

---

## Scenario 2: Code Generation with Claude Code

**Focus:** CLAUDE.md, plan mode, slash commands/skills

Key concepts tested:
- CLAUDE.md hierarchy and loading order
- When to use plan mode vs direct execution
- Skills/commands with frontmatter configuration
- @import syntax (max depth: 4)
- Path-specific rules in `.claude/rules/`
- Permission modes and Shift+Tab cycling
- Iterative refinement techniques (TDD, interview pattern)

**Common trap:** Thinking plan mode is always better. Direct execution is correct for single-file, obvious-scope changes.

---

## Scenario 3: Multi-Agent Research System

**Focus:** Coordinator-subagent patterns, error handling

Key concepts tested:
- Hub-and-spoke architecture (coordinator manages subagents)
- Context isolation (subagents do NOT inherit coordinator history)
- Explicit context passing with structured data
- Parallel subagent execution (multiple tool calls in single response)
- Decomposition risks (overly narrow decomposition)
- Iterative refinement loops (evaluate → identify gaps → re-delegate)
- Information provenance and claim-source mappings
- Handling conflicting data across subagent results

**Common trap:** Assuming subagents inherit context automatically. Everything must be explicitly passed.

---

## Scenario 4: Developer Productivity with Claude

**Focus:** Built-in tools, MCP servers, codebase exploration

Key concepts tested:
- Built-in tools (Grep, Glob, Read, Write, Edit, Bash + 40 more)
- Codebase exploration pattern (Grep → Read → follow imports → Read)
- MCP server configuration (`.mcp.json`, env var expansion)
- MCP architecture (Host → Client → Server)
- Transport types (stdio for local, Streamable HTTP for remote)
- Tool consolidation principle (fewer, well-designed tools)
- Subagent delegation for verbose exploration

**Common trap:** Reading all files upfront instead of iterative exploration. Also: giving one agent too many tools instead of distributing across subagents.

---

## Scenario 5: Claude Code for CI/CD

**Focus:** Structured output, batch API, multi-pass review

Key concepts tested:
- CLI flags (`-p`, `--output-format json`, `--json-schema`, `--max-turns`, `--max-budget-usd`)
- `--bare` for fast scripted starts (skips auto-discovery)
- Message Batches API (50% savings, 24h window, `custom_id` correlation)
- Resubmitting only failed batch items (not entire batch)
- Multi-pass review architecture (per-file → cross-file)
- Session context isolation (generation session ≠ review session)
- Structured output via `output_config.format` or `tool_use` + `tool_choice`

**Common trap:** Using batch API for blocking CI checks (needs synchronous). Also: resubmitting entire batch when only some items fail.

---

## Scenario 6: Structured Data Extraction

**Focus:** JSON schemas, tool_use, validation-retry

Key concepts tested:
- Structured output via tool_use with forced tool_choice
- Strict mode requirements (`additionalProperties: false`, all in `required`)
- Schema compliance vs semantic correctness (schema guarantees types, not math)
- Validation-retry loops (append errors to prompt, retry up to N times)
- Nullable fields for missing data (don't force fabrication)
- Per-document-type accuracy reporting (aggregate masks failures)
- Field-level confidence scores (must be calibrated)
- JSON Schema limitations in strict mode (no regex, no format validation)
- `output_config.format` as alternative to tool_use

**Common trap:** Thinking strict mode guarantees correctness. It only guarantees schema compliance — you still need semantic validation.

---

# Practice Questions (All Domains)

## Domain 1: Agentic Architecture

**Q1:** An agentic loop is processing customer requests. After Claude responds, what should the system check to determine the next action?

A) Whether the response contains text content  
B) The `stop_reason` field in the response  
C) The length of the response  
D) Whether Claude expressed confidence in its answer  

**Answer: B** — The `stop_reason` field determines whether to continue the loop (`tool_use`), end it (`end_turn`), or handle edge cases (`pause_turn`, `max_tokens`).

---

**Q2:** A coordinator agent needs to pass customer information to a subagent. What is the correct approach?

A) The subagent will automatically inherit the coordinator's conversation context  
B) Include all relevant customer information explicitly in the subagent's prompt  
C) Store the information in a shared database that both agents access  
D) Use environment variables to pass the context  

**Answer: B** — Subagents have isolated context and do not inherit the coordinator's conversation. Context must be explicitly passed in the prompt.

---

**Q3:** A financial services application needs to ensure identity verification occurs before any refund processing. Which enforcement mechanism should be used?

A) Add instructions to the system prompt requiring verification first  
B) Use a PreToolUse hook that blocks `process_refund` until verification is complete  
C) Train the model on examples where verification comes first  
D) Set a high temperature to encourage creative verification approaches  

**Answer: B** — Financial operations require deterministic, programmatic enforcement. Prompt-based instructions have a non-zero failure rate.

---

**Q4:** A coordinator agent decomposes "research creative industries" into only visual arts subtasks. What is the primary problem?

A) Too many subagents were spawned  
B) Overly narrow decomposition — missing music, writing, film, design  
C) The coordinator should have done the research itself  
D) Visual arts is not a creative industry  

**Answer: B** — Overly narrow decomposition is a tested anti-pattern. The fix is broader initial decomposition with explicit coverage checks.

---

**Q5:** To run three subagents in parallel, the coordinator should:

A) Send three separate messages, one per subagent  
B) Emit multiple Task/Agent tool calls in a single response  
C) Use threading to manage parallel execution  
D) Configure a parallel execution flag in the system prompt  

**Answer: B** — Parallel subagent execution requires multiple tool calls in a single coordinator response. Separate turns = sequential.

---

## Domain 2: Tool Design & MCP

**Q1:** An agent has many tools configured and is frequently selecting the wrong tool. What is the most effective solution?

A) Improve all tool descriptions to be more detailed  
B) Distribute the tools across specialized subagents with fewer tools each  
C) Set tool_choice to "any" to force tool usage  
D) Add examples to the system prompt showing correct tool selection  

**Answer: B** — Consolidating tools and distributing across specialized subagents is the key architectural principle. Fewer, well-designed tools reduce selection ambiguity.

---

**Q2:** A search tool returns an empty array. What should the agent communicate to the user?

A) "The search service is currently unavailable"  
B) "No results were found matching your query"  
C) Check the response status to distinguish between "no results" and "search failed"  
D) Retry the search with different parameters  

**Answer: C** — The agent must distinguish between a valid empty result (status 200, no matches) and a failed search (error/timeout). These require fundamentally different responses.

---

**Q3:** Where should MCP server credentials be stored in a project using `.mcp.json`?

A) Directly in the `.mcp.json` file  
B) In environment variables referenced via `${VAR_NAME}` syntax  
C) In a separate `.mcp-secrets.json` file  
D) In the project's `package.json`  

**Answer: B** — `.mcp.json` supports environment variable expansion. Credentials should never be committed to version control.

---

**Q4:** What is the correct way to signal a tool execution failure to Claude?

A) Return an empty result  
B) Set `is_error: true` in the `tool_result` with a descriptive error message  
C) Throw an exception in the tool handler  
D) Return a JSON object with `"status": "error"`  

**Answer: B** — Setting `is_error: true` in the `tool_result` helps Claude understand the failure and reason about recovery.

---

**Q5:** Which MCP transport is most appropriate for a local development tool?

A) Streamable HTTP  
B) WebSocket  
C) stdio  
D) SSE  

**Answer: C** — stdio is the standard transport for local process communication. No network overhead, most common for local tools.

---

## Domain 3: Claude Code Configuration

**Q1:** A team wants testing conventions to apply to all `*.test.ts` files throughout their codebase. What is the most maintainable approach?

A) Add a CLAUDE.md in every directory that contains test files  
B) Create a single rule file in `.claude/rules/` with `paths: ["**/*.test.ts"]` frontmatter  
C) Add testing rules to the project-level CLAUDE.md  
D) Create a `tests/` directory and put all tests there with a CLAUDE.md  

**Answer: B** — Path-specific rules in `.claude/rules/` are superior when conventions span multiple directories.

---

**Q2:** When should Claude Code use plan mode instead of direct execution?

A) When fixing a single bug with a clear stack trace  
B) When migrating a library that affects 45+ files across the project  
C) When adding a null check to a function  
D) When updating a configuration constant  

**Answer: B** — Plan mode is for multi-file changes with architectural implications and multiple valid approaches.

---

**Q3:** A CI pipeline needs to run Claude Code for automated code review. Which flag is essential?

A) --verbose  
B) -p / --print  
C) --force  
D) --no-cache  

**Answer: B** — The `-p`/`--print` flag enables non-interactive mode, mandatory in CI. Without it, Claude Code hangs waiting for input.

---

**Q4:** What is the maximum @import depth in CLAUDE.md files?

A) 3 hops  
B) 4 hops  
C) 5 hops  
D) Unlimited  

**Answer: B** — Maximum import depth is 4 hops (to prevent circular imports).

---

**Q5:** A skill needs to run in isolation without polluting the main conversation context. Which frontmatter field achieves this?

A) `isolation: true`  
B) `context: fork`  
C) `subagent: true`  
D) `separate-context: true`  

**Answer: B** — `context: fork` spawns the skill as a subagent with its own context. CLAUDE.md still loads but main conversation history does not.

---

## Domain 4: Prompt Engineering

**Q1:** A code review system flags "use your best judgment" as a criterion for reporting issues. What should be changed?

A) Add more examples of past reviews  
B) Replace with explicit criteria defining exactly what qualifies as a reportable issue  
C) Increase the model's temperature for more nuanced judgment  
D) Add a confidence score threshold  

**Answer: B** — Vague instructions produce inconsistent results. Replace with explicit criteria defining what qualifies and what doesn't.

---

**Q2:** An invoice extraction system uses strict mode but sometimes returns a total that doesn't match the sum of line items. What's happening?

A) Strict mode is broken  
B) The schema is misconfigured  
C) Strict mode guarantees schema compliance, not semantic correctness  
D) The model needs fine-tuning  

**Answer: C** — Strict mode ensures schema conformance (correct types, required fields) but cannot validate semantic correctness (whether math adds up).

---

**Q3:** A batch processing job has 1,000 documents. 50 fail. What's the correct approach?

A) Resubmit the entire batch of 1,000  
B) Use `custom_id` to identify and resubmit only the 50 failed documents  
C) Switch to synchronous processing  
D) Increase the batch timeout  

**Answer: B** — Use `custom_id` to correlate requests with responses and resubmit only failures.

---

**Q4:** Which method guarantees Claude returns structured JSON output?

A) Ask Claude nicely to return JSON in the system prompt  
B) Set `output_config.format` with `type: "json_schema"` or use `tool_choice` with a forced tool  
C) Use `temperature: 0` to get deterministic output  
D) Parse Claude's text response with regex  

**Answer: B** — `output_config.format` with a JSON schema, or forcing a tool via `tool_choice`, are the two guaranteed methods for structured output.

---

**Q5:** A validation-retry loop keeps retrying because the invoice has no PO number. What's wrong?

A) The retry limit is too low  
B) The schema should use a nullable field for optional data rather than forcing fabrication  
C) The model needs better examples  
D) The validation logic is too strict  

**Answer: B** — Retries are ineffective when information is absent from the source. Use nullable/optional fields and accept null for missing data.

---

## Domain 5: Context Management & Reliability

**Q1:** An agent is processing a customer support request. After context compression, the customer's order number was summarized as "a recent order." What should have been done?

A) Increase the context window size  
B) Extract critical transactional data into a persistent case facts block  
C) Disable context compression  
D) Repeat the order details in every message  

**Answer: B** — Extracting transactional facts into a structured block that persists through compression prevents loss of critical details.

---

**Q2:** A customer says "I AM SO ANGRY RIGHT NOW!!!" about a simple $5 billing charge. Should the agent escalate?

A) Yes, high sentiment indicates a complex issue  
B) Yes, capital letters indicate urgency  
C) No, sentiment intensity does not indicate issue complexity — resolve the simple issue  
D) No, but reduce the agent's confidence score  

**Answer: C** — Sentiment does NOT equal complexity. The agent should resolve the simple issue while acknowledging frustration.

---

**Q3:** An extraction system reports 97% overall accuracy. Is it ready for production?

A) Yes, 97% is excellent  
B) Check if accuracy is consistent across all document types and fields  
C) Run more documents to increase sample size  
D) Compare against competitor systems  

**Answer: B** — Aggregate accuracy can mask poor performance on specific types. 97% overall might hide 45% accuracy on handwritten documents.

---

**Q4:** Two research subagents return different values for the same metric. What should the coordinator do?

A) Average the two values  
B) Use the most recent value  
C) Annotate the conflict with source attribution and let the user decide  
D) Discard both and search again  

**Answer: C** — Conflicting data should be annotated with source attribution rather than silently resolved.

---

**Q5:** During a long session, Claude starts referencing "typical patterns" instead of specific code found earlier. What's happening?

A) The model is hallucinating  
B) Context degradation — earlier findings are being lost or compressed  
C) The codebase changed  
D) The model is lazy  

**Answer: B** — Context degradation in extended sessions. Mitigation: use scratchpad files, subagent delegation, or proactive `/compact`.

---

# 4-Week Study Plan

## Week 1: Core Architecture (Days 1-7)

**Daily goal:** 1 domain section + practice questions (1.5-2 hours/day)

| Day | Topic | Focus |
|-----|-------|-------|
| 1 | D1.1 | Agentic loops, `stop_reason` values, API response structure |
| 2 | D1.2-1.3 | Multi-agent orchestration, subagent spawning, context passing |
| 3 | D1.4-1.5 | Workflow enforcement (hooks vs prompts), hook events/types |
| 4 | D1.6-1.7 | Task decomposition, session management, branching |
| 5 | D2.1-2.2 | Tool descriptions, structured errors, empty vs failed |
| 6 | D2.3-2.4 | Tool distribution, tool_choice, MCP configuration |
| 7 | D2.5 + Review | Built-in tools, codebase exploration + Domain 1-2 practice questions |

## Week 2: Applied Skills (Days 8-14)

**Daily goal:** 1 domain section + build a small hands-on project using each concept

| Day | Topic | Focus |
|-----|-------|-------|
| 8 | D3.1 | CLAUDE.md hierarchy, @import (4 hops), rules directory |
| 9 | D3.2-3.3 | Skills/commands, frontmatter, path-specific rules |
| 10 | D3.4-3.5 | Plan mode vs direct, permission modes, iterative refinement |
| 11 | D3.6 | CI/CD integration, CLI flags, batch API |
| 12 | D4.1-4.2 | Explicit criteria, few-shot prompting |
| 13 | D4.3-4.4 | Structured output (tool_use, output_config), validation-retry |
| 14 | D4.5-4.6 + Review | Batch processing, multi-pass review + Domain 3-4 practice questions |

## Week 3: Reliability + Hands-On (Days 15-21)

**Daily goal:** Review weak areas + build end-to-end scenarios

| Day | Topic | Focus |
|-----|-------|-------|
| 15 | D5.1-5.2 | Context management, case facts blocks, escalation patterns |
| 16 | D5.3-5.4 | Error propagation, large codebase context strategies |
| 17 | D5.5-5.6 | Human review, provenance, conflicting data |
| 18 | Scenarios 1-2 | Customer Support + Code Generation walkthroughs |
| 19 | Scenarios 3-4 | Multi-Agent Research + Developer Productivity walkthroughs |
| 20 | Scenarios 5-6 | CI/CD + Structured Data Extraction walkthroughs |
| 21 | Domain 5 practice + Anti-patterns drill | All 10 anti-patterns cold |

## Week 4: Exam Prep (Days 22-28)

**Daily goal:** 1 full practice exam per day + targeted review

| Day | Topic | Focus |
|-----|-------|-------|
| 22 | Full Mock Exam 1 | 60 questions, 120 minutes, timed |
| 23 | Review Mock 1 | Analyze wrong answers, identify weak domains |
| 24 | Targeted study | Deep-dive weak areas from Mock 1 |
| 25 | Full Mock Exam 2 | 60 questions, 120 minutes, timed |
| 26 | Review Mock 2 | Pattern analysis — which traps caught you? |
| 27 | Full Mock Exam 3 | 60 questions, 120 minutes, timed (target: 75%+) |
| 28 | Final review | Anti-patterns, decision frameworks, cheat sheet, REST |

---

# Resources

| Resource | URL / Location |
|----------|---------------|
| Official Exam Guide v1.0 | Anthropic Partner Academy (effective July 2026) |
| Claude API Docs | platform.claude.com |
| Claude Code Docs | code.claude.com |
| MCP Specification | modelcontextprotocol.io |
| Claude Agent SDK (Python) | `pip install claude-agent-sdk` |
| Claude Agent SDK (TypeScript) | `npm install @anthropic-ai/claude-agent-sdk` |
| Building Effective Agents | Anthropic Research blog |
| Pearson VUE | Anthropic certification program |
| Anthropic Skilljar | Building with Claude API training |
| Claude Partner Network | Registration for exam access |

---

# Program Details (July 2026)

## Certification Family

| Credential | Code | Target Audience |
|------------|------|-----------------|
| Claude Certified Associate — Foundations | CCAO-F | Entry-level Claude users |
| Claude Certified Architect — Foundations | CCAR-F | Solution architects (this exam) |
| Claude Certified Architect — Professional | CCAR-P | Senior architects |
| Claude Certified Developer — Foundations | CCDV-F | Developers building with Claude |

## Exam Logistics

- **Delivery:** Pearson VUE (OnVUE online or test center), proctored
- **Registration:** Through Anthropic Partner Academy
- **Fee:** $125 per attempt
- **Validity:** 12 months
- **Attempts:** Up to 4 per rolling 12 months
  - After attempt 1: 14-day wait
  - After attempt 2: 30-day wait
  - After attempt 3: 90-day wait
- **Access:** Gated to Claude Partner Network members
- **Previous terms superseded:** "$99, first 5,000 partner employees free" beta terms no longer apply

## Target Candidate

Solution architect with 6+ months experience with:
- Claude APIs
- Claude Agent SDK
- Claude Code
- Model Context Protocol (MCP)

## Documentation Split (2026)

Anthropic's docs were split into:
- **platform.claude.com** — API documentation
- **code.claude.com** — Claude Code documentation

The SDK was renamed from "Claude Code SDK" to **Claude Agent SDK**.

---

# Exam-Day Quick Reference

## Top 5 Rules

1. **stop_reason drives the loop** — never parse text for termination
2. **Hooks for enforcement, prompts for guidance** — financial/safety = hooks
3. **Context is never inherited** — always pass explicitly to subagents
4. **Empty ≠ Failed** — distinguish valid empty results from errors
5. **Aggregate masks specifics** — always break down by type/field

## If You're Stuck on a Question

- Eliminate anti-patterns first (they're usually 2 of the 4 choices)
- Look for the "programmatic" answer for safety/financial scenarios
- Look for the "explicit" answer for context/communication scenarios
- "Sentiment" and "confidence" are almost always wrong triggers for escalation
- "All files at once" is almost always wrong for exploration/review

## Time Management

- 60 questions in 120 minutes = 2 minutes per question
- Flag difficult questions and return to them
- Don't spend more than 3 minutes on any single question
- No penalty for guessing — never leave blank

---

**Verified:** August 5, 2026  
**Sources:** platform.claude.com, code.claude.com, modelcontextprotocol.io (protocol v2026-07-28)
