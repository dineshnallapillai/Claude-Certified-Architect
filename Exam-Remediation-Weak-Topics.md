# Claude Certified Architect — Exam Remediation Notes
## Topics Scored 0% or Below 70% (August 16, 2026)

---

# SECTION A: TOPICS SCORED 0% (Critical Priority)

---

## 1. Configure Subagent Invocations with Tool Restrictions, Context Scoping & System Prompts

### The Core Exam Question Pattern

The exam tests whether you can **correctly configure a subagent definition** — not just describe it. You must know the exact parameters:

```python
# COMPLETE subagent configuration (all levers)
sub_agent = client.messages.create(
    model="claude-sonnet-4-6",              # 1. Model selection
    system="You are a code reviewer...",     # 2. System prompt (ROLE CONSTRAINT)
    tools=[only_these_tools],               # 3. Tool restriction (HARD boundary)
    messages=[{                             # 4. Context scoping (WHAT it sees)
        "role": "user",
        "content": scoped_task_description
    }],
    max_tokens=4096                         # 5. Output limit
)
```

### Three Configuration Levers (Exam Always Tests ONE)

| Lever | What It Controls | How It Constrains |
|-------|-----------------|-------------------|
| **System prompt** | Role, behavior, output format | Soft constraint — agent SHOULD follow |
| **Tool list** | Available capabilities | Hard constraint — agent CANNOT use unlisted tools |
| **Context (messages)** | What information the agent sees | Information boundary — agent doesn't know what it can't see |

### KEY EXAM DISTINCTION: System Prompt vs Tool Restrictions

```
SYSTEM PROMPT says: "You are a read-only analyst. Do not modify files."
→ SOFT constraint. Model may still call write tools if they're available.

TOOL LIST says: tools=[read_file, grep, glob]  (no write_file, no edit)
→ HARD constraint. Model literally CANNOT write. Not a choice — impossible.
```

**Exam trap:** If the question asks "how to PREVENT a subagent from doing X," the answer is ALWAYS tool restriction, NOT system prompt instruction. System prompts guide behavior; tool lists enforce boundaries.

### Context Scoping — What the Subagent Sees

```python
# BAD: Passing entire parent context (information leakage)
messages = parent_conversation_history  # Sub-agent sees everything

# GOOD: Scoped context with only what's needed
messages = [{
    "role": "user",
    "content": f"""
    Task: Review this function for SQL injection vulnerabilities.
    
    File: {file_path}
    Code:
    ```
    {function_code}
    ```
    
    Report: list each vulnerability with line number and fix.
    """
}]
```

**Why scope matters:**
- Security: sub-agent doesn't see secrets from other tasks
- Cost: fewer input tokens per sub-agent call
- Accuracy: less noise means better focus on the actual task
- Isolation: sub-agent can't reference/leak information from unrelated tasks

### System Prompt Patterns for Role Constraint

```python
# Pattern: Constrain role + output format + boundaries
system_prompt = """You are a security review specialist.

ROLE: Analyze code exclusively for security vulnerabilities.
DO NOT: Comment on style, performance, or architecture.
DO NOT: Suggest refactoring unrelated to security.

OUTPUT FORMAT:
- Severity: critical/high/medium/low
- Location: file:line
- Issue: one sentence
- Fix: one sentence

If no vulnerabilities found, respond with: {"findings": []}
"""
```

### Claude Code Agent Tool Configuration

```python
# In Claude Code workflows:
agent(
    prompt="Self-contained task description...",
    schema=OUTPUT_SCHEMA,        # Forces structured output
    label="security-review",     # Display name
    phase="Review",              # Progress group
    model="sonnet",              # Model selection
    effort="medium",             # Reasoning effort
    isolation="worktree",        # Git isolation (parallel file edits)
    agentType="code-reviewer"    # Custom agent type with its own system prompt
)
```

### Exam Decision Framework

| Scenario | System Prompt | Tool Restriction | Context Scope |
|----------|:---:|:---:|:---:|
| "Agent should only analyze, never modify" | ✓ guide | ✓✓ ENFORCE (no write tools) | Provide only relevant code |
| "Agent must output JSON only" | ✓ format instruction | — | — |
| "Agent shouldn't see customer data" | — | — | ✓✓ ENFORCE (exclude from messages) |
| "Agent shouldn't access the database" | ✓ guide | ✓✓ ENFORCE (no DB tools) | — |
| "Prevent prompt injection from tool results" | ✓ warn | — | ✓ sanitize inputs |

---

## 2. Select the Appropriate Agentic Review Architecture

### The Three Architectures (Exam Always Compares)

```
┌────────────────────────────────────────────────────────────────────┐
│  PLAN MODE                                                         │
│  ─────────                                                         │
│  • Agent READS and PLANS but does NOT execute                      │
│  • Human reviews plan → approves → agent executes                  │
│  • Single agent, single pass with approval gate                    │
│  • Best for: high-risk changes that need human sign-off            │
│  • Toggle: Shift+Tab (TUI) or --permission-mode plan (CLI)         │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  DIRECT EXECUTION                                                  │
│  ─────────────────                                                 │
│  • Agent reads, reasons, and executes in one continuous flow       │
│  • No explicit approval step (permissions still apply per-tool)    │
│  • Best for: low-risk, well-understood, reversible tasks           │
│  • Default mode in Claude Code                                     │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  MULTI-PHASE WORKFLOW                                              │
│  ─────────────────────                                             │
│  • Multiple agents in coordinated phases (find → verify → fix)     │
│  • Each phase has different tools, models, or review requirements  │
│  • Parallel and/or sequential orchestration                        │
│  • Best for: complex scope, needs multiple perspectives, or         │
│    tasks too large for single-agent context                        │
└────────────────────────────────────────────────────────────────────┘
```

### THE Decision Matrix (Exam Favorite)

| Factor | Plan Mode | Direct Execution | Multi-Phase Workflow |
|--------|:---------:|:----------------:|:--------------------:|
| **Task scope** | Medium (5-20 files) | Small (1-3 files) | Large (20+ files, multiple concerns) |
| **Risk level** | High (irreversible, security) | Low (reversible, safe) | Variable (isolated per phase) |
| **Human approval** | Required before execution | Not needed (trust agent) | Per-phase or final gate |
| **Multiple perspectives** | No (single analysis) | No | Yes (parallel reviewers) |
| **Latency tolerance** | Low urgency (human waits) | High urgency (needs it now) | Medium (parallel speeds it up) |
| **Context needed** | Fits in one context | Fits in one context | Exceeds single context |

### Critical Exam Scenarios

**Scenario: "Refactoring 3 files in a well-known pattern"**
→ **Direct execution.** Low risk, well-understood, reversible.

**Scenario: "Database migration on production with 50M rows"**
→ **Plan mode.** Irreversible, high risk, needs human review before execution.

**Scenario: "Security audit across the entire codebase checking for OWASP Top 10"**
→ **Multi-phase workflow.** Large scope (entire codebase), needs multiple focused reviewers (one per vulnerability category), parallel execution for latency.

**Scenario: "Add a test for an existing function"**
→ **Direct execution.** Single file, zero risk, one obvious approach.

**Scenario: "Redesign the authentication module to support OAuth2 + SAML"**
→ **Plan mode.** Multiple valid approaches, architectural decision, high risk, needs user buy-in.

**Scenario: "Review PR for correctness, security, and performance"**
→ **Multi-phase workflow.** Three separate concerns benefit from focused passes (one per dimension) to avoid recall trade-offs from competing concerns.

### TRAP: When Plan Mode Looks Right But Multi-Phase Is Better

Ask: "Does this task benefit from MULTIPLE INDEPENDENT PERSPECTIVES?"
- If YES → Multi-phase (parallel reviewers, each focused on one dimension)
- If NO (just needs approval gate) → Plan mode

Ask: "Does the scope exceed what one agent can hold in context?"
- If YES → Multi-phase (decompose into sub-tasks)
- If NO → Plan mode or direct execution

---

## 3. Evaluate Multi-Agent Orchestration Patterns

### The Three Patterns (Exam Tests Selection)

```
COORDINATOR-WORKER (Hub-and-Spoke)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
         ┌── Worker A (research)
Hub ─────┼── Worker B (code)
         └── Worker C (review)

• Central coordinator delegates, collects, synthesizes
• Workers are independent — don't talk to each other
• Hub makes all routing decisions
• BEST FOR: Complex tasks with heterogeneous subtasks

PARALLEL EXECUTION (Fan-Out/Fan-In)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    ┌── Agent A ──┐
──┼── Agent B ──┼── Merge
    └── Agent C ──┘

• Same or similar tasks run concurrently
• All results collected and merged at the end
• BEST FOR: Coverage (search broadly) and latency (do it all at once)

SEQUENTIAL PIPELINE
━━━━━━━━━━━━━━━━━━━
Agent A → Agent B → Agent C → Output

• Each agent transforms and passes forward
• Output of stage N = input of stage N+1
• BEST FOR: Tasks with hard data dependencies between stages
```

### Selection Criteria (THE EXAM TABLE)

| Requirement | Best Pattern | Why |
|-------------|-------------|-----|
| **Maximum research coverage** | Parallel | Multiple independent searchers find what one misses |
| **Minimum latency** | Parallel | Wall-clock = slowest single agent, not sum of all |
| **Maximum reliability** | Coordinator-Worker | Hub can retry failed workers, validate results, resolve conflicts |
| **Strong data dependencies** | Sequential Pipeline | Stage B literally needs stage A's output |
| **Heterogeneous tasks** | Coordinator-Worker | Different workers specialize (research ≠ code ≠ review) |
| **Homogeneous tasks at scale** | Parallel | Same task × many items (e.g., review 20 files) |
| **Need for synthesis/conflict resolution** | Coordinator-Worker | Hub reconciles disagreements |
| **Simple transformation chain** | Sequential Pipeline | Each stage enriches/transforms (extract → validate → format) |

### Critical Exam Traps

**TRAP 1: "Which pattern for maximum coverage?"**
- Answer: PARALLEL (not coordinator-worker)
- Why: Multiple independent search agents find different things. One agent can't search everything.

**TRAP 2: "Which pattern when stages have no dependencies?"**
- Answer: PARALLEL (not sequential pipeline)
- Why: If stages are independent, running them sequentially wastes time.

**TRAP 3: "Which pattern when you need to reconcile conflicting results?"**
- Answer: COORDINATOR-WORKER
- Why: Only a hub can see all results and make a judgment call. Parallel returns raw results without reconciliation.

**TRAP 4: "Latency vs. Reliability trade-off"**
- Parallel wins on LATENCY (everything runs at once)
- Coordinator-Worker wins on RELIABILITY (hub catches failures, retries, resolves conflicts)
- Sequential Pipeline loses on BOTH unless there are true data dependencies

### Combining Patterns (Advanced Exam Question)

```
Real-world systems combine patterns:

Coordinator-Worker + Parallel:
  Hub delegates to workers, workers run in parallel
  → Coverage + reliability

Sequential + Parallel:
  Pipeline stages where WITHIN each stage, work is parallel
  → Dependencies respected + latency optimized per stage

Example: Code Review System
  Phase 1 (parallel): Security reviewer + Performance reviewer + Logic reviewer
  Phase 2 (sequential): Synthesis agent merges all findings (needs Phase 1 complete)
  Phase 3 (coordinator): Hub presents to human, handles feedback
```

---

## 4. Select the Correct Claude Code Configuration Mechanism

### THE Decision Table (Exam Favorite — MEMORIZE THIS)

| Guidance Type | Mechanism | Why This One |
|---------------|-----------|-------------|
| **Always-on project rules** (conventions, stack, patterns) | `CLAUDE.md` (project root) | Loaded every session, shared with team |
| **Always-on personal preferences** | `~/.claude/CLAUDE.md` | Personal, all projects |
| **Rules that apply only to certain file types** | `.claude/rules/*.md` with `paths:` glob frontmatter | Loaded only when editing matching files |
| **On-demand workflows needing isolation** | Skills (`.claude/skills/*/SKILL.md`) | `context: fork` + `allowed-tools` |
| **Simple prompt templates** | Slash commands (`.claude/commands/*.md`) | No isolation, no tool restriction |
| **Hard enforcement of actions** (MUST happen, not advisory) | Hooks (`settings.json` → hooks) | Execute shell commands on events |
| **Tool permission allow/deny lists** | `settings.json` → permissions | Controls what tools can be used |
| **Machine-specific local config** | `CLAUDE.local.md` (gitignored) | Not committed, varies per developer |

### THE Critical Distinctions (Where Exam Catches You)

#### CLAUDE.md vs .claude/rules/ (Path-Scoped Rules)

```
CLAUDE.md:
• Always loaded (even if editing unrelated files)
• Good for: project-wide conventions
• Cost: every token loaded on every call

.claude/rules/ with paths: frontmatter:
• ONLY loaded when editing files matching the glob
• Good for: module-specific rules (API rules only when editing API code)
• Cost: zero when not relevant

EXAM QUESTION PATTERN:
"API conventions should apply only when editing src/api/**"
→ Answer: .claude/rules/api-conventions.md with paths: ["src/api/**"]
NOT: CLAUDE.md (wastes tokens when editing non-API files)
NOT: src/api/CLAUDE.md (this works too but .claude/rules/ is preferred for organized projects)
```

#### Skills vs Slash Commands

```
SKILL (.claude/skills/):
• Has frontmatter: context: fork, allowed-tools, argument-hint
• Can be ISOLATED (fork) — verbose output doesn't pollute main context
• Can have TOOL RESTRICTIONS — hard boundary on capabilities
• Use for: complex workflows, analysis, code generation with safety

SLASH COMMAND (.claude/commands/):
• NO frontmatter support (no fork, no tool restriction)
• Runs in main conversation context
• Simple text template with $ARGUMENTS substitution
• Use for: quick prompts, simple repeated instructions
```

#### Hooks vs CLAUDE.md Instructions

```
CLAUDE.md says: "Always run tests before committing"
→ ADVISORY. Claude SHOULD do this but might not.

Hook in settings.json:
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash(git commit*)",
      "command": "npm test",
      "blocking": true
    }]
  }
}
→ ENFORCED. Shell command executes BEFORE git commit. If it fails, commit is blocked.

EXAM RULE: If the question says "MUST", "ENFORCE", "GUARANTEE", or "PREVENT"
→ Answer is HOOKS or PERMISSIONS, not CLAUDE.md
```

#### Settings Permissions vs Tool Restrictions in Skills

```
Settings permissions (settings.json):
• GLOBAL — apply to all conversations in the project
• allow/deny patterns for tools
• Example: deny ["Bash(rm *)"] blocks rm everywhere

Skill allowed-tools:
• SCOPED — only applies when that skill is running
• Doesn't affect other conversations or skills
• Example: allowed-tools: [Read, Grep, Glob] makes skill read-only
```

### Exam Decision Flowchart

```
Does it need to be ENFORCED (not just advised)?
├── YES → Is it about tool permissions?
│         ├── YES → settings.json permissions (allow/deny)
│         └── NO → hooks in settings.json
└── NO → Is it always-on or on-demand?
          ├── ALWAYS-ON → Does it apply to specific file patterns only?
          │               ├── YES → .claude/rules/ with paths: glob
          │               └── NO → CLAUDE.md (project or user level)
          └── ON-DEMAND → Does it need isolation or tool restriction?
                          ├── YES → Skill (.claude/skills/)
                          └── NO → Slash command (.claude/commands/)
```

---

## 5. Design Synthesis Agent Behavior Preserving Source-Level Uncertainty

### What the Exam Tests

The exam tests whether you understand that a synthesis agent must **NOT collapse conflicting information into a single confident statement**. It must preserve:
- Which sources agree vs. disagree
- Confidence levels per finding
- The distinction between "established fact" and "contested claim"

### The Anti-Pattern (What Gets You 0%)

```
BAD SYNTHESIS (collapses uncertainty):
"The migration will take 3 months and cost $50K."

GOOD SYNTHESIS (preserves uncertainty):
"Migration timeline estimates conflict:
 - Cost analysis agent: 3 months, $50K (based on similar past projects, confidence: medium)
 - Technical assessment agent: 6 months, $120K (based on actual code complexity analysis, confidence: high)
 - The discrepancy stems from whether legacy data format conversion is included in scope.
 Recommendation: Use the 6-month estimate (higher confidence, accounts for data conversion)."
```

### Three Uncertainty Preservation Patterns

#### Pattern 1: Source-Attributed Claims

```python
synthesis_output = {
    "findings": [
        {
            "claim": "API handles 10K requests/second",
            "sources": ["load_test_agent"],
            "evidence": "Load test results from 2024-01-15",
            "confidence": "high",
            "status": "established"  # All sources agree
        },
        {
            "claim": "Database will need sharding at 100K users",
            "sources": ["capacity_agent"],
            "contradicted_by": ["performance_agent"],
            "evidence_for": "Linear growth projection from current 40K",
            "evidence_against": "Query optimization could defer to 500K",
            "confidence": "contested",
            "status": "disputed"  # Sources disagree
        }
    ]
}
```

#### Pattern 2: Confidence-Graduated Presentation

```
WELL-ESTABLISHED (all sources agree, direct evidence):
• "Authentication uses JWT with 24h expiry" [confirmed by code + config + docs]

PROBABLE (most sources agree, some indirect evidence):
• "Response times degrade above 5K concurrent users" [load test suggests, not production-confirmed]

CONTESTED (sources disagree):
• "Migrating to microservices would improve deployment speed"
  - FOR: DevOps agent cites independent deployment cycles
  - AGAINST: Architecture agent cites coordination overhead
  - UNRESOLVED: depends on team size (not assessed)

UNCERTAIN (insufficient evidence):
• "Current caching strategy is optimal" [no benchmarks available, only design review]
```

#### Pattern 3: Explicitly Flagging Collapsed Information

```python
# In the synthesis agent's system prompt:
system_prompt = """You are a research synthesis agent.

CRITICAL RULES:
1. NEVER state a contested finding as fact
2. When sources disagree, present BOTH positions with their evidence
3. Distinguish between:
   - ESTABLISHED: All sources agree, direct evidence exists
   - PROBABLE: Majority agree, indirect evidence supports
   - CONTESTED: Sources actively disagree
   - UNKNOWN: Insufficient evidence to assess
4. Always state your confidence level and what would change it
5. If collapsing nuance for brevity, explicitly say "simplified from conflicting data"
"""
```

### Exam Question Patterns

**Q: "Two research agents return conflicting market size estimates. How should synthesis handle this?"**

WRONG: Pick the one with higher confidence and present it as fact.
WRONG: Average the two numbers.
RIGHT: Present both estimates with their methodologies, note the discrepancy, explain what causes the difference, and let the human decide which methodology to trust.

**Q: "A synthesis agent combines findings from 5 sub-agents. One sub-agent's finding contradicts the other four. What should it do?"**

WRONG: Majority rules — drop the outlier.
RIGHT: Report the majority finding as "probable" (4/5 agree), report the dissenting finding separately with its reasoning, and flag the disagreement for human review. The outlier may have found something the others missed.

---

## 6. Apply Context Window Optimization Techniques

### The Four Techniques (Exam Tests Selection)

| Technique | What It Does | When to Use | Trade-off |
|-----------|-------------|-------------|-----------|
| **Summarization** | Compress past conversation into condensed form | Before starting new phase; at ~75% capacity | Loses specific details (line numbers, exact values) |
| **Sliding Window** | Keep only last N messages, drop oldest | Long-running conversations with recent-info priority | Loses early context entirely |
| **Structured State Objects** | Maintain explicit state dict updated each turn | Multi-step tasks needing specific tracked values | Requires discipline to update; overhead per turn |
| **Selective Retention** | Keep specific high-value messages, drop low-value ones | Mixed conversations (some turns critical, some noise) | Complexity in deciding what's "valuable" |

### Summarization (Most Common Exam Answer)

```python
# When: conversation approaches context limit
# How: compress history into key decisions + findings

BEFORE SUMMARIZATION:
  Turn 1: "Read auth module" → [3000 tokens of file content]
  Turn 2: "What patterns?" → [1500 tokens of analysis]
  Turn 3: "Read config" → [2000 tokens of file content]
  Turn 4: "Find vulnerabilities" → [2000 tokens of findings]
  Total: ~8500 tokens of history

AFTER SUMMARIZATION:
  "SUMMARY: Auth module (src/auth/) uses JWT with hardcoded secret.
   Config at src/config/auth.ts. Key vulnerability: secret in source code.
   Decision: will move to env var."
  Total: ~100 tokens

WHAT'S LOST: Exact file contents, line numbers, intermediate reasoning
WHAT'S KEPT: Decisions, findings, current state
```

### Sliding Window

```python
# Keep only last N turns in the message array
def sliding_window(messages, max_turns=20):
    system = messages[0]  # Always keep system prompt
    recent = messages[-max_turns:]  # Keep last N
    return [system] + recent

# WHEN TO USE:
# - Chat-like interactions where recent context matters most
# - Tasks where early turns are fully superseded by later work
# - NOT for tasks needing to reference early decisions
```

### Structured State Objects (Exam Differentiator)

```python
# Instead of relying on message history, maintain explicit state:
state = {
    "task": "Refactor auth module to use env vars",
    "files_modified": ["src/auth/token.ts", "src/config/auth.ts"],
    "files_remaining": ["src/auth/middleware.ts"],
    "decisions": [
        "Use dotenv for local, cloud secrets manager for prod",
        "Keep backward compat with JWT_SECRET env var name"
    ],
    "current_step": "Updating middleware to read from env",
    "blockers": []
}

# Inject state at beginning of each turn:
messages.append({
    "role": "user",
    "content": f"CURRENT STATE:\n{json.dumps(state, indent=2)}\n\nContinue with next step."
})
```

**Why structured state beats summarization for multi-step tasks:**
- No information loss on tracked values
- Explicit — nothing "interpreted" from vague summaries  
- Machine-readable — downstream agents can parse it
- Auditable — every state transition is deliberate

### Selective Retention

```python
# Score each message for retention value:
def retention_score(message, current_task):
    score = 0
    if contains_decision(message): score += 3      # Decisions are always valuable
    if contains_constraint(message): score += 3    # Constraints must be kept
    if is_recent(message, turns=5): score += 2     # Recency matters
    if references_current_task(message): score += 2  # Relevance
    if is_tool_output(message): score -= 1         # Raw output is replaceable
    if is_verbose_exploration(message): score -= 2 # Noise
    return score

# Keep messages above threshold, drop the rest
```

### Exam Selection Criteria

| Scenario | Best Technique | Why NOT the others |
|----------|---------------|-------------------|
| "Agent doing 50-step refactoring, needs to track which files are done" | Structured state object | Summarization might lose the file list; sliding window drops early progress |
| "Chat conversation that's been going 200 turns, user asks new question" | Sliding window | Early turns irrelevant to new question; state object is overkill for chat |
| "Agent switching from investigation phase to implementation phase" | Summarization | Investigation details (grep results) not needed, just the conclusions; state object too rigid for open-ended investigation |
| "System processes documents sequentially, must maintain running tally" | Structured state object | Need exact counts/values; summarization would approximate |
| "Long conversation with 3 critical constraint turns buried in middle" | Selective retention | Constraints must be kept verbatim; summarization might paraphrase them lossy; sliding window would drop them |

### The `/compact` Connection (Claude Code Specific)

```
/compact = SUMMARIZATION applied to Claude Code conversation

WHEN TO USE:
• Proactively at ~75% context capacity (BEFORE degradation)
• After verbose exploration phase (investigation complete → implementation starts)
• When you see degradation signals (vague answers, hallucinated specifics)

WHAT IT PRESERVES:
• Key decisions and findings
• Current task state
• Critical constraints from system prompt

WHAT IT DROPS:
• Raw tool outputs (file contents, grep results)
• Intermediate reasoning
• Superseded plans
```

---

## 7. Select the Most Reliable Structured Output Method

### Three Methods Ranked by Reliability

```
RELIABILITY RANKING (highest → lowest):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. TOOL USE WITH JSON SCHEMA ← Most reliable (schema-enforced)
2. PREFILLED RESPONSES       ← Medium (partial format lock-in)
3. PROMPT-BASED FORMATTING   ← Least reliable (model may deviate)
```

### Method 1: Tool Use with JSON Schema (HIGHEST RELIABILITY)

```python
# Define a "tool" whose purpose is purely to capture structured output
tools = [{
    "name": "submit_analysis",
    "description": "Submit the structured analysis results",
    "input_schema": {
        "type": "object",
        "properties": {
            "severity": {"type": "string", "enum": ["critical", "high", "medium", "low"]},
            "findings": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "file": {"type": "string"},
                        "line": {"type": "integer"},
                        "issue": {"type": "string"},
                        "fix": {"type": "string"}
                    },
                    "required": ["file", "line", "issue"]
                }
            },
            "summary": {"type": "string"}
        },
        "required": ["severity", "findings", "summary"]
    }
}]

# Force tool use — Claude MUST call this tool
response = client.messages.create(
    model="claude-sonnet-4-6",
    tools=tools,
    tool_choice={"type": "tool", "name": "submit_analysis"},  # FORCED
    messages=[{"role": "user", "content": "Review this code..."}]
)

# Result is GUARANTEED to match schema
structured_data = response.content[0].input  # Already valid JSON
```

**Why most reliable:**
- JSON Schema validation at the API layer
- `tool_choice: {"type": "tool", "name": "X"}` guarantees invocation
- Model cannot respond with prose — it MUST call the tool
- Schema defines types, required fields, enums — all enforced
- No parsing needed — result is already a Python dict

### Method 2: Prefilled Responses (MEDIUM RELIABILITY)

```python
# Start Claude's response with a partial JSON structure
response = client.messages.create(
    model="claude-sonnet-4-6",
    messages=[
        {"role": "user", "content": "Analyze this code for bugs..."},
        {"role": "assistant", "content": '{"findings": ['}  # PREFILL
    ]
)

# Claude continues from where prefill left off
# Result: '{"file": "auth.ts", "line": 42, ...}]}'
full_json = '{"findings": [' + response.content[0].text
```

**Why medium reliability:**
- Locks the opening format (model continues the pattern)
- But no schema validation — model might add unexpected fields
- Model might not close brackets correctly in complex cases
- Works well for simple formats, breaks on deep nesting

### Method 3: Prompt-Based Formatting (LOWEST RELIABILITY)

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    messages=[{
        "role": "user",
        "content": """Analyze this code. Respond ONLY in this exact JSON format:
        {"severity": "...", "findings": [...], "summary": "..."}
        
        Do not include any text outside the JSON."""
    }]
)

# Must parse — might have markdown fences, explanatory text, etc.
text = response.content[0].text
# Might be: ```json\n{"severity": ...}\n``` — needs cleanup
```

**Why least reliable:**
- Model might add explanatory text before/after JSON
- Model might wrap in markdown code fences
- Model might deviate from exact field names
- Model might add fields not in your spec
- No schema validation — any valid JSON passes, even wrong structure

### THE EXAM DECISION TABLE

| Requirement | Method | Why |
|-------------|--------|-----|
| **Downstream system requires exact schema compliance** | Tool use + JSON schema + forced tool_choice | Only method that GUARANTEES schema match |
| **Simple key-value extraction, low stakes** | Prompt-based (with parsing fallback) | Cheapest, simplest, usually works |
| **Need JSON but tool_choice not available** | Prefilled response | Forces JSON start without tool overhead |
| **Critical pipeline — failure = data loss** | Tool use (with retry on malformed) | Schema validation catches issues before they propagate |
| **Streaming required + structured output** | Tool use (input_json_delta events) | Streams tool input JSON progressively |

### Exam Trap Questions

**"When is prompt-based formatting acceptable?"**
→ Only when: (a) downstream can handle occasional failures gracefully, (b) you have a retry/fallback mechanism, (c) the format is very simple (single value or flat object).

**"What combination gives highest structured output guarantee?"**
→ Tool use + JSON schema + `tool_choice: {"type": "tool", "name": "X"}`. This triple guarantees: tool is called (tool_choice), output matches schema (JSON schema validation), and no prose alternative (forced invocation).

---

# SECTION B: TOPICS SCORED 50-67% (High Priority)

---

## 8. Subagent Delegation: Goal-Oriented vs Procedural Instructions (50%)

### The Two Styles

```
GOAL-ORIENTED (Declarative):
"Find and fix all SQL injection vulnerabilities in the auth module."
→ Agent decides HOW to accomplish this
→ Adaptive: can change approach based on what it finds
→ Higher autonomy, potentially better outcomes for complex tasks

PROCEDURAL (Imperative):
"1. Read src/auth/login.ts
 2. Check line 45 for unsanitized user input
 3. Replace raw SQL with parameterized query
 4. Run tests"
→ Agent follows exact steps
→ Predictable: coordinator knows exactly what will happen
→ Better visibility and control, but brittle if assumptions wrong
```

### When to Use Each (EXAM DECISION)

| Factor | Goal-Oriented | Procedural |
|--------|:---:|:---:|
| Task is well-understood with known solution | | ✓ |
| Task requires exploration/discovery | ✓ | |
| Need predictable execution for audit | | ✓ |
| Subagent has domain expertise | ✓ | |
| Steps might need adaptation mid-task | ✓ | |
| Need exact progress tracking | | ✓ |
| Task is novel — best approach unclear | ✓ | |
| Operating in high-risk environment | | ✓ |

### The Spectrum (Not Binary)

```
PURE PROCEDURAL          HYBRID                    PURE GOAL-ORIENTED
"Do step 1, 2, 3"    "Achieve X. Must include    "Fix the auth bugs."
                       Y. Don't touch Z."
                       
← More control                              More adaptive →
← More predictable                          Better outcomes for complex →
← Brittle to surprises                     Resilient to discoveries →
← Easier to debug                          Harder to predict exactly →
```

### Exam Trap: The Hybrid Answer

Most exam questions have the hybrid as the best answer:

```python
# BEST PRACTICE: Goal-oriented with guardrails
prompt = """
GOAL: Fix SQL injection vulnerabilities in the auth module.

CONSTRAINTS:
- Only modify files in src/auth/
- Use parameterized queries (not string escaping)
- Preserve existing function signatures
- Run existing tests after each change

REPORT: For each fix, state the file, line, vulnerability, and fix applied.
"""

# This gives adaptive behavior (goal) + visibility (report) + safety (constraints)
```

**Key insight for exam:** The question asks you to "select the approach that enables adaptive behavior WHILE maintaining coordinator visibility." The answer is almost always the hybrid — goal-oriented core with reporting requirements and constraints.

---

## 9. Apply Extraction Accuracy Patterns (50%)

### Three Pillars of Extraction Accuracy

```
1. STRUCTURED SCHEMAS (with optional fields)
   → Model knows exactly what to extract and what types
   → Optional/nullable fields prevent hallucination of missing data

2. FORMAT NORMALIZATION INSTRUCTIONS
   → "Dates should be YYYY-MM-DD regardless of input format"
   → "Phone numbers as +CC-XXX-XXX-XXXX"
   → Reduces inconsistency across documents

3. FEW-SHOT EXAMPLES
   → Show input document → expected extraction
   → Include edge cases (missing fields, ambiguous values)
   → Most effective for complex/nuanced extractions
```

### Optional Fields Prevent Hallucination

```python
# BAD: All fields required → model INVENTS values for missing data
schema = {
    "properties": {
        "company": {"type": "string"},
        "revenue": {"type": "number"},  # What if document doesn't mention revenue?
        "founded_year": {"type": "integer"}  # What if not stated?
    },
    "required": ["company", "revenue", "founded_year"]  # ← Forces hallucination
}

# GOOD: Optional fields → model can honestly say "not found"
schema = {
    "properties": {
        "company": {"type": "string"},
        "revenue": {"type": ["number", "null"]},  # null = not found in document
        "founded_year": {"type": ["integer", "null"]},
        "confidence": {"type": "string", "enum": ["high", "medium", "low"]}
    },
    "required": ["company", "confidence"]  # Only require what's always present
}
```

### Format Normalization Instructions

```python
system_prompt = """Extract company information from the document.

FORMAT RULES:
- Dates: Always YYYY-MM-DD (convert "March 5, 2024" → "2024-03-05")
- Currency: Always USD decimal (convert "€50M" → 55000000, "$2.5B" → 2500000000)
- Phone: Always +CC-XXX-XXX-XXXX
- Names: Title Case (convert "JOHN SMITH" → "John Smith")
- Percentages: Decimal (convert "45%" → 0.45)

If a value cannot be normalized with confidence, set it to null and add a 
note in the "extraction_notes" field.
"""
```

### Few-Shot Examples (The Accuracy Multiplier)

```python
messages = [
    {"role": "user", "content": """Extract from: "TechCorp, founded in Silicon Valley 2018, 
     reported $5.2M ARR in their Series A deck. CEO: jane doe"
     """},
    {"role": "assistant", "content": json.dumps({
        "company": "TechCorp",
        "founded_year": 2018,
        "revenue": 5200000,
        "revenue_type": "ARR",
        "ceo": "Jane Doe",
        "location": "Silicon Valley",
        "confidence": "high",
        "extraction_notes": null
    })},
    {"role": "user", "content": """Extract from: "Meeting notes: discussed partnership with 
     Acme Ltd. They might be doing around $10-15M. Founded sometime in the 2010s."
     """},
    {"role": "assistant", "content": json.dumps({
        "company": "Acme Ltd",
        "founded_year": null,  # "sometime in 2010s" is too vague
        "revenue": null,  # "$10-15M" is a range estimate, not confirmed
        "revenue_type": null,
        "ceo": null,
        "location": null,
        "confidence": "low",
        "extraction_notes": "Revenue mentioned as estimate range ($10-15M), not confirmed. Founded date is decade-level only."
    })},
    # NOW the actual extraction request:
    {"role": "user", "content": f"Extract from: {actual_document}"}
]
```

**Key few-shot principles:**
- Show a high-confidence example (clear data → clean extraction)
- Show a low-confidence example (vague data → nulls + notes)
- Show the EXACT output format you expect
- Edge cases in examples teach more than instructions

---

## 10. Configure Tool Distribution in Multi-Agent Systems (50%)

### The Principle: Minimum Viable Toolset Per Agent

```
WRONG (every agent gets everything):
┌────────────────────────────────────────┐
│ All Agents: [read, write, delete,      │
│  search, deploy, send_email, db_query] │
└────────────────────────────────────────┘
↑ One confused agent could deploy or delete

RIGHT (each agent gets only what its role needs):
┌─────────────────────────────────┐
│ Research Agent: [read, search]   │ ← Can't modify anything
│ Code Agent: [read, write, edit]  │ ← Can't deploy or delete
│ Review Agent: [read, grep, lint] │ ← Read-only + analysis
│ Deploy Agent: [deploy, rollback] │ ← Can't write code
└─────────────────────────────────┘
```

### Why This Matters (Two Reasons Tested on Exam)

**1. Reduces Decision Complexity:**
- Agent with 5 tools: high accuracy selecting the right one
- Agent with 30 tools: model confused, selects wrong tools, wastes turns

**2. Prevents Out-of-Role Tool Invocations:**
- Review agent with write tools might "helpfully" fix what it finds
- This violates the separation of concerns — reviewer shouldn't fix
- By removing write tools, the agent literally cannot overstep

### Configuration Methods

```python
# Method 1: Per-agent tool lists in orchestrator
AGENT_CONFIGS = {
    "research": {
        "system": "You are a research specialist...",
        "tools": [search_tool, read_tool],  # Only 2 tools
    },
    "developer": {
        "system": "You are a code implementation specialist...",
        "tools": [read_tool, write_tool, edit_tool, run_tests_tool],  # 4 tools
    },
    "reviewer": {
        "system": "You are a code review specialist...",
        "tools": [read_tool, grep_tool, lint_tool, comment_tool],  # 4 tools, all read-only
    }
}

# Method 2: Skills with allowed-tools (Claude Code)
# .claude/skills/review/SKILL.md
# ---
# allowed-tools:
#   - Read
#   - Grep
#   - Glob
# ---

# Method 3: MCP server with scoped capabilities
# Connect research agent to read-only DB server
# Connect deploy agent to deployment MCP server
```

### Exam Scenario Pattern

**"A code review agent in a multi-agent system occasionally modifies files it's reviewing instead of just reporting issues. How do you fix this?"**

WRONG: "Add to system prompt: do not modify files"
RIGHT: "Remove Write and Edit from the review agent's tool list. It should only have Read, Grep, Glob, and a 'report_finding' tool."

---

## 11. Design Human Review Routing Strategies (67%)

### Routing Criteria (Not Random Sampling)

```
ROUTING DIMENSIONS:
━━━━━━━━━━━━━━━━━━

1. CONFIDENCE SCORE
   • High confidence (>0.9): auto-approve, no human needed
   • Medium confidence (0.6-0.9): route to standard reviewer
   • Low confidence (<0.6): route to senior/expert reviewer

2. DOCUMENT CHARACTERISTICS  
   • Simple/standard format → auto-process
   • Complex/unusual format → human review
   • New document type never seen before → always human first

3. FIELD-LEVEL AMBIGUITY
   • Critical field uncertain (e.g., dollar amount unclear) → human review
   • Non-critical field uncertain (e.g., middle name missing) → auto-accept as null
```

### Routing Decision Matrix

```python
def route_extraction(extraction_result):
    # Rule 1: Overall confidence
    if extraction_result.overall_confidence > 0.9:
        return "auto_approve"
    
    # Rule 2: Critical field uncertainty
    critical_fields = ["amount", "date", "counterparty"]
    uncertain_critical = [
        f for f in critical_fields 
        if extraction_result.field_confidence[f] < 0.7
    ]
    if uncertain_critical:
        return "senior_reviewer"  # Expert needed for critical ambiguity
    
    # Rule 3: Document characteristics
    if extraction_result.document_type == "unknown":
        return "senior_reviewer"  # New format needs expert
    
    if extraction_result.document_quality == "poor":  # Blurry, handwritten
        return "specialist_reviewer"
    
    # Rule 4: Standard medium-confidence
    return "standard_reviewer"
```

### Key Exam Insight

**"Random sampling" is ALWAYS the wrong answer** for routing extractions to human review. The exam tests whether you understand that routing should be INTELLIGENT and based on:
- Where the model is uncertain (confidence-based)
- What kind of document it is (characteristic-based)  
- Which specific fields are ambiguous (field-level)

Random sampling is for VALIDATION (checking system accuracy over time), not for ROUTING (deciding which items need human attention).

---

## 12. Codebase Exploration with Grep, Glob, Read (60%)

### The Strategy: Build Understanding Incrementally

```
PHASE 1: ORIENT (Glob — find structure)
  Glob("**/src/**/*.ts") → understand project layout
  Glob("**/test*/**") → find test locations

PHASE 2: LOCATE (Grep — find specific things)
  Grep("function authenticate") → find entry points
  Grep("import.*auth") → find consumers

PHASE 3: UNDERSTAND (Read — read specific files)
  Read("src/auth/middleware.ts") → understand implementation
  Read("src/auth/types.ts") → understand data structures

PHASE 4: CONNECT (Grep again — follow references)
  Grep("TokenValidator") → who uses this class?
  Grep("refreshToken") → trace the flow
```

### Context Window Management During Exploration

```
RULE: Don't read entire files when you only need part of them

BAD: Read("src/app.ts")  → 2000 lines, 8000 tokens consumed
GOOD: Grep("auth") in src/app.ts → find relevant lines → Read with offset/limit

BAD: Read 20 files to "understand the architecture"
GOOD: Read 3-4 key files, use Grep to find connections

STRATEGY: Use Grep to LOCATE, then Read to UNDERSTAND (targeted)
```

### Tool Selection Guide

| Need | Tool | Why |
|------|------|-----|
| Find files by name/path pattern | **Glob** | Fast, pattern-based, returns paths only |
| Find specific content across codebase | **Grep** | Content search, regex support |
| Understand a specific file's implementation | **Read** | Full content with line numbers |
| Find definition of a function/class | **Grep** (pattern: `"def funcname\|function funcname\|class ClassName"`) | Cross-file symbol lookup |
| Understand project structure | **Glob** (`"**/*.ts"`, `"**/package.json"`) | File tree discovery |
| Find all usages of an API | **Grep** with specific pattern | Cross-reference discovery |

### Context Budget Rule

```
RULE OF THUMB:
• Each Read of a full file: ~1000-5000 tokens consumed
• Each Grep result: ~200-500 tokens
• Each Glob result: ~50-200 tokens

STRATEGY:
• Use Glob/Grep freely (cheap, focused results)
• Use Read sparingly and targeted (expensive, fills context)
• After 5-10 reads, consider: do I need to compact?
```

---

## 13. Extraction Schemas: Optional Fields, Nullable Values, Enums (67%)

### Representing Missing Information WITHOUT Hallucination

```json
{
  "type": "object",
  "properties": {
    "name": {"type": "string"},
    "email": {"type": ["string", "null"]},
    "age": {"type": ["integer", "null"]},
    "status": {
      "type": "string",
      "enum": ["active", "inactive", "unknown"]
    },
    "source_quality": {
      "type": "string", 
      "enum": ["clear", "ambiguous", "illegible"],
      "description": "Quality of source data for this extraction"
    }
  },
  "required": ["name", "status", "source_quality"]
}
```

### Three Principles

**1. Make fields nullable/optional when data might be absent:**
```json
"revenue": {"type": ["number", "null"]}
// null = "document doesn't mention revenue"
// vs. required field → model MUST invent a number → hallucination
```

**2. Include "unknown" or "unclear" in enums for ambiguous cases:**
```json
"contract_type": {
  "type": "string",
  "enum": ["full-time", "part-time", "contractor", "unknown"]
}
// "unknown" = honest representation of ambiguity
// Without it, model picks the closest match even when uncertain
```

**3. Pair extraction with confidence/quality metadata:**
```json
"fields": {
  "amount": {"value": 50000, "confidence": "high"},
  "date": {"value": "2024-03-15", "confidence": "medium"},  
  "counterparty": {"value": null, "confidence": "low", "note": "name partially illegible"}
}
```

### Exam Trap

**"Design a schema for extracting invoice data where some invoices may be missing the PO number."**

WRONG:
```json
"po_number": {"type": "string"}  // Required → model invents PO numbers
```

RIGHT:
```json
"po_number": {"type": ["string", "null"], "description": "Purchase order number if present, null if not stated"}
```

---

# QUICK REFERENCE: ALL WEAK TOPICS IN ONE TABLE

| Topic | Key Principle | Exam Trap |
|-------|-------------|-----------|
| Subagent tool restrictions | Tool list = HARD constraint; system prompt = SOFT | "How to prevent" → always tool restriction, not prompt |
| Review architecture selection | Plan=approval gate, Direct=low risk, Multi-phase=multiple perspectives | "Multiple concerns" → multi-phase, not plan mode |
| Multi-agent pattern selection | Parallel=coverage+latency, Hub=reliability, Pipeline=dependencies | "Maximum coverage" → parallel, not coordinator |
| Configuration mechanism | Enforcement → hooks/permissions; Advisory → CLAUDE.md; Path-specific → .claude/rules/ | "Must guarantee" → hooks, not CLAUDE.md |
| Synthesis uncertainty | NEVER collapse conflicting sources into single claim | "Two agents disagree" → present both, don't pick one |
| Context optimization | State objects for tracking, summarization for phase transitions, sliding window for chat | "Multi-step tracking" → structured state, not summarization |
| Structured output method | Tool use + schema + forced tool_choice = highest guarantee | "Guarantee schema compliance" → tool use, not prompting |
| Delegation style | Hybrid (goal + constraints + reporting) for adaptive+visible | "Adaptive + visible" → goal-oriented with guardrails |
| Extraction accuracy | Optional/nullable fields + format normalization + few-shot examples | "Required field missing in source" → make it nullable |
| Tool distribution | Minimum viable toolset per agent role | "Agent oversteps role" → remove tools, not add instructions |
| Human review routing | Route by confidence + document type + field ambiguity | "Random sampling" → always wrong for routing |
| Codebase exploration | Glob to orient → Grep to locate → Read targeted | Don't Read everything; Grep first to find what to Read |
| Extraction schemas | Nullable types + "unknown" enum values + confidence metadata | "All required" → forces hallucination on missing data |
