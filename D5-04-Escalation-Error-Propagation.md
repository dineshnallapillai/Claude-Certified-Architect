# Domain 5 | Topic 5.4: Escalation Patterns & Error Propagation

## 1. Core Concepts

### What is Escalation?

**Escalation** is the process of an agent recognizing it cannot resolve a situation autonomously and transferring control to a more capable entity (human, more powerful model, or specialized agent).

```
ESCALATION HIERARCHY
━━━━━━━━━━━━━━━━━━━━
Sub-agent (can't solve)
    ↓ escalate to
Orchestrator agent (can't solve)
    ↓ escalate to
Human operator (can't solve)
    ↓ escalate to
On-call engineer / domain expert
```

### When to Escalate

| Trigger | Example |
|---------|---------|
| **Confidence too low** | "I'm not sure if this migration is safe" |
| **Max retries exceeded** | Failed 3 times to fix the test |
| **Security decision required** | "Should I grant this permission?" |
| **Ambiguous requirements** | "I could do A or B, both are valid" |
| **Destructive action** | "This will delete production data" |
| **Out of scope** | "This requires AWS console access I don't have" |
| **Conflicting information** | Two sources disagree, can't resolve |

### Error Propagation

**Error propagation** describes how errors flow through a system of agents. Key concern: should errors be **handled locally** or **propagated upward**?

```
ERROR FLOW OPTIONS
━━━━━━━━━━━━━━━━━━
Option A: Local handling (agent fixes itself)
  Agent → Error → Agent retries → Success

Option B: Propagation (bubble up)
  Sub-agent → Error → Orchestrator → Error → Human

Option C: Hybrid (try locally, escalate if stuck)
  Agent → Error → Retry (max 3) → Still failing → Escalate
```

---

## 2. Design Patterns

### Pattern 1: Graduated Escalation

```python
class EscalationManager:
    def __init__(self):
        self.levels = [
            {"name": "self_correct", "max_attempts": 3},
            {"name": "alternate_approach", "max_attempts": 2},
            {"name": "parent_agent", "max_attempts": 1},
            {"name": "human_operator", "max_attempts": 1}
        ]
    
    def handle_failure(self, error, context, attempts):
        if attempts < 3:
            return {"action": "retry", "strategy": "self_correct",
                    "guidance": f"Previous error: {error}. Try different approach."}
        elif attempts < 5:
            return {"action": "alternate", "strategy": "alternate_approach",
                    "guidance": "Previous approaches failed. Try fundamentally different method."}
        elif attempts < 6:
            return {"action": "escalate_to_orchestrator",
                    "context": {"error": error, "attempts": attempts, "tried": context}}
        else:
            return {"action": "escalate_to_human",
                    "message": self.format_human_escalation(error, context)}
    
    def format_human_escalation(self, error, context):
        return f"""
🚨 Agent requires human assistance:
- Task: {context['task']}
- Error: {error}
- Attempts: {context['attempts']}
- Approaches tried: {context['tried']}
- What's needed: {context['needs']}
"""
```

### Pattern 2: Error Context Preservation

```python
class ErrorChain:
    """Preserve full error context as it propagates up."""
    
    def __init__(self):
        self.chain = []
    
    def add(self, level, error, action_taken, result):
        self.chain.append({
            "level": level,
            "error": str(error),
            "action": action_taken,
            "result": result,
            "timestamp": time.time()
        })
    
    def to_escalation_report(self):
        """Format for human review."""
        report = "Error escalation chain:\n"
        for i, entry in enumerate(self.chain):
            report += f"\n{i+1}. [{entry['level']}] Error: {entry['error']}\n"
            report += f"   Action taken: {entry['action']}\n"
            report += f"   Result: {entry['result']}\n"
        return report
```

### Pattern 3: Circuit Breaker with Escalation

```python
class AgentCircuitBreaker:
    """Stop retrying and escalate when pattern indicates systemic issue."""
    
    def __init__(self, threshold=5, window=60):
        self.errors = []
        self.threshold = threshold
        self.window = window  # seconds
    
    def record_error(self, error):
        self.errors.append({"error": error, "time": time.time()})
        self.cleanup_old()
        
        if len(self.errors) >= self.threshold:
            return EscalationRequired(
                reason="Circuit breaker tripped",
                error_count=len(self.errors),
                pattern=self.detect_pattern(),
                recommendation="Systemic issue detected — manual investigation needed"
            )
        return None
    
    def detect_pattern(self):
        """Identify if errors share a pattern."""
        types = [e["error"].__class__.__name__ for e in self.errors]
        most_common = max(set(types), key=types.count)
        return f"Repeated {most_common} errors ({types.count(most_common)}/{len(types)})"
```

### Pattern 4: Safe Propagation with Partial Results

```python
def orchestrate_with_safe_propagation(tasks):
    """Don't let one failure block all results."""
    results = {}
    errors = {}
    
    for task in tasks:
        try:
            results[task.id] = execute_task(task)
        except RecoverableError as e:
            # Try alternate approach
            try:
                results[task.id] = execute_alternate(task)
            except Exception as e2:
                errors[task.id] = ErrorChain([e, e2])
        except CriticalError as e:
            # Must escalate immediately
            errors[task.id] = e
            escalate_immediately(task, e)
    
    # Return partial results + escalation for failures
    return {
        "completed": results,
        "failed": errors,
        "success_rate": len(results) / len(tasks)
    }
```

---

## 3. Implementation Details

### Error Categories and Propagation Rules

| Error Type | Local Handle? | Propagate? | Escalate? |
|-----------|--------------|-----------|-----------|
| Transient (timeout, rate limit) | Yes (retry) | No | After N retries |
| Validation (bad input) | Yes (fix input) | No | If can't fix |
| Permission (access denied) | No | Yes | To entity with access |
| Logic (bug in agent code) | No | Yes | Always (needs fix) |
| External (service down) | Retry once | Yes | If persistent |
| Safety (security concern) | Never handle locally | Yes | Always to human |

### Escalation Message Format

```python
def format_escalation(level, error, context):
    """Structured escalation for clear handoff."""
    return {
        "severity": classify_severity(error),
        "summary": one_line_summary(error),
        "detail": {
            "what_happened": str(error),
            "what_was_tried": context.attempts_summary,
            "what_is_needed": context.needed_resolution,
            "current_state": context.system_state
        },
        "suggested_actions": generate_suggestions(error),
        "deadline": context.urgency_deadline,
        "artifacts": context.relevant_files
    }
```

### Error Propagation in Multi-Agent Systems

```
SUB-AGENT FAILURE HANDLING:
━━━━━━━━━━━━━━━━━━━━━━━━━━

Sub-agent fails → returns error to orchestrator

Orchestrator decides:
├── Retry same agent (transient error)
├── Try different agent (capability mismatch)
├── Retry with more context (information gap)
├── Accept partial result (non-critical failure)
└── Escalate to human (unresolvable)

Key: orchestrator NEVER silently drops errors
```

---

## 4. Security & Reliability

### Security-Critical Escalation

```python
ALWAYS_ESCALATE = [
    "credential_exposure",      # Potential secret leak
    "permission_escalation",    # Unexpected access gained
    "data_integrity_violation", # Data may be corrupted
    "authentication_bypass",    # Security control failed
    "unexpected_data_access",   # Accessed data outside scope
]

def check_security_escalation(event):
    """Some events MUST go to human regardless of agent capability."""
    if event.type in ALWAYS_ESCALATE:
        escalate_to_human(
            priority="critical",
            message=f"Security event: {event.type}",
            block_further_action=True  # Stop agent immediately
        )
```

### Preventing Error Swallowing

| Risk | Consequence | Prevention |
|------|------------|------------|
| Agent catches and ignores error | Proceeds with wrong state | All errors must be logged + handled |
| Orchestrator drops sub-agent error | Result missing, not noticed | Track expected vs received results |
| Error in error handler | Infinite loop or silent failure | Error handler must be simple + tested |
| Timeout treated as success | Missing data accepted | Explicit timeout handling |

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A code generation agent fails 3 times to produce code that passes tests. It's been trying the same general approach each time. What should happen next?

**A:** **Graduated escalation:**
1. After 3 same-approach failures: **change strategy** (not just retry)
   - "I've tried fixing the implementation 3 times. Let me reconsider the approach entirely."
2. Try a fundamentally different approach (different algorithm, library, or architecture)
3. If the alternate approach also fails (attempt 5): **escalate to user**
   - Report: what was tried, what failed, what the agent thinks is blocking

Key: 3 failures of the same approach ≠ retry again. It means the approach itself is wrong.

### Scenario 2
**Q:** In a multi-agent system, a sub-agent encounters a permissions error. Should it retry, handle locally, or escalate?

**A:** **Propagate to orchestrator (don't handle locally)** because:
- Sub-agent doesn't have the authority to grant itself permissions
- Retrying won't help (permission error is not transient)
- Orchestrator may have the needed permissions or can request them
- If orchestrator can't resolve, escalate to human

Permission errors always propagate — only an entity WITH the permission can resolve them.

### Scenario 3
**Q:** An orchestrator's sub-agent returns an error. The orchestrator has 4 other successful results. Should it wait/retry the failed agent or return partial results?

**A:** Depends on whether the failed result is critical:
- **If independent**: Return partial results immediately + note the gap + retry failed agent in background
- **If dependency**: Other results depend on it → retry the failed agent (with timeout) → escalate if still failing
- **In all cases**: Never silently drop the error. Always report what's missing and why.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Infinite retry without strategy change | Wastes tokens, never succeeds | Max 3 same-approach, then pivot |
| Silent error swallowing | System proceeds with bad state | All errors logged and handled |
| Escalating everything | Human overwhelmed | Graduated: self-fix → alternate → escalate |
| No context in escalation | Human can't help efficiently | Include: what happened, what tried, what's needed |
| Agent handles security decisions | Inappropriate authority | Always escalate security to human |
| Partial results without noting gaps | Incomplete work accepted as complete | Always flag what's missing |

---

## 7. Quick Reference / Cheat Sheet

```
ESCALATION TRIGGERS
━━━━━━━━━━━━━━━━━━━
• 3+ failures same approach
• Security-sensitive decision
• Ambiguous requirements
• Permission denied
• Destructive action needed
• Conflicting information unresolvable
• Out of scope/capability

GRADUATED ESCALATION
━━━━━━━━━━━━━━━━━━━━
Level 1: Self-correct (retry with learning)
Level 2: Change approach (fundamentally different)
Level 3: Escalate to parent/orchestrator
Level 4: Escalate to human

ERROR PROPAGATION RULES
━━━━━━━━━━━━━━━━━━━━━━━
Transient → Retry locally (max 3)
Validation → Fix input locally
Permission → Propagate up (can't self-fix)
Logic/Bug → Propagate up (needs fix)
Security → ALWAYS escalate to human

ESCALATION MESSAGE FORMAT
━━━━━━━━━━━━━━━━━━━━━━━━━
• What happened (the error)
• What was tried (approaches attempted)
• What's needed (what would resolve it)
• Current state (where things stand)
• Suggested next steps

NEVER
━━━━━
• Never silently drop errors
• Never retry indefinitely
• Never handle security decisions autonomously
• Never escalate without context
• Never block on non-critical failures
```
