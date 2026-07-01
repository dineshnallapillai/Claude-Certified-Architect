# Claude Certified Architect — Full Mock Exam #1

## Instructions
- 50 questions covering ALL 5 domains
- Scenario-based, select the BEST answer
- Passing score: 80% (40/50)
- Time limit: 60 minutes
- Answers at the bottom

---

## Questions

### Q1 [D1]
An agentic loop processes a user request. On iteration 7, `stop_reason` is `"end_turn"`. What should your code do?

A) Make another API call to confirm completion  
B) Exit the loop and return Claude's response to the user  
C) Check if all planned tasks were completed  
D) Summarize the conversation and start a new loop  

---

### Q2 [D2]
A tool description reads: "Database tool." Claude uses it for queries, inserts, and deletes — even when separate specialized tools exist. What fix has the HIGHEST impact?

A) Add rate limiting  
B) Rewrite the description with clear WHEN/WHEN-NOT triggers and explicit scope  
C) Remove the tool entirely  
D) Rename it to `generic_database_tool`  

---

### Q3 [D3]
Project settings deny `Bash(rm *)`. A developer's local settings allow `Bash(rm temp/*)`. The developer asks Claude to `rm temp/cache.txt`. What happens?

A) Blocked by project deny  
B) Allowed by local settings (higher priority overrides project deny)  
C) User is prompted to approve  
D) Error due to conflicting rules  

---

### Q4 [D4]
Claude generates a JSON response but includes trailing commas (invalid JSON). A validation-retry loop catches this. What should the retry feedback say?

A) "Try again"  
B) "Invalid JSON: trailing comma at position 245. JSON does not allow trailing commas. Remove the comma after the last item in arrays and objects."  
C) "Your response was wrong"  
D) "Generate valid output"  

---

### Q5 [D5]
An agent reads 30 files (average 3000 tokens each). Context is now 90,000 tokens of file content plus 10,000 for system/tools. Response quality is degrading. What is the PRIMARY cause?

A) Too many files for Claude to handle  
B) Context approaching degradation thresholds; earlier file contents in the "middle" receive less attention  
C) The model has a bug with large contexts  
D) File content is corrupting Claude's reasoning  

---

### Q6 [D1]
A hub-and-spoke system has an orchestrator (Opus), 3 code agents (Sonnet), and 2 validation agents (Haiku). A code agent needs help with a complex architectural decision. What should happen?

A) Code agent handles it alone  
B) Code agent escalates to orchestrator, which may handle it with its Opus reasoning or delegate to a specialized planning agent  
C) Code agent calls another code agent  
D) Validation agent makes the decision  

---

### Q7 [D2]
MCP server configuration in `.claude/settings.json`:
```json
{"mcpServers": {"db": {"env": {"PASSWORD": "hunter2"}}}}
```
This file is committed to git. What is the vulnerability and fix?

A) The server name "db" is too generic  
B) The password is in version control (exposed forever in git history); use `"PASSWORD": "${DB_PASSWORD}"` to reference environment variables  
C) The env section should be in a separate file  
D) MCP servers shouldn't have passwords  

---

### Q8 [D3]
A user types `/project:review`. The file `.claude/commands/review.md` contains review criteria. What role does this play compared to CLAUDE.md?

A) Same role — both are always active  
B) The slash command is ON-DEMAND (only when invoked); CLAUDE.md is ALWAYS ACTIVE  
C) Slash commands override CLAUDE.md  
D) CLAUDE.md is only active during reviews  

---

### Q9 [D4]
You want Claude to ALWAYS produce structured output (never conversational text). What API configuration achieves this with highest reliability?

A) System prompt: "Always respond with JSON"  
B) `tool_choice: {"type": "tool", "name": "structured_output"}` with a tool whose input_schema matches desired structure  
C) `tool_choice: {"type": "any"}`  
D) Validation-retry loop with JSON schema validation  

---

### Q10 [D5]
A system prompt is 4,000 tokens. With prompt caching in a 25-iteration agentic loop, approximately how much do you save vs. no caching?

A) ~25%  
B) ~50%  
C) ~85%  
D) ~95%  

---

### Q11 [D1]
An agent calls a tool that doesn't exist (`hallucinated_tool`). What should your system return?

A) An empty successful result  
B) A tool_result with `is_error: true` listing available tools so Claude can self-correct  
C) Terminate the conversation  
D) Ignore it and make the next API call  

---

### Q12 [D2]
A JSON Schema property has `"enum": ["draft", "published", "archived"]`. Claude always provides one of these values. If you used `"type": "string"` without enum, Claude might invent values like "active" or "live." What principle does this demonstrate?

A) Enums are faster to process  
B) Constrained schemas prevent hallucination by limiting the valid option space  
C) Free-text fields are a security risk  
D) Enums save tokens  

---

### Q13 [D3]
Which is the STRONGEST enforcement mechanism in Claude Code (hardest to bypass)?

A) CLAUDE.md instruction  
B) Settings deny list  
C) PreToolUse hook with non-zero exit  
D) System prompt constraint  

---

### Q14 [D4]
Few-shot examples in the system prompt add 3,000 tokens. Over a 20-iteration agentic loop, this costs 60,000 extra input tokens without caching. With prompt caching enabled, what's the effective cost?

A) 60,000 tokens (same — caching doesn't work on examples)  
B) ~8,700 tokens (3,000 full + 19 × 300 cached at 10%)  
C) 3,000 tokens (loaded once only)  
D) 30,000 tokens (half price)  

---

### Q15 [D5]
An orchestrator's sub-agent returns conflicting data: it says both "the service is healthy" and "error rate is 45%." How should this contradiction be handled?

A) Trust "service is healthy" because it's stated first  
B) Flag the contradiction, investigate which claim has better provenance (direct health check vs. metric query), and present both to human if unresolvable  
C) Average the two claims  
D) Retry the sub-agent  

---

### Q16 [D1]
An agentic loop has been running for 50 iterations and spent $5 in tokens. The agent is still working. What safeguard should have prevented this?

A) Higher quality model that works faster  
B) Token budget limit that triggers escalation or termination when exceeded  
C) Better tools that need fewer calls  
D) Smaller system prompt  

---

### Q17 [D2]
Tool A: "Search users by name or email"  
Tool B: "Search orders by customer"  
Claude uses Tool A when the user asks about order history. Why?

A) Tool A is listed first  
B) Tool A's description mentions searching and users — if the user asked about "their orders," Claude connects "user" to "their"  
C) Tool B's description doesn't include "history"  
D) Random tool selection  

---

### Q18 [D3]
A CLAUDE.md file is 500 tokens. Is this a problem?

A) Yes — it's too short to be useful  
B) No — under 1,500 tokens is the ideal range for a root CLAUDE.md  
C) Yes — it won't be cached (below 1024 minimum)  
D) No — but it should be converted to a settings file  

---

### Q19 [D4]
A batch of 100,000 items is submitted via the Messages Batches API. How long until results are available?

A) Immediately  
B) Within 24 hours  
C) Within 1 hour  
D) Within 1 week  

---

### Q20 [D5]
An agent makes a security-sensitive decision (deleting user data) without escalating to a human. According to escalation patterns, this is...

A) Acceptable if the agent is confident  
B) A violation — security-sensitive/irreversible actions should ALWAYS be escalated  
C) Acceptable if the user originally requested it  
D) Only wrong if the agent makes an error  

---

### Q21 [D1]
What is the PRIMARY benefit of using extended thinking (thinking budget) in an agentic system?

A) Reduces token count  
B) Improves reasoning quality for complex decisions (trade-off analysis, multi-step planning)  
C) Makes responses faster  
D) Eliminates the need for tool calls  

---

### Q22 [D2]
A tool result is 30,000 tokens (a large database query output). In the next 10 iterations of the agentic loop, this result is included in context. Total token cost from this one result:

A) 30,000 tokens  
B) 300,000 tokens (30K × 10 iterations)  
C) 60,000 tokens (30K × 2 for input and output)  
D) It's free after the first inclusion  

---

### Q23 [D3]
Plan mode produces a 10-step plan. After executing step 5, the user says "actually, I want to change the approach for steps 7-10." What should Claude do?

A) Complete the original plan, then address changes  
B) Stop execution, revise steps 7-10 based on user feedback, present updated plan for approval  
C) Undo steps 1-5 and start over  
D) Ignore the user and complete the plan  

---

### Q24 [D4]
A system prompt says "You are a code review agent." The user sends a message in French. Claude should:

A) Refuse because it's a "code review agent"  
B) Respond in French about code review topics — the role constrains the domain, not the language  
C) Respond only in English  
D) Ask the user to switch to English  

---

### Q25 [D5]
An agent needs to decide between two architectures. It has read 5 documents, queried 2 databases, and spoken with 3 sub-agents. Before presenting its recommendation, it should:

A) Just give the recommendation  
B) Attribute each supporting fact to its source (provenance), state confidence level, and flag any assumptions  
C) Present all raw data and let the human decide  
D) Only present the conclusion, not the sources  

---

### Q26 [D1]
A tool call fails with a rate limit (429) error. The error includes `retry_after: 5`. What should your agentic loop do?

A) Return the error to Claude immediately  
B) Wait 5 seconds, then automatically retry the same tool call without involving Claude  
C) Terminate the loop  
D) Ask Claude what to do  

---

### Q27 [D2]
A tool has `"required": ["query"]` and an optional `"limit"` with `"default": 10`. Claude calls the tool without providing "limit". What should your server do?

A) Reject the call (missing parameter)  
B) Use the default value (limit=10)  
C) Return all results (no limit)  
D) Ask Claude to provide limit  

---

### Q28 [D3]
A team wants Claude to auto-format all written files. What is the correct hook configuration?

A) PreToolUse hook on Write that formats first  
B) PostToolUse hook on Write and Edit that runs the formatter on `$FILE_PATH`  
C) Stop hook that formats all changed files  
D) Add "always format code" to CLAUDE.md  

---

### Q29 [D4]
Claude is generating code and you want to validate it passes TypeScript type checking. Where does `tsc --noEmit` fit in a validation-retry loop?

A) Before generation (pre-validation)  
B) After generation, as a validation step — if types fail, feed errors back for Claude to fix  
C) Only after all retries are exhausted  
D) In a PreToolUse hook  

---

### Q30 [D5]
What information is GUARANTEED to survive context compaction?

A) All message history  
B) System prompt, tool definitions, CLAUDE.md content  
C) Everything — compaction is lossless  
D) Only the most recent 5 messages  

---

### Q31 [D1]
Two approaches to building an agent: (A) Give it 30 tools and let it figure things out, (B) Give it 8 focused tools with great descriptions. Which is generally better?

A) A — more tools = more capable  
B) B — fewer, well-described tools = better selection accuracy and lower token cost  
C) They perform equally  
D) It depends entirely on the model  

---

### Q32 [D2]
An MCP server uses stdio transport. It's deployed on a remote machine. Can Claude Code connect to it?

A) Yes — stdio works over any connection  
B) No — stdio requires the server to be a local child process; remote servers need Streamable HTTP transport  
C) Yes — with SSH tunneling  
D) Only if Docker is used  

---

### Q33 [D3]
A developer adds `"allow": ["Write"]` (all writes, unrestricted) to their local settings. They later accidentally tell Claude to overwrite their `.env` file. What happened?

A) The deny list should have caught this  
B) Unrestricted `Write` permission has no path scoping — Claude could write anywhere. Should have used `"Write(src/**)"` with explicit path constraints  
C) `.env` files are always protected  
D) Claude wouldn't overwrite .env by default  

---

### Q34 [D4]
A system processes both "quick questions" (1 turn) and "deep analysis" (10+ turns). Should both use the same model?

A) Yes — consistency matters  
B) No — use Haiku for quick questions (fast + cheap) and Sonnet/Opus for deep analysis (quality matters more)  
C) Always use Opus for best quality  
D) Use the same model but vary max_tokens  

---

### Q35 [D5]
A human reviewer approves all agent actions without reading them (approval fatigue). This defeats the purpose of human-in-the-loop. What architectural fix helps?

A) Require longer approval messages  
B) Reduce approval frequency (only truly critical/risky items), make the approval presentation concise with key risks highlighted  
C) Remove human review entirely  
D) Add a second reviewer  

---

### Q36 [D1]
An agent enters a loop: call tool → get error → call same tool with same params → get same error. This repeats 8 times. What TWO safeguards are missing?

A) A longer timeout and bigger context  
B) Consecutive error detection (3 same errors = change strategy) AND a max iteration limit  
C) Better error messages from the tool  
D) A more capable model  

---

### Q37 [D2]
Tool description: "Creates a user and sends welcome email." The implementation also logs the creation to an audit system. Should the description mention the audit log?

A) No — it's an internal implementation detail  
B) Yes — the description should document ALL observable effects so Claude can inform users accurately  
C) Only if the user asks about auditing  
D) Only in the tool's code comments  

---

### Q38 [D3]
A project CLAUDE.md instructs: "Use yarn, not npm." The package-lock.json in the repo indicates npm was used historically. What should Claude do?

A) Follow CLAUDE.md (use yarn)  
B) Use npm since the project evidence shows npm  
C) Ask the user for clarification given the conflict  
D) Use whichever is faster  

---

### Q39 [D4]
A system prompt is: "You are a helpful assistant." Claude gives inconsistent results across sessions. What is the root cause?

A) Model non-determinism  
B) The system prompt provides no specific behavioral constraints, output format, or criteria — Claude fills the gaps with its own judgment each time  
C) Temperature is too high  
D) The context window is too small  

---

### Q40 [D5]
Token optimization priority order for an agentic system (biggest impact first)?

A) Better prompts → fewer iterations → smaller results  
B) Prompt caching (biggest: reduces repeated system content by 90%) → tool result truncation → model selection for sub-tasks  
C) Cheaper model → fewer tools → shorter prompts  
D) All optimizations are equally important  

---

### Q41 [D1]
A Plan-Execute-Verify agent verifies step 3 and finds the result is incorrect. Steps 1 and 2 were correct. What is the correct action?

A) Redo everything from step 1  
B) Revert step 3, re-execute it (possibly with adjusted approach), verify again  
C) Skip step 3 and proceed to step 4  
D) Report failure and stop  

---

### Q42 [D2]
A tool's input_schema has `"maxItems": 100` on an array field. Claude provides 150 items. What should your server do?

A) Accept all 150  
B) Validate against schema, return structured error: "validation failed: array exceeds maxItems (150 > 100)", with `is_error: true`  
C) Silently truncate to 100  
D) Increase the limit  

---

### Q43 [D3]
Settings hierarchy question: `.claude/settings.json` sets `"env": {"DEBUG": "false"}` and `~/.claude/settings.json` sets `"env": {"DEBUG": "true"}`. What value does DEBUG have?

A) "true" (global wins)  
B) "false" (project wins — higher priority than global)  
C) Error — conflicting values  
D) Undefined — they cancel out  

---

### Q44 [D4]
A prompt uses XML tags for clear structure: `<examples>`, `<task>`, `<constraints>`. What is the benefit over markdown headers?

A) XML is more machine-readable  
B) Clear semantic boundaries help Claude parse sections reliably; reduces cross-contamination between prompt sections  
C) XML uses fewer tokens  
D) Markdown isn't supported  

---

### Q45 [D5]
An agent has been working for 15 minutes. Its findings from minute 1 were based on a file read. That file has since been modified by another developer. What provenance principle applies?

A) Findings are always current  
B) Information freshness — 15-minute-old file reads may be stale in active repos; re-verify before acting on them  
C) Only re-read if the user asks  
D) The file system caches prevent this issue  

---

### Q46 [D1]
What does `stop_reason: "tool_use"` tell your application?

A) Claude finished and used tools earlier  
B) Claude wants to use one or more tools — execute them and return results so the loop continues  
C) A tool call failed  
D) Claude is waiting for user input  

---

### Q47 [D2]
A tool for sending emails has no rate limit. An agent in a loop sends 500 emails due to a bug. What safeguard should have prevented this?

A) Better system prompt instructions  
B) Rate limiting on the tool (e.g., max 5 emails per session) + confirmation gate for bulk actions  
C) A less capable model  
D) Removing email from the tool set  

---

### Q48 [D3]
A PostToolUse hook's command fails (exits non-zero). Does this block the tool execution?

A) Yes — like PreToolUse, it blocks  
B) No — PostToolUse runs AFTER the tool has already executed; it can't undo it  
C) Yes — it reverts the tool's changes  
D) It depends on the exit code  

---

### Q49 [D4]
Batch processing: 50,000 items classified using Haiku via Batches API (50% discount). Approximate cost at $0.80/M input, $4/M output, 200 input + 50 output tokens per item?

A) ~$40  
B) ~$9  
C) ~$4.50 (batch discount applied to both input and output)  
D) ~$1  

---

### Q50 [D5]
An agent is stuck in a loop of escalating retries. Error budget: 10 errors max. It's at 9 errors. What should happen on the 10th error?

A) Keep trying (11th time's the charm)  
B) Exhaust the budget → hard stop → escalate to human with full error chain context  
C) Reset the error counter  
D) Switch to a different model  

---

---

## Answers

### A1: **B** [D1]
`stop_reason: "end_turn"` means Claude has produced its final response — no more tool calls needed. Exit the loop and return the response to the user.

### A2: **B** [D2]
"Database tool" has no trigger conditions, no scope, no disambiguation. Adding "Use ONLY for X. Do NOT use for Y (use Z instead). Returns: {format}" gives Claude the information needed for correct selection.

### A3: **A** [D3]
Deny rules are CUMULATIVE and ABSOLUTE — a deny at ANY level blocks the action regardless of allow rules at other levels. The project deny on `Bash(rm *)` blocks all `rm` commands, and no local allow can override it. This is a security-first design. Critical safety enforcement via hooks provides additional defense-in-depth.

### A4: **B** [D4]
Specific, actionable error feedback enables Claude to fix the exact issue. It includes: what's wrong (trailing comma), where (position 245), why it's wrong (JSON spec), and how to fix it (remove comma). Vague feedback leads to more retries.

### A5: **B** [D5]
90K tokens of file content approaches degradation zones. Early file contents are in the "middle" (lowest attention). Fix: extract only relevant portions from files, summarize completed analyses, and compact proactively.

### A6: **B** [D1]
The code agent has limited context and may not handle complex architectural reasoning well. Escalating to the Opus orchestrator (which has broader context and better reasoning) is appropriate for decisions beyond the sub-agent's scope.

### A7: **B** [D2]
Credentials in git history are exposed permanently (even after removal, they exist in commit history). Environment variable references ensure the settings file is safe to commit while secrets stay in each developer's environment.

### A8: **B** [D3]
Key distinction: CLAUDE.md = always active (every conversation, every action). Slash commands = explicitly triggered by user typing the command. Use CLAUDE.md for persistent rules; commands for on-demand workflows.

### A9: **B** [D4]
Forced tool use with a matching schema is the most reliable approach for structured output. The tool's input becomes the structured data — no JSON parsing, no formatting issues, schema-validated by design.

### A10: **C** [D5]
Without caching: 4,000 × 25 = 100,000 tokens. With caching: 4,000 (first) + 24 × 400 (10% of 4,000) = 13,600. Savings: ~86%. Closest answer: C (~85%).

### A11: **B** [D1]
Return `is_error: true` with available tool names. This lets Claude self-correct (it likely misspelled or hallucinated the name) without crashing the session.

### A12: **B** [D2]
Enums constrain the output space. Without enum, Claude might generate any string. With enum, it's forced to pick from valid options — eliminating hallucinated values for fields with finite, known options.

### A13: **C** [D3]
PreToolUse hooks execute external code; non-zero exit physically blocks tool execution. CLAUDE.md (A) is interpretive (soft). Settings deny (B) is strong and cumulative (cannot be overridden), but hooks offer programmatic logic and dynamic validation that static deny rules cannot. System prompt (D) is strong but not code-enforced.

### A14: **B** [D4]
With prompt caching: first call = 3,000 × 1.25 (write premium) ≈ 3,750. Next 19 calls: 3,000 × 0.1 × 19 = 5,700. Total: ~9,450 effective. Closest simplified: B (3,000 full + 19×300 = 8,700). vs 60,000 without caching.

### A15: **B** [D5]
Contradictions should be flagged, not resolved by picking one. Check provenance: a direct health check endpoint vs. a metrics aggregation — which is more authoritative for this specific question? If unresolvable with available data, present both to a human.

### A16: **B** [D1]
A token budget (and/or cost budget) provides a hard ceiling. When exceeded: stop the agent, return partial results, and escalate. This prevents runaway costs from unbounded or stuck agents.

### A17: **B** [D2]
Claude associates "user" in Tool A's description with the user asking about "their" orders. Tool B says "customer" not "user." The description language should match how users phrase requests. Fix: Tool B should say "Search for a user's order history."

### A18: **B** [D3]
500 tokens for a root CLAUDE.md is well within the ideal range (< 1,500 tokens). It's concise, loads quickly, and doesn't bloat every API call. The 1024 minimum is for prompt caching, not for CLAUDE.md itself.

### A19: **B** [D4]
The Messages Batches API processes requests within 24 hours at 50% cost reduction. This is the stated SLA regardless of batch size (up to 100,000 items per batch).

### A20: **B** [D5]
Security-sensitive and irreversible actions should ALWAYS be escalated to a human, regardless of agent confidence. Deleting user data has legal, ethical, and compliance implications that require human judgment.

### A21: **B** [D1]
Extended thinking provides a reasoning budget for complex decisions — weighing options, planning multi-step approaches, considering trade-offs. Simple operations don't benefit; complex architectural or strategic decisions do.

### A22: **B** [D2]
In agentic loops, all message history (including tool results) is sent with every subsequent API call. A 30K token result × 10 iterations = 300,000 input tokens charged. This is why tool result truncation/summarization is critical.

### A23: **B** [D3]
Plan mode allows mid-execution adjustments. Stop at step 5 (current execution point), revise steps 7-10 per user feedback, present updated plan for approval, then continue. The plan is a living document during execution.

### A24: **B** [D4]
The role "code review agent" constrains the domain (only code-related topics) but not the language of interaction. Claude should help with code review in whatever language the user communicates in.

### A25: **B** [D5]
Provenance-aware recommendations attribute each fact to its source, state confidence levels, and explicitly flag assumptions. This lets the human decision-maker evaluate the reliability of each supporting point.

### A26: **B** [D1]
Rate limits are transient errors with explicit retry timing. The system should handle this automatically (wait the specified time, retry) without involving Claude in the decision. It's infrastructure-level handling.

### A27: **B** [D2]
Schema defaults should be applied server-side when optional parameters aren't provided. This is standard JSON Schema behavior — `"default": 10` means use 10 if not specified.

### A28: **B** [D3]
PostToolUse on Write/Edit is correct: format AFTER Claude writes the file. This ensures the formatter sees the final content, and Claude's subsequent reads see formatted code. PreToolUse (A) runs before the file exists.

### A29: **B** [D4]
Type checking is a VALIDATION step: generate code → run `tsc --noEmit` → if errors, feed them back for Claude to fix → retry. It sits in the validate position of the generate-validate-retry loop.

### A30: **B** [D5]
System prompt, tool definitions, and CLAUDE.md content are structural — they're always loaded regardless of compaction. Message history may be summarized or lost. This is why critical information belongs in persistent locations.

### A31: **B** [D1]
Research consistently shows that fewer, well-described tools produce better selection accuracy. 30 tools overwhelm reasoning; 8 focused tools with clear WHEN/WHEN-NOT descriptions achieve better results at lower token cost.

### A32: **B** [D2]
stdio transport uses stdin/stdout pipes between parent and child processes — it requires the server to be running locally as a subprocess. Remote servers must use Streamable HTTP transport for network communication (HTTP+SSE is legacy/deprecated).

### A33: **B** [D3]
Unrestricted `"Write"` allows writing to ANY path — including sensitive files. Should use scoped patterns: `"Write(src/**)"`, `"Write(tests/**)"` with explicit deny for sensitive paths. Broad permissions = broad risk.

### A34: **B** [D4]
Match model to task: Haiku for simple tasks (fast, cheap, sufficient quality), Sonnet for moderate tasks, Opus for complex reasoning. Using Opus for simple questions wastes money; using Haiku for complex analysis produces poor results.

### A35: **B** [D5]
Reduce false alarms/trivial approvals so humans only see genuinely critical decisions. Make presentations concise (summary + key risk + decision needed). This restores signal-to-noise ratio and makes human review effective.

### A36: **B** [D1]
Two safeguards needed: (1) consecutive error counter — after 3 identical errors, inject "change your approach" message; (2) max iteration limit — hard stop if the agent exceeds N total iterations regardless of error type.

### A37: **B** [D2]
Tool descriptions should reflect ALL observable effects. If the tool sends an email AND logs to audit, Claude needs to know both — it might tell the user "I'll create the user" without mentioning the email/audit, causing surprise.

### A38: **A** [D3]
CLAUDE.md represents the project owner's current instructions and takes precedence over historical artifacts. The team has migrated to yarn (as documented in CLAUDE.md). If truly uncertain, asking (C) is also reasonable.

### A39: **B** [D4]
"Helpful assistant" provides zero behavioral constraints. Claude fills the vacuum with variable decisions about format, length, depth, and style. Specific role + criteria + constraints = consistent behavior across sessions.

### A40: **B** [D5]
Priority order: (1) Prompt caching — system prompt is typically 40-60% of input cost and repeats every iteration; 90% savings is the single biggest win. (2) Tool result truncation — prevents context bloat. (3) Model selection — Haiku for simple sub-tasks saves per-call.

### A41: **B** [D1]
Plan-Execute-Verify: if verification fails, revert the failed step, fix the approach, re-execute that step, verify again. Don't redo everything (steps 1-2 were fine) and don't skip (step 4 may depend on step 3).

### A42: **B** [D2]
Schema validation should reject the input cleanly with a structured error. Return `is_error: true` with specific details about the violation. This lets Claude self-correct on retry. Never silently accept invalid input.

### A43: **B** [D3]
Project settings (`.claude/settings.json`) have higher priority than global (`~/.claude/settings.json`). Project: "false" wins over global: "true". Priority: local > project > global.

### A44: **B** [D4]
XML tags create unambiguous section boundaries. Claude reliably identifies `<examples>` vs `<task>` vs `<constraints>` as distinct sections, reducing confusion about which content serves which purpose. This is a prompt engineering best practice.

### A45: **B** [D5]
Information freshness: a file read from 15 minutes ago is a point-in-time observation. In active development, it may be outdated. The agent should re-read before making decisions or modifications based on that earlier reading.

### A46: **B** [D1]
`stop_reason: "tool_use"` means Claude's response contains tool_use blocks that need execution. Your code should: extract all tool_use blocks, execute each tool, return all results, and continue the loop.

### A47: **B** [D2]
External-effect tools (email, deploy, publish) need: rate limiting (max N per session), confirmation for bulk actions, and potentially human approval gates. An unbounded email tool in a loop is a recipe for disaster.

### A48: **B** [D3]
PostToolUse runs AFTER the tool has already executed — it cannot block or revert. It's for: logging, formatting, notifications, verification. Only PreToolUse can block (prevent execution). This is a critical distinction.

### A49: **B** [D4]
50,000 items × 200 input tokens = 10M input tokens. 50,000 items × 50 output tokens = 2.5M output tokens.
Haiku rates with 50% batch discount: input = $0.80/M × 0.5 = $0.40/M; output = $4/M × 0.5 = $2/M.
Input cost: 10M × $0.40/M = $4. Output cost: 2.5M × $2/M = $5. Total: $4 + $5 = $9.

### A50: **B** [D5]
Error budget exhaustion = hard stop. The agent has demonstrated it cannot resolve the issue. Escalate to human with: full error chain (what was tried, what failed each time), current state, and what's needed to resolve. Never exceed the budget.
