# Domain 2: Tool Design & Integration — Practice Quiz

## Instructions
- 25 scenario-based questions
- Select the BEST answer for each
- Answers with explanations at the bottom
- Time target: 30 minutes

---

## Questions

### Q1
Claude consistently selects `search_all` over the more specific `search_users` tool. Both descriptions start with "Search for...". What is the MOST effective fix?

A) Rename `search_users` to `find_user_by_criteria`  
B) Add "Use this tool ONLY for..." and "Do NOT use for..." guidance to both descriptions  
C) Remove `search_all` from the tool set  
D) Make `search_users` the first tool in the list  

---

### Q2
A tool description reads: "Gets data from the system." Claude uses it for 5 different scenarios, often incorrectly. What is the ROOT problem?

A) The tool name is too generic  
B) The description lacks trigger conditions (WHEN to use) and scope boundaries  
C) Claude needs more training on tool selection  
D) There are too many tools competing for attention  

---

### Q3
A tool returns the string `"Error: User not found"` without `is_error: true`. Claude responds to the user: "The user's status is 'Error: User not found'." What caused this?

A) Claude's reasoning is flawed  
B) The error was returned as successful data, and Claude interpreted it as the user's status field value  
C) The tool description doesn't mention error cases  
D) Claude doesn't understand error messages  

---

### Q4
Your tool's `input_schema` specifies `"date": {"type": "string"}` without a pattern or description. Claude sometimes provides "March 15, 2024" and sometimes "2024-03-15". How do you ensure consistent ISO 8601 format?

A) Add `"pattern": "^\\d{4}-\\d{2}-\\d{2}$"` and `"description": "Date in ISO 8601 format: YYYY-MM-DD (e.g., '2024-03-15')"`  
B) Add only `"format": "date"` to the schema  
C) Parse any format server-side  
D) Add validation in a PreToolUse hook  

---

### Q5
You have 35 tools defined. Claude's tool selection accuracy has dropped to 60%. What architectural change provides the BIGGEST improvement?

A) Improve all 35 tool descriptions  
B) Implement contextual tool loading — only present 5-8 relevant tools per task  
C) Switch to Opus for better reasoning  
D) Add mutual exclusion clauses to every description  

---

### Q6
An MCP server provides database access. The configuration hardcodes the password in `.claude/settings.json` (committed to git). What is the security issue and fix?

A) The password should be in `.claude/settings.local.json` instead  
B) The password is exposed in git history; use `"env": {"DB_PASS": "${DATABASE_PASSWORD}"}` to reference environment variables  
C) Encrypt the password in the settings file  
D) Use a separate credentials file and add it to .gitignore  

---

### Q7
A tool's structured error response includes: `{"status": "error", "error_type": "validation", "message": "Invalid email format", "field": "email", "received": "not-email", "suggestion": "Provide valid email like user@example.com"}`. How does this help Claude compared to just `"Error"`?

A) It doesn't help — Claude treats all errors the same  
B) Claude can identify the specific field, understand what went wrong, and retry with the correct format  
C) It reduces token usage on retries  
D) It prevents Claude from retrying  

---

### Q8
An MCP server running via stdio crashes silently. Claude's next tool call hangs indefinitely. What architectural pattern prevents this?

A) Run the MCP server in Docker  
B) Health check ping + timeout + auto-restart + graceful error on unavailability  
C) Use HTTP+SSE transport instead of stdio  
D) Add a retry to every tool call  

---

### Q9
Two MCP servers provide similar tools: `server-a` has `search_docs` and `server-b` has `find_documents`. Claude gets confused between them. What is the BEST resolution?

A) Remove one server  
B) Rename tools to be clearly distinct and add disambiguation in descriptions  
C) Add a priority system for tool selection  
D) Configure one server as primary and the other as fallback  

---

### Q10
A JSON Schema uses `"additionalProperties": true` in a tool's input_schema. Claude sometimes adds unexpected extra fields like `"notes": "..."` or `"reasoning": "..."`. What should you do?

A) Remove `additionalProperties` or set it to `false`  
B) Add all possible fields to the schema explicitly  
C) Filter extra fields server-side before processing  
D) Add "do not include extra fields" to the description  

---

### Q11
A tool chain: `search_code` → `read_file` → `edit_file` → `run_tests`. The `read_file` result is 15,000 tokens (large file). What is the cost implication and mitigation?

A) The 15K tokens are only charged once  
B) The 15K tokens accumulate in context and are charged on every subsequent API call; mitigation: extract only relevant sections  
C) Tool results don't count toward token usage  
D) The cost is fixed regardless of result size  

---

### Q12
You want Claude to ALWAYS extract structured data (never respond conversationally). Which API configuration ensures this?

A) Add "Always use tools" to the system prompt  
B) Use `tool_choice: {"type": "tool", "name": "extract_data"}`  
C) Use `tool_choice: {"type": "any"}`  
D) Only provide one tool and no conversational option  

---

### Q13
A composite tool `search_and_summarize` combines 3 operations. When should you use a composite tool vs. separate tools?

A) Always use composite tools to reduce costs  
B) Composite when steps always happen together and don't need Claude's judgment between them; separate when Claude needs to decide between steps  
C) Composite for read operations, separate for write operations  
D) Composite for Haiku, separate for Opus  

---

### Q14
An MCP server exposes a `run_sql` tool. A malicious prompt injection in user content could cause Claude to execute `DROP TABLE users`. What is the MOST effective defense?

A) Add "never run DROP statements" to Claude's system prompt  
B) MCP server uses read-only database credentials + query allowlist + parameterized queries  
C) Add input validation in a PreToolUse hook  
D) Filter the word "DROP" from all tool inputs  

---

### Q15
Your tool definition has 25 properties in its input_schema. Claude frequently provides incorrect values for properties deep in the schema. What is the BEST refactoring?

A) Add more detailed descriptions to every property  
B) Split into multiple focused tools with 5-8 properties each, or group into nested objects  
C) Make all properties required so Claude doesn't skip them  
D) Switch to Opus for better schema adherence  

---

### Q16
MCP server `filesystem` is configured with `"args": ["/Users/me/project"]`. Claude requests to read `/etc/passwd`. What should happen?

A) The MCP server returns the file content  
B) The MCP server blocks the request since `/etc/passwd` is outside the allowed path  
C) Claude's system prompt should prevent this request  
D) A PreToolUse hook should catch this  

---

### Q17
A tool returns `{"retryable": true, "error_type": "timeout"}` but Claude doesn't retry. Why, and what's the fix?

A) Claude doesn't understand `retryable` — rename it  
B) `retryable` is metadata for YOUR system to handle (retry logic); add "You may retry this operation" to the suggestion field for Claude to act on  
C) Set `is_error: false` so Claude doesn't think it failed  
D) Claude never retries — it always moves to a different approach  

---

### Q18
Three parallel tool calls are made: `read_file("a.ts")`, `read_file("b.ts")`, `read_file("c.ts")`. How should results be returned?

A) Three separate user messages, one per result  
B) One user message with an array of three tool_result blocks, each matched by tool_use_id  
C) A single tool_result combining all three files  
D) In any order — the API sorts them automatically  

---

### Q19
Tool description: "Searches for users." Claude uses it to search for products. What element is MISSING from the description?

A) The tool name should mention "users"  
B) A "Do NOT use for..." clause disambiguating from product/order search  
C) The return type specification  
D) Example queries  

---

### Q20
An MCP server's tool returns sensitive data (SSN, credit card numbers) in its result. This data flows into Claude's context. What is the architectural fix?

A) Tell Claude "never mention sensitive data in responses"  
B) The MCP server should redact/mask sensitive fields before returning results to the host  
C) Encrypt the data in the tool result  
D) Use a less capable model that can't extract patterns from data  

---

### Q21
You're choosing between stdio and HTTP+SSE transport for an MCP server that will be shared across a team. The server runs on a central host. Which transport and why?

A) stdio — it's faster and simpler  
B) HTTP+SSE — required for remote servers since stdio only works for local processes  
C) Either works — transport is just a protocol detail  
D) stdio with SSH tunneling for security  

---

### Q22
A tool's error response includes a raw Python traceback with file paths and library versions. What security risks does this create?

A) No risk — tracebacks are useful for debugging  
B) Information disclosure: reveals internal architecture, file paths, dependency versions that attackers can exploit  
C) Token waste — tracebacks are long  
D) Claude may try to fix the server-side code  

---

### Q23
Claude makes a tool call with `"name": "search_documentss"` (typo — extra 's'). Your system has no tool by that name. What should you return?

A) An empty successful result  
B) Nothing — skip it  
C) A tool_result with `is_error: true` listing available tool names so Claude can self-correct  
D) Terminate the conversation  

---

### Q24
A tool chain has 5 steps. Step 3 fails with a validation error. Steps 1 and 2 produced useful data. What is the correct handling?

A) Discard all results and start over  
B) Return the step 3 error to Claude with context from steps 1-2, allowing it to fix step 3 and continue  
C) Skip step 3 and proceed to step 4  
D) Return partial results and tell the user "task incomplete"  

---

### Q25
Your team uses 3 MCP servers: GitHub, Jira, and Slack. All use the same API token stored in one environment variable. What is the security concern?

A) No concern — shared tokens are convenient  
B) Compromise of one token grants access to all three services; use separate credentials per server (principle of least privilege)  
C) The token will expire faster with three servers using it  
D) Rate limits will be shared across all three  

---

---

## Answers

### A1: **B**
Disambiguation requires explicit WHEN/WHEN-NOT clauses in BOTH tool descriptions. `search_users`: "Use ONLY when looking up user profiles/accounts." `search_all`: "Use only when the query doesn't match a specific search tool. Do NOT use for user lookups (use search_users instead)."

### A2: **B**
"Gets data from the system" tells Claude NOTHING about when to use it. A proper description needs: trigger conditions ("Use when the user asks about X"), scope ("Only for Y data"), and disambiguation ("Not for Z — use tool_W instead").

### A3: **B**
Without `is_error: true`, the error string is treated as successful tool output — data that Claude incorporates into its response. Claude has no way to distinguish "Error: User not found" (an error) from data where the status field contains that string. Always use `is_error: true` for failures.

### A4: **A**
Both `pattern` (hard validation) and `description` with example (Claude guidance) together provide the highest success rate. The pattern catches invalid formats; the description with example guides Claude to generate correctly on the first try. Server-side parsing (C) doesn't prevent Claude from sending the wrong format.

### A5: **B**
With 35 tools, Claude spends significant reasoning on tool selection and often picks wrong. Contextual tool loading (showing only 5-8 relevant tools per query) dramatically reduces confusion. This is more impactful than improving descriptions (which helps but doesn't solve the core information overload problem).

### A6: **B**
Credentials committed to git are in the history permanently (even if later removed). The fix: use environment variable references (`${DATABASE_PASSWORD}`) so the settings file contains a pointer, not the actual secret. Each developer sets the variable in their own environment.

### A7: **B**
Structured errors give Claude everything it needs to self-correct: which field failed (email), what was wrong (invalid format), what was received (not-email), and what to do instead (use user@example.com format). This typically results in a successful retry on the first attempt.

### A8: **B**
Multiple defenses: health checks detect crashes quickly, timeouts prevent infinite hangs, auto-restart recovers the server, and graceful error responses let Claude know the tool is temporarily unavailable (rather than hanging). This is comprehensive reliability engineering.

### A9: **B**
Duplicate/similar tools across MCP servers confuse Claude. Either consolidate (remove one) or make them clearly distinct through renaming and description disambiguation. Claude selects based on name + description, so both must be unambiguous.

### A10: **A**
`additionalProperties: true` (or undefined — which defaults to allowing extras) lets Claude add arbitrary fields. Setting it to `false` or removing it (and only defining expected properties) constrains Claude's output to the schema. Server-side filtering (C) works but doesn't prevent wasted tokens.

### A11: **B**
In agentic loops, ALL message history (including tool results) is sent with every subsequent API call. A 15K token result is charged on every iteration after it's added. Mitigation: extract only the relevant portion of the file (specific functions/lines) rather than the entire file.

### A12: **B**
`tool_choice: {"type": "tool", "name": "extract_data"}` forces Claude to ALWAYS use that specific tool — it cannot respond with plain text. This guarantees structured output matching the tool's input_schema. Option C forces some tool but Claude picks which one.

### A13: **B**
Composite tools are optimal when steps are always performed together and Claude's reasoning isn't needed between them (e.g., search always followed by read). Keep separate when Claude needs to make decisions based on intermediate results (e.g., search → decide what to read based on results).

### A14: **B**
Defense in depth at the server level: read-only credentials (can't DROP anything), query allowlist (only approved patterns execute), and parameterized queries (prevent injection even if malicious SQL reaches the server). System prompt instructions (A) are soft guardrails, not security guarantees.

### A15: **B**
25 properties in one tool overwhelms Claude's reasoning. Split into multiple focused tools (e.g., `create_basic_alert` + `configure_alert_conditions` + `configure_notifications`) or group related properties into nested objects. This reduces cognitive load and error rates.

### A16: **B**
The MCP filesystem server is configured with an allowed directory (`/Users/me/project`). Requests for paths outside this directory should be rejected by the server with a permission/scope error. This is infrastructure-level enforcement.

### A17: **B**
`retryable` is typically metadata for the application layer's retry logic, not something Claude automatically acts on. If you want Claude to retry, include it in the `suggestion` field: "This was a timeout — you may retry the same operation." Claude reads and acts on suggestions.

### A18: **B**
When Claude makes parallel tool calls (multiple tool_use blocks in one response), all results must be returned in a single user message containing an array of tool_result objects, each with the matching `tool_use_id`. This is the API contract.

### A19: **B**
Without explicit "Do NOT use for product/order searches (use search_products instead)", Claude sees "searches" in the description and applies it to any search intent. Negative guidance (when NOT to use) is as important as positive guidance (when to use) for disambiguation.

### A20: **B**
Data minimization at the source: the MCP server should mask sensitive fields (SSN → ***-**-1234, CC → ****4242) before returning results. Once sensitive data enters Claude's context, it's extremely difficult to prevent it from appearing in responses.

### A21: **B**
stdio transport uses stdin/stdout pipes — it only works when the server is a local child process of the host. For remote servers shared across a team on a central host, HTTP+SSE transport is required (client sends HTTP POST, server sends Server-Sent Events).

### A22: **B**
Raw tracebacks expose: internal file paths (/app/server/auth/handler.py), library versions (Django 4.2.1, which may have known CVEs), database structure, and system architecture. This is information disclosure — always sanitize errors before returning them.

### A23: **C**
Return a tool_result with `is_error: true` containing the available tool names: `"Tool 'search_documentss' not found. Available tools: search_documents, search_users, search_code"`. This lets Claude self-correct (likely a hallucinated typo) on the next iteration.

### A24: **B**
The error should be returned to Claude WITH the successful context from steps 1-2. Claude can then fix the step 3 input (using information from steps 1-2) and retry. Don't discard prior work; don't skip steps that may have dependencies.

### A25: **B**
Shared credentials violate least privilege: if the token is compromised (e.g., through a Jira vulnerability), the attacker also gets GitHub and Slack access. Each MCP server should have its own dedicated credentials with minimum required permissions for that specific service.
