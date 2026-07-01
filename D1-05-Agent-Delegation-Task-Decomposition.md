# Domain 1 | Topic 1.5: Agent Delegation & Task Decomposition

## 1. Core Concepts

### Task Decomposition

**Task decomposition** is breaking a complex user request into smaller, manageable sub-tasks that can be:
- Executed independently or sequentially
- Delegated to specialized agents
- Validated individually
- Tracked for progress

### Decomposition Strategies

```
USER REQUEST: "Refactor the authentication module to use OAuth2"

STRATEGY 1: Functional Decomposition
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
├── Analyze current auth implementation
├── Design OAuth2 integration architecture
├── Implement token management
├── Update API endpoints
├── Write tests
└── Update documentation

STRATEGY 2: Layer-Based Decomposition
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
├── Database layer (new token tables)
├── Service layer (OAuth2 logic)
├── API layer (endpoint changes)
├── Client layer (token refresh handling)
└── Config layer (OAuth provider settings)

STRATEGY 3: Risk-Based Decomposition
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
├── [Low Risk] Read and analyze current code
├── [Low Risk] Write tests for expected behavior
├── [Medium Risk] Implement new OAuth2 module alongside old
├── [High Risk] Switch endpoints to new module
└── [High Risk] Remove old authentication code
```

### Delegation Principles

| Principle | Description |
|-----------|-------------|
| **Single Responsibility** | Each sub-agent handles one clear task |
| **Clear Interfaces** | Well-defined inputs/outputs between agents |
| **Appropriate Granularity** | Not too big (overloaded) or too small (overhead) |
| **Independence** | Sub-tasks should minimize dependencies |
| **Verifiability** | Each sub-task has clear success criteria |

### When to Delegate vs. Do Directly

| Delegate When... | Do Directly When... |
|-----------------|---------------------|
| Task requires different expertise | Simple, single-step task |
| Tasks can run in parallel | Strong sequential dependency |
| Different permission levels needed | Context switching cost > benefit |
| Task is well-bounded and clear | Task needs tight iterative feedback |
| Isolation improves safety | Overhead of delegation > time saved |

---

## 2. Design Patterns

### Pattern 1: Plan-Then-Execute

```python
def plan_then_execute(request):
    # Phase 1: Planning (use high-capability model)
    plan = planning_agent.create_plan(
        request=request,
        constraints=["max 5 steps", "each step independently verifiable"]
    )
    
    # Phase 2: Execution (delegate each step)
    results = []
    for step in plan.steps:
        result = execute_step(step)
        results.append(result)
        
        # Verify before continuing
        if not verify(step, result):
            # Re-plan from current state
            plan = planning_agent.replan(request, results, step.failure_reason)
    
    # Phase 3: Synthesis
    return synthesize_results(results)
```

### Pattern 2: Progressive Disclosure

```
Level 0: Try single agent with all tools
    ↓ (if too complex)
Level 1: Decompose into 2-3 sub-tasks
    ↓ (if still too complex)
Level 2: Further decompose + parallel agents
    ↓ (if still too complex)
Level 3: Hierarchical multi-agent with planning
```

Start simple, escalate complexity only as needed.

### Pattern 3: Dependency-Aware Scheduling

```python
# Build a DAG of task dependencies
tasks = {
    "A": Task(name="Analyze code", deps=[]),
    "B": Task(name="Write tests", deps=["A"]),
    "C": Task(name="Design solution", deps=["A"]),
    "D": Task(name="Implement", deps=["B", "C"]),
    "E": Task(name="Integration test", deps=["D"]),
}

# Execute respecting dependencies, parallelize where possible
# A runs first
# B and C run in parallel (both only depend on A)
# D waits for both B and C
# E waits for D
```

### Pattern 4: Result-Based Re-Delegation

```python
def smart_delegation(task):
    result = delegate_to(fast_agent, task)
    
    if result.confidence < 0.8:
        # Low confidence → re-delegate to more capable agent
        result = delegate_to(thorough_agent, task, 
                           context=f"Previous attempt: {result.summary}")
    
    if result.needs_human_review:
        # Agent knows its limits → escalate
        return escalate_to_human(task, result)
    
    return result
```

---

## 3. Implementation Details

### Task Definition Schema

```python
@dataclass
class SubTask:
    id: str
    description: str
    agent_type: str  # "research", "code", "review"
    dependencies: list[str]  # IDs of tasks that must complete first
    input_context: dict  # What this agent needs to know
    expected_output: str  # What success looks like
    max_iterations: int = 5
    timeout_seconds: int = 120
```

### Claude Code's Agent Tool (Built-in Delegation)

```
Agent(
    description="Brief task summary",
    prompt="Detailed instructions for the sub-agent",
    subagent_type="general-purpose",  # or "Explore", "Plan", etc.
    model="sonnet",  # Match model to task
    isolation="worktree"  # Optional: git isolation
)
```

Key aspects:
- Sub-agent gets NO context from parent conversation
- Prompt must be self-contained
- Result returned as single message
- Can run in background for parallel work

### Effective Delegation Prompts

```python
# BAD: Too vague, relies on context sub-agent doesn't have
prompt = "Fix the bug I mentioned"

# GOOD: Self-contained, specific, includes context
prompt = """
Find and fix the null pointer exception in src/auth/token_validator.py.

Context:
- The error occurs on line 42 when `user.session` is None
- This happens for users who haven't logged in yet
- The fix should return a 401 response instead of crashing

Expected output:
- Edit the file to handle the None case
- Run the existing tests to verify the fix
- Report what you changed and test results
"""
```

### Tracking Delegation Results

```python
class DelegationTracker:
    def __init__(self):
        self.tasks = {}
        self.results = {}
    
    def delegate(self, task: SubTask):
        self.tasks[task.id] = {"status": "in_progress", "started": now()}
        result = run_agent(task)
        self.tasks[task.id] = {"status": "completed", "duration": elapsed()}
        self.results[task.id] = result
        return result
    
    def get_progress(self):
        completed = sum(1 for t in self.tasks.values() if t["status"] == "completed")
        return f"{completed}/{len(self.tasks)} tasks complete"
```

---

## 4. Security & Reliability

### Delegation Trust Model

```
TRUST DECREASES WITH DELEGATION DEPTH
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
User Request (fully trusted)
  └→ Orchestrator (trusted, has full context)
       └→ Sub-Agent Level 1 (semi-trusted, limited context)
            └→ Sub-Agent Level 2 (lower trust, validate outputs)
                 └→ [STOP] — max depth reached
```

### Risks of Deep Delegation

| Risk | Consequence | Mitigation |
|------|------------|------------|
| Context loss | Sub-agents miss crucial constraints | Explicit constraint passing |
| Error amplification | Small error at level 1 → big error at level 3 | Verify at each level |
| Accountability gap | Which agent caused the bug? | Full audit trail |
| Prompt injection | Malicious tool result → sub-agent compromised | Input sanitization |
| Resource exhaustion | N agents × M iterations = explosion | Budget limits per delegation |

### Safe Decomposition Rules

1. **Never delegate security decisions** — keep auth/permissions at orchestrator level
2. **Validate at boundaries** — check sub-agent output before passing to next agent
3. **Limit delegation depth** — max 2-3 levels
4. **Timeout everything** — no sub-task should run indefinitely
5. **Fail safely** — partial results > cascading failure

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A user asks an agent to "update all microservices to use the new logging library." The agent has access to 12 repositories. What is the MOST effective decomposition strategy?

**A:** **Parallel fan-out with template-based delegation:**
1. Orchestrator creates a change template (what to update, how)
2. Spawns 12 parallel sub-agents, one per repository
3. Each sub-agent: reads repo → applies template → runs tests → reports results
4. Orchestrator collects results, identifies failures, reports summary

**Why this works:**
- Repositories are independent (no cross-repo dependencies)
- Same change pattern → reusable template
- Parallel execution dramatically reduces wall-clock time
- Individual failures don't block others

### Scenario 2
**Q:** An orchestrator's sub-agent returns a result that contradicts information from a previous step. What should the orchestrator do?

**A:** The orchestrator should:
1. **Identify the contradiction** explicitly
2. **Check the source of truth** — is there a more authoritative source?
3. **Re-delegate with additional context** — give the sub-agent the contradicting information and ask it to reconcile
4. If still unresolved, **escalate to human** with both versions and the contradiction clearly stated

**Never:** silently pick one, ignore the contradiction, or retry hoping for a different answer.

### Scenario 3
**Q:** You're decomposing a task: "Build a REST API with auth, CRUD operations, and tests." Which decomposition results in the LEAST rework?

**A:** **Dependency-aware decomposition:**
1. Design API schema & data models (prerequisite for everything)
2. Implement auth middleware (prerequisite for protected routes)
3. Implement CRUD endpoints (depends on 1 & 2)
4. Write tests (depends on 3 being stable)

**Why not parallel all?** If CRUD is built without the auth middleware, it'll need rework. If tests are written before the API is stable, they'll break. Respecting dependencies minimizes rework.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Premature decomposition | Simple task → 5 agents → overhead | Try single agent first, decompose only if needed |
| Over-decomposition | 20 tiny tasks with high coordination cost | Right-size tasks (meaningful work per agent) |
| Unclear ownership | Two agents modify same file | One owner per resource |
| Missing dependency tracking | Agent B starts before A finishes | Explicit dependency DAG |
| Fire-and-forget delegation | No validation of sub-agent results | Always verify before using results |
| Context starvation | Sub-agent lacks info to succeed | Include all necessary context in prompt |

---

## 7. Quick Reference / Cheat Sheet

```
DECOMPOSITION STRATEGIES
━━━━━━━━━━━━━━━━━━━━━━━
• Functional: by capability needed
• Layer-based: by system layer
• Risk-based: by impact level
• Dependency-based: by execution order

DELEGATION CHECKLIST
━━━━━━━━━━━━━━━━━━━━
✓ Self-contained prompt (no assumed context)
✓ Clear success criteria
✓ Appropriate model for task complexity
✓ Timeout and iteration limits
✓ Result validation plan
✓ Fallback if delegation fails

TASK SIZING GUIDELINES
━━━━━━━━━━━━━━━━━━━━━━
Too small: < 1 tool call → just do it directly
Right size: 3-10 tool calls, clear outcome
Too large: > 20 tool calls, multiple concerns → decompose further

EXECUTION ORDER
━━━━━━━━━━━━━━━
1. Identify all sub-tasks
2. Map dependencies (DAG)
3. Find independent tasks (parallelize these)
4. Execute respecting dependencies
5. Validate each result
6. Synthesize final output

ESCALATION TRIGGERS
━━━━━━━━━━━━━━━━━━━
• Confidence below threshold
• Contradicting results from agents
• Max retries exceeded
• Security-sensitive decision needed
• Ambiguous requirements
```
