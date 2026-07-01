# Domain 5 | Topic 5.1: Progressive Summarization & Compaction

## 1. Core Concepts

### What is Progressive Summarization?

**Progressive summarization** is the technique of condensing earlier conversation context into summaries as the context window fills up, preserving the most important information while discarding details.

```
FULL CONTEXT                    AFTER COMPACTION
━━━━━━━━━━━━━━                 ━━━━━━━━━━━━━━━━━
[System prompt]                 [System prompt]
[Message 1 - 500 tokens]       [Summary of msgs 1-10: 200 tokens]
[Message 2 - 300 tokens]       [Message 11 - 400 tokens]
[Message 3 - 800 tokens]       [Message 12 - 350 tokens]
[...10 messages...]             [Message 13 - current]
[Message 13 - current]
Total: ~8000 tokens             Total: ~3000 tokens
```

### Why Compaction is Necessary

| Problem | Context Window Limit |
|---------|---------------------|
| Fable 5, Opus 4.8/4.7/4.6, Sonnet 5/4.6 | 1M tokens |
| Haiku 4.5 | 200K tokens |
| Long conversations | Can exceed in many exchanges |
| Agentic loops | Tool results accumulate rapidly |
| Large codebases | Multiple file reads fill context |

### What Gets Preserved vs Lost

| Preserved (High Priority) | Lost (Low Priority) |
|---------------------------|---------------------|
| Key decisions and their reasons | Verbatim tool outputs |
| Current task and progress | Intermediate reasoning |
| Important constraints discovered | Exact file contents (re-readable) |
| Error patterns encountered | Exploratory dead ends |
| User preferences expressed | Routine acknowledgments |

### Compaction Strategies

```
STRATEGY 1: Summarize Old Messages
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Keep recent N messages intact, summarize everything before.

STRATEGY 2: Selective Retention
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Keep: decisions, errors, key findings
Drop: routine reads, verbose tool output

STRATEGY 3: Tiered Compression
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Tier 1 (recent): Full detail
Tier 2 (older): Summarized
Tier 3 (oldest): Key facts only
```

---

## 2. Design Patterns

### Pattern 1: Automatic Context Compaction (Claude Code)

Claude Code automatically detects approaching context limits and:
1. Identifies messages that can be summarized
2. Generates a summary preserving key information
3. Replaces original messages with the summary
4. Continues the session with compressed context

This is invisible to the user but maintains continuity.

### Pattern 2: Manual Summarization in Agentic Loops

```python
def manage_context(messages, max_tokens=150000):
    """Compact context when approaching limit."""
    current_tokens = count_tokens(messages)
    
    if current_tokens > max_tokens * 0.8:  # 80% threshold
        # Summarize all but the last 5 messages
        old_messages = messages[:-5]
        recent_messages = messages[-5:]
        
        summary = client.messages.create(
            model="claude-haiku-4-5",  # Cheap model for summarization
            max_tokens=1000,
            messages=[{
                "role": "user",
                "content": f"Summarize this conversation context, preserving: key decisions, current task state, important errors, and constraints. Be concise.\n\n{format_messages(old_messages)}"
            }]
        )
        
        # Replace old messages with summary
        messages = [
            {"role": "user", "content": f"[Context summary: {summary.content[0].text}]"},
            *recent_messages
        ]
    
    return messages
```

### Pattern 3: Structured State Tracking

```python
class AgentState:
    """Explicit state that persists across compaction."""
    
    def __init__(self):
        self.task_description = ""
        self.completed_steps = []
        self.pending_steps = []
        self.key_findings = []
        self.errors_encountered = []
        self.constraints = []
    
    def to_context(self):
        return f"""
CURRENT STATE:
Task: {self.task_description}
Completed: {self.completed_steps}
Pending: {self.pending_steps}
Key Findings: {self.key_findings}
Errors: {self.errors_encountered}
Constraints: {self.constraints}
"""
    
    def inject_into_messages(self, messages):
        """Add state as first message after compaction."""
        state_msg = {"role": "user", "content": self.to_context()}
        return [state_msg] + messages
```

### Pattern 4: Information That Survives Compaction

```
ALWAYS SURVIVES (built into architecture):
├── System prompt (CLAUDE.md content)
├── Tool definitions
├── Memory files (if loaded)
└── Active plan (if in plan mode)

SURVIVES THROUGH SUMMARIZATION:
├── Key decisions ("We chose approach X because Y")
├── Discovered constraints ("File X is read-only")
├── Progress tracking ("Steps 1-5 complete, step 6 pending")
└── Critical errors ("Auth module has circular dependency")

LOST (must be re-read if needed):
├── Full file contents (can be re-read)
├── Complete tool outputs (verbose)
├── Intermediate reasoning steps
└── Dead-end explorations
```

---

## 3. Implementation Details

### Token Counting for Compaction Decisions

```python
import anthropic

def should_compact(messages, system_prompt, max_context=200000):
    """Determine if compaction is needed."""
    # Rough estimation (actual tokenization is model-specific)
    total_tokens = estimate_tokens(system_prompt)
    for msg in messages:
        total_tokens += estimate_tokens(msg)
    
    # Trigger compaction at 75% capacity
    return total_tokens > max_context * 0.75

def estimate_tokens(text):
    """Rough token estimate: ~4 chars per token for English."""
    if isinstance(text, str):
        return len(text) // 4
    elif isinstance(text, dict):
        return estimate_tokens(json.dumps(text))
    return 0
```

### Compaction Quality Metrics

| Metric | Good Compaction | Bad Compaction |
|--------|----------------|----------------|
| Task continuity | Agent continues work without confusion | Agent re-asks questions or repeats work |
| Decision preservation | Remembers WHY choices were made | Revisits decided questions |
| Error memory | Doesn't repeat failed approaches | Tries same failing thing again |
| Information loss | Only loses re-derivable details | Loses critical constraints or decisions |

### What to Include in Summaries

```
GOOD SUMMARY COMPONENTS:
━━━━━━━━━━━━━━━━━━━━━━━
1. Task objective: What are we trying to accomplish
2. Progress: What's been done, what's remaining
3. Key findings: Important discoveries that affect the work
4. Decisions: Choices made and their rationale
5. Blockers: Current obstacles or constraints
6. State: Current files being worked on, their status

BAD SUMMARY COMPONENTS:
━━━━━━━━━━━━━━━━━━━━━━━
- Full file contents (re-readable)
- Verbose tool output (re-runnable)
- "I said X, then you said Y" narrative
- Unchanged context (system prompt already has it)
```

---

## 4. Security & Reliability

### Compaction Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Lost security constraints | Agent violates rule it forgot | Keep constraints in system prompt (not messages) |
| Summarization hallucination | False "facts" in summary | Validate summaries against source |
| Losing error context | Agent retries same failing approach | Track errors in explicit state |
| Over-compression | Agent loses crucial nuance | Conservative compaction threshold |
| Permission forgetting | Re-requests already-granted permissions | Permissions are session-state, not message-based |

### Reliability Strategies

1. **Critical info in persistent locations** — CLAUDE.md, memory files, NOT just messages
2. **Explicit progress tracking** — plans/tasks survive compaction
3. **Conservative compaction** — only compress when truly needed
4. **Verification after compaction** — "Let me confirm my understanding of the current state"
5. **Idempotent operations** — if context is lost, re-running is safe

---

## 5. Exam Scenarios

### Scenario 1
**Q:** An agent is working on a 15-file refactoring. After context compaction, it starts modifying a file it already completed. What architectural pattern prevents this?

**A:** **Explicit progress tracking** that persists across compaction:
1. Use plan mode with a checklist of files
2. Mark files as "done" in the plan after completing each
3. Plan state persists across compaction (it's structural, not message-based)
4. Before modifying any file, agent checks the plan for its status

Alternative: Track progress in an external file (like a TODO.md) that the agent reads after compaction.

### Scenario 2
**Q:** What information should NEVER rely on message history (and should instead be in system prompt or persistent files)?

**A:**
- **Security constraints** → system prompt (always present)
- **Project conventions** → CLAUDE.md (always loaded)
- **User preferences** → memory files (loaded per session)
- **Current task progress** → plan mode or task tracker
- **Critical decisions** → documented in code comments or commit messages

Anything whose loss could cause harm or significant rework should not depend on message history surviving compaction.

### Scenario 3
**Q:** An agentic loop processes large files (10K+ tokens each). After reading 20 files, context is 80% full. How should the system manage this?

**A:** **Multi-strategy approach:**
1. Don't keep full file contents in context — extract only relevant portions
2. Summarize each file's relevant findings immediately after reading
3. Use structured state: `{"file": "auth.py", "relevant_functions": [...], "issues": [...]}`
4. Trigger compaction before reading more files (don't wait until 100%)
5. Re-read specific files if details are needed later (cheaper than keeping in context)

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Keeping all tool output in context | Rapid context fill | Summarize or discard verbose output |
| Critical rules only in messages | Lost on compaction | Put in system prompt or CLAUDE.md |
| No explicit progress tracking | Agent forgets what's done | Plan mode, task lists, state objects |
| Compacting too aggressively | Loses important nuance | Conservative threshold (75-80%) |
| Never compacting | Hit hard limit, lose everything | Proactive compaction before limit |
| Summary without structure | Narrative blob, hard to use | Structured: task/progress/findings/decisions |

---

## 7. Quick Reference / Cheat Sheet

```
COMPACTION TIMING
━━━━━━━━━━━━━━━━
• Trigger at 75-80% context usage
• Summarize oldest messages first
• Keep recent 5-10 messages intact
• Preserve system prompt (never compacted)

WHAT SURVIVES
━━━━━━━━━━━━━
Always: System prompt, tools, CLAUDE.md, memory files
Summarized: Decisions, progress, findings, errors
Lost: Full file contents, verbose tool output, dead ends

INFORMATION HIERARCHY
━━━━━━━━━━━━━━━━━━━━━
Critical → System prompt (guaranteed)
Important → Explicit state/plan (structural)
Useful → Summary (compressed)
Derivable → Drop (can re-read/re-run)

PREVENTION STRATEGIES
━━━━━━━━━━━━━━━━━━━━━
• CLAUDE.md for persistent rules/context
• Plan mode for progress tracking
• Structured state objects for complex tasks
• Extract-then-discard for large file reads
• Summary after each major step

COMPACTION RED FLAGS
━━━━━━━━━━━━━━━━━━━
⚠ Agent re-asking answered questions
⚠ Agent repeating completed work
⚠ Agent violating previously-stated constraints
⚠ Agent trying previously-failed approaches
→ All indicate information lost that should have been preserved
```
