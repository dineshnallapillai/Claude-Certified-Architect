# Domain 1 | Topic 1.2: Multi-Agent Orchestration (Hub-and-Spoke)

## 1. Core Concepts

### What is Multi-Agent Orchestration?

Multi-agent orchestration involves coordinating **multiple Claude instances** (agents), each with specialized roles, tools, and system prompts, to collaboratively solve complex tasks that exceed a single agent's capabilities.

### Orchestration Topologies

```
┌─────────────────────────────────────────────────────────┐
│  HUB-AND-SPOKE (Star Pattern)                           │
│                                                         │
│              ┌──── Agent A (Research)                    │
│              │                                          │
│  User ──► Router/Hub ──── Agent B (Code)                │
│              │                                          │
│              └──── Agent C (Review)                      │
│                                                         │
│  • Central orchestrator delegates to specialists         │
│  • Agents don't talk to each other directly              │
│  • Hub synthesizes results                              │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  PIPELINE (Sequential)                                  │
│                                                         │
│  Input → Agent A → Agent B → Agent C → Output           │
│                                                         │
│  • Each agent transforms and passes forward              │
│  • Good for staged processing (draft → review → edit)   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  PARALLEL (Fan-out/Fan-in)                              │
│                                                         │
│         ┌── Agent A ──┐                                 │
│  Input ─┼── Agent B ──┼─► Merge → Output                │
│         └── Agent C ──┘                                 │
│                                                         │
│  • Same task, different perspectives or data splits      │
│  • Results merged by orchestrator                       │
└─────────────────────────────────────────────────────────┘
```

### Hub-and-Spoke Architecture (Primary Exam Focus)

The **hub (orchestrator)** is itself a Claude agent that:
1. Receives the user's request
2. Decomposes it into sub-tasks
3. Delegates sub-tasks to specialist agents
4. Collects and synthesizes results
5. Returns unified response to user

**Spoke agents** are specialized:
- Narrow system prompts focused on one capability
- Limited tool sets (principle of least privilege)
- No knowledge of other agents
- Return structured results to hub

### Key Terminology

| Term | Definition |
|------|-----------|
| **Orchestrator** | The central agent that coordinates others |
| **Sub-agent** | A specialist agent invoked by the orchestrator |
| **Delegation** | Passing a sub-task to a specialist agent |
| **Synthesis** | Combining results from multiple agents |
| **Handoff** | Transferring control from one agent to another |
| **Context isolation** | Each agent has its own message history |

---

## 2. Design Patterns

### Pattern 1: Router-Based Hub

```python
# The hub decides which specialist to invoke based on the task
AGENTS = {
    "researcher": {
        "system": "You are a research specialist...",
        "tools": [web_search, read_document]
    },
    "coder": {
        "system": "You are a code specialist...",
        "tools": [write_file, run_tests]
    },
    "reviewer": {
        "system": "You are a code reviewer...",
        "tools": [read_file, lint]
    }
}

def orchestrate(user_request):
    # Hub decides which agents to invoke
    plan = hub_agent.plan(user_request)
    
    results = {}
    for step in plan.steps:
        agent_config = AGENTS[step.agent_type]
        result = run_sub_agent(
            system=agent_config["system"],
            tools=agent_config["tools"],
            task=step.task_description
        )
        results[step.name] = result
    
    # Hub synthesizes all results
    return hub_agent.synthesize(results)
```

### Pattern 2: Hierarchical Delegation (Claude Code Style)

```
Orchestrator (Opus)
├── Research Agent (Sonnet) — fast lookups
├── Code Agent (Sonnet) — writes/edits code  
├── Review Agent (Haiku) — quick validation
└── Planning Agent (Opus) — complex architecture decisions
```

- Use more capable models for complex reasoning
- Use faster/cheaper models for routine tasks
- Match model capability to task complexity

### Pattern 3: Agent as Tool

The cleanest integration pattern — sub-agents are exposed as tools to the hub:

```python
tools = [
    {
        "name": "research_agent",
        "description": "Delegate a research task to a specialist. Use when you need to find information from documents or the web.",
        "input_schema": {
            "type": "object",
            "properties": {
                "task": {"type": "string", "description": "What to research"},
                "scope": {"type": "string", "enum": ["web", "docs", "code"]}
            },
            "required": ["task"]
        }
    },
    {
        "name": "code_agent", 
        "description": "Delegate a coding task to a specialist. Use for writing, modifying, or debugging code.",
        "input_schema": {
            "type": "object",
            "properties": {
                "task": {"type": "string"},
                "files": {"type": "array", "items": {"type": "string"}}
            },
            "required": ["task"]
        }
    }
]
```

### Pattern 4: Parallel Fan-Out

```python
import asyncio

async def parallel_agents(tasks):
    """Run multiple agents concurrently for independent sub-tasks."""
    results = await asyncio.gather(*[
        run_sub_agent(task) for task in tasks
    ])
    return results
```

- Use when sub-tasks are independent
- Dramatically reduces wall-clock time
- Hub merges results after all complete

---

## 3. Implementation Details

### Context Isolation

Each sub-agent gets a **fresh message history**. The orchestrator must explicitly pass:
- The sub-task description
- Any necessary context from prior steps
- Constraints and expected output format

```python
def run_sub_agent(system, tools, task, context=None):
    messages = [{"role": "user", "content": f"""
Task: {task}

{f'Context: {context}' if context else ''}

Respond with your findings in JSON format.
"""}]
    
    response = client.messages.create(
        model="claude-sonnet-4-6",
        system=system,
        tools=tools,
        messages=messages,
        max_tokens=4096
    )
    return response
```

### Information Passing Between Agents

| Strategy | When to Use |
|----------|------------|
| Full context forwarding | Small tasks, need complete picture |
| Summary only | Large outputs, downstream agent needs gist |
| Structured data | When format matters (JSON, schemas) |
| Selective extraction | Only relevant portions passed forward |

### Model Selection for Sub-Agents

| Task Type | Recommended Model | Reason |
|-----------|-------------------|--------|
| Complex planning | Opus | Best reasoning |
| Code generation | Sonnet | Good balance of quality/speed |
| Simple classification | Haiku | Fast, cheap, sufficient |
| Document analysis | Sonnet | Good at extraction |
| Validation/checking | Haiku | Quick yes/no decisions |

---

## 4. Security & Reliability

### Trust Boundaries

```
┌─────────────────────────────────────┐
│  TRUST BOUNDARY                     │
│                                     │
│  Orchestrator (full access)         │
│  ├── Sub-agent A (read-only tools)  │
│  ├── Sub-agent B (write tools)      │ ← Higher risk
│  └── Sub-agent C (no tools)         │
│                                     │
└─────────────────────────────────────┘
```

### Security Principles

1. **Least privilege per agent** — each agent gets only the tools it needs
2. **Validate inter-agent messages** — don't blindly trust sub-agent output
3. **Separate concerns** — agent that reads sensitive data ≠ agent that has network access
4. **Audit trail** — log all delegations and results
5. **Timeout per sub-agent** — prevent any single agent from blocking the system

### Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|-----------|
| Sub-agent timeout | Orchestrator blocks | Timeouts + fallback |
| Sub-agent hallucination | Bad data propagates | Validation agent or checks |
| Circular delegation | Infinite loop | Depth limit on delegation |
| Context overflow | Sub-agent gets confused | Summarize before passing |
| Conflicting results | Hub can't synthesize | Voting, confidence scores, or escalation |

---

## 5. Exam Scenarios

### Scenario 1
**Q:** You're designing a system where users submit legal documents for analysis. The system must: extract key clauses, check compliance against regulations, and generate a summary report. Which multi-agent architecture is MOST appropriate?

**A:** **Pipeline architecture** — sequential processing where:
1. Extraction Agent (Sonnet) → identifies and structures key clauses
2. Compliance Agent (Opus) → checks each clause against regulation database
3. Report Agent (Sonnet) → generates human-readable summary

**Why pipeline:** Each step depends on the previous one's output. The compliance check needs extracted clauses, and the report needs compliance results. Parallel won't work here due to data dependencies.

### Scenario 2
**Q:** An orchestrator is delegating to 5 sub-agents in parallel. Two return conflicting recommendations. What is the BEST resolution strategy?

**A:** The orchestrator should:
1. Present both recommendations with their reasoning to a **tie-breaking agent** (or itself with elevated thinking budget)
2. Consider confidence signals from each agent
3. If still ambiguous, escalate to human review with both options clearly presented

**Not:** picking randomly, always preferring one agent, or re-running until they agree.

### Scenario 3
**Q:** Your hub-and-spoke system's sub-agents are consuming excessive tokens because the orchestrator passes the full conversation history to each. What optimization reduces costs MOST?

**A:** **Selective context forwarding** — the orchestrator should summarize only the relevant portions for each sub-agent's specific task, rather than forwarding the entire message history. Each sub-agent only needs context directly relevant to its narrow task.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Over-delegation | Simple tasks split across 5 agents = slow + expensive | Single agent for simple tasks |
| God orchestrator | Hub does all reasoning, sub-agents are trivial | Meaningful delegation with smart specialists |
| Shared mutable state | Agents modify same data concurrently | Immutable data, hub merges |
| No context isolation | Sub-agent gets confused by irrelevant history | Fresh context per sub-agent |
| Uniform model for all | Haiku for planning OR Opus for classification = waste | Match model to task complexity |
| Deep nesting | Agent → agent → agent → agent | Max 2-3 levels of delegation |

---

## 7. Quick Reference / Cheat Sheet

```
MULTI-AGENT PATTERNS
━━━━━━━━━━━━━━━━━━━
• Hub-and-Spoke: Central orchestrator → specialist agents
• Pipeline: Agent A → Agent B → Agent C (sequential)
• Parallel: Fan-out to N agents → merge results
• Hierarchical: Orchestrator(Opus) → Workers(Sonnet/Haiku)

HUB RESPONSIBILITIES
━━━━━━━━━━━━━━━━━━━
• Decompose user request into sub-tasks
• Select appropriate agent per sub-task
• Pass minimal relevant context
• Synthesize results into coherent response
• Handle failures and conflicts

DESIGN PRINCIPLES
━━━━━━━━━━━━━━━━━
• Least privilege per agent
• Context isolation (fresh history per sub-agent)
• Match model to task complexity
• Max 2-3 delegation depth
• Timeouts on every sub-agent call
• Validate sub-agent outputs

MODEL SELECTION
━━━━━━━━━━━━━━
• Opus → complex reasoning, planning, architecture
• Sonnet → code, analysis, general tasks
• Haiku → classification, validation, simple lookups

WHEN TO USE MULTI-AGENT
━━━━━━━━━━━━━━━━━━━━━━
✓ Task requires multiple specialized skills
✓ Sub-tasks can be clearly decomposed
✓ Different trust/permission levels needed
✓ Parallel processing would help latency
✗ Simple single-step tasks
✗ Tasks requiring tight iterative collaboration
```
