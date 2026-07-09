# Domain 1 | Topic 1.3: Hooks, Workflows & Session Management

## 1. Core Concepts

### Hooks

**Hooks** are user-defined shell commands that execute automatically in response to specific events in Claude Code. They enable automation, validation, and custom workflows without modifying Claude's core behavior.

#### Hook Types

| Hook Event | When It Fires | Use Case |
|-----------|---------------|----------|
| `PreToolUse` | Before a tool is executed | Block dangerous commands, validate inputs |
| `PostToolUse` | After a tool completes | Log results, trigger follow-up actions |
| `Notification` | When Claude sends a notification | Custom notification routing |
| `Stop` | When Claude finishes responding | Post-processing, auto-commit |

#### Hook Configuration (settings.json)

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "python validate_command.py"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "python format_written_file.py"
      }
    ],
    "Stop": [
      {
        "command": "python notify_done.py"
      }
    ]
  }
}
```

#### Hook Behavior
- Hooks run **synchronously** — Claude waits for them to complete
- **Exit code 0** = success, hook passes
- **Non-zero exit code** = failure, blocks the tool execution (for PreToolUse)
- Hook stdout is fed back to Claude as context
- Hooks receive data via **STDIN as a JSON object** (NOT environment variables)

#### Hook Input (STDIN JSON Object)

Hooks receive a JSON object on STDIN containing contextual fields:

| Field | Description | Available In |
|-------|-------------|--------------|
| `tool_name` | Name of the tool being called | PreToolUse, PostToolUse |
| `tool_input` | Object with tool input parameters | PreToolUse, PostToolUse |
| `tool_output` | Tool execution result | PostToolUse only |
| `session_id` | Current session identifier | All hooks |
| `transcript_path` | Path to the conversation transcript | Stop |

Example hook script reading from STDIN:

```python
#!/usr/bin/env python3
import json
import sys

# Hook data is passed via STDIN as a JSON object
hook_data = json.loads(sys.stdin.read())

tool_name = hook_data.get("tool_name")
tool_input = hook_data.get("tool_input", {})
session_id = hook_data.get("session_id")

# Example: block dangerous commands
if tool_name == "Bash":
    command = tool_input.get("command", "")
    if "rm -rf" in command:
        print("Blocked: dangerous rm -rf command")
        sys.exit(1)

sys.exit(0)  # Allow execution
```

### Workflows

**Workflows** in the Claude architect context refer to structured sequences of operations that combine:
- Multiple tool calls in a defined order
- Conditional branching based on results
- Human-in-the-loop checkpoints
- Retry and error handling logic

#### Workflow Patterns

```
LINEAR WORKFLOW
━━━━━━━━━━━━━━
Step 1 → Step 2 → Step 3 → Done

CONDITIONAL WORKFLOW
━━━━━━━━━━━━━━━━━━
Step 1 → [Condition] → Step 2A (if true)
                     → Step 2B (if false)

LOOP WORKFLOW
━━━━━━━━━━━━
Step 1 → Step 2 → [Check] → Done (if pass)
                          → Step 1 (if fail, retry)

PARALLEL WORKFLOW
━━━━━━━━━━━━━━━━
         ┌→ Step 2A ─┐
Step 1 ──┤            ├──→ Step 3
         └→ Step 2B ─┘
```

### Session Management

**Sessions** represent a continuous interaction with Claude, maintaining:
- Message history (the conversation context)
- Working directory state
- Tool permissions granted
- Active plan state

#### Session Lifecycle

```
Start Session → Load Context → [Interactive Loop] → End Session
                     ↑                                    │
                     └── Resume (with compaction) ←────────┘
```

#### Context Compaction

When a session approaches context limits:
1. System detects approaching limit
2. Prior messages are **summarized** (progressive summarization)
3. Summary replaces original messages
4. Session continues with condensed context

---

## 2. Design Patterns

### Pattern 1: Pre-Commit Validation Hook

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "python scripts/validate_no_secrets.py"
      }
    ]
  }
}
```

The hook script reads the tool input from STDIN:
```python
#!/usr/bin/env python3
# scripts/validate_no_secrets.py
import json, sys, re

hook_data = json.loads(sys.stdin.read())
command = hook_data.get("tool_input", {}).get("command", "")

# Check for secrets patterns in the command
secret_patterns = [r'password\s*=', r'api_key\s*=', r'secret\s*=']
for pattern in secret_patterns:
    if re.search(pattern, command, re.IGNORECASE):
        print(f"Blocked: potential secret in command")
        sys.exit(1)

sys.exit(0)
```

- Blocks commits containing secrets
- Runs before every Bash command
- Returns non-zero to prevent execution

### Pattern 2: Auto-Format on Write

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "python scripts/format_on_write.py"
      }
    ]
  }
}
```

The hook script extracts the file path from STDIN JSON and formats it:
```python
#!/usr/bin/env python3
# scripts/format_on_write.py
import json, sys, subprocess

hook_data = json.loads(sys.stdin.read())
file_path = hook_data.get("tool_input", {}).get("file_path", "")

if file_path.endswith((".js", ".ts", ".jsx", ".tsx")):
    subprocess.run(["npx", "prettier", "--write", file_path])
    subprocess.run(["npx", "eslint", "--fix", file_path])
```

- Automatically formats files after Claude writes them
- Ensures consistent code style
- Claude sees the formatted result

### Pattern 3: Workflow with Human Checkpoint

```python
def review_workflow(pr_url):
    # Step 1: Automated analysis
    analysis = agent.analyze_pr(pr_url)
    
    # Step 2: Human checkpoint
    if analysis.risk_level == "high":
        approval = get_human_approval(analysis.summary)
        if not approval:
            return "Blocked by human review"
    
    # Step 3: Continue with approved action
    return agent.apply_suggestions(analysis)
```

### Pattern 4: Session Continuity with Memory

```
Session 1: User works on feature X
  → Claude builds context about codebase
  → Session ends (context saved/summarized)

Session 2: User returns to continue
  → CLAUDE.md provides persistent context
  → Memory files provide learned preferences
  → Reduced ramp-up time
```

---

## 3. Implementation Details

### Hook Matcher Syntax

| Matcher | Matches |
|---------|---------|
| `"Bash"` | All Bash tool calls |
| `"Write"` | All Write tool calls |
| `"Edit"` | All Edit tool calls |
| `"Read"` | All Read tool calls |
| `"*"` | All tool calls |
| `"Bash(npm *)"` | Bash commands starting with `npm` |

### Settings Hierarchy for Hooks

```
~/.claude/settings.json          (User-level, all projects)
.claude/settings.json            (Project-level, shared via git)
.claude/settings.local.json      (Project-level, local only)
```

- More specific settings override general ones
- Project-level hooks run IN ADDITION to user-level hooks
- Local settings take precedence over shared project settings

### Subagent Invocation

**Subagents** are autonomous agents spawned by a coordinator agent to handle specialized tasks. The coordinator maintains context and orchestrates subagent execution.

#### Task Tool: Spawning Subagents

The **Task tool** is how coordinators spawn subagents:

```python
# Coordinator agent invokes Task tool to spawn subagent
response = client.messages.create(
    model="claude-opus-4-6-20250514",
    messages=[
        {"role": "user", "content": "Analyze this PR for security issues"}
    ],
    tools=[
        {
            "name": "task",
            "description": "Spawn a subagent to handle a specific task",
            "input_schema": {
                "type": "object",
                "properties": {
                    "type": {"type": "string", "enum": ["task"]},
                    "agent_type": {"type": "string", "description": "Type of subagent (e.g., 'security_reviewer')"},
                    "prompt": {"type": "string", "description": "Task prompt for the subagent"},
                },
                "required": ["type", "agent_type", "prompt"]
            }
        }
    ]
)
```

#### allowedTools Requirement

**CRITICAL:** For a coordinator to spawn subagents, its `allowedTools` configuration MUST include `"task"`:

```python
# Agent definition for coordinator
coordinator = {
    "name": "coordinator",
    "description": "Orchestrates multi-agent research system",
    "allowedTools": [
        "task",           # ← MUST include to spawn subagents
        "read_document",  # Coordinator's own tools
        "search_web"
    ]
}

# Without "task" in allowedTools, the agent cannot invoke Task tool
```

#### Context Passing to Subagents

Subagents receive **isolated context** — they do NOT automatically inherit the coordinator's conversation history. Pass context explicitly:

```python
# WRONG: Context not passed
subagent_prompt = "Analyze this codebase for bugs"

# RIGHT: Context passed explicitly with findings
web_search_results = "[Search results for 'async patterns']"
prior_findings = "Found: Memory leak in connection pooling, Race condition in cache invalidation"

subagent_prompt = f"""Analyze this codebase for bugs.

Prior findings from other agents:
{prior_findings}

Web search context:
{web_search_results}

Your task: Identify security-related bugs."""
```

**Key patterns:**
- Include complete findings from prior agents
- Pass metadata (source URLs, document names, page numbers)
- Use structured format (YAML/JSON) for clarity
- Coordinator must re-inject context on each subagent invocation

#### Parallel Subagent Spawning

Multiple subagents can be spawned in parallel by emitting multiple Task tool calls in one coordinator response:

```python
# Coordinator response with 3 parallel Task calls
response = {
    "role": "assistant",
    "content": [
        {
            "type": "tool_use",
            "id": "toolu_01",
            "name": "task",
            "input": {
                "agent_type": "search_agent",
                "prompt": "Search for: async/await patterns in Node.js"
            }
        },
        {
            "type": "tool_use",
            "id": "toolu_02",
            "name": "task",
            "input": {
                "agent_type": "document_analyzer",
                "prompt": "Analyze: Best practices documentation"
            }
        },
        {
            "type": "tool_use",
            "id": "toolu_03",
            "name": "task",
            "input": {
                "agent_type": "synthesizer",
                "prompt": "Synthesize findings (will receive results from above)"
            }
        }
    ]
}
# All 3 subagents execute in parallel, results returned together
```

#### Subagent Return Format

Subagents return structured results to the coordinator:

```python
# Subagent returns findings as tool_result
{
    "type": "tool_result",
    "tool_use_id": "toolu_01",
    "content": json.dumps({
        "findings": [
            {
                "claim": "Async patterns reduce callback nesting",
                "source": "https://nodejs.org/en/docs/guides/",
                "confidence": 0.95
            }
        ],
        "coverage": "80% of async patterns covered",
        "gaps": "Missing: Promise.race() patterns"
    })
}
```

### Session State Management

```
Conversation State:
├── messages[]          — Full message history
├── system_prompt       — From CLAUDE.md + settings
├── tool_permissions    — What's been allowed this session
├── working_directory   — Current filesystem context
├── plan               — Active plan (if in plan mode)
└── subagents[]        — Spawned subagent contexts (isolated)
```

### Workflow Implementation with Tool Chaining

```python
# Claude Code implements workflows through tool chaining in the agentic loop
# Each tool result informs the next tool call

# Example: Test-Fix-Verify workflow
# 1. Run tests → get failures
# 2. Read failing test file → understand expectation
# 3. Read source file → find bug
# 4. Edit source file → apply fix
# 5. Run tests again → verify fix
# Loop back to step 2 if still failing
```

---

## 4. Security & Reliability

### Hook Security Considerations

| Risk | Mitigation |
|------|-----------|
| Malicious hooks in shared settings | Review .claude/settings.json in PRs |
| Hook command injection | Validate STDIN JSON fields before use |
| Hook timeout | Set reasonable timeouts |
| Hook accessing secrets | Limit hook process environment |
| Hooks bypassed | Critical validation should be in CI, not just hooks |

### Session Security

- **Permission grants are session-scoped** — new session requires re-approval
- **No automatic carry-over of dangerous permissions**
- **Session isolation** — one session can't access another's state
- **Context compaction** — summarization may lose security-relevant details

### Workflow Reliability

| Concern | Solution |
|---------|----------|
| Step fails mid-workflow | Retry logic + idempotent operations |
| Context lost between steps | Explicit state passing between steps |
| Human checkpoint timeout | Default-deny after timeout period |
| Non-deterministic results | Validation step after each action |

### Subagent Reliability

| Risk | Mitigation |
|------|-----------|
| Subagent lacks context | Pass all prior findings explicitly in prompt |
| Subagent hallucinating | Include source data in prompt, verify output |
| Coordinator loses results | Use tool_use_id matching to correlate results |
| Context explosion | Subagents have isolated context, coordinator synthesizes |
| Stale context | Inject timestamp and freshness checks in subagent prompts |

---

## Session Resumption & Forking

### Session Resumption with --resume

**Named sessions** allow resuming prior conversations with context preservation:

```bash
# Start named session
claude --session research-async "Analyze async patterns in Node.js"

# Session ends, context saved
# Later: resume the session
claude --resume research-async
```

#### How Resumption Works

```
Turn 1: Start session "research-async"
  ├─ Upload codebase context
  ├─ Ask: "Analyze async patterns"
  └─ Claude: [Analysis + observations]
      └─ Context saved to .claude/sessions/research-async

Turn 2: Resume session (days/weeks later)
  ├─ Load context from .claude/sessions/research-async
  ├─ Messages: [prior conversation]
  ├─ Ask: "Now check file async-utils.ts"
  └─ Claude: [With prior context about patterns already analyzed]
```

#### When to Resume vs Start Fresh

| Scenario | Approach | Reason |
|----------|----------|--------|
| Continuing investigation same day | --resume | Prior context still valid |
| After code changes to prior files | Start fresh with summary | Tool results stale |
| Different angle on same codebase | --fork-session | Divergent exploration |
| After infrastructure changes | Start fresh | Context invalidated |

#### Informing Session About Changes

When resuming after code modifications, inform the agent:

```bash
# Resume session
claude --resume research-async "Updated files: src/utils/async.ts (new retry logic added)"
```

Claude then knows to re-analyze specific files rather than relying on stale tool results.

### fork_session for Divergent Exploration

**fork_session** creates an independent branch from a shared analysis baseline:

```bash
# Initial analysis
claude --session main-analysis "Analyze refactoring approaches for legacy code"
# Claude explores, provides analysis

# Fork to explore approach A
claude --fork-session main-analysis branch-a "Explore approach A: microservices"
# Divergent investigation in isolated context

# Fork to explore approach B
claude --fork-session main-analysis branch-b "Explore approach B: modular monolith"
# Parallel investigation without interfering

# Return to main session to compare both approaches
claude --resume main-analysis "Compare branch-a and branch-b findings"
```

#### fork_session Use Cases

- **A/B testing strategies:** Compare two refactoring approaches in parallel
- **Risk mitigation:** Explore a risky change in isolation before committing
- **Learning:** Test multiple explanations without polluting main session
- **Collaboration:** Different team members explore different branches

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A team wants to ensure that Claude never executes `rm -rf` commands in their project. What is the MOST effective approach?

**A:** Configure a `PreToolUse` hook with matcher `"Bash"` that reads the tool input from STDIN and inspects it for dangerous patterns like `rm -rf`, `rm -r /`, etc. The hook script exits with non-zero code if a dangerous command is detected, blocking execution.

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "command": "python .claude/scripts/block_dangerous.py"
    }]
  }
}
```

The hook reads the command from the STDIN JSON object:
```python
#!/usr/bin/env python3
# .claude/scripts/block_dangerous.py
import json, sys

hook_data = json.loads(sys.stdin.read())
command = hook_data.get("tool_input", {}).get("command", "")

dangerous = ["rm -rf", "rm -r /", "mkfs", "dd if="]
if any(d in command for d in dangerous):
    print(f"BLOCKED: dangerous command detected")
    sys.exit(1)
sys.exit(0)
```

**Why not other approaches:**
- Relying on Claude's system prompt alone = not a hard guarantee
- Permission mode helps but user might approve by accident
- Post-execution hook = too late, damage done

### Scenario 2
**Q:** A developer's Claude Code sessions frequently hit context limits during long refactoring tasks. What architectural approach BEST maintains continuity?

**A:** Combine:
1. **CLAUDE.md documentation** for persistent project context that survives compaction
2. **Plan mode** to maintain structured progress tracking
3. **Progressive summarization** (built-in) to compress old messages
4. **Task decomposition** — break large refactors into smaller, session-sized chunks

The key insight: critical context belongs in persistent files (CLAUDE.md, memory), not just in message history.

### Scenario 3
**Q:** A workflow requires: (1) generate code, (2) run linter, (3) fix lint errors, (4) run tests, (5) fix test failures. Steps 3-4-5 may need to repeat. What pattern applies?

**A:** This is a **validation-retry loop** (a.k.a. fix-verify cycle):

```
Generate → Lint → [Pass?] → Test → [Pass?] → Done
                    ↓ No              ↓ No
                  Fix lint          Fix test
                    ↓                 ↓
                  Lint again       Test again
```

Implement with a bounded retry (max 3 attempts per fix cycle) and escalation if the agent can't resolve after retries.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Heavy hooks on every tool | Slows Claude dramatically | Target hooks to risky tools only |
| Security only in hooks | Hooks can be overridden locally | Defense in depth (hooks + permissions + CI) |
| No hook timeouts | Stuck hook blocks Claude indefinitely | Always set timeouts |
| Relying on session memory for critical context | Lost on compaction | Store in CLAUDE.md or memory files |
| Manual multi-step workflows | Error-prone, inconsistent | Encode as hook chains or slash commands |
| Unbounded retry workflows | Infinite fix-test loops | Max retries + escalation |

---

## 7. Quick Reference / Cheat Sheet

```
HOOK EVENTS
━━━━━━━━━━━
• PreToolUse  — before tool runs (can block)
• PostToolUse — after tool completes
• Notification — when Claude notifies
• Stop         — when Claude finishes turn

HOOK MECHANICS
━━━━━━━━━━━━━━
• Exit 0 = pass, non-zero = block (PreToolUse)
• Stdout → fed back to Claude as context
• Runs synchronously (Claude waits)
• Data passed via STDIN as JSON object
• STDIN fields: tool_name, tool_input, tool_output, session_id

SETTINGS HIERARCHY
━━━━━━━━━━━━━━━━━━
~/.claude/settings.json        (global)
.claude/settings.json          (project, git-tracked)
.claude/settings.local.json    (project, local-only)
Priority: local > project > global

WORKFLOW PATTERNS
━━━━━━━━━━━━━━━━
• Linear: A → B → C
• Conditional: A → [test] → B or C
• Loop/Retry: A → B → [check] → retry or done
• Parallel: A → [B, C] → merge → D

SESSION MANAGEMENT
━━━━━━━━━━━━━━━━━━
• Context compaction = automatic summarization near limits
• Permissions are session-scoped (reset on new session)
• Persistent state lives in CLAUDE.md + memory files
• Plan mode tracks progress across compaction
```
