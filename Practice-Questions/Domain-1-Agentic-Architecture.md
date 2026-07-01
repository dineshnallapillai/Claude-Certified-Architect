# Domain 1: Agentic Architecture & Multi-Agent Systems — Practice Quiz

## Instructions
- 25 scenario-based questions
- Select the BEST answer for each
- Answers with explanations at the bottom
- Time target: 30 minutes

---

## Questions

### Q1
An agentic loop receives `stop_reason: "tool_use"` with three `tool_use` blocks in the response. Your implementation currently processes only the first block. What is the consequence?

A) Claude will re-request the missing tools on the next iteration  
B) The other two tool calls are permanently lost and Claude won't know they failed  
C) The API will return an error on the next call due to missing tool_results  
D) Claude will automatically retry the missing tools internally  

---

### Q2
You're building an agent that searches a knowledge base, reads relevant documents, and synthesizes an answer. The agent averages 8 tool calls per request. After deployment, you notice costs are 3x higher than expected. Which optimization gives the LARGEST cost reduction?

A) Switch from Opus to Haiku for all calls  
B) Enable prompt caching on the system prompt and tool definitions  
C) Reduce max_tokens from 4096 to 1024  
D) Combine search + read + summarize into a single composite tool  

---

### Q3
A production agent occasionally enters infinite loops, making 50+ tool calls without producing a final answer. Which combination of safeguards is MOST appropriate?

A) Set max_tokens very low to force termination  
B) Max iteration count + token budget + consecutive error detection  
C) Add a stop_sequence that triggers after 10 iterations  
D) Use tool_choice: "auto" to let Claude decide when to stop  

---

### Q4
In a hub-and-spoke multi-agent system, the orchestrator sends the full 20-message conversation history to each sub-agent. What is the PRIMARY problem?

A) Sub-agents will respond to messages meant for the orchestrator  
B) Token cost multiplied across all sub-agents, with mostly irrelevant context  
C) Sub-agents may hallucinate responses to earlier user messages  
D) The API will reject requests that exceed context limits  

---

### Q5
An orchestrator delegates tasks to three sub-agents in parallel. Agent A returns in 2 seconds, Agent B in 5 seconds, Agent C times out after 30 seconds. What is the BEST handling?

A) Wait for all three before proceeding  
B) Return results from A and B immediately, retry C with a shorter task  
C) Cancel agents B and C when A returns since it's fastest  
D) Return A and B results, mark C as failed, and proceed with partial synthesis  

---

### Q6
You need to process customer refund requests. The workflow requires: verify purchase, check refund policy, calculate amount, process payment, send confirmation email. Which architecture is MOST appropriate?

A) Parallel fan-out to 5 independent agents  
B) Single agent with all 5 tools in an agentic loop  
C) Pipeline of 5 specialized agents in sequence  
D) Hub-and-spoke with orchestrator delegating each step  

---

### Q7
A sub-agent returns: `{"status": "error", "error_type": "permission", "message": "Cannot access production database"}`. What should the orchestrator do?

A) Retry the same sub-agent with elevated permissions  
B) Propagate the error to the human user with context  
C) Try a different sub-agent that might have access  
D) Modify the sub-agent's tools to include production access  

---

### Q8
What is the PRIMARY difference between the `is_error: true` flag on a tool_result and simply including "error" in the content string?

A) `is_error: true` costs fewer tokens  
B) `is_error: true` explicitly signals failure to Claude's reasoning; content strings may be interpreted as data  
C) `is_error: true` automatically triggers a retry mechanism  
D) There is no functional difference; both are equivalent  

---

### Q9
An agent uses extended thinking (thinking budget: 10000 tokens). In which scenario does this provide the MOST value?

A) Simple file reads and writes  
B) Complex multi-step planning where the agent must reason about trade-offs  
C) Classification tasks with clear categories  
D) Formatting output to match a template  

---

### Q10
Your agentic system needs to handle both simple queries (1-2 tool calls) and complex investigations (15+ tool calls). What is the BEST architectural approach?

A) Always use the complex pipeline for consistency  
B) Progressive disclosure: start simple, escalate complexity only when needed  
C) Route all requests through a planning phase first  
D) Use separate systems for simple and complex tasks  

---

### Q11
A multi-agent system uses Opus for the orchestrator, Sonnet for code agents, and Haiku for validation. The code agent consistently produces low-quality output. What should you try FIRST?

A) Upgrade the code agent to Opus  
B) Add more detailed instructions in the code agent's system prompt  
C) Add few-shot examples to the code agent's prompt  
D) Replace the code agent with a pipeline of smaller steps  

---

### Q12
In a ReAct (Reason + Act) pattern, Claude includes its reasoning in text blocks before making tool calls. What is the PRIMARY benefit?

A) The reasoning text is cheaper than thinking tokens  
B) Explicit reasoning traces improve reliability and debuggability of complex decisions  
C) It reduces the number of tool calls needed  
D) The user can see Claude's thought process in real-time  

---

### Q13
An agent's `stop_reason` is `"max_tokens"` mid-response. The partial response contains a tool_use block with truncated input JSON. What should your system do?

A) Execute the tool with the partial input  
B) Increase max_tokens and retry the call  
C) Discard the response and re-send with higher max_tokens  
D) Append the partial response and ask Claude to continue  

---

### Q14
You're designing error handling for a tool that queries an external API. The API returns: connection timeouts (5%), rate limits (2%), and malformed responses (0.5%). What retry strategy fits BEST?

A) Retry all errors with the same fixed delay  
B) Timeouts: retry with backoff; Rate limits: wait for retry-after; Malformed: don't retry (report error)  
C) Don't retry any — report all errors to Claude for decision  
D) Retry everything up to 10 times with exponential backoff  

---

### Q15
A circuit breaker for a flaky tool has tripped (5 failures in 60 seconds). The agent still needs that tool's functionality. What should happen?

A) Force-reset the circuit breaker and try again  
B) Return a structured error explaining the tool is temporarily unavailable and suggest alternatives  
C) Wait 60 seconds then retry automatically  
D) Terminate the agent session entirely  

---

### Q16
An orchestrator delegates a code review task to a sub-agent. The sub-agent returns a review that contradicts the project's coding standards (documented in CLAUDE.md). What went wrong?

A) The sub-agent hallucinated coding standards  
B) The sub-agent has its own context and didn't receive the project's CLAUDE.md content  
C) The orchestrator should have reviewed the sub-agent's output  
D) The sub-agent's model is too weak for code review  

---

### Q17
What is the MAXIMUM recommended depth of agent delegation (agent calling agent calling agent)?

A) 1 level (orchestrator + sub-agents only)  
B) 2-3 levels maximum  
C) 5 levels for complex tasks  
D) No limit if each level has clear purpose  

---

### Q18
A task decomposition results in 20 tiny sub-tasks, each requiring only 1-2 tool calls. What is the architectural problem?

A) Sub-tasks are too small — coordination overhead exceeds the benefit of delegation  
B) Too many sub-agents will exhaust the API rate limit  
C) The orchestrator can't track 20 concurrent tasks  
D) Claude can only call 10 tools per response  

---

### Q19
An agent encounters the same error 3 consecutive times (trying to write to a read-only file). What should the system do?

A) Retry 2 more times in case the file permission changes  
B) Inject a message: "You've failed 3 times. This file appears read-only. Try a different approach or inform the user."  
C) Silently switch to a backup file path  
D) Terminate the session with an error  

---

### Q20
In a Plan-Execute-Verify pattern, the verification step after step 3 reveals that step 2's output was incorrect. What is the BEST action?

A) Continue with steps 4-5 and fix everything at the end  
B) Revert step 3, fix step 2's output, then re-execute step 3  
C) Start over from step 1  
D) Skip to the final step and see if it works  

---

### Q21
A streaming agentic loop processes events from `messages.stream()`. A `content_block_start` event indicates `type: "tool_use"`. What should your code do at this point?

A) Execute the tool immediately  
B) Wait until the full input JSON is received (content_block_stop) before executing  
C) Display the tool name to the user and prepare for execution  
D) Cancel the stream and switch to non-streaming mode  

---

### Q22
Your agent handles customer data and uses tools that return PII. A sub-agent responsible for writing summaries should NOT see raw PII. How do you enforce this?

A) Add "don't include PII" to the sub-agent's system prompt  
B) Redact PII from data before passing it to the summary sub-agent  
C) Use a less capable model for the summary agent so it can't memorize PII  
D) Add a PostToolUse hook that filters PII from results  

---

### Q23
What is the CORRECT message order when Claude makes a tool call?

A) user → assistant(tool_use) → assistant(tool_result) → user  
B) user → assistant(tool_use) → user(tool_result) → assistant  
C) user → assistant(tool_use) → system(tool_result) → assistant  
D) user → assistant → user(tool_use) → assistant(tool_result)  

---

### Q24
An agentic system costs $50/day. Analysis shows 60% of tokens are spent on the system prompt repeated across iterations. What single change provides the BIGGEST savings?

A) Shorten the system prompt by 50%  
B) Enable prompt caching on the system prompt  
C) Switch to a cheaper model  
D) Reduce the number of iterations per task  

---

### Q25
A hub agent needs to synthesize results from 4 sub-agents. Two returned data, one returned an error, one timed out. What is the BEST synthesis approach?

A) Wait indefinitely for the timed-out agent  
B) Report only the 2 successful results, ignoring failures  
C) Synthesize available results, explicitly note what's missing and why, suggest next steps  
D) Retry both failed agents before synthesizing anything  

---

---

## Answers

### A1: **C**
The API requires a `tool_result` for every `tool_use` block. If you only return 1 result for 3 tool calls, the next API call will fail with a validation error. You must loop through ALL tool_use blocks and return a result for each.

### A2: **B**
Prompt caching saves ~90% on repeated system prompt + tool definitions. In an 8-iteration loop with 8K of system+tools: without caching = 64K tokens at full price; with caching = 8K full + 56K at 10% = ~13.6K effective (79% savings on that portion). Composite tools (D) also help but affect fewer tokens.

### A3: **B**
Multiple safeguards work together: max iterations prevents infinite loops, token budget prevents cost explosion, and consecutive error detection identifies when the agent is stuck (making the same mistake repeatedly). A alone is too blunt; C isn't a real feature; D is already the default but doesn't prevent loops.

### A4: **B**
Each sub-agent receives 20 messages of context it doesn't need, multiplied across all sub-agents. This wastes tokens (cost) and may confuse sub-agents with irrelevant information. The fix: pass only the relevant sub-task context to each sub-agent (selective context forwarding).

### A5: **D**
Return partial results from A and B immediately — don't block the user waiting for C. Mark C as failed (timeout), include in the synthesis that one data source is unavailable, and let the orchestrator decide if it's critical enough to retry or if partial results are sufficient.

### A6: **B**
This is a sequential workflow with data dependencies (verify → check policy → calculate → process → confirm). Each step needs the previous step's output. A single agent with all tools in an agentic loop naturally handles this sequential dependency. Pipeline (C) adds unnecessary overhead for a straightforward sequence. Parallel (A) is wrong since steps depend on each other.

### A7: **B**
Permission errors cannot be resolved by retrying or by the agent itself. The orchestrator should propagate to the human with context: "I need to access the production database to complete this task, but the agent doesn't have permission. Please grant access or provide an alternative."

### A8: **B**
`is_error: true` is a semantic signal that explicitly tells Claude "this tool FAILED." Without it, an error message in the content string might be misinterpreted as successful data about an error (e.g., "The error status of user X is..."). This distinction is critical for Claude's decision-making.

### A9: **B**
Extended thinking provides the most value for complex reasoning tasks where Claude needs to consider multiple options, weigh trade-offs, and plan multi-step approaches. Simple operations (A, C, D) don't benefit from additional reasoning budget.

### A10: **B**
Progressive disclosure: start with a single agent handling the request. If it's simple (1-2 tool calls), it resolves directly. If it turns complex (many steps needed), the system escalates to multi-agent or plan-execute patterns. This avoids the overhead of complex architecture for simple tasks.

### A11: **B**
Before upgrading the model (expensive), try improving the instructions. Low-quality output often means the code agent lacks clear criteria, conventions, or examples in its system prompt. Better prompting is free and often sufficient. Try few-shot examples (C) if instructions alone don't work.

### A12: **B**
Explicit reasoning traces in ReAct patterns make the agent's decision-making visible, which improves debugging, auditability, and reliability. When you can see WHY the agent chose a particular tool, you can identify and fix reasoning errors.

### A13: **C**
A truncated tool_use block has invalid/incomplete input JSON — executing it would fail or produce undefined behavior. The correct approach: discard the truncated response, increase max_tokens, and retry the call so Claude can produce the complete tool request.

### A14: **B**
Different error types require different strategies: timeouts are transient (retry with backoff), rate limits have explicit retry timing (honor retry-after header), and malformed responses indicate a server bug (retrying won't help, report to Claude for decision). One-size-fits-all retry wastes tokens on non-retryable errors.

### A15: **B**
Return a structured error to Claude: `{"error": "circuit_breaker_open", "message": "Tool X temporarily unavailable (5 failures in 60s)", "suggestion": "Try alternative tool Y or inform user this data is temporarily inaccessible"}`. This lets Claude adapt its approach without crashing.

### A16: **B**
Sub-agents have isolated context — they don't automatically inherit the parent's CLAUDE.md or conversation history. The orchestrator must explicitly pass relevant project standards to the sub-agent's prompt. This is a fundamental principle of context isolation in multi-agent systems.

### A17: **B**
2-3 levels maximum. Deeper nesting leads to: context loss (each level loses nuance), debugging difficulty (which level caused the bug?), latency multiplication, and accountability gaps. Most systems work with orchestrator → specialist (1 level) or orchestrator → coordinator → worker (2 levels).

### A18: **A**
Over-decomposition: 20 tasks each requiring 1-2 tool calls means the coordination overhead (delegating, waiting, collecting results) far exceeds the benefit. These should be consolidated into 3-5 meaningful tasks or handled by a single agent directly.

### A19: **B**
After 3 consecutive identical errors, inject guidance that forces a strategy change. The agent is stuck and will keep repeating the same failure. The injected message gives Claude the context to try a different approach (different file, different method, or ask the user).

### A20: **B**
Revert the dependent step (3), fix the root cause (step 2's output), then re-execute step 3 with correct input. Don't continue forward with bad foundations (A) and don't restart everything (C) since step 1 was correct. This is the "verify and correct" principle.

### A21: **B**
Tool input JSON streams incrementally via `input_json_delta` events. You must wait for the complete JSON (signaled by `content_block_stop`) before executing, since partial JSON is invalid and will cause execution errors.

### A22: **B**
Architectural enforcement (redacting PII before passing to the sub-agent) is the only reliable approach. System prompt instructions (A) can be ignored or bypassed. The data the sub-agent never receives cannot be leaked. This is the principle of data minimization.

### A23: **B**
Correct order: user sends message → assistant responds with tool_use → user sends tool_result (it goes in the "user" role) → assistant continues with final response or more tool calls. Tool results are always in the "user" role, matched by tool_use_id.

### A24: **B**
Prompt caching on the system prompt. If 60% of $50/day = $30/day is system prompt repetition, caching at 90% discount saves $27/day. That's a 54% total cost reduction from a single configuration change. Shortening the prompt (A) saves less and loses functionality.

### A25: **C**
The best synthesis acknowledges what's available and what's missing: "Based on results from agents A and B: [synthesis]. Note: Agent C timed out (data source X unavailable) and Agent D encountered a permission error (database Y inaccessible). The synthesis may be incomplete in areas X and Y. Recommended: retry or proceed with partial information."
