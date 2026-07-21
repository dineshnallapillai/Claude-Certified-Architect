# CCA-F Exam — Comprehensive Notes, Traps & Decision Frameworks

> Consolidated from: pass reports (Medium, LinkedIn, Substack, note.com), ExamTopics community threads, GitHub Ambassador repo, official exam guide excerpts, and multiple certification prep platforms.

> **Exam Format Reminder:** 60 scenario-based questions, 120 minutes, 720/1000 passing (scaled), 4 of 6+ scenarios randomly assigned, single best answer, no guessing penalty.

---

## PART 1: THE FIVE RECURRING EXAM PATTERNS

Every passer independently identifies the same five meta-patterns. If you internalize these, you can eliminate 2 options instantly on most questions.

---

### Pattern 1: Programmatic Enforcement vs. Prompt-Based Guidance

**The Core Rule:** When something MUST happen → use code (hooks, gates, validation). When something SHOULD happen → use prompts (instructions, examples).

| Requirement Type | Enforcement Mechanism | Compliance Rate |
|---|---|---|
| Non-negotiable (financial, security, compliance) | PreToolUse hook, prerequisite gate, validation layer | 100% deterministic |
| Best practice (style, formatting, approach) | System prompt instructions | ~85-97% probabilistic |
| Strong preference (ordering, tone) | Few-shot examples in prompt | ~90-95% probabilistic |

**How this appears on the exam:**
- Scenario describes a 5-15% failure rate on a critical path
- One option adds/improves prompt instructions
- One option adds a programmatic hook or gate
- The programmatic option is correct when the failure has financial/security/compliance consequences

**Trap:** The prompt-based option will sound reasonable ("add clear instructions that X must happen first"). It IS reasonable for low-stakes preferences. It is WRONG for non-negotiable business logic.

**Decision test:** "If this fails 3% of the time, is that acceptable?" If NO → code enforcement. If YES → prompt is fine.

---

### Pattern 2: Context Isolation (Subagents Get Nothing)

**The Core Rule:** Subagents start with an EMPTY conversation. They inherit NOTHING from the coordinator — not the conversation history, not the CLAUDE.md, not the system prompt, not the tools, not the state.

**How this appears on the exam:**
- Scenario asks how a coordinator should pass information to a subagent
- Trap option says "rely on context inheritance" or "subagent accesses coordinator's memory"
- Correct answer involves explicitly passing context in the subagent's prompt

**Critical implications:**
- Coordinator MUST serialize all needed context into the subagent's initial prompt
- Each subagent should receive ONLY the context relevant to its specific task (data minimization)
- Passing "everything" is wrong — it wastes tokens AND dilutes attention
- Subagents cannot "call back" to the coordinator mid-execution in standard patterns

**The most common wrong answer on the exam** (per community forums): "Rely on automatic context inheritance between parent and child agents." This gets 35% of votes in ExamTopics discussions because it's how people ASSUME it works.

---

### Pattern 3: Tool Scoping & The Degradation Cliff

**The Core Rule:** Tool routing accuracy degrades past ~5 tools per agent. At 18 tools, expect 25-40% misrouting.

| Tools per Agent | Routing Accuracy | Recommendation |
|---|---|---|
| 1-5 | High (~95%+) | Sweet spot for specialist agents |
| 6-10 | Moderate (~85%) | Acceptable with excellent descriptions |
| 11-15 | Degraded (~70%) | Split into specialist subagents |
| 16+ | Severely degraded (~60%) | Architectural problem |

**How this appears on the exam:**
- Scenario describes tool selection failures, wrong tools being called
- Options include: improve descriptions, add examples, reduce tool count, add routing layer
- The answer depends on CURRENT tool count and whether descriptions have been optimized

**Decision framework:**
1. If tool count > 10 → scoping/splitting is the answer (reduce per-agent tool count)
2. If tool count ≤ 5 AND descriptions are minimal → improve descriptions first
3. Few-shot examples are SECONDARY reinforcement, not primary fix
4. Routing classifiers are OVER-ENGINEERED when simpler fixes haven't been tried

---

### Pattern 4: Batch API — Latency vs. Cost Tradeoff

**The Core Rule:** Batch API = 50% cheaper but NO latency guarantee (up to 24 hours). Only appropriate for non-blocking, latency-tolerant workflows.

| Workflow Type | API Choice | Reason |
|---|---|---|
| Developer waiting for merge approval | Real-time | Blocking, latency-sensitive |
| Overnight tech debt report | Batch | Non-blocking, read next morning |
| Interactive chat | Real-time | User expects immediate response |
| Weekly compliance audit | Batch | Scheduled, no urgency |
| CI/CD gate (blocks pipeline) | Real-time | Blocking, developers waiting |
| Bulk document classification | Batch | High volume, no urgency |

**How this appears on the exam:**
- Scenario has TWO workflows — one latency-sensitive, one tolerant
- Option says "use batch for both" (wrong — saves cost but blocks users)
- Correct answer matches API type to workflow latency requirements

**Trap:** "Batch with polling fallback to real-time" sounds clever but adds complexity without benefit. Simpler: use the right API for each workflow from the start.

---

### Pattern 5: Simplest Effective Solution First

**The Core Rule:** The exam rewards the SIMPLEST option that fully solves the stated problem. Over-engineering is a wrong answer.

**Escalation ladder (try in order):**
1. Fix the prompt/description (cheapest, fastest)
2. Add validation/retry loop (moderate effort)
3. Add programmatic gate/hook (when #1 can't give deterministic guarantees)
4. Add routing classifier/ML model (only when #1-3 are demonstrably insufficient)
5. Redesign architecture (only when the problem is architectural)

**How this appears on the exam:**
- Multiple options solve the problem
- One is a simple prompt fix, one adds a classifier, one redesigns the system
- If the prompt fix hasn't been tried yet → it's the "first step" answer
- If the scenario says "despite clear instructions, 12% failure rate" → prompt already failed, escalate to code

**Key phrase in question stems:**
- "Most effective FIRST STEP" → simplest solution
- "Most RELIABLE solution" → may justify more engineering
- "Given that instructions have already been added" → prompt approach already failed

---

## PART 2: NUMBERED ANTI-PATTERNS (Referenced in Exam Options)

These anti-patterns appear as TRAP ANSWERS. If you see these patterns in an option, it's almost certainly wrong.

---

### AP-1: Text-Based Loop Termination

**Wrong:** Scanning Claude's text for "I'm done" or "Here's your answer" to decide when to stop the loop.
**Right:** Check `stop_reason` field. `"end_turn"` = stop. `"tool_use"` = continue. `"max_tokens"` = truncated.

**Why it fails:** Claude might say "Here's what I found so far" mid-loop without being done. Text parsing is fragile and non-deterministic.

---

### AP-2: Prompt-Only Enforcement for Financial Operations

**Wrong:** "Add instructions that refund amount must not exceed $500 without manager approval"
**Right:** PreToolUse hook that checks refund amount and blocks execution with exit code 2 if over threshold.

**Why it fails:** Prompts leak ~3-15% of the time. A 3% leak on $500+ refunds = real financial exposure.

---

### AP-3: Sentiment-Based Escalation

**Wrong:** "Detect customer frustration via sentiment analysis and escalate when negative sentiment exceeds threshold"
**Right:** Escalate based on CASE COMPLEXITY or POLICY GAPS, not customer emotion.

**Why it fails:** An angry customer may have a trivial issue (simple refund). A calm customer may have an impossibly complex issue. Sentiment ≠ complexity.

---

### AP-4: Self-Reported Confidence Scores

**Wrong:** "Have the agent rate its own confidence 1-10 and route to humans below threshold"
**Right:** Explicit escalation criteria based on observable conditions (policy not found, exception required, customer explicitly requests human).

**Why it fails:** LLMs are poorly calibrated. An overconfident model rates complex cases 8/10. An underconfident model rates routine cases 4/10. Confidence scores don't correlate with actual case difficulty.

---

### AP-5: Same-Session Self-Review

**Wrong:** "After generating code, have the same Claude instance review it for bugs"
**Right:** Spawn a SEPARATE Claude instance (fresh context, no inherited bias) for independent review.

**Why it fails:** Same-session review = confirmation bias. The model that wrote the code has the same blind spots when reviewing it. A fresh instance with no context of how the code was generated provides truly independent analysis.

---

### AP-6: Summarizing Critical State

**Wrong:** Letting conversation compaction summarize customer details, case facts, or reference data.
**Right:** Pin critical facts in an immutable CASE_FACTS block that's re-injected every turn and never summarized.

**Why it fails:** Summarization is lossy. If a customer said their order was #12345 twenty turns ago, that specific number may be lost to summarization. Pin it.

---

### AP-7: Full Context Dump to Subagents

**Wrong:** "Pass all 90K tokens of research to the synthesis subagent"
**Right:** Pass only the specific findings, summaries, and source metadata relevant to the synthesis task.

**Why it fails:** Attention dilution (lost-in-the-middle effect). The subagent's quality degrades as irrelevant context grows. Also wastes tokens/money on every call.

---

### AP-8: Prompt-Only JSON Output

**Wrong:** "Include in system prompt: 'Always respond in valid JSON format'"
**Right:** Use `tool_choice` with a specific tool whose schema defines the exact JSON structure.

| Approach | Structural Compliance |
|---|---|
| Prompt-only ("respond in JSON") | ~85% — fails 15% of the time |
| Forced `tool_choice` with schema | 100% — structurally guaranteed |

**Why it fails:** Claude occasionally adds markdown formatting, explanatory text, or invalid JSON syntax when relying on prompt instructions alone.

---

### AP-9: Required Non-Nullable Fields in Extraction

**Wrong:** Making all fields `required` and `non-nullable` in extraction schemas.
**Right:** Make fields nullable or include enum escapes like `"not_found"`, `"unclear"`, `"N/A"`.

| Schema Design | Fabrication Rate |
|---|---|
| All fields required, non-nullable | ~7% hallucinated values |
| Nullable fields + enum escapes | <1% fabrication |

**Why it fails:** When a field is required but the information doesn't exist in the source, Claude fabricates a plausible value rather than returning nothing.

---

### AP-10: Single-Pass Review of Large PRs

**Wrong:** Reviewing 14+ files in a single Claude call and expecting consistent quality.
**Right:** Multi-pass: file-by-file local analysis + separate integration/cross-file pass.

**Why it fails:** Attention dilution. The model gives detailed feedback on the first few files but becomes increasingly superficial. May also produce contradictory feedback (flagging a pattern in file 3 while approving identical code in file 11).

---

### AP-11: Generic Retry Feedback

**Wrong:** "Your output was invalid. Please try again."
**Right:** "Invalid JSON at position 245: trailing comma after last array item. JSON does not allow trailing commas. Remove the comma after 'value3'."

| Feedback Type | Convergence |
|---|---|
| Generic ("try again") | Does not converge — same error repeated |
| Specific (exact error + location + fix) | Converges in 2-3 attempts |

---

### AP-12: Over-Provisioning Tools "Just in Case"

**Wrong:** Giving every agent access to all available tools so it "has what it needs."
**Right:** Each agent gets ONLY the tools for its specific role. 3-5 tools per specialist agent.

**Why it fails:** Every additional tool adds decision complexity. Irrelevant tools get called when descriptions partially match. The cognitive overhead of evaluating 18 options per decision compounds into frequent errors.

---

### AP-13: Coordinator Gives Procedural Steps Instead of Goals

**Wrong:** "Step 1: Search Google Scholar. Step 2: Filter by citation count. Step 3: Read abstracts..."
**Right:** "Find 10+ authoritative, recent sources on [topic]. Quality criteria: peer-reviewed, post-2022, relevant methodology."

**Why it fails:** Procedural instructions prevent adaptation. If Google Scholar has no results, the agent gives up instead of trying alternative databases. Goal-based prompting lets the agent exercise judgment on HOW to achieve the outcome.

---

### AP-14: Suppressing Errors as Success

**Wrong:** Catching a tool timeout and returning empty results marked as `is_error: false` (successful).
**Right:** Return structured error with `is_error: true`, failure type, what was attempted, and partial results if any.

**Why it fails:** The coordinator/agent can't make recovery decisions if it doesn't know a failure occurred. Suppressed errors produce silently incomplete outputs.

---

### AP-15: Terminating Entire Workflow on Single Failure

**Wrong:** Top-level exception handler that kills the entire multi-agent pipeline when one subagent times out.
**Right:** Structured error propagation to coordinator, which evaluates whether to retry, substitute, or proceed without that component.

**Why it fails:** Multi-agent systems should be resilient. One subagent's timeout shouldn't kill 4 other successful subagents' work. The coordinator should make intelligent recovery decisions.

---

## PART 3: CRITICAL NUMBERS & THRESHOLDS

These specific numbers appear in exam questions. Memorize them.

---

### API & Model Facts

| Fact | Value | Context |
|---|---|---|
| stop_reason for loop control | `"tool_use"` / `"end_turn"` / `"max_tokens"` | ONLY reliable loop signal |
| Forced tool_choice compliance | 100% | Structural guarantee |
| Prompt-only JSON compliance | ~85% | 15% failure rate |
| Max tools before degradation | ~5 per agent | Accuracy drops ~8%/tool after this |
| Max items per batch | 100,000 | Hard API limit |
| Batch discount | 50% | Off real-time pricing |
| Batch SLA | 24 hours | NO latency guarantee within that window |
| Cache read discount | 90% | Off input token cost |
| Cache write premium (5-min) | 25% | One-time cost to establish cache |
| Cache write premium (1-hour) | 100% | Higher TTL = higher write cost |
| Cache TTL (standard) | 5 minutes | Default ephemeral |
| Cache TTL (extended) | 1 hour | Must specify `"ttl": "1h"` |
| Max cache breakpoints | 4 per request | Can't cache more than 4 segments |
| Min cacheable (Opus/Haiku 4.5) | 4096 tokens | Below this = never cached |
| Min cacheable (Fable 5/Sonnet 4.6) | 2048 tokens | Model-dependent minimum |
| Min cacheable (Sonnet 4.5/4.1) | 1024 tokens | Lowest threshold |
| Fabrication with required fields | ~7% | When field is required but data missing |
| Fabrication with nullable fields | <1% | Adding escape hatch nearly eliminates it |
| Validation-retry convergence | 2-3 attempts | WITH specific error feedback |

---

### Hook Exit Codes

| Exit Code | Effect | Use Case |
|---|---|---|
| 0 | Allow (proceed with tool execution) | Default pass-through |
| 2 | BLOCK (prevent tool execution) | Security gates, amount limits |
| Non-zero (other) | Error (treated as hook failure) | Unexpected hook crash |
| stdout output | Fed back to Claude as context | Explain WHY blocked, suggest alternative |

---

### HTTP Error Codes

| Code | Meaning | Action | Exam Trap |
|---|---|---|---|
| 400 | Invalid request (malformed, context overflow) | Fix request, don't retry | Context too long = 400, NOT 413 |
| 401 | Invalid API key | Fix credentials | — |
| 403 | Permission denied | Check access | — |
| 413 | Payload too large (raw HTTP size) | Reduce payload | NOT context overflow |
| 429 | Rate limited | Retry with exponential backoff | — |
| 500 | Internal server error | Retry cautiously | — |
| 529 | Overloaded (Anthropic-specific) | Retry with backoff + jitter | Unique to Anthropic |

**Critical distinction:** Context window overflow (too many tokens) = **400** not 413. The 413 is for raw HTTP payload size limits (e.g., sending a 100MB image).

---

### MCP Primitives (Who Controls What)

| Primitive | Controlled By | Exposed To | Purpose |
|---|---|---|---|
| **Tools** | Model (Claude decides when to call) | Model | Actions Claude can take |
| **Resources** | Application (client-side code) | Application | Data the app can read |
| **Prompts** | User (explicitly selected) | User | Pre-built templates |

**Exam trap:** Confusing who controls each primitive. Tools are MODEL-controlled (Claude decides). Resources are APPLICATION-controlled (your code decides). Prompts are USER-controlled (user explicitly selects).

---

### Model Selection Tiers

| Tier | Models | Best For | Tokens/sec | Relative Cost |
|---|---|---|---|---|
| Flagship | Opus 4.8, Fable 5 | Complex reasoning, multi-step | Slowest | Highest |
| Balanced | Sonnet 5, Sonnet 4.6 | General work, most tasks | Medium | Medium |
| Speed/Cost | Haiku 4.5 | Routing, classification, simple tasks | Fastest | Lowest |

**Exam application:** The answer is almost never "use the most expensive model." Test first, benchmark quality/speed/cost, select the tier that meets ALL constraints.

---

## PART 4: DECISION FRAMEWORKS FOR AMBIGUOUS QUESTIONS

---

### Framework 1: The Three-Point Priority

When two answers both seem correct, apply this hierarchy:

1. **FACT** — Is the option grounded in a verifiable technical truth? → Always wins
2. **GUIDELINE** — Does it follow Anthropic's documented best practices? → Wins over procedure
3. **PROCEDURE** — Does it describe correct implementation steps? → Lowest priority

**Example:** Option A says "Use stop_reason to control the loop" (FACT — this field exists and works this way). Option B says "Follow the recommended pattern of checking response completeness" (GUIDELINE — correct but vaguer). **A wins** because it's grounded in a specific technical truth.

---

### Framework 2: Root Cause Layer Analysis

When a multi-agent system produces bad output, diagnose by layer:

```
Layer 1: COORDINATOR (task decomposition)
  ↓ Was the task decomposed correctly?
Layer 2: DELEGATION (context passing)
  ↓ Did each subagent receive the right context?
Layer 3: EXECUTION (subagent behavior)
  ↓ Did each subagent perform its task correctly?
Layer 4: SYNTHESIS (result combination)
  ↓ Were results combined correctly?
```

**Exam application:** The scenario will describe a symptom (incomplete output). You must identify which LAYER failed. Common trap: blaming Layer 3 (execution) when the real problem is Layer 1 (decomposition).

**Example:** Final report covers only visual arts, missing music/film. Subagents executed correctly. Problem is Layer 1 — coordinator decomposed "creative industries" into only visual subtasks.

---

### Framework 3: The Constraint Binding Test

When choosing between two plausible options:

1. Identify the **binding constraint** in the scenario (cost? latency? security? accuracy?)
2. Ask: which option satisfies the binding constraint?
3. The option that addresses the binding constraint wins, even if the other option is generally "better"

**Example:** Scenario says "budget is limited but quality must be acceptable." Binding constraint = budget. Option A (expensive but highest quality) loses to Option B (test and find cheapest tier that meets quality bar).

---

### Framework 4: The "Has This Been Tried?" Test

For "first step" or "most effective fix" questions:

1. Has the simplest fix been attempted? (prompt improvement, description clarity)
2. If YES (scenario says "despite clear instructions...") → escalate to programmatic enforcement
3. If NO (scenario describes a new problem) → try the simple fix first

**Trap:** Jumping to ML classifiers or architectural redesigns when improving a tool description would solve it.

---

### Framework 5: Proportionality Test

The solution should be proportional to the problem:

| Problem Severity | Appropriate Response |
|---|---|
| 3-5% occasional incorrect formatting | Improve prompt/examples |
| 12% skip rate on critical verification | Programmatic hook/gate |
| 40% wrong tool selection | Architectural change (tool scoping) |
| Complete architectural mismatch | Redesign |

---

## PART 5: DOMAIN-SPECIFIC TRAPS

---

### Domain 1: Agentic Architecture (27% weight)

**TRAP 1.1:** "The agent should handle everything autonomously to maximize first-contact resolution"
- **CORRECT:** Define clear escalation boundaries. Autonomy WITHOUT boundaries leads to worse outcomes (the agent attempts complex cases it can't handle while escalating simple ones it could).

**TRAP 1.2:** "`response.is_finished`" or "`response.complete`" or "`response.done`"
- **CORRECT:** None of these fields exist. Only `stop_reason` determines loop control.

**TRAP 1.3:** "Hub-and-spoke topology means subagents communicate with each other directly"
- **CORRECT:** In hub-and-spoke, ALL communication routes through the coordinator. Subagents never communicate directly. If they need information from another subagent, the coordinator mediates.

**TRAP 1.4:** "Use the same model for all agents in a multi-agent system"
- **CORRECT:** Use tiered models — Haiku for routing/classification, Sonnet for general work, Opus for complex reasoning. Match model capability to task complexity.

**TRAP 1.5:** "Subagents can access the coordinator's tool set"
- **CORRECT:** Each agent (coordinator and subagents) has its own explicitly-defined tool set. Tools are scoped per agent. A subagent sees ONLY the tools assigned to it.

---

### Domain 2: Tool Design & MCP (18% weight)

**TRAP 2.1:** "Tool descriptions only need to say what the tool does"
- **CORRECT:** Tool descriptions must include: WHAT it does + WHEN to use it + WHEN NOT to use it + what it RETURNS + example inputs. The "when NOT to use it" boundary is what prevents tool confusion.

**TRAP 2.2:** "`is_error: true` is optional — just putting 'Error:' in the content works"
- **CORRECT:** Without `is_error: true`, Claude may interpret error text as successful data. Example: `"Error: User not found"` without the flag might be treated as the user's name being "Error: User not found."

**TRAP 2.3:** "MCP Resources are controlled by the model (like tools)"
- **CORRECT:** Resources are APPLICATION-controlled. The application code decides when to read resources, not Claude. Only Tools are model-controlled.

**TRAP 2.4:** "HTTP+SSE is the recommended remote MCP transport"
- **CORRECT:** HTTP+SSE is LEGACY/deprecated. Streamable HTTP is the current recommended transport for remote MCP servers. stdio for local servers.

**TRAP 2.5:** "More tools = more capable agent"
- **CORRECT:** More tools = more confusion. Accuracy drops ~8% per tool past 5. A focused agent with 4 well-described tools outperforms a "Swiss Army knife" with 18 tools.

---

### Domain 3: Claude Code Configuration (20% weight)

**TRAP 3.1:** "Type `/plan` to enter plan mode"
- **CORRECT:** Plan mode is toggled with `Shift+Tab` in interactive TUI. In CLI/headless: `--plan` flag. There is no `/plan` slash command.

**TRAP 3.2:** "Local settings override project deny rules"
- **CORRECT:** Deny rules are CUMULATIVE and ABSOLUTE. A deny at ANY level blocks, regardless of allow at any other level. Priority only applies to allow/general settings, NOT deny rules.

**TRAP 3.3:** "Use `CLAUDE_HEADLESS=true` for CI/CD"
- **CORRECT:** This environment variable doesn't exist. Use `-p` (or `--print`) flag for non-interactive/headless mode.

**TRAP 3.4:** "Team commands go in `~/.claude/commands/`"
- **CORRECT:** `~/.claude/commands/` = personal, not version-controlled. Team commands go in `.claude/commands/` in the project repo (git-tracked, shared via clone/pull).

**TRAP 3.5:** "Subdirectory CLAUDE.md replaces/overrides root CLAUDE.md"
- **CORRECT:** CLAUDE.md files are ADDITIVE. All applicable files load simultaneously. Subdirectory files activate IN ADDITION TO root, not instead of.

**TRAP 3.6:** "PostToolUse hooks can prevent dangerous operations"
- **CORRECT:** PostToolUse runs AFTER the tool already executed. It CANNOT prevent/block. Only PreToolUse can block (exit code 2). PostToolUse is for logging, formatting, notification.

**TRAP 3.7:** "Hooks receive data via environment variables like $TOOL_NAME"
- **CORRECT:** Hooks receive data via STDIN as JSON. Must parse JSON from stdin containing `tool_name`, `tool_input`, `session_id`, `transcript_path`.

**TRAP 3.8:** "`--batch` flag for CI/CD pipeline execution"
- **CORRECT:** `--batch` doesn't exist. Use `-p` / `--print`. Don't confuse CLI flags with the Message Batches API (completely different concept).

---

### Domain 4: Prompt Engineering & Structured Output (20% weight)

**TRAP 4.1:** "Ask Claude to output JSON in the system prompt for reliable structured output"
- **CORRECT:** Use `tool_choice` with a forced tool whose schema defines the JSON structure. This gives 100% structural compliance vs ~85% with prompt-only.

**TRAP 4.2:** "Few-shot examples are the most effective first fix for any output quality issue"
- **CORRECT:** Few-shot examples are SECONDARY reinforcement. First fix tool descriptions (for tool selection issues) or system prompt structure (for formatting issues). Examples add context but don't replace clear instructions.

**TRAP 4.3:** "Larger context windows solve attention quality issues"
- **CORRECT:** Larger windows don't fix attention dilution — they may worsen it. The problem is attentional focus, not capacity. A 1M-token window with 14 files still has "lost in the middle" effects.

**TRAP 4.4:** "Batch API results come back in the same order they were submitted"
- **CORRECT:** Results are NON-DETERMINISTIC order. Use `custom_id` field to match results to original requests.

**TRAP 4.5:** "Combine caching with batch for maximum savings (always)"
- **CORRECT:** Caching + batch CAN combine for ~95% savings, but caching requires repeated identical prefixes. For heterogeneous batch items with different system prompts, caching provides no benefit.

**TRAP 4.6:** "100% test coverage ensures production reliability"
- **CORRECT:** 100% coverage of all possible inputs is IMPOSSIBLE for natural language systems. The input space is infinite. Use representative sampling + edge case testing + production monitoring.

---

### Domain 5: Context Management & Reliability (15% weight)

**TRAP 5.1:** "Progressive summarization preserves all important information"
- **CORRECT:** Summarization is LOSSY by definition. Specific facts (order numbers, dates, names) can be lost. Critical facts must be pinned (not summarized) using immutable blocks or system-level context.

**TRAP 5.2:** "Put the most important information in the middle of the context for safety"
- **CORRECT:** WORST possible placement. Lost-in-the-middle effect means middle content gets least attention. Critical info goes at the BEGINNING (system prompt) or END (recent messages).

**TRAP 5.3:** "More context = better answers"
- **CORRECT:** False past a threshold. Irrelevant context dilutes attention. Quality often IMPROVES when you remove irrelevant content, even if it reduces total context. Quality = relevance × depth, not volume.

**TRAP 5.4:** "Cache everything to save money"
- **CORRECT:** Only content that repeats across requests benefits from caching. Dynamic, per-request content (user messages, unique queries) cannot be cached effectively. Also: minimum cacheable thresholds mean small content is never cached.

**TRAP 5.5:** "System prompt is just another message the user can override"
- **CORRECT:** System prompt has OPERATOR authority. User messages CANNOT override system prompt instructions. This is a security property, not a suggestion. Authority hierarchy: Anthropic > System prompt (operator) > User messages.

---

## PART 6: SPECIFIC TECHNICAL FACTS THAT APPEAR AS QUESTIONS

These appear as direct knowledge questions or as the basis for eliminating wrong options.

---

### Agentic Loop Implementation

```python
# CORRECT loop pattern (what the exam tests)
while True:
    response = client.messages.create(...)
    
    if response.stop_reason == "end_turn":
        return response.content  # DONE — exit loop
    
    elif response.stop_reason == "tool_use":
        # Extract tool calls, execute, feed results back
        tool_results = execute_tools(response.content)
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})
        # Continue loop
    
    elif response.stop_reason == "max_tokens":
        # Output truncated — may need continuation
        handle_truncation(response)
```

**What's testable:**
- `stop_reason` is the ONLY reliable signal (not text parsing)
- `tool_result` goes in the `user` role (not assistant or system)
- Multiple parallel tool results go in ONE user message (not separate messages)
- Each `tool_result` must include matching `tool_use_id`

---

### Forced Structured Output

```python
# CORRECT — 100% structural compliance
response = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    tools=[{
        "name": "output_data",
        "description": "Output structured data",
        "input_schema": {
            "type": "object",
            "properties": {
                "name": {"type": "string"},
                "age": {"type": ["integer", "null"]},  # NULLABLE — prevents fabrication
                "status": {"type": "string", "enum": ["active", "inactive", "unknown"]}  # ENUM ESCAPE
            },
            "required": ["name", "age", "status"],
            "additionalProperties": false
        }
    }],
    tool_choice={"type": "tool", "name": "output_data"},  # FORCE this specific tool
    messages=[...]
)
```

**What's testable:**
- `tool_choice` forces 100% compliance vs ~85% prompt-only
- Nullable types (`["integer", "null"]`) prevent fabrication
- Enum escapes (`"unknown"`, `"N/A"`) give Claude an honest exit
- `additionalProperties: false` prevents extra fields

---

### PreToolUse Hook (Blocking Gate)

```bash
#!/bin/bash
# PreToolUse hook — blocks refunds over $500

# Read JSON from STDIN (NOT environment variables!)
INPUT=$(cat)
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name')
AMOUNT=$(echo "$INPUT" | jq -r '.tool_input.amount // 0')

if [ "$TOOL_NAME" = "process_refund" ] && [ "$AMOUNT" -gt 500 ]; then
    echo "Refund of \$$AMOUNT exceeds $500 limit. Escalate to manager."
    exit 2  # EXIT 2 = BLOCK the tool call
fi

exit 0  # EXIT 0 = ALLOW
```

**What's testable:**
- Data comes via STDIN as JSON (NOT env vars)
- Exit 2 = block (not exit 1, which is generic error)
- stdout = feedback message sent back to Claude
- Hook is SYNCHRONOUS — agent waits for it to complete

---

### Prompt Caching (Prefix Matching)

```
REQUEST STRUCTURE (cache-friendly):
┌─────────────────────────────────┐
│ System prompt (STATIC)          │ ← cache_control: ephemeral
│ - Identity, rules, constraints  │
├─────────────────────────────────┤
│ Tool definitions (STATIC)       │ ← cache_control: ephemeral
│ - All tool schemas              │
├─────────────────────────────────┤
│ Few-shot examples (STATIC)      │ ← cache_control: ephemeral
│ - Reference examples            │
├─────────────────────────────────┤
│ Dynamic content (CHANGES)       │ ← NO cache (invalidates everything after)
│ - User message, conversation    │
└─────────────────────────────────┘
```

**What's testable:**
- Cache is PREFIX-based — any byte change invalidates everything AFTER the change point
- Static content MUST come before dynamic content
- Max 4 breakpoints per request
- Below minimum threshold (1024-4096 depending on model) = NEVER cached
- Write cost: 25% premium (5-min TTL) or 100% premium (1-hour TTL)
- Read discount: 90% off input cost

---

### CLAUDE.md Hierarchy & Activation

```
~/.claude/CLAUDE.md                    (ALWAYS loaded — personal preferences)
    ↓ ADDITIVE
project/CLAUDE.md                      (ALWAYS loaded — project standards)
    ↓ ADDITIVE (when working in src/)
project/src/CLAUDE.md                  (CONDITIONAL — activates per file path)
    ↓ ADDITIVE (when working in src/api/)
project/src/api/CLAUDE.md              (CONDITIONAL — deepest match)
```

**What's testable:**
- Files are ADDITIVE, not override
- Subdirectory files activate when Claude operates on files in that subtree
- `@path/to/file.md` for modular includes (just @ followed by path)
- Team standards go in PROJECT-level (`.claude/settings.json`, committed)
- Personal preferences go in USER-level (`~/.claude/settings.json`)
- CLAUDE.md survives context compaction (treated as system prompt level)

---

## PART 7: QUESTION STEM KEYWORDS → ANSWER PATTERNS

The exam uses specific phrasing that maps to expected answer types:

| Stem Phrase | Points Toward |
|---|---|
| "Most effective FIRST STEP" | Simplest proportional fix (not over-engineered) |
| "Most RELIABLE solution" | Programmatic enforcement / deterministic mechanism |
| "ROOT CAUSE" | Layer analysis — don't blame symptoms |
| "Despite clear instructions, X% failure" | Prompt approach already failed → escalate to code |
| "MOST LIKELY root cause" | Identify the layer that actually failed |
| "What should you evaluate FIRST" | Start simple → tool descriptions, prompt structure |
| "Which approach BEST enables recovery" | Structured context for intelligent decisions |
| "WITHOUT modifying existing infrastructure" | Incremental improvement, not redesign |
| "MINIMIZE false positives" | Precision over recall — fewer but more accurate flags |
| "What is the PRIMARY concern" | Security > correctness > performance > cost |
| "In a PRODUCTION environment" | Reliability, monitoring, error handling matter most |
| "NOT a recommended approach" | Identify the anti-pattern among correct practices |

---

## PART 8: THE 15 MOST TESTABLE FACTS (Final Review)

If you remember nothing else, remember these:

1. **`stop_reason`** is the ONLY reliable loop termination signal
2. **Subagents inherit NOTHING** — all context must be explicitly passed
3. **Tool accuracy degrades past 5 tools** per agent (~8% per additional tool)
4. **Forced `tool_choice`** = 100% structural compliance (vs 85% prompt-only)
5. **Nullable + enum escapes** reduce fabrication from 7% to <1%
6. **PreToolUse exit 2** = block; hooks receive STDIN JSON (not env vars)
7. **Batch API** = 50% discount, 24h SLA, NO latency guarantee
8. **Cache is PREFIX-matched** — static first, dynamic last, max 4 breakpoints
9. **Deny rules are CUMULATIVE** — deny anywhere = blocked everywhere
10. **Plan mode** = `Shift+Tab` (not `/plan`); headless = `-p` flag (not `--batch`)
11. **MCP Tools** = model-controlled; **Resources** = app-controlled; **Prompts** = user-controlled
12. **Streamable HTTP** = current MCP remote transport (HTTP+SSE = deprecated)
13. **Lost in the middle** — critical info at START or END, never middle
14. **System prompt** = operator authority; user messages CANNOT override it
15. **Specific error feedback** converges in 2-3 retries; generic feedback does NOT converge

---

## PART 9: EXAM-DAY ELIMINATION STRATEGY

### Step 1: Eliminate Impossible Options (30 seconds)

Look for these instant disqualifiers:
- Non-existent API fields (`response.is_finished`, `response.done`, `response.complete`)
- Non-existent CLI flags (`--batch`, `--headless`, `CLAUDE_HEADLESS=true`)
- Non-existent configuration keys (plausible-sounding settings that don't exist)
- Architecturally impossible claims ("subagents inherit context", "PostToolUse can block")

### Step 2: Identify the Binding Constraint (30 seconds)

What constraint makes this scenario specific?
- Financial consequence → programmatic enforcement
- Latency requirement → real-time API
- Tool selection failure → descriptions or scoping
- Incomplete output → layer analysis (decomposition? execution? synthesis?)

### Step 3: Apply the Proportionality Test (30 seconds)

Is the proposed solution proportional to the problem?
- Simple problem + complex solution = wrong (over-engineering trap)
- Critical problem + simple solution = wrong (under-engineering trap)
- Problem severity should match solution complexity

### Step 4: Choose Between Final Two (30 seconds)

When you've eliminated 2 options and have 2 plausible remaining:
- Apply the FACT > GUIDELINE > PROCEDURE hierarchy
- The option grounded in a specific technical truth wins
- The option that addresses the binding constraint wins
- The simpler option wins UNLESS the scenario explicitly states simple approaches have failed

---

## PART 10: TOPICS THE EXAM DOES NOT TEST

Confirmed by multiple sources — don't waste study time:

- API syntax memorization (exact parameter names/values)
- Model version history or changelog details
- RAG implementation specifics
- Pricing calculations or exact dollar amounts
- Constitutional AI research or RLHF details
- Batch API retry mechanics (internal implementation)
- Specific programming language syntax
- Infrastructure/cloud provider details (unless MCP-related)
- Token count arithmetic
- Anthropic company history or organizational structure

---

*Last updated: Based on research from 12+ credible sources including pass reports through July 2026*
