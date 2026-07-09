# Domain 5 | Topic 5.2: Context Window Degradation Handling

## 1. Core Concepts

### What is Context Degradation?

As a conversation grows, Claude's ability to attend to all information degrades. Even within the context window, **information at the edges** (very early or very middle) receives less attention than recent information.

```
ATTENTION DISTRIBUTION (simplified)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Early messages: ████████░░  (moderate attention - primacy)
Middle messages: ████░░░░░░  (least attention - "lost in the middle")
Recent messages: ██████████  (highest attention - recency)
System prompt:  ██████████  (always high - structural position)
```

### "Lost in the Middle" Problem

Research shows LLMs attend less to information in the middle of long contexts:
- **Beginning** — good attention (primacy effect)
- **Middle** — weakest attention (information gets "buried")
- **End** — best attention (recency effect)
- **System prompt** — high attention (special position)

### Degradation Signals

| Signal | Indicates |
|--------|-----------|
| Claude contradicts earlier information | Middle context lost |
| Claude re-asks answered questions | Relevant message buried |
| Decreasing response quality | Context overload |
| Hallucinated details | Can't find actual info, fills gaps |
| Ignoring constraints stated earlier | Constraint buried in middle |

---

## 2. Design Patterns

### Pattern 1: Information Positioning Strategy

```
PUT CRITICAL INFO IN HIGH-ATTENTION POSITIONS:

System prompt (always attended):
├── Constraints, rules, format requirements
├── Role definition
└── Critical context

Recent messages (high attention):
├── Current task details
├── Most relevant context
└── Recent decisions/findings

AVOID burying important info in the middle of long tool outputs
or between many routine messages.
```

### Pattern 2: Context Window Budgeting

```python
class ContextBudget:
    """Allocate context window space intentionally."""
    
    def __init__(self, max_tokens=200000):
        self.max_tokens = max_tokens
        self.budget = {
            "system_prompt": 5000,     # 2.5%
            "tools": 10000,            # 5%
            "conversation_state": 5000, # 2.5%
            "working_context": 150000,  # 75%
            "safety_margin": 30000     # 15% buffer
        }
    
    def available_for_tools(self):
        """How much space for tool results."""
        used = self.current_usage()
        return self.budget["working_context"] - used.working
    
    def should_truncate_result(self, result_tokens):
        """Should we summarize this tool result?"""
        return result_tokens > 5000  # Truncate results over 5K tokens
```

### Pattern 3: Strategic Repetition

```python
# Repeat critical constraints at regular intervals
def inject_reminders(messages, constraints, every_n_turns=10):
    """Re-inject important constraints to counteract degradation."""
    
    if len(messages) % every_n_turns == 0:
        messages.append({
            "role": "user",
            "content": f"[Reminder: {constraints}]"
        })
    
    return messages
```

### Pattern 4: Structured Context Anchors

```python
# Place key information where it gets most attention
system = """[System prompt - always attended]
Critical rules and constraints here.
"""

# When injecting retrieved context:
messages.append({
    "role": "user",
    "content": f"""
IMPORTANT CONTEXT (please attend carefully):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
{critical_context}

Current task:
{task_description}
"""
})
```

---

## 3. Implementation Details

### Detecting Degradation Programmatically

```python
class DegradationDetector:
    """Detect signs of context degradation."""
    
    def __init__(self):
        self.previous_responses = []
        self.stated_constraints = set()
    
    def check_response(self, response, messages):
        # Signal 1: Contradiction with earlier statements
        for prev in self.previous_responses[-5:]:
            if contradicts(response, prev):
                return "contradiction_detected"
        
        # Signal 2: Violating stated constraints
        for constraint in self.stated_constraints:
            if violates(response, constraint):
                return "constraint_violation"
        
        # Signal 3: Asking previously-answered questions
        if asks_answered_question(response, messages):
            return "memory_loss"
        
        return "healthy"
```

### Managing Large Tool Results

```python
def handle_large_result(tool_result, max_result_tokens=3000):
    """Prevent large tool results from pushing important context out."""
    
    tokens = estimate_tokens(tool_result)
    
    if tokens <= max_result_tokens:
        return tool_result
    
    # Strategy 1: Truncate with summary
    return f"""[Result truncated from {tokens} tokens to {max_result_tokens}]
    
Key information from result:
{summarize(tool_result, max_tokens=max_result_tokens)}

[Full result available via tool re-read if needed]"""
```

### Context Window Sizing by Model

| Model | Context Window | Max Output | Recommended Working Limit* |
|-------|---------------|------------|---------------------------|
| Claude Fable 5, Opus 4.8/4.7/4.6, Sonnet 5/4.6 | 1M tokens | 128K | Use compaction well before hitting the limit |
| Claude Haiku 4.5 | 200K tokens | 64K | ~150K for complex agentic tasks |

*These are approximate guidelines based on observed behavior in complex agentic workflows, not official Anthropic specifications. Actual degradation points vary by task type, information density, and how context is structured. Simple retrieval tasks may perform well at higher token counts, while multi-step reasoning tasks with many constraints may benefit from staying within these recommended working limits.

---

## 4. Security & Reliability

### Security Implications of Degradation

| Risk | Scenario | Mitigation |
|------|----------|-----------|
| Forgotten security constraint | "Never expose PII" buried in middle | Put in system prompt, not messages |
| Permission creep | Agent forgets it was denied something | Permissions are structural, not message-based |
| Confused identity | Long conversation makes agent drift | Strong system prompt identity |
| Prompt injection via volume | Attacker floods with text to bury constraints | System prompt constraints can't be buried |

### Reliability Under Degradation

1. **System prompt for non-negotiables** — always attended regardless of context length
2. **Structured state** — explicit state objects don't degrade like natural language
3. **Periodic verification** — agent re-checks critical constraints
4. **Bounded conversations** — break long tasks into sessions
5. **Monitoring** — detect and respond to degradation signals

---

## 5. Exam Scenarios

### Scenario 1
**Q:** An agent has a 100-message conversation. A security constraint stated in message #5 is being violated in message #95. Where should the constraint live?

**A:** In the **system prompt**, not in a user message. System prompt information:
- Is always in a high-attention position
- Never gets compacted or summarized away
- Is structurally prioritized by the model

If it must also be in messages, repeat it periodically or include it in the most recent context when relevant actions are being taken.

### Scenario 2
**Q:** A tool returns a 50,000 token result. After this, Claude's response quality drops. What is happening and how do you fix it?

**A:** The large tool result is **displacing other context** and creating a "lost in the middle" effect for everything before it.

Fix:
1. **Truncate/summarize** tool results over a threshold (e.g., 3000 tokens)
2. Extract only relevant portions from large results
3. If the full result is needed, break the task into smaller queries that return less data
4. Consider: does Claude need the full 50K tokens, or just a summary?

### Scenario 3
**Q:** An agentic system maintains quality for the first 10 iterations but degrades around iteration 15-20. What architectural changes help?

**A:**
1. **Proactive compaction** at iteration 10 (before degradation, not after)
2. **Summarize tool results** immediately rather than keeping raw output
3. **Structured state injection** — prepend current state summary to each turn
4. **Budget context** — allocate fixed space for each type of information
5. **Break into sub-tasks** — use fresh agent (sub-agent) for independent work to avoid context buildup

---

## 5b. Large Codebase Exploration — Task 5.4 (Exam-Critical)

### Scratchpad Files

When sessions are long, **write key findings to files** so they survive context compaction:

```
PATTERN:
1. Agent discovers important info (architecture, key classes, flows)
2. Writes to scratchpad file: scratch/findings.md or NOTES.md
3. Later iterations reference the scratchpad when context is degraded
4. Findings persist even after /compact or context summarization

WHY: Context compaction may summarize away specifics (class names, line numbers).
Scratchpad files are permanent — agent can Read them back anytime.
```

### The Explore Subagent

The **Explore subagent** isolates verbose discovery output from the main agent:

```
WITHOUT EXPLORE:                    WITH EXPLORE:
┌──────────────────────┐           ┌──────────────────────┐
│ Main Agent           │           │ Main Agent           │
│ [reads 40 files]     │           │ "Find auth flows"    │──► Explore subagent
│ [greps 20 patterns]  │           │                      │    [reads 40 files]
│ [10K tokens of noise]│           │ Receives: 500-token  │◄── [greps 20 patterns]
│ ← Context FULL       │           │ structured summary   │    [returns summary]
└──────────────────────┘           └──────────────────────┘
```

**Key points:**
- Explore subagent handles verbose discovery (file reads, greps)
- Returns only a structured summary to the main agent
- Main agent's context stays clean for coordination/implementation
- Referenced in exam Task 3.4 (plan mode) and Task 5.4

### /compact Command

Manually triggers context compaction to free space:

```
WHEN TO USE /compact:
• Before starting a new phase (investigation → implementation)
• When degradation signals appear (vague answers, "typical patterns")
• Proactively at ~75% context capacity
• After verbose exploration that's now fully processed

WHAT IT DOES:
• Summarizes prior conversation into condensed form
• Removes raw tool outputs that have been processed
• Preserves key decisions and findings
• Frees context for new work
```

### Structured State Persistence for Crash Recovery

For multi-agent systems that may crash or need to resume:

```
CRASH RECOVERY PATTERN:
1. Each agent exports state to a known file location periodically
2. Coordinator maintains a MANIFEST tracking all agent states
3. On crash/resume: load manifest → inject state → continue

MANIFEST EXAMPLE:
{
  "agents": {
    "search-agent": {"status": "completed", "findings": "state/search.json"},
    "analysis-agent": {"status": "partial", "completed": "papers 1-8", "remaining": "9-15"}
  }
}

ON RESUME:
• Coordinator reads manifest
• Completed results injected into downstream agents
• Partial agents restart from checkpoint (not from scratch)
```

### Combined Strategy for Large Codebases

```
1. Spawn Explore subagent → isolates verbose discovery
2. Explore returns structured summary → main context stays clean
3. Write key findings to scratchpad file → survives compaction
4. Use /compact if context grows large → free space
5. Spawn focused subagents for specific questions
6. Reference scratchpad for consistency
7. On crash → load manifest → resume from checkpoints
```

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Keeping all history verbatim | Hits degradation zone | Summarize/compact proactively |
| Important rules only in messages | Lost in middle | System prompt for critical rules |
| Huge tool results untruncated | Displaces other context | Summarize large results |
| Ignoring degradation signals | Quality silently drops | Monitor and respond to signals |
| Very long single conversations | Inevitable degradation | Break into sessions, use persistent state |
| No context budget | Uncontrolled growth | Allocate token budget per category |
| Exploring in main agent context | Context fills with verbose output | Use Explore subagent |
| No scratchpad for long sessions | Findings lost to compaction | Write findings to files |
| No crash recovery plan | Must restart from scratch | Use structured state manifests |
| Using /compact too late | Quality already degraded | Compact proactively at 75% |

---

## 7. Quick Reference / Cheat Sheet

```
ATTENTION POSITIONS (best → worst)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. System prompt (always high)
2. Last few messages (recency)
3. First few messages (primacy)
4. Middle of conversation (lowest)

DEGRADATION SIGNALS
━━━━━━━━━━━━━━━━━━━
• Contradicts earlier info
• Re-asks answered questions
• Violates stated constraints
• Hallucinate details ("typical patterns" instead of specific classes)
• Decreasing response quality
• References generic patterns instead of discovered specifics

MITIGATION STRATEGIES
━━━━━━━━━━━━━━━━━━━━━
1. Critical info → system prompt
2. Proactive compaction (/compact before degradation)
3. Truncate large tool results
4. Strategic repetition of constraints
5. Budget context allocation
6. Break long tasks into sessions
7. Scratchpad files for key findings
8. Explore subagent for verbose discovery

LARGE CODEBASE EXPLORATION (Task 5.4)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• Scratchpad files → write findings to file (survive compaction)
• Explore subagent → isolates verbose reads/greps, returns summary
• /compact → manually trigger compaction when context fills
• Crash recovery → export state to manifest, reload on resume

CONTEXT BUDGETING
━━━━━━━━━━━━━━━━━
System prompt: ~5K tokens (2.5%)
Tools: ~10K tokens (5%)
State/context: ~5K tokens (2.5%)
Working space: ~150K tokens (75%)
Safety margin: ~30K tokens (15%)

PRACTICAL LIMITS
━━━━━━━━━━━━━━━━
• Quality stable up to ~100-150K tokens
• Proactive compaction at 75% capacity
• Tool results: truncate over 3-5K tokens
• Re-inject constraints every 10 turns
• Scratchpad when session > 20 iterations
```
