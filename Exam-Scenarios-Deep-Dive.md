# CCA-F Exam — The 6 Scenarios (Deep Understanding)

> The exam randomly assigns 4 of 6 scenarios. Each scenario generates ~15 questions. Understanding the ARCHITECTURAL CONTEXT of each scenario is more important than memorizing answers — the exam tests whether you can make correct tradeoff decisions WITHIN a specific production context.

---

## How Scenarios Work on the Exam

1. You receive a **paragraph-long scenario description** establishing the system context
2. All questions in that scenario share the same system — they build on each other
3. Questions ask: "Given THIS system with THESE constraints, what's the best decision?"
4. The scenario CONSTRAINS what counts as correct — a good answer for Scenario 1 may be wrong for Scenario 3

**The exam tests: "Can you think like an architect who inherited THIS specific system?"**

---

---

## SCENARIO 1: Customer Support Resolution Agent

### The System You're Architecting

```
┌─────────────────────────────────────────────────────────┐
│                   CUSTOMER SUPPORT AGENT                  │
├─────────────────────────────────────────────────────────┤
│ SDK: Claude Agent SDK                                    │
│ Model: Sonnet (general) / Opus (complex disputes)        │
│ Target: 80%+ first-contact resolution                    │
│                                                          │
│ MCP Tools:                                               │
│   • get_customer — verify identity, pull account data    │
│   • lookup_order — retrieve order details, status        │
│   • process_refund — execute financial transactions      │
│   • escalate_to_human — transfer to human agent          │
│                                                          │
│ Request Types:                                           │
│   • Returns (standard + edge cases)                      │
│   • Billing disputes (amounts, charges, credits)         │
│   • Account issues (access, settings, verification)      │
│                                                          │
│ Constraints:                                             │
│   • Financial operations (refunds = real money)          │
│   • Identity verification required before actions        │
│   • Policy boundaries the agent must respect             │
│   • Escalation when agent cannot/should not resolve      │
└─────────────────────────────────────────────────────────┘
```

### Why This Scenario Exists

This is the **canonical agentic system** — it has everything:
- A loop (customer back-and-forth)
- Tools with sequencing requirements (verify BEFORE refund)
- Financial stakes (wrong refund = real loss)
- Ambiguity (when to resolve vs. escalate)
- State management (customer context across turns)

### What the Exam Tests Here

| Topic | What They're Really Asking |
|-------|---------------------------|
| **Tool sequencing** | Should you ENFORCE order with code or SUGGEST order with prompts? |
| **Escalation boundaries** | How do you DEFINE when to escalate (not sentiment, not confidence)? |
| **Tool descriptions** | If the agent picks wrong tools, is it a description problem or a count problem? |
| **Financial gates** | How do you GUARANTEE a $500 refund limit is never exceeded? |
| **State persistence** | How do you keep customer ID/order# from being lost over 20 turns? |
| **Error handling** | What happens when `get_customer` returns "not found"? |

### Key Architectural Decisions in This Scenario

**Decision 1: Tool Sequencing — Prompt vs. Code**

The scenario WILL describe a failure: "12% of the time, the agent skips `get_customer` and calls `lookup_order` directly using the customer's stated name."

- **Wrong answer:** "Add clearer instructions to always verify first" → prompts are probabilistic (~85-97%), and 12% failure on financial operations is unacceptable
- **Right answer:** Programmatic prerequisite gate — `lookup_order` and `process_refund` are BLOCKED until `get_customer` has returned a verified customer ID

**Why this is the right answer:** Financial consequences. If refunding the wrong customer costs $500, a 3% leak rate = $15 per 100 interactions. Code gives 100% enforcement.

---

**Decision 2: Escalation Criteria — What Triggers It?**

The scenario WILL describe inverse escalation: "Agent escalates easy cases (standard replacements) while attempting hard cases (policy exceptions) autonomously."

| Wrong Signal | Right Signal |
|---|---|
| Customer frustration/sentiment | Policy gap (no rule covers this case) |
| Self-reported confidence score | Exception required (above refund limit) |
| Request complexity (vague) | Customer explicitly requests human |
| Conversation length | Agent cannot progress (stuck in loop) |

- **Wrong answer:** "Add confidence scoring" → LLMs are poorly calibrated; the model that wrongly handles complex cases rates itself 8/10
- **Right answer:** Explicit escalation criteria with few-shot examples showing borderline cases

---

**Decision 3: State Management — Customer Identity Across Turns**

The scenario WILL present: "After 15 turns, the agent processes a refund for the wrong customer because it lost track of the verified customer ID."

- **Wrong answer:** "Extend context window" or "Summarize more carefully"
- **Right answer:** Pin a CASE_FACTS block (immutable, re-injected every turn) containing verified customer ID, order number, and issue type

---

**Decision 4: Tool Description Quality**

The scenario WILL present: "Agent calls `get_customer` when user asks about orders, and `lookup_order` when user asks about account settings."

- **Wrong answer:** "Add few-shot examples of correct routing" → secondary reinforcement
- **Right answer:** Expand tool descriptions with explicit boundaries: "Use `lookup_order` when user mentions order numbers, shipping, delivery status. Do NOT use for account-level queries."

---

### Traps Specific to This Scenario

1. **"Add sentiment analysis for escalation"** → Sentiment ≠ complexity. Angry customer ≠ hard case.
2. **"Have the agent rate its confidence"** → Self-reported confidence is unreliable.
3. **"Consolidate tools into one `handle_customer` tool"** → Hides individual failure modes, reduces observability.
4. **"Add more few-shot examples of tool order"** → Examples don't enforce; they suggest. Financial operations need guarantees.
5. **"Extend the context window"** → Doesn't fix summarization loss. Pin critical state.

### Mental Model for This Scenario

Think of it as: **"I'm building a system where a wrong decision costs real money. What MUST be deterministic vs. what CAN be probabilistic?"**

- Customer identity verification → MUST be deterministic (hook/gate)
- Refund amount limits → MUST be deterministic (PreToolUse hook)
- Choosing the right tool → SHOULD be reliable (good descriptions)
- Conversational tone → CAN be probabilistic (prompt instructions fine)

---

---

## SCENARIO 2: Code Generation with Claude Code

### The System You're Architecting

```
┌─────────────────────────────────────────────────────────┐
│                 CLAUDE CODE FOR DEVELOPMENT               │
├─────────────────────────────────────────────────────────┤
│ Environment: Developer's local machine                    │
│ Tools: Built-in (Read, Write, Edit, Bash, Grep, Glob)    │
│                                                          │
│ Configuration:                                           │
│   • CLAUDE.md hierarchy (root + subdirectories)          │
│   • .claude/rules/ with glob patterns                    │
│   • .claude/commands/ (team custom slash commands)        │
│   • .claude/settings.json (permissions, MCP, hooks)      │
│                                                          │
│ Tasks:                                                   │
│   • Code generation (new features)                       │
│   • Refactoring (restructuring existing code)            │
│   • Debugging (finding and fixing bugs)                  │
│   • Documentation (generating docs from code)            │
│                                                          │
│ Modes:                                                   │
│   • Plan mode (exploration, architecture, design)        │
│   • Direct execution (implementation, quick fixes)       │
│                                                          │
│ Collaboration:                                           │
│   • Multiple developers using same config               │
│   • Version-controlled conventions                       │
│   • Personal vs team settings                            │
└─────────────────────────────────────────────────────────┘
```

### Why This Scenario Exists

This tests whether you understand **how to configure Claude Code for team productivity** — not how to write code, but how to set up the ENVIRONMENT so Claude generates the RIGHT code consistently.

### What the Exam Tests Here

| Topic | What They're Really Asking |
|-------|---------------------------|
| **CLAUDE.md placement** | Where do team standards vs personal preferences go? |
| **Plan mode triggers** | When should you explore vs when should you just do? |
| **Custom commands** | How to share team tooling via version control? |
| **Path-specific rules** | How to apply different conventions to different file types? |
| **Permissions** | How do deny/allow rules interact across levels? |
| **Context preservation** | How does CLAUDE.md survive context compaction? |

### Key Architectural Decisions in This Scenario

**Decision 1: Plan Mode vs Direct Execution**

| Use Plan Mode | Use Direct Execution |
|---|---|
| Monolith → microservice decomposition | Fix a typo in a string |
| "How should we restructure auth?" | "Add input validation to this function" |
| Multiple valid approaches exist | One obvious correct approach |
| Changes span dozens of files | Changes in 1-3 files |
| Architectural decisions needed | Implementation is straightforward |
| Exploring unfamiliar codebase | Working in familiar code |

- **Exam trigger:** "restructure," "redesign," "multiple approaches," "architectural decision" → Plan mode
- **Exam trigger:** "fix," "add," "update," "implement this specific thing" → Direct execution

---

**Decision 2: Where to Put What**

```
Team coding standards         → project/CLAUDE.md (committed, shared)
Personal preferences          → ~/.claude/CLAUDE.md (not shared)
API handler conventions       → project/src/api/CLAUDE.md (subdirectory, conditional)
Test file conventions         → .claude/rules/tests.yaml with glob: "**/*.test.tsx"
Custom review command         → .claude/commands/review.md (committed, shared via git)
Personal shortcuts            → ~/.claude/commands/ (not shared)
MCP server configs            → .claude/settings.json (committed) + .local.json for secrets
```

**The key distinction the exam tests:**
- File-TYPE conventions (all test files, regardless of directory) → `.claude/rules/` with globs
- Directory-SPECIFIC conventions (everything in `src/api/`) → subdirectory `CLAUDE.md`
- Team-SHARED tooling → `.claude/commands/` in project repo
- Personal tooling → `~/.claude/commands/` in home directory

---

**Decision 3: Permission Conflicts**

Scenario: "Project settings deny `Bash(rm *)`. Developer's local settings allow `Bash(rm temp/*)`. What happens?"

- **Answer:** BLOCKED. Deny rules are cumulative and absolute. A deny at ANY level blocks regardless of allow at any other level. The local allow cannot override the project deny.
- **Trap:** "Local has higher priority so it overrides" → WRONG for deny rules. Priority applies to allow/general settings only.

---

**Decision 4: Context Survival**

Scenario: "After 50 turns, Claude seems to forget the project's coding conventions."

- **Answer:** CLAUDE.md content survives context compaction (system-prompt level). If Claude forgets conventions, they weren't in CLAUDE.md — they were mentioned in conversation and got summarized away.
- **Fix:** Move critical conventions from conversation messages into CLAUDE.md files.

---

### Traps Specific to This Scenario

1. **"`/plan` enters plan mode"** → WRONG. It's `Shift+Tab`. No `/plan` command exists.
2. **"Put team commands in `~/.claude/commands/`"** → WRONG. That's personal, not version-controlled.
3. **"Subdirectory CLAUDE.md overrides root"** → WRONG. Additive, not override.
4. **"Use `.claude/config.json` with a `commands` array"** → WRONG. No such file/format.
5. **"Use a separate CLAUDE.md in every test directory"** → WRONG for file-type conventions. Use `.claude/rules/` with glob patterns instead.

### Mental Model for This Scenario

Think of it as: **"I'm the engineering manager setting up Claude Code for my team of 10 developers. How do I ensure consistency without requiring manual setup on each machine?"**

- Shared via git clone → `.claude/commands/`, `.claude/settings.json`, `CLAUDE.md`, `.claude/rules/`
- Personal, not shared → `~/.claude/`, `.claude/settings.local.json`
- Automatic enforcement → permissions (deny rules), hooks
- Guideline enforcement → CLAUDE.md instructions, path-specific rules

---

---

## SCENARIO 3: Multi-Agent Research System

### The System You're Architecting

```
┌─────────────────────────────────────────────────────────┐
│              MULTI-AGENT RESEARCH SYSTEM                  │
├─────────────────────────────────────────────────────────┤
│ Architecture: Hub-and-spoke (coordinator + subagents)     │
│                                                          │
│ Coordinator (Opus):                                      │
│   • Decomposes research topics into subtasks             │
│   • Delegates to specialized subagents                   │
│   • Evaluates gaps and re-delegates                      │
│   • Synthesizes final output                             │
│                                                          │
│ Subagents:                                               │
│   • Web Search Agent (Sonnet) — finds sources            │
│   • Document Analysis Agent (Sonnet) — summarizes papers │
│   • Synthesis Agent (Sonnet) — combines findings         │
│   • Report Generator (Haiku) — formats final output      │
│                                                          │
│ Data Flow:                                               │
│   Topic → Decompose → Search → Analyze → Synthesize →   │
│   → Report                                               │
│                                                          │
│ Challenges:                                              │
│   • Coverage completeness (is the topic fully explored?) │
│   • Source attribution (provenance through pipeline)     │
│   • Error recovery (subagent timeout/failure)            │
│   • Context passing (what does each subagent receive?)   │
│   • Token efficiency (120K → 3K compression)             │
└─────────────────────────────────────────────────────────┘
```

### Why This Scenario Exists

This is the **most architecturally complex scenario** — it tests your understanding of multi-agent coordination, context isolation, error propagation, and decomposition quality. Every architectural concept from D1 + D2 + D5 appears here.

### What the Exam Tests Here

| Topic | What They're Really Asking |
|-------|---------------------------|
| **Decomposition quality** | If output is incomplete, is it the coordinator's fault or the subagent's? |
| **Context isolation** | How does information flow between agents that CAN'T share memory? |
| **Error propagation** | When one subagent fails, what should the coordinator do? |
| **Tool scoping** | Should each subagent have all 18 tools or just 3-4? |
| **Source attribution** | How do you track provenance through a summarization pipeline? |
| **Recovery patterns** | Retry? Reformulate? Proceed without? Escalate? |

### Key Architectural Decisions in This Scenario

**Decision 1: Diagnosing Incomplete Output**

Scenario: "Final report on 'AI in creative industries' covers only visual arts, missing music, writing, and film."

**Layer analysis:**
```
Layer 1: COORDINATOR decomposed "creative industries" into:
  - "AI in digital art creation"
  - "AI in graphic design"  
  - "AI in photography"
  ← PROBLEM IS HERE (too narrow decomposition)

Layer 2: Subagents received these tasks and executed correctly
Layer 3: Each subagent found relevant sources for its assigned task
Layer 4: Synthesis combined what it received correctly
```

- **Wrong answer:** "Synthesis agent should detect gaps" → blames wrong layer
- **Wrong answer:** "Web search needs broader queries" → search was never asked about music
- **Right answer:** "Coordinator's task decomposition is too narrow" → the root cause

**Key principle:** When multi-agent output is incomplete, diagnose the COORDINATOR first. Downstream agents are usually victims of narrow assignments.

---

**Decision 2: Context Passing to Subagents**

Scenario: "Coordinator needs to invoke synthesis agent with results from search and analysis phases."

**Critical fact:** Subagents inherit NOTHING. No conversation history, no CLAUDE.md, no coordinator state.

- **Wrong answer (most common):** "Rely on automatic context inheritance" → gets 35% votes in forums because people ASSUME it works like nested function scopes. It doesn't.
- **Wrong answer:** "Use shared memory with reference IDs" → requires infrastructure beyond basic SDK
- **Right answer:** "Include complete findings directly in the synthesis agent's initial prompt" → explicit context injection

**What to pass:**
- Document analysis summaries (structured, not raw)
- Source metadata (URLs, page numbers) SEPARATE from content
- Research questions/goals
- Quality criteria for synthesis

**What NOT to pass:**
- Full 120K raw search results (attention dilution)
- Coordinator's internal reasoning (irrelevant to synthesis)
- Complete conversation history (wasteful, dilutive)

---

**Decision 3: Error Recovery When Subagent Fails**

Scenario: "Web search subagent times out while researching a complex topic."

| Approach | Verdict |
|---|---|
| Return structured error (type, attempted query, partial results, alternatives) | **CORRECT** — enables intelligent recovery |
| Automatic retry with exponential backoff, then "unavailable" | Wrong — hides context from coordinator |
| Return empty results as successful | **WORST** — suppresses failure entirely |
| Terminate entire workflow | Wrong — kills other successful subagents' work |

The coordinator needs: WHAT failed, WHY it failed, WHAT was attempted, WHAT partial results exist, WHAT alternatives might work.

---

**Decision 4: Tool Distribution Across Subagents**

Scenario: "All subagents have access to all 18 system tools. Citation formatter gets called during search. Search tool invoked during synthesis."

- **Root cause:** 18 tools exceeds the degradation cliff (~5 tools/agent for reliable selection)
- **Wrong answer:** "Improve descriptions" → helps but doesn't solve fundamental overload
- **Right answer:** Scope tools per agent role:
  - Web Search Agent: 3-4 search/retrieval tools
  - Document Analysis: 2-3 summarization/extraction tools
  - Synthesis: 2-3 formatting/citation tools
  - Report Generator: 1-2 output tools

---

**Decision 5: Source Attribution Through Pipeline**

Scenario: "Final reports occasionally contain claims without proper source attribution. Investigation shows metadata gets lost during summarization."

- **Wrong answer:** "Inline citations in markdown footnotes" → summarization rewrites text and drops footnotes
- **Right answer:** "Structured data separating content from metadata" → summarization touches content field only; metadata passes through untouched in its own field

```json
{
  "content": "AI image generators have seen 400% growth...",
  "sources": [
    {"url": "...", "page": 12, "excerpt": "The market grew from..."}
  ]
}
```

---

**Decision 6: The 80/20 Scoped Tool Pattern**

Scenario: "Synthesis agent frequently needs to verify claims. 85% are simple fact-checks. Currently routes through coordinator (adds 40% latency)."

- **Right answer:** Give synthesis a lightweight `verify_fact` tool for simple lookups (85% case). Complex verifications (15%) still route through coordinator to web search agent.
- **Wrong answer:** Give synthesis full search access → over-provisioning, violates separation of concerns
- **Wrong answer:** Batch all verifications to end → synthesis depends on earlier verified facts; can't wait

---

### Traps Specific to This Scenario

1. **"Subagents communicate directly with each other"** → WRONG. Hub-and-spoke = all through coordinator.
2. **"Context inheritance between parent and child"** → WRONG. Complete isolation.
3. **"Give every agent all tools for flexibility"** → WRONG. Degrades past 5 tools.
4. **"Synthesis should detect coverage gaps"** → WRONG layer. Coordinator owns decomposition.
5. **"Larger context window fixes quality"** → WRONG. Attention dilution, not capacity.
6. **"Re-query sources fresh during synthesis"** → WRONG. Discards work, adds latency, non-deterministic.

### Mental Model for This Scenario

Think of it as: **"I'm the architect of a research organization. The coordinator is the project manager. Subagents are specialist researchers. Each researcher gets a specific brief, works independently, and returns structured results. They can't peek at each other's notes."**

- Project manager writes bad brief → researchers deliver wrong work (decomposition failure)
- Researchers can't see each other's work → no "inheritance" (context isolation)
- One researcher gets sick → project manager finds alternative, doesn't cancel everyone (error recovery)
- Each researcher has their own tools → librarian doesn't have a microscope (tool scoping)

---

---

## SCENARIO 4: Developer Productivity with Claude

### The System You're Architecting

```
┌─────────────────────────────────────────────────────────┐
│           DEVELOPER PRODUCTIVITY AGENT                    │
├─────────────────────────────────────────────────────────┤
│ SDK: Claude Agent SDK                                    │
│ Users: Software engineers on your team                    │
│                                                          │
│ Built-in Tools:                                          │
│   • Read — read file contents                            │
│   • Write — create/overwrite files                       │
│   • Edit — modify existing files (diff-based)            │
│   • Bash — execute shell commands                        │
│   • Grep — search file contents                          │
│   • Glob — find files by pattern                         │
│                                                          │
│ MCP Servers:                                             │
│   • GitHub (PRs, issues, reviews)                        │
│   • Database (schema exploration, queries)               │
│   • Internal docs (wiki, Confluence)                     │
│   • CI/CD (pipeline status, logs)                        │
│                                                          │
│ Use Cases:                                               │
│   • Explore unfamiliar codebases                         │
│   • Understand legacy systems                            │
│   • Generate boilerplate code                            │
│   • Automate repetitive tasks                            │
│   • Debug production issues                              │
│                                                          │
│ Constraints:                                             │
│   • Security (no secrets in prompts/outputs)             │
│   • Codebase size (100K+ file repos)                     │
│   • Convention adherence (team standards)                 │
│   • Tool selection (right tool for right task)           │
└─────────────────────────────────────────────────────────┘
```

### Why This Scenario Exists

This tests the **intersection of tool design, MCP configuration, and agentic behavior** in a developer context. It's less about financial risk and more about:
- Correct tool selection among many options
- MCP server setup and security
- Agent behavior when exploring large codebases
- Balancing autonomy with safety

### What the Exam Tests Here

| Topic | What They're Really Asking |
|-------|---------------------------|
| **Tool descriptions** | How do you make the agent pick Grep over Bash for searching? |
| **MCP configuration** | How do you securely configure database access without leaking credentials? |
| **Exploration strategy** | Should the agent read everything or grep first? |
| **Tool annotations** | How do you mark destructive vs read-only operations? |
| **Built-in tool selection** | When is Glob better than Bash `find`? When is Grep better than Bash `grep`? |
| **Security boundaries** | How do you prevent secrets from leaking into conversation/output? |

### Key Architectural Decisions in This Scenario

**Decision 1: Tool Description Clarity**

Scenario: "Agent uses Bash to run `grep` instead of using the built-in Grep tool, resulting in permission prompts and slower execution."

- **Root cause:** Tool descriptions don't clearly differentiate when to use built-in Grep vs Bash
- **Fix:** Expand Grep description: "Use for ALL file content searches. Faster than bash grep, no permission prompt, supports regex. Use INSTEAD OF `bash grep/rg/find`."
- The "WHEN NOT to use" boundary matters as much as "WHEN to use"

---

**Decision 2: MCP Server Security**

Scenario: "Database MCP server needs credentials. Team commits `.claude/settings.json` to git."

```json
// WRONG — credential in committed file
{"mcpServers": {"db": {"env": {"PASSWORD": "hunter2"}}}}

// RIGHT — reference environment variable
{"mcpServers": {"db": {"env": {"PASSWORD": "${DB_PASSWORD}"}}}}

// OR — put secrets in .local.json (not committed)
// .claude/settings.local.json (in .gitignore)
{"mcpServers": {"db": {"env": {"PASSWORD": "hunter2"}}}}
```

---

**Decision 3: Exploration Strategy for Large Codebases**

Scenario: "Agent reads 30 files averaging 3000 tokens each. Context is now 90K tokens. Quality degrades."

- **Wrong approach:** Read everything → context overflow, attention dilution
- **Right approach:** Grep-first strategy → find relevant files, read only what's needed
- **Pattern:** `Glob` (find files by pattern) → `Grep` (find relevant content) → `Read` (targeted reading)

---

**Decision 4: Tool Annotations for Safety**

When configuring MCP tools, annotations signal risk:

| Operation | readOnly | destructive | idempotent |
|---|---|---|---|
| `read_file` | true | false | true |
| `delete_branch` | false | **true** | false |
| `run_query` (SELECT) | true | false | true |
| `run_query` (DELETE) | false | **true** | false |
| `deploy_service` | false | true | false |

These annotations inform permission systems and user approval gates.

---

### Traps Specific to This Scenario

1. **"Use Bash for all file operations"** → WRONG. Built-in tools (Read, Grep, Glob) are preferred — no permission prompts, better UX.
2. **"Read entire codebase into context"** → WRONG. Grep-first, then targeted reads.
3. **"Store MCP credentials in settings.json"** → WRONG. Use env vars or `.local.json`.
4. **"Give the agent full database write access for productivity"** → WRONG. Read-only for exploration; write only when explicitly needed with approval gates.

### Mental Model for This Scenario

Think of it as: **"I'm setting up a powerful assistant for my dev team. How do I make it productive WITHOUT giving it the keys to production databases and deployment systems?"**

- Principle of least privilege → scoped MCP access
- Prefer specialized tools → built-in Grep over Bash grep
- Explore efficiently → don't read everything; search first
- Secure by default → secrets in env vars, destructive ops require approval

---

---

## SCENARIO 5: Claude Code for Continuous Integration

### The System You're Architecting

```
┌─────────────────────────────────────────────────────────┐
│              CLAUDE CODE IN CI/CD PIPELINE                │
├─────────────────────────────────────────────────────────┤
│ Environment: GitHub Actions / GitLab CI / Jenkins         │
│ Execution: Non-interactive (no human at keyboard)         │
│                                                          │
│ Workflows:                                               │
│   • Automated code review on every PR                    │
│   • Test generation for new/changed code                 │
│   • Security vulnerability scanning                      │
│   • Documentation update verification                    │
│                                                          │
│ Constraints:                                             │
│   • Non-interactive (no prompts, no approval dialogs)    │
│   • Deterministic (same input → consistent output)       │
│   • Cost-bounded (can't let costs spiral on every PR)    │
│   • Low false-positive rate (developers ignore noisy CI) │
│   • Latency-appropriate (blocking vs non-blocking)       │
│                                                          │
│ Integration:                                             │
│   • PR comments with review feedback                     │
│   • Exit codes for pass/fail gates                       │
│   • JSON output for pipeline consumption                 │
│   • Budget caps per invocation                           │
│                                                          │
│ Cost Optimization:                                       │
│   • Batch API for non-blocking workflows                 │
│   • Prompt caching for repeated schemas/tools            │
│   • Model tiering (Haiku for triage, Sonnet for review)  │
└─────────────────────────────────────────────────────────┘
```

### Why This Scenario Exists

This tests the **operational side** of Claude Code — running it without humans, in pipelines, at scale, with cost controls. It's where Domain 3 (configuration) meets Domain 4 (prompt engineering for reliability).

### What the Exam Tests Here

| Topic | What They're Really Asking |
|-------|---------------------------|
| **Non-interactive mode** | How do you run Claude Code in a pipeline without it hanging? |
| **Output formats** | How do you get structured output a pipeline can parse? |
| **Batch vs real-time** | Which workflows can tolerate 24h latency for 50% savings? |
| **False positive management** | How do you make CI feedback useful, not noisy? |
| **Multi-pass review** | How do you review 14 files without attention dilution? |
| **Cost control** | How do you cap spending per PR review? |

### Key Architectural Decisions in This Scenario

**Decision 1: The Hanging Pipeline**

Scenario: "Pipeline script runs `claude "Analyze this PR"` but the job hangs indefinitely."

- **Answer:** Use `-p` flag: `claude -p "Analyze this pull request for security issues"`
- **Why it hangs:** Without `-p`, Claude Code enters interactive mode and waits for user input
- **Traps:** `CLAUDE_HEADLESS=true` (doesn't exist), `--batch` (doesn't exist), `< /dev/null` (doesn't properly enable non-interactive mode)

**Full CI invocation pattern:**
```bash
claude -p \
  --output-format json \
  --max-budget-usd 0.50 \
  --model claude-sonnet-5 \
  --allowedTools "Read,Grep,Glob" \
  "Review this PR for security vulnerabilities. Output findings as JSON."
```

---

**Decision 2: Batch API for Cost Savings**

Scenario: "Two workflows: (1) blocking pre-merge check, (2) overnight tech debt report. Manager proposes batch for both."

| Workflow | Blocking? | Latency Tolerant? | API Choice |
|---|---|---|---|
| Pre-merge check | YES (devs wait) | NO | **Real-time** |
| Overnight tech debt | NO | YES (read next morning) | **Batch (50% savings)** |

- **Right answer:** Batch for tech debt only; real-time for pre-merge
- **Wrong answer:** "Batch both with polling" → adds complexity, unreliable for blocking workflows
- **Wrong answer:** "Real-time for both" → wastes 50% savings on latency-tolerant workflow

---

**Decision 3: Multi-Pass Review for Large PRs**

Scenario: "Single-pass review of 14 files produces inconsistent results — detailed on some files, superficial on others, contradictions between files."

- **Root cause:** Attention dilution — too much content in one pass
- **Solution:** Split into passes:
  1. **Local pass:** Review each file individually for local issues (bugs, style, logic)
  2. **Integration pass:** Examine cross-file data flow, dependency consistency, API contract alignment
- **Wrong answer:** "Larger context window" → doesn't fix attention quality
- **Wrong answer:** "Run 3 passes, flag only issues appearing in 2+" → suppresses real bugs that are intermittently detected

---

**Decision 4: Minimizing False Positives**

Scenario: "Developers are ignoring CI review comments because too many are nitpicks or incorrect."

Strategies:
- Focus on high-severity issues only (bugs, security) — skip style nits
- Require concrete evidence (line number, specific failure scenario)
- Use validation step: after finding issues, verify each one by reasoning about whether it's actually reachable
- Separate "blocking" findings (must fix) from "advisory" findings (consider fixing)

---

**Decision 5: Structured Output for Pipeline Consumption**

```bash
# Output JSON for pipeline parsing
claude -p --output-format json "Review this PR..."

# Pipeline consumes JSON
FINDINGS=$(claude -p --output-format json "..." | jq '.findings')
if [ $(echo "$FINDINGS" | jq 'length') -gt 0 ]; then
  exit 1  # Fail the gate
fi
```

---

### Traps Specific to This Scenario

1. **`--batch` flag** → Doesn't exist. Use `-p` for non-interactive.
2. **"Batch API for blocking checks"** → WRONG. 24h SLA is inappropriate for developers waiting.
3. **"Single pass for all files"** → Leads to attention dilution on large PRs.
4. **"Run 3 independent reviews, intersect results"** → Suppresses intermittent bug detection.
5. **"Auto-merge if no issues found"** → WRONG. Always require human approval gate.
6. **"Same-session self-review"** → Confirmation bias. Use separate Claude instance.

### Mental Model for This Scenario

Think of it as: **"I'm the DevOps engineer making Claude Code useful in our CI pipeline. It must run unattended, stay within budget, and produce feedback developers actually read."**

- Non-interactive → `-p` flag
- Cost control → `--max-budget-usd`, batch for tolerant workflows
- Quality → multi-pass review, verify findings before reporting
- Trust → low false positives, evidence-backed findings, human gate before merge

---

---

## SCENARIO 6: Structured Data Extraction

### The System You're Architecting

```
┌─────────────────────────────────────────────────────────┐
│           STRUCTURED DATA EXTRACTION SYSTEM               │
├─────────────────────────────────────────────────────────┤
│ Input: Unstructured documents (invoices, contracts,       │
│        medical records, emails, PDFs)                     │
│                                                          │
│ Output: Validated JSON matching defined schemas           │
│                                                          │
│ Pipeline:                                                │
│   Document → Extract → Validate → Retry if invalid →     │
│   → Output to downstream system                          │
│                                                          │
│ Key Mechanisms:                                          │
│   • Forced tool_choice for structural compliance         │
│   • JSON Schema with nullable fields + enum escapes      │
│   • Validation-retry loops with specific feedback        │
│   • Batch processing for high volume                     │
│   • Prompt caching for repeated schemas                  │
│                                                          │
│ Challenges:                                              │
│   • Missing data (field not present in source)           │
│   • Ambiguous data (multiple interpretations)            │
│   • Edge cases (unusual formats, languages)              │
│   • Scale (10,000+ documents per day)                    │
│   • Accuracy (downstream systems depend on output)       │
│                                                          │
│ Quality Metrics:                                         │
│   • Structural validity (100% with forced tool_choice)   │
│   • Factual accuracy (<1% fabrication target)            │
│   • Coverage (% of fields successfully extracted)        │
│   • Throughput (documents per hour)                      │
└─────────────────────────────────────────────────────────┘
```

### Why This Scenario Exists

This is the **purest prompt engineering + structured output** scenario. It tests whether you understand how to get Claude to produce reliable, validated, honest output at scale — and what to do when information is genuinely missing.

### What the Exam Tests Here

| Topic | What They're Really Asking |
|-------|---------------------------|
| **Forced tool_choice** | Why is this better than "respond in JSON" in the prompt? |
| **Schema design** | How do nullable fields and enum escapes prevent fabrication? |
| **Validation-retry** | What makes retry feedback effective vs ineffective? |
| **Batch processing** | When to use Batch API for high-volume extraction? |
| **Prompt caching** | How to cache the schema/examples for repeated extractions? |
| **Handling missing data** | What happens when a required field isn't in the source? |

### Key Architectural Decisions in This Scenario

**Decision 1: Ensuring Structural Compliance**

| Approach | Compliance Rate | When to Use |
|---|---|---|
| "Respond in JSON" (prompt instruction) | ~85% | Never for production pipelines |
| `tool_choice: {"type": "tool", "name": "extract"}` | **100%** | Always for production |

**Why prompt-only fails:** Claude occasionally adds markdown formatting (```json), explanatory text, or invalid syntax (trailing commas). With forced `tool_choice`, output MUST match the schema — the SDK validates at the API layer.

---

**Decision 2: Schema Design to Prevent Fabrication**

Scenario: "The system extracts invoice data. 7% of extracted phone numbers are fabricated — they look plausible but aren't in the source document."

**Root cause:** All fields are `required` and non-nullable. When data is missing, Claude invents a plausible value.

**Fix — Nullable fields + enum escapes:**

```json
{
  "properties": {
    "phone": {
      "type": ["string", "null"],
      "description": "Customer phone from document. null if not present."
    },
    "payment_status": {
      "type": "string",
      "enum": ["paid", "unpaid", "partial", "unclear"],
      "description": "Payment status. Use 'unclear' if ambiguous in source."
    }
  }
}
```

| Schema Design | Fabrication Rate |
|---|---|
| All required, non-nullable | ~7% (Claude invents values) |
| Nullable + enum escapes ("unknown", "unclear", null) | **<1%** |

**Key principle:** Give Claude an honest exit. If it CAN say "I don't know" (null) or "this is ambiguous" (unclear), it will. If forced to provide a value, it fabricates.

---

**Decision 3: Validation-Retry Loop Design**

```
Attempt 1: Generate extraction
    ↓
Validate: JSON schema check → business rule check → cross-field consistency
    ↓ (if invalid)
Retry with SPECIFIC feedback:
    "Field 'invoice_date' has value '2024-13-45' which is not a valid date.
     Dates must be ISO 8601 format (YYYY-MM-DD). Re-read the document
     and extract the actual date from the header section."
    ↓
Attempt 2: Usually succeeds (converges in 2-3 with specific feedback)
    ↓ (if still invalid)
Attempt 3: Final try with worked example
    ↓ (if still fails)
Flag for human review (don't retry forever)
```

| Feedback Type | Convergence | Example |
|---|---|---|
| Generic: "Try again" | Does NOT converge | Same error repeated |
| Specific: "Invalid date at field X, format should be Y" | 2-3 attempts | Fixes the specific issue |
| Worked example: "For this document type, here's correct extraction..." | 1-2 attempts | Best for novel formats |

**Validation order (cheapest first):**
1. JSON schema validation (instant, free)
2. Business rules (regex, range checks — fast)
3. Cross-field consistency (dates, amounts — moderate)
4. LLM-based semantic check (expensive — last resort)

---

**Decision 4: Batch Processing for Scale**

Scenario: "10,000 invoices per day. Each extraction costs ~$0.05 real-time. Manager wants to reduce costs."

- **Batch API:** 50% discount → $0.025/invoice → saves $250/day
- **Appropriate because:** Invoice extraction is latency-tolerant (not blocking anyone in real-time)
- **Add caching:** Schema + few-shot examples are identical across all 10K invocations → cache the prefix

**Combined savings:**
```
Real-time, no caching:     $0.05 × 10,000 = $500/day
Batch only:                $0.025 × 10,000 = $250/day  (50% savings)
Batch + caching:           ~$0.005 × 10,000 = $50/day  (~90% savings)
```

---

**Decision 5: Handling Edge Cases Gracefully**

| Edge Case | Wrong Response | Right Response |
|---|---|---|
| Field missing from document | Fabricate plausible value | Return `null` |
| Ambiguous data | Pick one interpretation | Return `"unclear"` + raw text |
| Multiple values possible | Pick first one | Return array or most confident |
| Unrecognized format | Fail silently | Return `is_error: true` with details |
| Partial extraction possible | Return nothing | Return what you CAN extract + list what's missing |

**Key principle:** Transparent partial failure > silent fabrication. Downstream systems can handle null/unclear/missing. They can't handle plausible-looking fabricated data.

---

**Decision 6: Prompt Caching for Repeated Extractions**

```
REQUEST STRUCTURE (10,000 identical schema requests):

[System prompt + extraction instructions]     ← CACHED (90% savings)
[Tool schema (the extraction tool)]           ← CACHED (90% savings)  
[Few-shot examples (3-5 reference docs)]      ← CACHED (90% savings)
[THIS specific document to extract from]      ← DYNAMIC (different each request)
```

- Same schema + examples across all 10K documents = massive cache hits
- Document content is the only dynamic part
- Static prefix must come BEFORE dynamic content (prefix matching)
- Minimum cacheable size applies (1024-4096 tokens depending on model)

---

### Traps Specific to This Scenario

1. **"Just ask Claude to respond in JSON"** → 15% failure rate. Use forced tool_choice.
2. **"Make all fields required"** → Forces 7% fabrication. Use nullable types.
3. **"Retry with 'try again'"** → Generic feedback doesn't converge. Be specific.
4. **"Use real-time API for all 10K daily extractions"** → Wastes 50% on latency-tolerant workload.
5. **"Cache the entire request"** → Dynamic document content invalidates cache. Only cache static prefix.
6. **"100% test coverage ensures extraction accuracy"** → Impossible for natural language. Use sampling + monitoring.
7. **"Validate after all documents processed"** → Too late. Validate per-document in the loop.

### Mental Model for This Scenario

Think of it as: **"I'm building a production data pipeline. Every fabricated value pollutes my database. Every structural error crashes my downstream system. I need GUARANTEES, not hopes."**

- Structural compliance → forced tool_choice (100%)
- Factual honesty → nullable fields + enum escapes (<1% fabrication)
- Error recovery → specific validation feedback (converges in 2-3)
- Cost at scale → batch API + caching (90%+ savings)
- Edge cases → transparent failure over silent fabrication

---

---

## CROSS-SCENARIO COMPARISON

### Which Principles Apply Where?

| Principle | S1 (Support) | S2 (Claude Code) | S3 (Research) | S4 (Dev Prod) | S5 (CI/CD) | S6 (Extraction) |
|---|---|---|---|---|---|---|
| Programmatic enforcement | ★★★ | ★★ | ★ | ★★ | ★★ | ★★★ |
| Tool descriptions | ★★★ | ★★ | ★★ | ★★★ | ★ | ★ |
| Context isolation | ★ | ★ | ★★★ | ★ | ★ | ★ |
| Escalation criteria | ★★★ | ★ | ★★ | ★ | ★ | ★ |
| Batch vs real-time | ★ | ★ | ★ | ★ | ★★★ | ★★★ |
| Schema design | ★ | ★ | ★ | ★ | ★★ | ★★★ |
| Tool scoping | ★★ | ★★ | ★★★ | ★★ | ★ | ★ |
| Multi-pass review | ★ | ★ | ★★ | ★ | ★★★ | ★ |
| Prompt caching | ★ | ★ | ★ | ★ | ★★ | ★★★ |
| Plan mode | ★ | ★★★ | ★ | ★★ | ★ | ★ |

### The Universal Decision Rule

Across ALL scenarios, the same meta-rule applies:

> **How likely is the failure? How bad is the consequence?**

| Failure Rate | Consequence | Appropriate Response |
|---|---|---|
| Low (3%) | Low (wrong tone) | Prompt instruction |
| Low (3%) | High (wrong refund) | Programmatic gate |
| High (15%) | Low (format issue) | Validation-retry |
| High (15%) | High (data fabrication) | Schema redesign + forced tool_choice |

---

## EXAM-DAY QUICK CHECK

When you read a scenario question:

1. **Which scenario am I in?** → Determines which principles are most relevant
2. **What's the binding constraint?** → Financial risk? Latency? Cost? Coverage? Accuracy?
3. **What layer failed?** → Decomposition? Context passing? Execution? Synthesis?
4. **Is the proposed fix proportional?** → Simple enough? Too simple? Over-engineered?
5. **Does the option describe something that exists?** → Eliminate non-existent APIs/flags

Then pick the answer that satisfies the binding constraint with appropriate proportionality.

---

*These 6 scenarios cover all 5 domains. Master the architectural thinking within each, and you'll recognize the pattern in any question the exam throws.*
