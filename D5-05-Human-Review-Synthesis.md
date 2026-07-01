# Domain 5 | Topic 5.5: Human Review & Synthesis Strategies

## 1. Core Concepts

### Human-in-the-Loop Patterns

**Human-in-the-loop (HITL)** systems keep humans involved at critical decision points, balancing automation speed with human judgment for high-stakes or ambiguous situations.

```
HITL ARCHITECTURES
━━━━━━━━━━━━━━━━━━

1. APPROVAL GATE:
   Agent works → produces output → Human approves/rejects → proceed/redo

2. CHECKPOINT REVIEW:
   Agent: step 1 → [checkpoint] → step 2 → [checkpoint] → step 3

3. EXCEPTION HANDLING:
   Agent works autonomously → flags exceptions → Human handles exceptions

4. COLLABORATIVE:
   Human provides direction → Agent executes → Human refines → Agent adjusts
```

### When Humans Are Needed

| Scenario | Why Human Required |
|----------|-------------------|
| Ambiguous intent | Only user knows what they actually want |
| Value judgments | Trade-offs require human priorities |
| Novel situations | No precedent for agent to follow |
| Irreversible actions | Can't undo (deploy, delete, send) |
| Safety-critical | Security, compliance, legal implications |
| Quality bar unclear | "Good enough" is subjective |

### Synthesis = Combining Multiple Sources

When an agent gathers information from multiple sources (tools, agents, documents), **synthesis** is combining it into coherent, actionable output for human consumption.

---

## 2. Design Patterns

### Pattern 1: Approval Gate Before Destructive Actions

```python
def deploy_with_approval(changes):
    """Agent prepares, human approves."""
    
    # Agent does the prep work
    plan = agent.create_deploy_plan(changes)
    tests = agent.run_test_suite()
    risk_assessment = agent.assess_risk(changes)
    
    # Present to human for approval
    approval_request = {
        "action": "deploy to production",
        "changes_summary": plan.summary,
        "test_results": tests.summary,
        "risk_level": risk_assessment.level,
        "affected_services": plan.services,
        "rollback_plan": plan.rollback,
        "question": "Approve this deployment?"
    }
    
    human_decision = await get_human_approval(approval_request)
    
    if human_decision.approved:
        return agent.execute_deploy(plan)
    else:
        return agent.handle_rejection(human_decision.feedback)
```

### Pattern 2: Synthesis Report for Human Decision

```python
def synthesize_research(findings):
    """Combine multiple agent findings into human-readable report."""
    
    report = {
        "question": "Should we migrate from PostgreSQL to DynamoDB?",
        "sources_consulted": [
            {"source": "cost_analysis_agent", "finding": findings["cost"]},
            {"source": "performance_agent", "finding": findings["perf"]},
            {"source": "migration_risk_agent", "finding": findings["risk"]},
            {"source": "team_capability_agent", "finding": findings["team"]}
        ],
        "synthesis": {
            "pro": ["50% cost reduction at scale", "Better horizontal scaling"],
            "con": ["6-month migration effort", "Team unfamiliar with DynamoDB", "Loss of complex query capability"],
            "recommendation": "Defer migration — cost savings don't justify risk given current team expertise",
            "confidence": "Medium (based on estimates, not benchmarks)"
        },
        "decision_needed": "Proceed with migration or invest in PostgreSQL optimization?"
    }
    
    return report
```

### Pattern 3: Progressive Human Involvement

```
LOW RISK (Agent Autonomous):
  Simple code changes, test additions, formatting
  → Agent does it, reports what it did

MEDIUM RISK (Agent Proposes, Human Confirms):
  Architecture changes, dependency updates, API changes  
  → Agent creates plan, human reviews and approves

HIGH RISK (Human Directs, Agent Assists):
  Security changes, data migrations, production deployments
  → Human makes decision, agent helps execute safely

CRITICAL (Human Only):
  Compliance decisions, user data deletion, legal implications
  → Agent provides information, human makes and executes decision
```

### Pattern 4: Conflict Resolution via Human

```python
def resolve_conflict(agent_a_result, agent_b_result, context):
    """When agents disagree, present both to human with analysis."""
    
    conflict_report = {
        "situation": "Two agents reached different conclusions",
        "agent_a": {
            "conclusion": agent_a_result.conclusion,
            "reasoning": agent_a_result.reasoning,
            "evidence": agent_a_result.evidence,
            "confidence": agent_a_result.confidence
        },
        "agent_b": {
            "conclusion": agent_b_result.conclusion,
            "reasoning": agent_b_result.reasoning,
            "evidence": agent_b_result.evidence,
            "confidence": agent_b_result.confidence
        },
        "analysis": {
            "where_they_agree": find_agreement(agent_a_result, agent_b_result),
            "core_disagreement": identify_disagreement(agent_a_result, agent_b_result),
            "possible_resolution": suggest_resolution(context)
        },
        "question": "Which approach should we take, or should we gather more information?"
    }
    
    return await present_to_human(conflict_report)
```

---

## 3. Implementation Details

### Effective Synthesis Structure

```
SYNTHESIS REPORT TEMPLATE
━━━━━━━━━━━━━━━━━━━━━━━━

## Summary (1-2 sentences)
The key finding/recommendation in plain language.

## Evidence
- Source 1: [finding] (confidence: high/medium/low)
- Source 2: [finding] (confidence: high/medium/low)
- Source 3: [finding] (confidence: high/medium/low)

## Analysis
- Points of agreement across sources
- Points of conflict and why
- Gaps: what we don't know

## Recommendation
- Proposed action
- Confidence level
- Risk if wrong

## Decision Needed
- Specific question for human
- Options available
- Time sensitivity
```

### Human Review Checkpoint Design

```python
class ReviewCheckpoint:
    def __init__(self, name, criteria, blocking=True):
        self.name = name
        self.criteria = criteria  # What human should check
        self.blocking = blocking  # Must human approve to continue?
    
    async def present(self, work_so_far, agent_assessment):
        return {
            "checkpoint": self.name,
            "work_completed": work_so_far,
            "agent_assessment": agent_assessment,
            "review_criteria": self.criteria,
            "question": "Approve to continue?" if self.blocking else "Any feedback?",
            "options": ["approve", "reject", "modify", "defer"]
        }

# Define checkpoints for a workflow
DEPLOY_CHECKPOINTS = [
    ReviewCheckpoint("pre_deploy", ["Tests pass?", "No breaking changes?", "Rollback plan?"], blocking=True),
    ReviewCheckpoint("canary_check", ["Error rate stable?", "Latency acceptable?"], blocking=True),
    ReviewCheckpoint("full_rollout", ["Canary healthy for 30min?"], blocking=True)
]
```

### Presenting Information for Human Decision

| Principle | Implementation |
|-----------|---------------|
| **Lead with the decision** | Start with "Should we X or Y?" not background |
| **Provide context, not noise** | Only relevant information |
| **Show trade-offs clearly** | Pro/con format for each option |
| **Include agent's recommendation** | Human can agree or override |
| **State confidence** | "I'm 80% sure" helps human calibrate |
| **Make options concrete** | "Approve deployment" not "What should I do?" |

---

## 4. Security & Reliability

### Security of Human-in-the-Loop

| Risk | Mitigation |
|------|-----------|
| Approval fatigue | Reduce false alarms, batch non-critical reviews |
| Social engineering via agent output | Verify agent's claims before approving |
| Bypassing human review | Architectural enforcement (can't skip checkpoint) |
| Delayed response blocking system | Timeout with safe default (deny) |
| Human approves based on incomplete info | Agent must present all relevant risks |

### Reliability Patterns

1. **Timeout with safe default** — if human doesn't respond in X time, default to safe action (deny/wait)
2. **Audit trail** — log all human decisions for accountability
3. **Multiple approvers for critical actions** — no single point of approval failure
4. **Async review** — don't block automation for non-urgent reviews
5. **Clear escalation path** — if first reviewer is unavailable, who's next?

---

## 5. Exam Scenarios

### Scenario 1
**Q:** An agent has analyzed 10 data sources and needs to present a recommendation to a human decision-maker. What is the BEST format for the synthesis?

**A:** Structured synthesis report:
1. **One-line recommendation** (what to do)
2. **Key evidence** (3-5 most relevant findings, sourced)
3. **Trade-offs** (pro/con of the recommendation)
4. **Confidence level** (how certain the agent is)
5. **Decision question** (specific, actionable question for the human)
6. **Time sensitivity** (when must the decision be made)

**Not:** raw data dump, verbose analysis, or vague "what do you think?"

### Scenario 2
**Q:** An agentic system processes 100 requests/hour. Most are routine, but ~5% need human judgment. How should the system handle the 5%?

**A:** **Exception-based HITL:**
1. Agent processes all requests autonomously
2. For the 5% needing judgment: agent queues them with context
3. Human reviewer sees a prioritized queue of exceptions
4. Each exception includes: what happened, what the agent would do, why it's uncertain
5. Human approves/modifies/rejects
6. System learns from patterns in human decisions (if ML pipeline exists)

Key: don't slow down the 95% waiting for human review of the 5%.

### Scenario 3
**Q:** A human reviewer is presented with an agent's plan that includes a risky database migration. The agent says "risk: low." Should the human trust this assessment?

**A:** **No — the human should verify the agent's risk assessment** by checking:
1. What did the agent base "low risk" on? (provenance)
2. Did it consider data volume? (50M row table = different risk than 1K)
3. Is there a rollback plan?
4. Was this tested against production-like data?
5. What's the blast radius if it fails?

The agent's assessment is a starting point for the human's evaluation, not a final answer. The human adds judgment the agent can't — organizational context, past incident history, current team capacity.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Asking human about everything | Bottleneck, defeats automation | Only escalate when needed (risk-based) |
| Binary approve/reject only | Misses "modify" option | Offer: approve, reject, modify, defer |
| Raw data instead of synthesis | Human overwhelmed | Synthesize into actionable format |
| No recommendation from agent | Puts all cognitive load on human | Always include agent's suggestion |
| Blocking automation for non-urgent review | Slows everything down | Async review queue for non-critical |
| No timeout on human response | System hangs forever | Default-deny after timeout period |

---

## 7. Quick Reference / Cheat Sheet

```
HITL PATTERNS
━━━━━━━━━━━━━
• Approval Gate: agent proposes → human approves
• Checkpoint: periodic review during execution
• Exception: autonomous + flag anomalies for human
• Collaborative: human directs, agent executes

RISK-BASED INVOLVEMENT
━━━━━━━━━━━━━━━━━━━━━━━
Low risk → Agent autonomous
Medium risk → Agent proposes, human confirms
High risk → Human directs, agent assists
Critical → Human decides and executes

SYNTHESIS FORMAT
━━━━━━━━━━━━━━━━
1. Summary (1-2 sentences)
2. Key evidence (sourced)
3. Trade-offs (pro/con)
4. Recommendation + confidence
5. Specific decision question
6. Time sensitivity

HUMAN REVIEW PRINCIPLES
━━━━━━━━━━━━━━━━━━━━━━━━
✓ Lead with the decision needed
✓ Include agent's recommendation
✓ Show relevant evidence only
✓ State confidence level
✓ Offer concrete options (not open-ended)
✓ Include time sensitivity
✓ Provide safe default if no response
```
