# Domain 5: Context Management & Reliability — Practice Quiz

## Instructions
- 25 scenario-based questions
- Select the BEST answer for each
- Answers with explanations at the bottom
- Time target: 30 minutes

---

## Questions

### Q1
After context compaction, an agent starts re-asking questions that were already answered 30 messages ago. What information was lost and where should it have been stored?

A) The information was in tool results that got truncated  
B) Important findings/decisions should have been in CLAUDE.md or plan state (not just message history)  
C) Context compaction never loses information  
D) The agent should have a longer context window  

---

### Q2
An agent reads a 50,000-token file. After this read, Claude's responses become less coherent and miss previously-stated constraints. What is happening?

A) The file content is corrupted  
B) The large tool result displaces other context ("lost in the middle" effect) and may push the conversation past quality degradation thresholds  
C) Claude can't process files that large  
D) The system prompt was overridden  

---

### Q3
A critical security constraint is stated in message #5 of a 100-message conversation. At message #95, Claude violates it. Where should this constraint live?

A) In a more recent message  
B) In the system prompt — it's always in a high-attention position regardless of conversation length  
C) In a pinned message at the top  
D) Repeated every 10 messages  

---

### Q4
An agent claims "the API supports rate limiting" without having used any tool to verify this. What provenance concern does this raise?

A) No concern — Claude's knowledge is reliable  
B) This is an assumption from training data (lowest trust level) being stated as fact; it should be flagged as unverified  
C) This is always correct information  
D) The agent should never mention things it hasn't verified  

---

### Q5
In a multi-agent system, Agent A says "the database has 50,000 users" (from a query 30 minutes ago) and Agent B says "the database has 52,000 users" (from a query just now). Which should the orchestrator trust?

A) Agent A — it was first  
B) Agent B — more recent data has higher trust  
C) Average both: 51,000  
D) Re-query to get current number  

---

### Q6
An agent encounters 3 consecutive validation errors on the same tool with the same approach. According to escalation patterns, what should happen?

A) Retry 3 more times  
B) Change strategy (fundamentally different approach), not just retry  
C) Immediately escalate to human  
D) Disable the tool  

---

### Q7
A sub-agent encounters a permission error (can't access a database). Should it handle this locally or propagate?

A) Handle locally — retry with different credentials  
B) Propagate to orchestrator — permission errors can't be self-resolved; need entity with the right access  
C) Handle locally — skip the database query and use cached data  
D) Terminate the session  

---

### Q8
An orchestrator needs to present results from 4 sub-agents to a human decision-maker. Two agents agree, two disagree. What synthesis format is BEST?

A) Present only the majority opinion  
B) Structured synthesis: points of agreement, core disagreement, evidence from each side, confidence levels, and a specific decision question  
C) Raw dump of all four agent outputs  
D) Let the orchestrator pick the best answer  

---

### Q9
Context compaction is triggered at what percentage of context window usage?

A) 100% (when full)  
B) 50% (early compaction)  
C) 75-80% (proactive, before degradation)  
D) It's never triggered automatically  

---

### Q10
A system prompt is 5,000 tokens. With prompt caching in a 20-iteration loop, what is the approximate effective token cost for the system prompt?

A) 100,000 tokens (5K × 20)  
B) 5,000 tokens (loaded once)  
C) ~14,500 tokens (5K full + 19 × 5K × 0.1)  
D) 25,000 tokens (5K × 5 full + 15 cached)  

---

### Q11
An agent reads a config file, makes 15 other tool calls, then modifies the config file based on its earlier reading. What reliability risk exists?

A) None — file reads are reliable  
B) Stale data risk — the file may have changed in the 15 calls since reading; the agent should re-read before modifying  
C) The agent shouldn't modify config files  
D) 15 tool calls always exceeds rate limits  

---

### Q12
A human-in-the-loop checkpoint for a deployment is set with a 30-minute timeout. The reviewer hasn't responded. What is the SAFE default?

A) Proceed with deployment (assume approval)  
B) Deny/wait — safe default for irreversible actions is to NOT proceed without explicit approval  
C) Escalate to a different reviewer  
D) Send another notification and reset the timer  

---

### Q13
"Lost in the middle" effect: where does Claude attend LEAST in a long conversation?

A) The beginning (first few messages)  
B) The middle of the conversation  
C) The end (most recent messages)  
D) The system prompt  

---

### Q14
An agent has made 20 tool calls. Token usage: 60% system prompt (repeated), 25% tool results, 15% Claude responses. What single optimization saves the most?

A) Shorter Claude responses  
B) Prompt caching on system prompt (reduces 60% portion by 90%)  
C) Smaller tool results  
D) Fewer tool calls  

---

### Q15
Agent A found information via `grep` (direct tool call). Agent B inferred information from a code comment. The findings conflict. Which has higher provenance trust?

A) Agent B — comments are written by developers  
B) Agent A — direct observation (tool call) has higher trust than inference from secondary sources  
C) Equal trust  
D) Neither — both should be re-verified  

---

### Q16
A circuit breaker for a tool trips after 5 failures in 60 seconds. The agent needs an alternative. What is the correct architecture?

A) Force-reset the circuit breaker  
B) Return structured error to Claude explaining unavailability + suggest alternative tools or approaches  
C) Wait 60 seconds silently  
D) Terminate the agent  

---

### Q17
An agent needs to make a security-sensitive decision: whether to grant elevated database access to another agent. Who should make this decision?

A) The agent — it understands the context  
B) The orchestrator — it has broader visibility  
C) A human — security decisions should always be escalated  
D) The sub-agent that needs access  

---

### Q18
Progressive summarization preserves "key decisions and their reasons" but loses "verbatim tool outputs." An agent later needs the exact output of a tool from 30 messages ago. What should it do?

A) Guess based on the summary  
B) Re-run the tool to get fresh, accurate data  
C) Tell the user the information is lost  
D) Search the summary for approximate data  

---

### Q19
A batch of 1000 API calls uses the Messages Batches API. The result shows 950 succeeded and 50 errored. What is the correct handling?

A) Discard the batch and redo all 1000  
B) Process 950 successful results normally; create a new retry batch for 50 failed items  
C) Ignore the 50 failures  
D) Mark the entire batch as failed  

---

### Q20
An agent's response quality is excellent for the first 10 iterations but degrades around iteration 15. What architectural changes help MOST?

A) Use a bigger model  
B) Proactive compaction at iteration 10, summarize tool results immediately, budget context allocation  
C) Set max iterations to 10  
D) Switch to a different model after iteration 10  

---

### Q21
A system processes both critical requests (security alerts) and routine requests (info queries). A human reviewer handles exceptions. How should the queue be organized?

A) FIFO (first in, first out)  
B) Priority queue: critical items surface immediately, routine items queue normally  
C) Random order for fairness  
D) Batch all exceptions and review daily  

---

### Q22
An orchestrator's sub-agent times out. The orchestrator has partial results from 3 other agents. The timed-out agent's task was: "verify there are no security vulnerabilities." Should the orchestrator proceed?

A) Yes — 3 out of 4 results is sufficient  
B) No — the missing result is security-critical; cannot proceed without it; retry or escalate  
C) Yes — just note that security wasn't checked  
D) Proceed but add a disclaimer  

---

### Q23
Token optimization: a tool returns 20,000 tokens of log data. The agent only needs error lines. What is the MOST efficient approach?

A) Return all 20K tokens and let Claude find the errors  
B) Server-side: filter log data to only error lines BEFORE returning to Claude  
C) Truncate to first 2000 tokens  
D) Return a count of errors only  

---

### Q24
Information provenance: Claude states "Based on the file I read earlier, the function returns null on error." The file was read 25 minutes ago in a fast-moving codebase. How reliable is this statement?

A) Highly reliable — file contents don't change that fast  
B) Moderate reliability — in active development, files can change; if the agent acts on this (modifying the function), it should re-read first  
C) Unreliable — always re-read before any statement  
D) The reliability doesn't matter  

---

### Q25
An escalation message says: "Error occurred." No additional context is provided. Why is this problematic for the human recipient?

A) It's fine — the human can investigate  
B) Insufficient context: the human needs to know what happened, what was tried, what's needed, and current state to effectively resolve the issue  
C) Error messages should never reach humans  
D) The human should ask the agent for more details  

---

---

## Answers

### A1: **B**
Context compaction summarizes old messages — specific Q&A details can be lost. Important findings, decisions, and answers should be stored in persistent locations: CLAUDE.md, plan state, memory files, or explicit state objects. These survive compaction; message history does not.

### A2: **B**
A 50,000-token tool result displaces other context (earlier messages, constraints, instructions get pushed into the "middle" of the context where attention is lowest). Plus, approaching the context limit triggers degradation. Fix: truncate/summarize large results to relevant portions.

### A3: **B**
The system prompt occupies a privileged position in Claude's architecture — it always receives high attention regardless of conversation length. Critical security constraints should NEVER be only in message history (which can be compacted, buried in middle, or forgotten). System prompt is the authoritative location.

### A4: **B**
Claude's training data knowledge is the LOWEST trust level in the provenance hierarchy. It may be outdated, incorrect, or hallucinated. The agent should either: (1) verify by checking actual config/docs, or (2) clearly label it: "I believe the API supports rate limiting, but I haven't verified this — let me check."

### A5: **B**
In provenance-based trust, more recent data from the same source type has higher reliability. Agent B's query is 30 minutes more current. In a live system, 2,000 users could have been added in 30 minutes. If a critical decision depends on this, option D (re-query for absolute freshness) is even better.

### A6: **B**
After 3 consecutive identical errors, retrying the same approach won't succeed. The escalation pattern says: change strategy fundamentally. "You've tried X three times and it failed. Try a completely different approach." Direct escalation to human (C) is premature — first try an alternate strategy.

### A7: **B**
Permission errors CANNOT be self-resolved — the agent doesn't have the authority to grant itself access. Propagate to orchestrator (which may have broader permissions or can request access) or ultimately to a human who can authorize the access.

### A8: **B**
Effective synthesis for human decision-making requires structure: where agents agree (shared confidence), where they disagree (the actual decision point), evidence from each side (for human judgment), and a specific question ("Should we do X or Y?"). Raw data dumps (C) and hidden disagreements (A) don't serve the human.

### A9: **C**
75-80% is the recommended threshold — proactive compaction BEFORE degradation hits. At 100% (A), it's too late (hard failure or severe quality loss). At 50% (B), it's premature (losing information unnecessarily). 75-80% balances information preservation with quality maintenance.

### A10: **C**
With caching: first call = 5,000 tokens at full price (+ 25% write = 6,250 effective). Remaining 19 calls: 5,000 × 0.1 × 19 = 9,500 effective tokens. Total: ~15,750 effective vs. 100,000 without caching. Approximately 85% savings. Closest answer: C (~14,500).

### A11: **B**
Stale data risk: the file may have been modified by another user, another agent, or CI process during the 15 intervening tool calls. Before modifying any file based on an earlier reading, the agent should re-read it to verify the current state matches expectations (optimistic locking principle).

### A12: **B**
For irreversible actions (deployment), the safe default on timeout is to NOT proceed. "Assume denial" prevents accidental deployment when the reviewer is unavailable. The alternative (C: escalate to different reviewer) is acceptable as a secondary action after the initial timeout.

### A13: **B**
The "lost in the middle" phenomenon: LLMs attend most to the beginning (primacy) and end (recency) of their context. Information in the middle receives the least attention. This is why important constraints should be in the system prompt (always high attention) or recent messages (recency).

### A14: **B**
System prompt is 60% of cost and repeats every call. Prompt caching reduces this by 90% on subsequent calls: 60% × 90% savings = 54% total cost reduction from a single configuration change. This dwarfs savings from other optimizations.

### A15: **B**
Direct observation (grep tool call = directly examining the code) has higher provenance trust than inference from a code comment (which may be outdated, incorrect, or misleading). Comments are secondary sources; tool results are primary sources.

### A16: **B**
When a circuit breaker trips, return a structured error to Claude: `{"error": "circuit_breaker_open", "tool": "database_query", "message": "Temporarily unavailable", "suggestion": "Try cache_lookup tool or inform user data is temporarily unavailable"}`. This lets Claude adapt.

### A17: **C**
Security decisions should ALWAYS be escalated to a human. Granting elevated database access is a security-sensitive action with potential for data breach. No agent should autonomously make security access decisions — the blast radius of a mistake is too high.

### A18: **B**
If verbatim tool output was lost in compaction but the agent needs the exact data, the correct action is to re-run the tool. Tool calls are re-executable and will return current (possibly more accurate) data. This is why compaction works — derivable information can be re-obtained.

### A19: **B**
Process the 950 successes (don't block them waiting for the 50). Create a retry batch for the 50 failures. If the retry also fails for some items, log them to a dead letter queue for investigation. Never discard successful results because of partial failures.

### A20: **B**
Quality degradation at iteration 15 indicates context approaching degradation thresholds. Fix: compact proactively at iteration 10 (before degradation), summarize tool results immediately (prevent context bloat), and budget context (allocate fixed space per category). These keep quality stable through long sessions.

### A21: **B**
Priority queue: security alerts need immediate human attention (lives/data at risk). Routine queries can wait. FIFO would make a critical security alert wait behind 50 routine questions — unacceptable. Prioritization ensures the most impactful items get reviewed first.

### A22: **B**
The missing result is specifically about SECURITY VULNERABILITIES — this is safety-critical information. Proceeding without it means potentially deploying vulnerable code. The correct action: retry the security check agent (with timeout), or escalate to a human security reviewer. Never skip security checks.

### A23: **B**
Server-side filtering before returning to Claude is the most token-efficient: only error lines reach the context (maybe 200 tokens instead of 20,000). This preserves context space for other information and reduces cost by 99%. Never return data the agent doesn't need.

### A24: **B**
25 minutes in an active codebase means the file may have been modified (commits, rebases, other developers). The information has moderate reliability — good enough for discussion but NOT good enough for acting on (especially modification). Re-read before modifying; acceptable for informational statements with a freshness caveat.

### A25: **B**
"Error occurred" gives the human NOTHING to work with — no context about what system, what operation, what was tried, what the current state is, or what's needed. Effective escalation requires: what happened + what was tried + what's needed + current state. Without this, the human must investigate from scratch, wasting time.
