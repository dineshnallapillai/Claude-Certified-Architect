# Exercise 1: Build a Multi-Tool Agent with Escalation Logic

**Source:** Claude Certified Architect – Foundations Exam Guide v1.0, Section 8  
**Objective:** Practice designing an agentic loop with tool integration, structured error handling, and escalation patterns.  
**Domains Reinforced:** D1 (Agentic Architecture & Orchestration), D2 (Tool Design & MCP Integration), D5 (Context Management & Reliability)  
**Estimated Duration:** 3-4 hours  
**Difficulty:** Medium

---

## Overview

You will build a customer support resolution agent using the Claude Agent SDK. This exercise reinforces:

- Agentic loop design (stop_reason handling)
- Tool interface design (clear descriptions, boundary conditions)
- Structured error responses (error categories, retryable flags)
- Agent SDK hooks for business rule enforcement
- Multi-concern request decomposition
- Escalation workflows

---

## Part 1: Define Tool Specifications

### Step 1.1: Define 3-4 MCP Tools with Clear Descriptions

Your agent will have these tools:

**Tool 1: `get_customer`**
```
Name: get_customer
Description: Retrieve customer account information by ID or email. 
  Returns: customer_id, name, email, account_status, customer_tier, account_created_date
  Input: customer_id (string) OR email (string, must be exact match)
  Edge case: Returns empty if customer not found; does not match on partial email
  Use this: As the FIRST step before ANY order operations to verify customer identity
  Do NOT use: For order lookups; use lookup_order instead
  Performance: <500ms typical, <2s worst-case
```

**Tool 2: `lookup_order`**
```
Name: lookup_order
Description: Retrieve order details by order ID or customer order history.
  Returns: order_id, customer_id, order_date, status, items (list), total_amount, shipping_address
  Input: order_id (string) OR customer_id (string, returns last 10 orders)
  Edge case: order_id takes precedence; if order_id provided, customer_id is ignored
  Use this: When customer provides specific order ID or when reviewing account history
  Do NOT use: For customer verification; use get_customer first
  Example queries: "What's my order status?" → use order_id from context; "Show my orders" → use customer_id
```

**Tool 3: `process_refund`**
```
Name: process_refund
Description: Process a refund for a customer order.
  Input: order_id (string), refund_amount (decimal), refund_reason (string), notes (string)
  Returns: refund_id, refund_status (pending/approved/processed), processing_time_hours
  Prerequisites: MUST have verified customer_id from get_customer first
  Authorization: Refunds under $500 auto-approved; >$500 require escalation
  Policy: Full refund within 90 days of order; partial refund up to 180 days
  Use this: After verification, only when customer explicitly requests refund
```

**Tool 4: `escalate_to_human`**
```
Name: escalate_to_human
Description: Escalate case to human support agent with structured context.
  Input: 
    - case_summary (string, required): Concise summary of issue
    - customer_id (string, required): Verified customer ID
    - attempted_resolution (string): What agent tried and why it failed
    - priority (enum: normal/high/urgent): Based on customer urgency, not sentiment
    - context_data (json): Relevant order IDs, amounts, policies referenced
  Returns: escalation_id, queue_position, estimated_wait_time_minutes
  Use this: When policy is ambiguous, customer requests human, or agent hits blocker
  Do NOT use: For straightforward cases within agent authority
```

### Step 1.2: Identify Ambiguity & Clarify Boundaries

For each tool pair, write down how you'd describe them to avoid selection confusion:

**Ambiguity: `get_customer` vs `lookup_order`**
- **Problem:** Both can retrieve information; a new agent might call `lookup_order` for customer info
- **Solution:** 
  - `get_customer` description emphasizes: "Customer identity verification (required before ANY operations)"
  - `lookup_order` description emphasizes: "Order-specific details, order history access"
  - Example queries in descriptions: "How do I verify your account?" → get_customer; "What's my last order?" → lookup_order

**Ambiguity: Similar error cases**
- **Problem:** Both `process_refund` and `escalate_to_human` might be called for policy exceptions
- **Solution:**
  - `process_refund` description: "Automatic refunds within policy ($<500, within 90 days)"
  - `escalate_to_human` description: "Policy exceptions, amounts >$500, or customer requests"

---

## Part 2: Implement the Agentic Loop

### Step 2.1: Agentic Loop with stop_reason Handling

Implement the core loop (pseudocode):

```python
def run_agent_loop(customer_message: str):
    """
    Agentic loop that continues while stop_reason is 'tool_use' 
    and terminates when stop_reason is 'end_turn'
    """
    conversation_history = [
        {"role": "user", "content": customer_message}
    ]
    
    while True:
        response = claude.messages.create(
            model="claude-opus-4-8",
            max_tokens=1024,
            system=SYSTEM_PROMPT,  # See below
            tools=TOOLS,  # See Part 3
            messages=conversation_history
        )
        
        # Check stop reason
        if response.stop_reason == "end_turn":
            # Agent has finished; extract and return final response
            final_text = next(
                (block.text for block in response.content if hasattr(block, 'text')),
                None
            )
            return final_text
        
        elif response.stop_reason == "tool_use":
            # Agent is requesting tools; execute them
            tool_calls = [block for block in response.content if block.type == "tool_use"]
            
            # Add assistant's response to history
            conversation_history.append({
                "role": "assistant",
                "content": response.content
            })
            
            # Execute each tool call
            tool_results = []
            for tool_call in tool_calls:
                result = execute_tool(tool_call.name, tool_call.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_call.id,
                    "content": json.dumps(result)
                })
            
            # Add tool results to history for next iteration
            conversation_history.append({
                "role": "user",
                "content": tool_results
            })
        
        else:
            # Unexpected stop reason
            raise ValueError(f"Unexpected stop_reason: {response.stop_reason}")
```

### Step 2.2: System Prompt for Tool Selection & Sequencing

```
You are a customer support resolution agent. Your goal is to resolve customer issues 
on first contact when possible, or escalate appropriately when needed.

CRITICAL TOOL SEQUENCING RULES:
1. Always call get_customer FIRST, before any order operations
2. Never skip customer verification even if customer volunteers order details
3. Only call process_refund after get_customer has verified customer_id
4. Escalate before attempting unauthorized refunds (>$500, outside policy timeframes)

TOOL SELECTION GUIDANCE:
- get_customer: Customer identity verification, account status checks
- lookup_order: Order details, order history, order-specific information
- process_refund: Process customer-requested refunds within policy
- escalate_to_human: Policy exceptions, amounts >$500, customer requests for human, inability to make progress

MULTI-CONCERN DECOMPOSITION:
If a customer mentions multiple issues (e.g., "I want to return this item AND I was overcharged"), 
treat each as a separate concern. Investigate each independently using shared customer context, 
then synthesize a unified resolution.

ESCALATION CRITERIA:
Escalate when:
- Customer explicitly requests a human agent
- Policy is ambiguous or silent on their specific situation
- Refund exceeds $500 or falls outside standard timeframes (90/180 day windows)
- You cannot make meaningful progress toward resolution

Do NOT escalate:
- Standard refunds within policy and amounts
- Straightforward order status inquiries
- Cases where you have sufficient information to resolve

Format escalation summaries with: customer_id, root_cause, attempted_resolution, recommended_action
```

---

## Part 3: Add Structured Error Responses

### Step 3.1: Define Error Categories

When your tools encounter errors, return structured metadata:

```python
ERROR_CATEGORIES = {
    "transient": {
        "description": "Temporary failure, safe to retry",
        "examples": ["Database timeout", "Service temporarily unavailable"],
        "isRetryable": True
    },
    "validation": {
        "description": "Invalid input, retrying won't help",
        "examples": ["Invalid customer_id format", "Missing required field"],
        "isRetryable": False
    },
    "permission": {
        "description": "Access denied, may require escalation",
        "examples": ["Refund amount exceeds authorization", "Account suspended"],
        "isRetryable": False
    },
    "business": {
        "description": "Business rule violated, requires policy exception",
        "examples": ["Refund outside 90-day window", "Refund >$500"],
        "isRetryable": False
    }
}
```

### Step 3.2: Implement Structured Error Responses

For each tool, implement error handling that returns structured metadata:

```python
def get_customer(customer_id: str = None, email: str = None) -> dict:
    """Return structured response with error category and retryable flag"""
    try:
        if not customer_id and not email:
            return {
                "success": False,
                "isError": True,
                "errorCategory": "validation",
                "errorMessage": "Must provide customer_id OR email",
                "isRetryable": False
            }
        
        customer = db.customers.find_one({"_id": customer_id} or {"email": email})
        if not customer:
            return {
                "success": False,
                "isError": True,
                "errorCategory": "validation",
                "errorMessage": f"Customer not found: {customer_id or email}",
                "isRetryable": False
            }
        
        return {
            "success": True,
            "customer_id": customer["_id"],
            "name": customer["name"],
            "email": customer["email"],
            "account_status": customer["status"],
            "customer_tier": customer["tier"]
        }
    
    except TimeoutError:
        return {
            "success": False,
            "isError": True,
            "errorCategory": "transient",
            "errorMessage": "Database timeout; service temporarily unavailable",
            "isRetryable": True
        }
    
    except Exception as e:
        return {
            "success": False,
            "isError": True,
            "errorCategory": "transient",
            "errorMessage": f"Unexpected error: {str(e)}",
            "isRetryable": True
        }

def process_refund(order_id: str, refund_amount: float, refund_reason: str, notes: str = "") -> dict:
    """Structured refund response with authorization checks"""
    # Validate order exists
    order = db.orders.find_one({"order_id": order_id})
    if not order:
        return {
            "success": False,
            "isError": True,
            "errorCategory": "validation",
            "errorMessage": f"Order not found: {order_id}",
            "isRetryable": False
        }
    
    # Check policy: refund window
    days_since_order = (datetime.now() - order["order_date"]).days
    if days_since_order > 90:
        return {
            "success": False,
            "isError": True,
            "errorCategory": "business",
            "errorMessage": f"Refund outside 90-day window ({days_since_order} days since order)",
            "isRetryable": False,
            "policy_reference": "Standard refund policy: within 90 days"
        }
    
    # Check authorization: amount threshold
    if refund_amount > 500:
        return {
            "success": False,
            "isError": True,
            "errorCategory": "permission",
            "errorMessage": f"Refund amount ${refund_amount} exceeds agent authorization limit ($500)",
            "isRetryable": False,
            "required_action": "Escalate to management for approval"
        }
    
    # Process refund
    refund_id = generate_refund_id()
    db.refunds.insert_one({
        "refund_id": refund_id,
        "order_id": order_id,
        "amount": refund_amount,
        "reason": refund_reason,
        "notes": notes,
        "status": "approved",
        "created_at": datetime.now()
    })
    
    return {
        "success": True,
        "refund_id": refund_id,
        "refund_amount": refund_amount,
        "refund_status": "approved",
        "processing_time_hours": 24
    }
```

### Step 3.3: Agent Responds to Structured Errors

Update the system prompt to guide error recovery:

```
HANDLING TOOL ERRORS:
1. When a tool returns isError=True:
   - Check errorCategory to decide next action
   - If isRetryable=True: Retry with adjusted parameters (e.g., different query format)
   - If isRetryable=False: 
     - For validation errors: Explain to customer, ask for correct info
     - For business errors: Explain policy, escalate if needed
     - For permission errors: Escalate to human

2. Always communicate error implications to customer in natural language
   (Don't repeat raw error JSON)

3. If a tool returns errorCategory="business", escalate with policy context
```

---

## Part 4: Implement Programmatic Prerequisite Enforcement

### Step 4.1: Hook for Tool Call Interception

Implement a PostToolUse hook that blocks certain operations until prerequisites are met:

```python
from anthropic.sdk.types.beta.agent_tool_use_event import AgentToolUseEvent

def prerequisite_enforcement_hook(tool_call: AgentToolUseEvent):
    """
    Intercept tool calls to enforce business rules
    Blocks process_refund and escalate_to_human until get_customer completes
    """
    
    # Track which tools have been called in this session
    if not hasattr(prerequisite_enforcement_hook, "tools_executed"):
        prerequisite_enforcement_hook.tools_executed = set()
    
    tool_name = tool_call.tool_name
    input_params = tool_call.tool_input
    
    # Rule 1: process_refund requires get_customer first
    if tool_name == "process_refund":
        if "get_customer" not in prerequisite_enforcement_hook.tools_executed:
            return {
                "success": False,
                "blocked": True,
                "errorCategory": "permission",
                "errorMessage": "Customer verification required before refund processing. Call get_customer first.",
                "required_action": "Execute get_customer to verify customer_id"
            }
    
    # Rule 2: Refunds >$500 require escalation, not autonomous processing
    if tool_name == "process_refund":
        refund_amount = input_params.get("refund_amount", 0)
        if refund_amount > 500:
            return {
                "success": False,
                "blocked": True,
                "errorCategory": "permission",
                "errorMessage": f"Refund ${refund_amount} exceeds autonomous authorization ($500 limit)",
                "required_action": "Escalate to human for approval"
            }
    
    # Record that this tool was executed
    prerequisite_enforcement_hook.tools_executed.add(tool_name)
    return {"success": True, "allowed": True}
```

### Step 4.2: Integrate Hook into Agent Initialization

```python
from anthropic.sdk.beta.agents import Agent, AgentDefinition

agent = Agent(
    name="customer_support_agent",
    definition=AgentDefinition(
        model="claude-opus-4-8",
        description="Customer support resolution agent with escalation logic",
        system_prompt=SYSTEM_PROMPT,
        tools=TOOLS,
        hooks={
            "tool_call_interception": prerequisite_enforcement_hook
        }
    )
)
```

---

## Part 5: Test Multi-Concern Request Decomposition

### Step 5.1: Test Case 1 — Multiple Independent Issues

**Input:** "Hi, I want to return my order #12345 for a refund, AND I was charged twice for order #12346. Can you help?"

**Expected behavior:**
1. Agent calls get_customer (prerequisite)
2. Agent calls lookup_order for #12345 (return investigation)
3. Agent calls lookup_order for #12346 (duplicate charge investigation)
4. Agent processes refund for #12345 (if within policy)
5. Agent escalates #12346 duplicate charge (out of standard scope) OR explains finding
6. Agent synthesizes unified response addressing both concerns

**Assertions:**
- Both orders investigated independently
- Customer context (verified customer_id) shared across both investigations
- Refund processed for #12345 if policy allows
- Escalation or explanation for #12346

### Step 5.2: Test Case 2 — Policy Exception Required

**Input:** "I ordered this on August 1st and want a refund. I just realized it doesn't work for my use case."

**Expected behavior:**
1. Agent calls get_customer (verify identity)
2. Agent calls lookup_order (find order from August 1st)
3. Agent calculates days since order (>90 days, outside policy)
4. Agent explains policy
5. Agent escalates to human with context (customer_id, order_id, root_cause: out of window)

**Assertions:**
- Escalation includes customer_id and order details
- Escalation reason is clear (policy window exceeded)
- Agent doesn't attempt unauthorized refund

### Step 5.3: Test Case 3 — Authorization Threshold

**Input:** "I need a full refund of $750 for my order. Here's order #99999."

**Expected behavior:**
1. Agent calls get_customer
2. Agent calls lookup_order for #99999
3. Hook blocks process_refund (amount >$500)
4. Agent escalates to human with:
   - customer_id
   - order_id, amount
   - reason: "Authorization threshold exceeded"
5. Agent explains process to customer

**Assertions:**
- Refund not processed autonomously
- Escalation includes all context
- Customer informed of escalation

---

## Part 6: Validation Checklist

### Agentic Loop ✅
- [ ] stop_reason correctly handles "tool_use" (continue) vs "end_turn" (terminate)
- [ ] Tool results appended to conversation history
- [ ] No hardcoded iteration caps; loop terminates on "end_turn" only
- [ ] Agent reasoning flows between iterations (not reset)

### Tool Interface Design ✅
- [ ] Each tool description includes purpose, inputs, outputs, edge cases, boundaries
- [ ] Descriptions clearly differentiate similar tools (get_customer vs lookup_order)
- [ ] Example queries included to guide selection
- [ ] Descriptions state when NOT to use each tool

### Structured Error Responses ✅
- [ ] errorCategory properly classifies (transient/validation/business/permission)
- [ ] isRetryable flag accurately reflects retry viability
- [ ] Error messages are customer-facing and actionable
- [ ] Tool results include both success and error paths

### Programmatic Enforcement ✅
- [ ] Hooks block policy-violating actions (refunds >$500, skipped verification)
- [ ] Hook returns structured context explaining why action was blocked
- [ ] Agent routes blocked actions to escalation workflow
- [ ] No reliance on prompt instructions for financial authorization

### Multi-Concern Handling ✅
- [ ] Agent decomposes multi-issue requests into separate investigations
- [ ] Shared context (customer_id) used across all investigations
- [ ] Each issue investigated in parallel (efficiency)
- [ ] Final response synthesizes findings for all issues

### Escalation Workflows ✅
- [ ] Escalation summaries include: customer_id, root cause, attempted resolution, recommended action
- [ ] Policy exceptions trigger escalation (not autonomous processing)
- [ ] Customer requests for human honored immediately
- [ ] Escalations include all context needed by human agent (no need to review chat history)

---

## 🎯 Success Criteria

Your implementation is complete when:

1. **Agentic loop** runs without hardcoded iteration caps; terminates on "end_turn"
2. **Tool descriptions** are sufficiently detailed that an LLM reliably selects the right tool
3. **Error responses** are structured with error categories and retryable flags
4. **Prerequisite enforcement** hook blocks unauthorized refunds and verification skips
5. **Multi-concern messages** are decomposed, investigated, and synthesized correctly
6. **All three test cases** pass without falling back to prompt-only enforcement

---

## 📚 Related Study Materials

- D1-01: Agentic Loops & Core API Functionality
- D1-03: Hooks, Workflows & Session Management
- D1-05: Agent Delegation & Task Decomposition
- D1-06: Error Handling in Agentic Systems
- D2-01: Tool Description Best Practices
- D2-02: Structured Error Responses
- D5-02: Escalation Patterns & Error Propagation

---

**Prepared:** July 15, 2026  
**Difficulty:** Medium  
**Estimated Time:** 3-4 hours  
**Status:** Ready for implementation
