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
        "command": "python validate_command.py \"$TOOL_INPUT\""
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "npx prettier --write \"$FILE_PATH\""
      }
    ],
    "Stop": [
      {
        "command": "echo 'Claude finished' | notify-send"
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
- Hooks have access to environment variables with tool metadata

#### Hook Environment Variables

| Variable | Description |
|----------|-------------|
| `$TOOL_NAME` | Name of the tool being called |
| `$TOOL_INPUT` | JSON string of tool input parameters |
| `$FILE_PATH` | File path (for file-related tools) |
| `$TOOL_OUTPUT` | Tool output (PostToolUse only) |

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

#### Session Resumption with --resume

The `--resume` CLI flag allows continuing a previous session by name:

```bash
# Session 1: Start a named investigation session
claude --session analysis-pr-123 "Analyze this pull request for performance issues"
# Work continues interactively until user exits

# Session 2: Later, resume the same session (same day)
claude --resume analysis-pr-123 "Check if there are any concurrency bugs"
# Continues with full message history from previous session
# Context is preserved across interruptions
```

**Session resumption behavior:**
- Full conversation history is restored
- Working directory and file context preserved
- Tool permissions remain the same
- If code has changed, you should inform Claude: "I've updated file X since we last worked on this"

**Key limitation:** Sessions expire after 24-48 hours. After expiration, `--resume` won't work — start a new session instead.

**When to use resumption vs. starting fresh:**
- **Resume:** Prior analysis is mostly valid, you're adding new questions, incremental refinement
- **Fresh session:** Code has significantly changed, you want a "fresh eyes" review, prior context would introduce bias

#### fork_session for Divergent Exploration

The `fork_session` mechanism creates independent branches from a shared analysis baseline. This is different from resumption — you're exploring multiple divergent approaches in parallel.

```bash
# Main session: Analyze codebase
claude --session main "Analyze this microservices architecture"
# → Shared understanding of system

# Fork 1: Explore migration strategy A
claude --fork-from main "Now propose a migration to serverless with minimal changes"
# → Independent exploration of approach A

# Fork 2: Explore migration strategy B (from the SAME main baseline)
claude --fork-from main "Now propose a migration to serverless with comprehensive redesign"
# → Independent exploration of approach B (doesn't see fork 1's reasoning)

# Main session: Compare approaches
claude --resume main "Compare the two migration approaches from forks and recommend one"
# → See results from both forks, synthesize
```

**fork_session use cases:**
- A/B testing approaches (two implementation strategies)
- Comparing architectural decisions
- Exploring "what-if" scenarios
- Brainstorming alternatives from a shared baseline

**Key difference from parallel thinking:**
- **fork_session:** Independent sessions, don't see each other's reasoning, then synthesize
- **Within-session multi-pass:** Same session analyzing the same code from different angles (sees prior reasoning)

---

### Subagent Invocation via Task Tool

The **Task tool** is how a coordinator agent spawns subagents. This is the primary mechanism in the Claude Agent SDK for multi-agent orchestration.

#### Task Tool Basics

```python
# Coordinator agent can call the Task tool if allowedTools includes "Task"
response = client.messages.create(
    model="claude-opus-4-6-20250514",
    tools=[
        {
            "name": "task",
            "description": "Spawn a subagent to handle a specific task",
            "input_schema": {
                "type": "object",
                "properties": {
                    "subagent_type": {
                        "type": "string",
                        "description": "Type of subagent (e.g., 'research', 'code', 'review')"
                    },
                    "prompt": {
                        "type": "string",
                        "description": "Task description for the subagent"
                    }
                },
                "required": ["subagent_type", "prompt"]
            }
        }
    ],
    messages=[{"role": "user", "content": "Research topic X and summarize findings"}]
)
```

#### allowedTools Configuration

For a coordinator agent to invoke subagents, its Agent Definition must grant it access to the Task tool:

```python
coordinator_agent = {
    "name": "research-coordinator",
    "description": "Orchestrates research across multiple subagents",
    "instructions": "You coordinate web search, document analysis, and synthesis...",
    "allowedTools": ["Task"]  # ← REQUIRED for subagent spawning
}

subagent_research = {
    "name": "researcher",
    "description": "Performs web searches and summarizes results",
    "instructions": "Search the web for the provided query...",
    "allowedTools": ["WebSearch", "WebFetch"]  # ← Subagent's own tools
}
```

**Critical requirement:** The coordinator's `allowedTools` array MUST include `"Task"` for subagent spawning to work.

#### Context Passing to Subagents

Subagents do NOT automatically inherit parent context. The coordinator must explicitly pass all relevant information in the prompt:

```python
# ❌ WRONG: Coordinator assumes subagent can access prior findings
coordinator_response = task_tool({
    "subagent_type": "synthesis",
    "prompt": "Synthesize the research findings"
})
# Subagent has no idea what findings the coordinator collected

# ✓ CORRECT: Explicit context passing
prior_research = "From search: [findings 1, 2, 3]...\nFrom docs: [findings A, B]..."
coordinator_response = task_tool({
    "subagent_type": "synthesis",
    "prompt": f"""
    Synthesize these research findings into a comprehensive report:
    
    {prior_research}
    
    Create a structured report with: executive summary, key findings, sources.
    """
})
```

**Best practice:** Structured data separates content from metadata:

```python
search_results = [
    {"claim": "X increases Y by 30%", "source": "study-url.com", "date": "2024"},
    {"claim": "X also improves Z", "source": "article.com", "excerpt": "..."}
]

synthesis_prompt = f"""
Synthesize these claims into a report. Preserve source attribution:

{json.dumps(search_results, indent=2)}

Format: Each finding should cite its source URL and publication date.
"""
```

#### Parallel Subagent Spawning

Multiple subagents can be invoked in parallel by emitting multiple Task tool calls in a single coordinator response:

```python
# Coordinator response with 3 parallel task calls
response_content = [
    {
        "type": "tool_use",
        "id": "task_1",
        "name": "task",
        "input": {
            "subagent_type": "researcher",
            "prompt": "Search for recent AI trends in healthcare..."
        }
    },
    {
        "type": "tool_use",
        "id": "task_2",
        "name": "task",
        "input": {
            "subagent_type": "researcher",
            "prompt": "Search for regulatory changes in medical AI..."
        }
    },
    {
        "type": "tool_use",
        "id": "task_3",
        "name": "task",
        "input": {
            "subagent_type": "researcher",
            "prompt": "Analyze market adoption of AI in healthcare..."
        }
    }
]

# All 3 execute in parallel; coordinator receives all results at once
```

**Latency benefit:** 3 parallel searches (1 minute each) takes 1 minute total, vs. 3 sequential searches (3 minutes).

#### fork_session for Divergent Exploration

The `fork_session` mechanism allows exploring multiple approaches from a shared baseline:

```python
# Main coordinator session analyzes codebase
baseline_analysis = "File structure: [...]  Key dependencies: [...]"

# Fork 1: Explore migration approach A
fork_1_prompt = f"""
{baseline_analysis}

Propose a migration strategy that prioritizes speed over stability.
Focus on: minimal changes, quick rollout, accept technical debt.
"""
subagent_A = task(subagent_type="architect", prompt=fork_1_prompt)

# Fork 2: Explore migration approach B (independent branch)
fork_2_prompt = f"""
{baseline_analysis}

Propose a migration strategy that prioritizes stability over speed.
Focus on: comprehensive testing, gradual rollout, zero technical debt.
"""
subagent_B = task(subagent_type="architect", prompt=fork_2_prompt)

# Compare approaches without one influencing the other
comparison = coordinator.synthesize(subagent_A, subagent_B)
```

**Key difference from parallel spawning:**
- **Parallel:** Same task, split across agents (speed)
- **Fork:** Different approaches to same problem (exploration)

---

## 2. Design Patterns

### Pattern 1: Pre-Commit Validation Hook

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "python scripts/validate_no_secrets.py \"$TOOL_INPUT\""
      }
    ]
  }
}
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
        "command": "npx prettier --write \"$FILE_PATH\" && npx eslint --fix \"$FILE_PATH\""
      }
    ]
  }
}
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

### Pattern 5: Coordinator-Subagent Orchestration with Task Tool

```python
from anthropic import Anthropic

client = Anthropic()

def run_research_coordinator(topic: str):
    """Coordinator spawns subagents for parallel research."""
    
    # Coordinator's agentic loop
    messages = [{
        "role": "user",
        "content": f"Research: {topic}. Use subagents to search, analyze, and synthesize."
    }]
    
    system_prompt = """You are a research coordinator.
    - Spawn researcher subagents for different angles
    - Pass ALL findings explicitly to synthesis subagent
    - You have access to the Task tool for spawning subagents
    """
    
    while True:
        response = client.messages.create(
            model="claude-opus-4-6-20250514",
            max_tokens=4096,
            system=system_prompt,
            tools=[
                {
                    "name": "task",
                    "description": "Spawn a subagent",
                    "input_schema": {
                        "type": "object",
                        "properties": {
                            "subagent_type": {"type": "string"},
                            "prompt": {"type": "string"}
                        },
                        "required": ["subagent_type", "prompt"]
                    }
                }
            ],
            messages=messages
        )
        
        messages.append({"role": "assistant", "content": response.content})
        
        if response.stop_reason == "tool_use":
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    # Execute subagent
                    subagent_result = run_subagent(
                        subagent_type=block.input["subagent_type"],
                        prompt=block.input["prompt"]
                    )
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": subagent_result
                    })
            
            messages.append({"role": "user", "content": tool_results})
        else:
            # Coordinator produced final response
            return response.content[0].text

def run_subagent(subagent_type: str, prompt: str) -> str:
    """Run a specialized subagent."""
    
    subagent_config = {
        "researcher": {
            "system": "You are a research specialist. Search for accurate information. Return structured findings with sources.",
            "tools": ["WebSearch", "WebFetch"]
        },
        "analyzer": {
            "system": "You are a data analyst. Analyze the provided information. Return structured insights.",
            "tools": ["none"]  # No external tools
        }
    }
    
    config = subagent_config[subagent_type]
    
    # Subagent runs in isolated context (no coordinator's prior messages)
    response = client.messages.create(
        model="claude-sonnet-4-6-20250514",
        max_tokens=2048,
        system=config["system"],
        messages=[{"role": "user", "content": prompt}]
    )
    
    return response.content[0].text
```

**Key pattern elements:**
- Coordinator has `"Task"` in its tools
- Subagents have NO `Task` tool (can't spawn deeper subagents)
- Subagent context is isolated (fresh session per subagent)
- Coordinator explicitly passes findings to downstream agents
- Parallel Task calls in one response execute concurrently

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

### Session State Management

```
Conversation State:
├── messages[]          — Full message history
├── system_prompt       — From CLAUDE.md + settings
├── tool_permissions    — What's been allowed this session
├── working_directory   — Current filesystem context
└── plan               — Active plan (if in plan mode)
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
| Hook command injection | Sanitize `$TOOL_INPUT` before use |
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

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A team wants to ensure that Claude never executes `rm -rf` commands in their project. What is the MOST effective approach?

**A:** Configure a `PreToolUse` hook with matcher `"Bash"` that inspects `$TOOL_INPUT` for dangerous patterns like `rm -rf`, `rm -r /`, etc. The hook script exits with non-zero code if a dangerous command is detected, blocking execution.

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "command": "python .claude/scripts/block_dangerous.py \"$TOOL_INPUT\""
    }]
  }
}
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
• Env vars: $TOOL_NAME, $TOOL_INPUT, $FILE_PATH, $TOOL_OUTPUT

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
