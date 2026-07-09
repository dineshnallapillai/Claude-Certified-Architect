# Supplementary Practice Questions — Gap Coverage

## Instructions
- 12 scenario-based questions covering topics NOT tested in existing practice files
- Select the BEST answer for each
- Answers with explanations at the bottom
- These fill specific exam guide task statement gaps

---

## Questions

### Q1 [D3 — Task 3.2: SKILL.md context: fork]
A team creates a codebase analysis skill that reads 40+ files and runs 20 grep patterns. After using it, developers find that their subsequent questions in the same session get vague, generic answers because context is filled with file contents. What is the BEST fix?

A) Add "summarize findings in under 500 words" to the skill instructions  
B) Add `context: fork` to the SKILL.md frontmatter so it runs in an isolated sub-agent context  
C) Split the skill into 8 smaller skills that each read fewer files  
D) Tell developers to run `/compact` after using the skill  

---

### Q2 [D3 — Task 3.2: SKILL.md allowed-tools]
A test generation skill should read source files, write test files, and search for existing patterns — but must NOT execute commands, make git commits, or access the network. Which SKILL.md frontmatter configuration enforces this?

A) Add "Do not run bash commands or access the network" to the skill instructions  
B) Set `allowed-tools: [Read, Write, Edit, Grep, Glob]` in the SKILL.md frontmatter  
C) Configure `deny: ["Bash(*)"]` in `.claude/settings.json`  
D) Use `context: fork` which automatically restricts tool access  

---

### Q3 [D3 — Task 3.2: SKILL.md argument-hint]
Developers invoke `/security-audit` without specifying a target. The skill then analyzes the entire repository (expensive and slow) instead of the intended module. What frontmatter addition prevents this?

A) `required: true`  
B) `argument-hint: "target file path, module name, or PR number to audit"`  
C) `default-target: "."`  
D) `validate-args: true`  

---

### Q4 [D3 — Task 3.5: Interview Pattern]
You ask Claude to "add a caching layer to the API." Claude immediately implements a basic in-memory LRU cache. In production, the team discovers it doesn't handle multi-node deployment, cache invalidation, or PII exclusion. Which iterative refinement technique would have PREVENTED this?

A) Providing 3 concrete input/output examples showing cached vs uncached responses  
B) Using the interview pattern — having Claude ask questions about requirements before implementing  
C) Writing a complete test suite covering all caching scenarios first  
D) Giving Claude a detailed specification document up front  

---

### Q5 [D3 — Task 3.5: Interacting vs Independent Issues]
A developer finds three bugs: (1) retry logic uses fixed 1-second delay instead of exponential backoff, (2) a typo in an error message string ("recieved" → "received"), and (3) the retry timeout doesn't account for the new backoff delays. How should these be reported to Claude?

A) All three in a single message — they were found together  
B) #1 and #3 together (they interact), then #2 separately (independent)  
C) All three sequentially, one per turn  
D) Create a numbered list and ask Claude to fix them in priority order  

---

### Q6 [D4 — Task 4.6: Self-Review Limitation]
Your CI pipeline uses Claude to generate test code and then immediately reviews it in the same session by appending "Now review the tests you just wrote for correctness." Reviews consistently approve the generated tests, but manual review later finds logical gaps. What is the ROOT CAUSE?

A) The review prompt needs more specific criteria  
B) The same session retains reasoning context from generation, biasing the review toward confirming its own decisions  
C) The model isn't capable enough to review test code  
D) Extended thinking should be enabled for the review step  

---

### Q7 [D4 — Task 4.6: Multi-Pass Review]
A single-pass automated code review of a 14-file PR produces detailed feedback for files 1-4, superficial comments for files 5-10, and contradictory findings (flagging a pattern in file 2 but approving identical code in file 12). What restructuring BEST addresses this?

A) Switch to Opus for better attention over long inputs  
B) Split into per-file local analysis passes, then a separate cross-file integration pass for data flow and consistency  
C) Run three independent reviews of the full PR and flag issues appearing in 2+ runs  
D) Require developers to submit PRs with no more than 5 files  

---

### Q8 [D2 — Task 2.4: MCP Server Scoping]
A developer is experimenting with a custom MCP server they built for personal productivity. They want to test it without affecting their teammates. Where should they configure it?

A) `.mcp.json` in the project root with a feature flag  
B) `~/.claude/settings.json` (user-level configuration, applies to all projects)  
C) `.claude/settings.local.json` in the project (local only, gitignored)  
D) `.claude/settings.json` with a comment noting it's experimental  

---

### Q9 [D2 — Task 2.4: MCP Resources]
An MCP server provides tools for querying a Jira board. Agents frequently make exploratory calls like `search_tickets("")` just to discover what's available. What MCP capability would REDUCE these wasteful calls?

A) Add a `list_all_tickets` tool that returns everything  
B) Expose the sprint board and issue categories as MCP **resources** (content catalogs agents can browse)  
C) Add a `get_available_data` meta-tool  
D) Document all available queries in the tool description  

---

### Q10 [D5 — Task 5.4: Scratchpad Files]
An agent exploring a large legacy codebase maintains quality for the first 15 iterations but starts giving vague answers by iteration 25, referencing "typical patterns" instead of the specific classes it discovered earlier. What technique BEST preserves findings across context degradation?

A) Increase the model's context window  
B) Have the agent write key findings to a scratchpad file that it can Read back in later iterations  
C) Repeat all findings in every message  
D) Use extended thinking to improve recall  

---

### Q11 [D5 — Task 5.4: Explore Subagent]
A main agent needs to understand the authentication flow in a large codebase. Investigation requires reading 30+ files and running many grep patterns. If this discovery happens in the main agent's context, it will consume most of the available tokens. What is the BEST architectural approach?

A) Read only the 5 most important files and skip the rest  
B) Spawn an Explore subagent for the verbose discovery phase, which returns only a structured summary to the main agent  
C) Use /compact after every 5 file reads  
D) Split the investigation into 6 separate conversations  

---

### Q12 [D1 — Task 1.4: Programmatic Prerequisite Enforcement]
Production data shows that in 8% of cases, your customer support agent calls `process_refund` without first calling `get_customer` to verify the customer's identity. This has led to 3 incorrect refunds. What change MOST reliably fixes this?

A) Strengthen the system prompt: "You MUST always call get_customer before process_refund"  
B) Add few-shot examples showing get_customer being called first in every scenario  
C) Implement a programmatic PreToolUse hook that blocks `process_refund` unless `get_customer` has already returned a verified customer ID in the current session  
D) Add a routing classifier that only enables refund tools after identity verification  

---

---

## Answers

### A1: **B** [D3 — Task 3.2]
`context: fork` runs the skill in an isolated sub-agent context. All verbose intermediate output (40 file reads, 20 grep results) stays in the fork. Only the final summary returns to the main conversation, keeping context clean for subsequent questions. Option A may help but doesn't solve the context pollution problem. Option C adds complexity without isolation. Option D puts the burden on developers and still leaves verbose output in context.

### A2: **B** [D3 — Task 3.2]
`allowed-tools` in SKILL.md frontmatter provides a **hard restriction** — the skill literally cannot use tools not in the list. By excluding Bash, the skill cannot run commands, make commits, or access the network. Option A is prompt-based (probabilistic, can be ignored). Option C is global (affects all operations, not just this skill). Option D isolates context but does NOT restrict tool access.

### A3: **B** [D3 — Task 3.2]
`argument-hint` is the SKILL.md frontmatter field that specifies what to prompt the user for when they invoke without arguments. When the developer types `/security-audit` with no target, Claude displays "Please provide: target file path, module name, or PR number to audit" — giving them a chance to specify the scope before execution begins.

### A4: **B** [D3 — Task 3.5]
The interview pattern has Claude ask questions BEFORE implementing to surface requirements you haven't considered. Questions like "Single-node or multi-node?", "Cache invalidation strategy?", "Any data that must never be cached?" would have revealed the multi-node, invalidation, and PII requirements upfront. Input/output examples (A) don't help when the requirements are unknown. Tests (C) require knowing the requirements first. A detailed spec (D) assumes you already know everything — which was the problem.

### A5: **B** [D3 — Task 3.5]
Issues #1 (backoff logic) and #3 (timeout accounting for backoff) **interact** — fixing the retry delay directly affects what the timeout should be. Issue #2 (typo) is completely **independent** — fixing it has zero effect on retry logic. Grouping interacting issues prevents Claude from fixing #1 in a way that conflicts with #3. Separating independent issues keeps scope clean. The decision rule: "Would fixing A change how I fix B?" YES → together.

### A6: **B** [D4 — Task 4.6]
Self-review limitation: the same session retains reasoning context from code generation. Claude is anchored to its original intent and is less likely to question decisions it just made. An **independent instance** (separate API call with only the code, no generation history) provides a fresh perspective that catches issues the generator missed. This is a fundamental exam concept — same session reviews are systematically weaker than independent reviews.

### A7: **B** [D4 — Task 4.6]
Multi-pass review directly addresses attention dilution. Per-file local passes give each file full attention (consistent depth, no superficial comments). A separate cross-file integration pass examines data flow, API contracts, and consistency across files. Option A misunderstands that larger context doesn't fix attention quality. Option C (consensus) suppresses real bugs that may be intermittently detected. Option D shifts burden to developers.

### A8: **C** [D2 — Task 2.4]
`.claude/settings.local.json` is the **project-level local** configuration — gitignored, so it won't affect teammates. You add MCP servers under the `mcpServers` key. Perfect for experimental servers scoped to this project only. `.mcp.json` (A) is project-level and would be committed to git, affecting the whole team. `~/.claude/settings.json` (B) is user-level but applies to ALL projects globally, not just the one you're experimenting in. `.claude/settings.json` (D) is project-level and git-tracked, so teammates would see it.

### A9: **B** [D2 — Task 2.4]
MCP **resources** are content catalogs that agents can browse without making action-oriented tool calls. Exposing the sprint board and categories as resources gives agents visibility into what data exists, so they can make targeted tool calls instead of exploratory searches. This is the exam-specific concept: "MCP resources reduce exploratory tool calls." A meta-tool (C) works but isn't the MCP-native solution. Documentation (D) helps but doesn't give agents programmatic access to browse.

### A10: **B** [D5 — Task 5.4]
Scratchpad files persist findings outside of conversation context. When context is compacted or degraded, the agent can Read the scratchpad file to recover specific details (class names, line numbers, architectural decisions) that would otherwise be lost to summarization. This directly counteracts the "vague answers" degradation signal. Larger context (A) only delays the problem. Repeating findings (C) wastes tokens. Extended thinking (D) doesn't help recall external facts.

### A11: **B** [D5 — Task 5.4]
The Explore subagent isolates verbose discovery (30+ file reads, many grep patterns) in a separate context. It returns only a structured summary to the main agent, preserving the main agent's context for coordination and implementation. This is the exam-tested pattern for "isolating verbose exploration output while the main agent coordinates high-level understanding." Option D loses continuity across conversations.

### A12: **C** [D1 — Task 1.4]
When a specific tool sequence is required for critical business logic (identity verification before refund processing), **programmatic enforcement provides deterministic guarantees** that prompt-based approaches cannot. A PreToolUse hook that checks whether `get_customer` has returned a verified ID before allowing `process_refund` has a 0% failure rate. System prompt (A) and few-shot (B) are probabilistic (~95%) — unacceptable when errors have financial consequences. This matches the exam guide's sample question #1 exactly.

---

---

## Additional Weak Spot Questions (Q13–Q16)

---

### Q13 [D1 — Task 1.7: fork_session vs Resumption]
A developer has been analyzing a microservices codebase in a Claude Code session for 30 minutes. They now want to explore two competing migration strategies — one conservative (minimal changes) and one aggressive (complete redesign) — without either exploration contaminating the other. What is the BEST approach?

A) Use `--resume` to continue the same session and explore both sequentially  
B) Use `--fork-session` (boolean flag with `--resume`) to create an independent session copy from the current shared analysis baseline. Invoke separately for each branch.  
C) Start two completely new sessions from scratch, re-doing the initial analysis in each  
D) Ask Claude to evaluate both strategies in the same session using a comparison table  

---

### Q14 [D3 — Task 3.3: Path-Specific Rules with Glob Patterns]
A project has React components in `src/components/`, API handlers in `src/api/`, and test files spread throughout (e.g., `src/components/Button.test.tsx`, `src/api/users.test.ts`). All test files should follow the same testing conventions regardless of directory. What is the MOST maintainable configuration?

A) Place a CLAUDE.md in each directory that contains test files  
B) Create a `.claude/rules/testing.md` file with `paths: ["**/*.test.{ts,tsx}"]` frontmatter  
C) Add all testing conventions to the root CLAUDE.md  
D) Create a testing skill that developers invoke before writing tests  

---

### Q15 [D5 — Task 5.5: Stratified Sampling & Confidence Calibration]
A structured data extraction system achieves 97% overall accuracy. The team decides to reduce human review and auto-approve all high-confidence extractions. After deployment, they discover 40% error rates on a specific document type (handwritten forms) that represents only 3% of volume. What review pattern would have caught this?

A) Random sampling of 5% of all extractions  
B) Stratified random sampling by document type, measuring error rates per segment before reducing review  
C) Only reviewing extractions where the model reports low confidence  
D) Increasing the overall accuracy threshold from 97% to 99%  

---

### Q16 [D5 — Task 5.6: Information Provenance & Conflicting Sources]
A multi-agent research system finds two credible sources reporting different statistics: Source A (published 2024) says "market size is $4.2B" and Source B (published 2025) says "market size is $3.8B." The synthesis agent must produce a final report. What is the CORRECT handling?

A) Use Source B since it's more recent  
B) Average the two values: $4.0B  
C) Include both values with source attribution and publication dates, noting the discrepancy and letting the reader decide  
D) Discard both and mark the field as "unable to determine"  

---

---

## Answers (Q13–Q16)

### A13: **B** [D1 — Task 1.7]
`--fork-session` is a boolean flag used with `--resume` or `--continue` that creates a new session ID while preserving the original session's context. Each fork requires a separate invocation. Both forks start with the same 30-minute codebase understanding, but explore different strategies without contaminating each other. The developer can then compare results from both forks. Option A would let strategy 1's reasoning influence strategy 2 (no independence). Option C wastes 30 minutes re-doing analysis in each. Option D risks the strategies influencing each other within one context.

### A14: **B** [D3 — Task 3.3]
`.claude/rules/` files with YAML frontmatter `paths: ["**/*.test.{ts,tsx}"]` activate only when Claude edits matching files — regardless of directory location. This is the exam-tested advantage over subdirectory CLAUDE.md files: conventions that span multiple directories. Option A requires CLAUDE.md in every directory with tests (unmaintainable). Option C loads testing rules even when editing non-test files (wasteful). Option D requires manual invocation.

### A15: **B** [D5 — Task 5.5]
Stratified random sampling by document type reveals per-segment error rates. Aggregate metrics (97% overall) mask poor performance on minority document types. By measuring accuracy per type BEFORE reducing review, the team would have discovered the 40% error rate on handwritten forms. Option A (random 5%) likely undersamples the 3% segment. Option C misses cases where the model is confidently WRONG. Option D doesn't address segment-level variation.

### A16: **C** [D5 — Task 5.6]
When credible sources conflict, the synthesis must preserve BOTH values with source attribution and publication dates — not arbitrarily select one. This lets the reader understand the discrepancy and make their own judgment. The temporal difference (2024 vs 2025) might explain the change (market contraction) or might be methodological. Annotating conflicts rather than resolving them is the exam-tested principle. Option A loses information. Option B fabricates a value. Option D is overly conservative.
