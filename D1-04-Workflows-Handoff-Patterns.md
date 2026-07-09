# Domain 1 | Topic 1.4: Multi-Step Workflows with Enforcement & Handoff Patterns

## 1. Core Concepts

### Programmatic Enforcement vs Prompt-Based Guidance

When a workflow requires deterministic compliance (financial operations, security gates), there are two approaches:

```
PROMPT-BASED (Probabilistic)
┌──────────────────────────────────────────┐
│ System Prompt: "Always verify customer   │
│ before processing refund"                │
│                                          │
│ Claude: Follows ~95% of the time         │
│ Failure rate: ~5% (occasional skips)     │
│ Enforcement: Soft, advisory              │
└──────────────────────────────────────────┘

PROGRAMMATIC (Deterministic)
┌──────────────────────────────────────────┐
│ PreToolUse Hook:                         │
│ IF process_refund is called              │
│   AND no prior get_customer call         │
│ THEN: BLOCK execution                    │
│                                          │
│ Claude: Cannot bypass verification       │
│ Failure rate: 0% (guaranteed)            │
│ Enforcement: Hard, architectural         │
└──────────────────────────────────────────┘
```

**Key principle:** When errors have financial/security consequences, use **programmatic enforcement**. The 5% failure rate of prompt guidance is unacceptable.

### Workflow Patterns

#### Pattern 1: Linear Sequential
```
Step 1 → Step 2 → Step 3 → Done
(Output of step N is input to step N+1)
```
**Use case:** Refund processing (verify → check policy → calculate → process → confirm)

#### Pattern 2: Parallel with Synchronization
```
         ┌── Task A ──┐
Start ──┤            ├── Synthesize
         └── Task B ──┘
```
**Use case:** Investigating multi-concern requests (check shipping, check return policy, check refund eligibility in parallel, then synthesize)

#### Pattern 3: Conditional Branching
```
Check → [Decision] → Path A
                  └─ Path B
```
**Use case:** Escalation gates (if simple refund → process; if policy exception → escalate)

### Handoff Protocols

A **handoff** transfers control from an agent to a human (or another system) with complete context about:
- What was attempted
- Why the agent couldn't resolve it
- Root cause analysis
- Recommended next action
- All relevant facts for the human to continue work

**Handoff components:**
```
HANDOFF SUMMARY
│
├── CONTEXT (What happened)
│   ├── Customer ID
│   ├── Request summary
│   └── Timeline of attempts
│
├── ROOT CAUSE ANALYSIS
│   ├── What was investigated
│   ├── What was discovered
│   └── Why agent cannot proceed
│
├── RECOMMENDED ACTION
│   ├── Primary recommendation
│   └── Alternative approaches
│
└── SUPPORTING FACTS
    ├── Amounts (dollar values)
    ├── Policy references
    └── System state at handoff
```

---

## 2. Design Patterns

### Pattern 1: Programmatic Prerequisite Gates

Enforce required sequences via hooks:

```python
from anthropic import Anthropic

client = Anthropic()

# Track which prerequisites have been completed
completed_steps = {}

def handle_tool_use(tool_name, tool_input):
    """Execute tool with prerequisite enforcement."""
    
    # Define prerequisites
    prerequisites = {
        "process_refund": ["get_customer"],      # Must verify customer first
        "update_order": ["get_customer"],        # Must know who we're talking to
        "send_confirmation": ["process_refund"]  # Only after refund processes
    }
    
    # Check if prerequisites are met
    if tool_name in prerequisites:
        required_steps = prerequisites[tool_name]
        for step in required_steps:
            if not completed_steps.get(step):
                return {
                    "error": f"Prerequisite not met: {step} must complete before {tool_name}",
                    "required_first": required_steps,
                    "completed_so_far": list(completed_steps.keys())
                }
    
    # Execute the tool
    result = execute_tool(tool_name, tool_input)
    
    # Mark this step as complete
    completed_steps[tool_name] = True
    
    return result

def run_refund_agent():
    """Process a customer refund with enforced workflow."""
    
    messages = [{
        "role": "user",
        "content": "Customer #12345 wants to return their order and get a refund"
    }]
    
    while True:
        response = client.messages.create(
            model="claude-opus-4-6-20250514",
            max_tokens=2048,
            system="""You are a refund agent. Process refunds by:
1. Verifying the customer (get_customer)
2. Looking up their order (lookup_order)
3. Checking the refund policy (check_refund_policy)
4. Calculating the refund amount (calculate_refund)
5. Processing the refund (process_refund)
6. Sending confirmation (send_confirmation)

Always follow this sequence. Do not skip steps.""",
            tools=[
                {
                    "name": "get_customer",
                    "description": "Retrieve verified customer information by ID"
                },
                {
                    "name": "lookup_order",
                    "description": "Look up order details for a customer"
                },
                {
                    "name": "process_refund",
                    "description": "Process the refund (only after verification)"
                },
                # ... other tools
            ],
            messages=messages
        )
        
        messages.append({"role": "assistant", "content": response.content})
        
        if response.stop_reason == "tool_use":
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    # ENFORCEMENT HAPPENS HERE
                    result = handle_tool_use(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(result)
                    })
            
            messages.append({"role": "user", "content": tool_results})
        else:
            # Agent produced final response
            return response.content[0].text
```

**Key enforcement mechanism:**
- `handle_tool_use()` checks prerequisites BEFORE executing
- If prerequisites aren't met, returns error (not the tool result)
- Claude sees the error and must take correct action (call prerequisite first)
- This is guaranteed to work — no way to bypass

### Pattern 2: Structured Handoff to Human

When the agent cannot proceed, create a comprehensive handoff:

```python
def create_handoff_summary(customer_id, request_type, investigation_results, 
                          root_cause, recommended_action):
    """Create a structured handoff for human agents."""
    
    handoff = {
        "timestamp": datetime.utcnow().isoformat(),
        "handoff_reason": "Policy exception requires human judgment",
        
        "customer_context": {
            "customer_id": customer_id,
            "verified": True,
            "account_status": "active",
            "lifetime_value": "$5,420"
        },
        
        "request_details": {
            "type": request_type,
            "original_request": "Refund requested due to damaged item (customer claims packaging was torn)",
            "amount_requested": 89.99,
            "order_id": "ORD-2024-12345",
            "purchase_date": "2024-06-15",
            "damage_evidence": "Customer provided photo showing visible damage"
        },
        
        "investigation_results": {
            "order_verified": True,
            "within_return_window": True,
            "damage_claim_validated": "Photo matches damaged item description",
            "prior_refunds_for_customer": 0,
            "policy_allows_damage_refund": True
        },
        
        "root_cause_analysis": {
            "issue": "Policy permits damage refund, but amount exceeds auto-approval threshold ($50)",
            "why_agent_cannot_proceed": "Agent lacks authority to approve refunds > $50; requires manager review",
            "severity": "low - routine exception, not customer error"
        },
        
        "recommended_action": {
            "primary": "Approve full refund of $89.99 (damage claim is valid, within policy)",
            "rationale": "Damage is legitimate, return window open, no prior exploits by customer",
            "alternative": "Offer $50 auto-refund + $39.99 store credit if full refund denied",
            "escalation_not_needed_for": ["shipping issues", "minor damage"]
        },
        
        "required_actions": [
            "Authorize refund",
            "Process return shipping label",
            "Send confirmation email to customer"
        ]
    }
    
    return handoff
```

**Handoff structure ensures:**
- Human has all facts without reading conversation history
- Root cause is explicit (not just "customer wants refund")
- Recommendation is actionable (not vague "discuss with manager")
- Human can approve/deny quickly (structured data, not prose)

### Pattern 3: Multi-Concern Decomposition

Customer requests often involve multiple independent issues:

```python
def decompose_multi_concern_request(request):
    """Break complex request into independent concerns."""
    
    # Request: "My order hasn't arrived AND the item is wrong. 
    # Also charge my card twice."
    
    concerns = [
        {
            "id": "concern_1",
            "type": "shipping",
            "description": "Order hasn't arrived",
            "investigation": "Check tracking, verify delivery address, estimate arrival",
            "resolution_path": "Reship or refund depending on carrier status"
        },
        {
            "id": "concern_2",
            "type": "wrong_item",
            "description": "Received incorrect item",
            "investigation": "Verify order contents, get photo from customer if needed",
            "resolution_path": "Send replacement or arrange return+refund"
        },
        {
            "id": "concern_3",
            "type": "billing",
            "description": "Card charged twice",
            "investigation": "Check payment history, verify both charges, identify duplicate",
            "resolution_path": "Reverse duplicate charge"
        }
    ]
    
    # Investigate ALL concerns in parallel with shared context
    all_context = get_customer_context(customer_id)
    
    results = {}
    for concern in concerns:
        results[concern["id"]] = investigate_concern(
            concern, all_context
        )
    
    # Synthesize: single unified response addressing all three
    return synthesize_multi_concern_response(results)
```

**Benefits:**
- Each concern is investigated independently (no missed issues)
- Parallel investigation (faster)
- Shared context prevents duplication (efficient)
- Unified response (customer sees one solution, not three separate answers)

### Pattern 4: Escalation Decision Tree

```
START: Customer Request
  │
  ├─ [Escalation Trigger 1: Customer explicitly asks for human?]
  │  └─ YES → Escalate immediately (honor customer preference)
  │  └─ NO → Continue
  │
  ├─ [Escalation Trigger 2: Policy ambiguous or silent on this case?]
  │  └─ YES → Escalate (requires judgment call)
  │  └─ NO → Continue
  │
  ├─ [Escalation Trigger 3: Multiple matches returned (ambiguous customer)?]
  │  └─ YES → Ask for clarification (don't guess)
  │  └─ NO → Continue
  │
  ├─ [Escalation Trigger 4: Root cause is outside agent's tools?]
  │  └─ YES → Escalate (need manual intervention)
  │  └─ NO → Continue
  │
  └─ [Escalation Trigger 5: Consecutive failures?]
     └─ YES (3+ retries failed) → Escalate (stuck)
     └─ NO → Resolve autonomously
```

---

## 3. Implementation Details

### Prerequisite DAG (Directed Acyclic Graph)

For complex workflows, visualize dependencies:

```
    ┌─────────────────┐
    │  get_customer   │
    │ (MANDATORY)     │
    └────────┬────────┘
             │
      ┌──────┴──────┐
      │             │
    ┌─▼──────┐   ┌──▼────────┐
    │lookup_ │   │check_fraud │
    │order   │   │_status     │
    └─┬──────┘   └──┬────────┘
      │             │
      └──────┬──────┘
             │
        ┌────▼───────────────┐
        │ calculate_refund   │
        │ (depends on both)  │
        └────┬───────────────┘
             │
        ┌────▼──────────┐
        │process_refund │
        │ (FINAL STEP)  │
        └───────────────┘
```

**Key principle:** Enforce at decision points, not as suggestions.

### Error Categorization for Handoffs

When escalating, categorize the failure:

```python
escalation_categories = {
    "policy_exception": {
        "description": "Policy doesn't cover this case",
        "example": "Refund > $1000 requires manager approval",
        "handoff_urgency": "medium",
        "decision_type": "judgment_call"
    },
    "customer_ambiguity": {
        "description": "Cannot identify customer uniquely",
        "example": "Multiple John Smiths found",
        "handoff_urgency": "high",
        "decision_type": "clarification_needed"
    },
    "system_limitation": {
        "description": "Agent lacks capability or permissions",
        "example": "Cannot issue refund > $500 without override",
        "handoff_urgency": "medium",
        "decision_type": "authorization_needed"
    },
    "data_inconsistency": {
        "description": "Conflicting information in systems",
        "example": "Order says shipped but tracking shows cancelled",
        "handoff_urgency": "high",
        "decision_type": "investigation_needed"
    }
}
```

---

## 4. Security & Reliability

### Verification Before Financial Operations

Critical workflows should verify state at each gate:

```python
def safe_refund_gate(customer_id, order_id, amount):
    """Verify conditions before allowing refund."""
    
    verifications = [
        verify_customer_exists(customer_id),
        verify_order_exists(order_id),
        verify_order_belongs_to_customer(order_id, customer_id),
        verify_within_return_window(order_id),
        verify_refund_not_already_issued(order_id),
        verify_amount_within_limit(amount)
    ]
    
    # ALL verifications must pass
    for verification in verifications:
        if not verification["passed"]:
            return {
                "allowed": False,
                "reason": verification["failure_reason"],
                "escalate_to": "human_review"
            }
    
    # All checks passed
    return {
        "allowed": True,
        "proceed_with_refund": True
    }
```

### Logging for Audit Trail

Every handoff should be logged:

```python
def log_handoff(handoff_summary):
    """Create immutable audit trail."""
    
    audit_entry = {
        "timestamp": datetime.utcnow().isoformat(),
        "event_type": "handoff_to_human",
        "customer_id": handoff_summary["customer_context"]["customer_id"],
        "reason": handoff_summary["handoff_reason"],
        "agent_investigation": handoff_summary["investigation_results"],
        "recommended_action": handoff_summary["recommended_action"],
        "human_assigned": None,  # Filled in when human takes action
        "resolution": None  # Filled in when resolved
    }
    
    # Log to audit system (immutable)
    save_to_audit_log(audit_entry)
    
    return audit_entry["id"]
```

---

## 5. Exam Scenarios

### Scenario 1: Prerequisite Enforcement
**Q:** A refund agent occasionally calls `process_refund` without calling `get_customer` first, leading to incorrect refunds. The system prompt says "always verify first" but it doesn't work. What's the BEST fix?

**A:** Implement a PreToolUse hook that blocks `process_refund` calls unless `get_customer` has already been called in the conversation. This provides **deterministic enforcement** — impossible to bypass. Prompt instructions are probabilistic (~95% compliance); hooks are deterministic (100% compliance).

### Scenario 2: Multi-Concern Decomposition
**Q:** A customer messages: "My order is late AND they sent the wrong item AND I was charged twice." The agent addresses only the late order, missing the other issues. Why?

**A:** The agent didn't decompose the multi-concern request. It addressed the first concern and considered the task complete. **Fix:** Explicitly decompose into 3 independent concerns, investigate all three in parallel with shared context, then synthesize a single unified response.

### Scenario 3: Handoff Timing
**Q:** An agent spends 5 minutes investigating a request, then escalates to a human with the message "Customer wants refund, not sure what to do." The human complains they have no context. What should change?

**A:** The agent should create a **structured handoff summary** including: what was investigated, what was discovered, why escalation is needed, and recommended action. This lets the human decide immediately without re-investigating.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Relying on prompt for financial gates | ~5% failure rate unacceptable | Programmatic prerequisite hooks |
| Escalating without context | Human has to re-investigate | Structured handoff summary |
| Treating all escalations the same | Urgent != all urgent | Categorize by type (policy exception vs. ambiguity vs. system limit) |
| Investigating one concern at a time | Misses multi-concern requests | Decompose, investigate parallel, synthesize |
| Blocking customer immediately on ambiguity | Bad experience | Ask clarifying question first, escalate only if customer doesn't respond |
| No audit trail | Can't trace decisions | Log every handoff with full context |

---

## 7. Quick Reference / Cheat Sheet

```
ENFORCEMENT VS GUIDANCE
━━━━━━━━━━━━━━━━━━━━━━
Prompt-based: ~95% compliance (probabilistic)
Hooks-based: 100% compliance (deterministic)

Use hooks for: financial, security, identity verification
Use prompts for: style, format, helpful suggestions

HANDOFF COMPONENTS
━━━━━━━━━━━━━━━━━━
✓ Customer context (ID, account status, lifetime value)
✓ Request details (what, when, amount)
✓ Investigation results (what was checked)
✓ Root cause (why agent can't proceed)
✓ Recommendation (actionable next step)
✓ Supporting facts (specific amounts, policies)

PREREQUISITES PATTERN
━━━━━━━━━━━━━━━━━━━━
Define: prerequisites = {target_tool: [required_tools]}
Check: IF target_tool in prerequisites, verify all required_tools completed
Enforce: PreToolUse hook checks before allowing execution
Result: Claude cannot bypass the sequence

MULTI-CONCERN DECOMPOSITION
━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Parse request for multiple issues
2. Create concern objects (id, type, description)
3. Investigate ALL concerns in parallel (shared context)
4. Synthesize unified response addressing all
5. Avoid: addressing one and forgetting others

ESCALATION TRIGGERS
━━━━━━━━━━━━━━━━━━━
• Customer explicitly asks for human → Escalate immediately
• Policy ambiguous or silent → Escalate (judgment needed)
• Ambiguous identifier match → Ask clarification first
• Outside agent's capabilities → Escalate (manual intervention)
• Multiple failures → Escalate (stuck)

CATEGORIES
━━━━━━━━━━
policy_exception: Judgment call needed
customer_ambiguity: Clarification needed
system_limitation: Authorization/permissions needed
data_inconsistency: Investigation needed
```

---

## Key Takeaways for Exam

✅ **Enforcement >> Guidance** — When stakes are high, use programmatic hooks not prompts  
✅ **Handoffs need structure** — Not prose; use facts, root cause, recommendation  
✅ **Multi-concerns are tricky** — Decompose, investigate parallel, synthesize once  
✅ **Escalation must have rules** — Not just "when it's hard"; specific triggers  
✅ **Audit trail matters** — Log every handoff for compliance and learning
