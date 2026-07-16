# Official Sample Questions (12) — CCAR-F Exam Guide v1.0

**Source:** Claude Certified Architect – Foundations Exam Guide (July 2026), Section 9  
**Format:** Multiple-choice, scenario-based  
**Time:** ~2 minutes per question (24 minutes total for all 12)  
**Purpose:** Official practice questions with detailed explanations and exam alignment

---

## Scenario 1: Customer Support Resolution Agent

**Context:** Building a customer support resolution agent using Claude Agent SDK with MCP tools: `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`. Target: 80%+ first-contact resolution with appropriate escalation.

---

### Question 1: Tool Sequencing & Programmatic Enforcement

**Difficulty:** Medium | **Domains:** D1.1 (Agentic Loops), D1.5 (Hooks)  
**Task Statements:** 1.1, 1.5

Production data shows that in 12% of cases, your agent skips `get_customer` entirely and calls `lookup_order` using only the customer's stated name, occasionally leading to misidentified accounts and incorrect refunds. What change would most effectively address this reliability issue?

**A.** Add a programmatic prerequisite that blocks `lookup_order` and `process_refund` calls until `get_customer` has returned a verified customer ID.

**B.** Enhance the system prompt to state that customer verification via `get_customer` is mandatory before any order operations.

**C.** Add few-shot examples showing the agent always calling `get_customer` first, even when customers volunteer order details.

**D.** Implement a routing classifier that analyzes each request and enables only the subset of tools appropriate for that request type.

---

**Correct Answer: A**

**Explanation:**

When a specific tool sequence is required for critical business logic (like verifying customer identity before processing refunds), **programmatic enforcement provides deterministic guarantees** that prompt-based approaches cannot.

Why the other options fail:

- **Option B** relies on probabilistic LLM compliance—prompt instructions are never 100% reliable, especially under edge cases. With 12% skip-rate, the LLM is already ignoring the instruction probabilistically.
- **Option C** adds context via examples, but few-shot examples don't enforce sequencing; they just make patterns more likely.
- **Option D** restricts tool availability but doesn't enforce tool sequencing—the problem is the *order* tools are called, not *which* tools are available.

**Key Takeaway:** When errors have financial consequences (incorrect refunds), use programmatic hooks or prerequisites, not prompt-based guidance. This is a **prerequisite gate**, not a prompt clarification.

**Exam Alignment:**
- **Task 1.1:** Agentic loop design with proper control flow
- **Task 1.5:** Agent SDK hooks for tool call interception and enforcement
- **Your Study Note:** D1-03-Hooks-Workflows-Sessions.md (PostToolUse hooks, tool call interception), D1-06-Error-Handling-Agentic-Systems.md (prerequisite enforcement)

**Related Sample Scenario:** Customer Support Resolution (Scenario 1)

---

### Question 2: Tool Description Clarity & Selection Reliability

**Difficulty:** Medium | **Domains:** D2.1 (Tool Interface Design)  
**Task Statement:** 2.1

Production logs show the agent frequently calls `get_customer` when users ask about orders (e.g., "check my order #12345"), instead of calling `lookup_order`. Both tools have minimal descriptions ("Retrieves customer information" / "Retrieves order details") and accept similar identifier formats. What's the most effective first step to improve tool selection reliability?

**A.** Add few-shot examples to the system prompt demonstrating correct tool selection patterns, with 5-8 examples showing order-related queries routing to `lookup_order`.

**B.** Expand each tool's description to include input formats it handles, example queries, edge cases, and boundaries explaining when to use it versus similar tools.

**C.** Implement a routing layer that parses user input before each turn and pre-selects the appropriate tool based on detected keywords and identifier patterns.

**D.** Consolidate both tools into a single `lookup_entity` tool that accepts any identifier and internally determines which backend to query.

---

**Correct Answer: B**

**Explanation:**

Tool descriptions are the **primary mechanism LLMs use for tool selection**. When descriptions are minimal, models lack the context to differentiate between similar tools.

**Why B is the best first step:**

- **Option B directly addresses the root cause** (minimal descriptions) with low effort and high leverage. Expanding descriptions to include edge cases, example queries, and boundary conditions gives the model the information it needs to choose correctly.
- **Option A** adds token overhead without fixing the underlying issue. Few-shot examples help but don't replace clear tool descriptions.
- **Option C** is over-engineered. It bypasses the LLM's natural language understanding and adds implementation complexity when the immediate problem is inadequate tool documentation.
- **Option D** is a valid architectural choice but requires more work than a "first step" warrants when improved descriptions haven't been tried yet.

**Key Takeaway:** When tool selection fails, fix tool *descriptions* first. They're the LLM's primary decision-making mechanism. Few-shot examples are secondary reinforcement, not the primary fix.

**Exam Alignment:**
- **Task 2.1:** Design effective tool interfaces with clear descriptions and boundaries
- **Your Study Note:** D2-01-Tool-Description-Best-Practices.md

**Related Sample Scenario:** Customer Support Resolution (Scenario 1)

---

### Question 3: Escalation Calibration & Decision Criteria

**Difficulty:** Medium | **Domains:** D5.2 (Escalation & Ambiguity Resolution)  
**Task Statement:** 5.2

Your agent achieves 55% first-contact resolution, well below the 80% target. Logs show it escalates straightforward cases (standard damage replacements with photo evidence) while attempting to autonomously handle complex situations requiring policy exceptions. What's the most effective way to improve escalation calibration?

**A.** Add explicit escalation criteria to your system prompt with few-shot examples demonstrating when to escalate versus resolve autonomously.

**B.** Have the agent self-report a confidence score (1-10) before each response and automatically route requests to humans when confidence falls below a threshold.

**C.** Deploy a separate classifier model trained on historical tickets to predict which requests need escalation before the main agent begins processing.

**D.** Implement sentiment analysis to detect customer frustration levels and automatically escalate when negative sentiment exceeds a threshold.

---

**Correct Answer: A**

**Explanation:**

Adding explicit escalation criteria with few-shot examples directly addresses the root cause: **unclear decision boundaries**. This is the proportionate first response before adding infrastructure.

**Why the other options fail:**

- **Option B** fails because LLM self-reported confidence is poorly calibrated. The agent is already incorrectly confident on hard cases (the problem), so asking it to report confidence won't fix the underlying judgment error. Confidence scores measure calibration, not actual case complexity.
- **Option C** is over-engineered, requiring labeled data and ML infrastructure when prompt optimization hasn't been tried. Start with prompt-based rules before deploying classifiers.
- **Option D** solves a different problem entirely. Sentiment (customer frustration) doesn't correlate with case complexity (whether the agent can resolve it). An angry customer might have a straightforward issue, and a patient customer might have a complex one.

**Key Takeaway:** For escalation, establish **explicit decision criteria** (policy gaps, customer requests, inability to progress) with **few-shot examples** showing ambiguous borderline cases. Confidence scores and sentiment analysis are not reliable proxies for case complexity.

**Exam Alignment:**
- **Task 5.2:** Design effective escalation and ambiguity resolution patterns
- **Your Study Note:** D5-02-Escalation-Patterns.md, D5-04-Escalation-Patterns-Error-Propagation.md

**Related Sample Scenario:** Customer Support Resolution (Scenario 1)

---

## Scenario 2: Code Generation with Claude Code

**Context:** Using Claude Code for code generation, refactoring, debugging, and documentation. Integration into team development workflows with custom slash commands, CLAUDE.md configurations, plan mode, and understanding when to use which execution mode.

---

### Question 4: Custom Slash Command Scope & Version Control

**Difficulty:** Easy | **Domains:** D3.1 (CLAUDE.md Hierarchy & Scoping)  
**Task Statement:** 3.1, 3.2

You want to create a custom `/review` slash command that runs your team's standard code review checklist. This command should be available to every developer when they clone or pull the repository. Where should you create this command file?

**A.** In the `.claude/commands/` directory in the project repository

**B.** In `~/.claude/commands/` in each developer's home directory

**C.** In the `CLAUDE.md` file at the project root

**D.** In a `.claude/config.json` file with a `commands` array

---

**Correct Answer: A**

**Explanation:**

Project-scoped custom slash commands should be stored in the `.claude/commands/` directory **within the repository**. These commands are version-controlled and automatically available to all developers when they clone or pull the repo.

**Why the other options are wrong:**

- **Option B** (`~/.claude/commands/`) is for **personal commands** that aren't shared via version control. Use this for individual developer customizations, not team-shared tooling.
- **Option C** (`CLAUDE.md`) is for project instructions, context, and conventions—not command definitions. Commands have their own file structure.
- **Option D** describes a configuration mechanism that doesn't exist in Claude Code.

**Key Takeaway:** `.claude/commands/` = shared team tooling (version-controlled). `~/.claude/commands/` = personal tooling (not shared).

**Exam Alignment:**
- **Task 3.1:** Configure CLAUDE.md with appropriate hierarchy and scoping
- **Task 3.2:** Create and configure custom slash commands and skills
- **Your Study Note:** D3-01-CLAUDE-md-Hierarchy.md, D3-02-Custom-Slash-Commands-Skills.md

**Related Sample Scenario:** Code Generation with Claude Code (Scenario 2)

---

### Question 5: Plan Mode vs Direct Execution for Architectural Changes

**Difficulty:** Medium | **Domains:** D3.4 (Plan Mode vs Direct Execution)  
**Task Statement:** 3.4

You've been assigned to restructure the team's monolithic application into microservices. This will involve changes across dozens of files and requires decisions about service boundaries and module dependencies. Which approach should you take?

**A.** Enter plan mode to explore the codebase, understand dependencies, and design an implementation approach before making changes.

**B.** Start with direct execution and make changes incrementally, letting the implementation reveal the natural service boundaries.

**C.** Use direct execution with comprehensive upfront instructions detailing exactly how each service should be structured.

**D.** Begin in direct execution mode and only switch to plan mode if you encounter unexpected complexity during implementation.

---

**Correct Answer: A**

**Explanation:**

Plan mode is designed for **complex tasks involving large-scale changes, multiple valid approaches, and architectural decisions**—exactly what monolith-to-microservices restructuring requires. It enables safe codebase exploration and design **before committing to changes**, preventing costly rework.

**Why the other options fail:**

- **Option B** risks costly rework when dependencies are discovered late. Unplanned incremental changes can lead to inconsistent service boundaries that require re-architecting.
- **Option C** assumes you already know the right structure without exploring the code. Microservices decomposition requires understanding actual dependencies, not just top-down design.
- **Option D** ignores that the complexity is **already stated in the requirements**, not something that might emerge later. With "dozens of files" and architectural implications, start with plan mode.

**Key Takeaway:** **Architectural decisions + large-scale changes = plan mode first.** Don't discover complexity; design for it upfront.

**Exam Alignment:**
- **Task 3.4:** Determine when to use plan mode vs direct execution
- **Your Study Note:** D3-04-Plan-Mode-vs-Direct-Execution.md

**Related Sample Scenario:** Code Generation with Claude Code (Scenario 2)

---

### Question 6: Path-Specific Rules with Glob Patterns

**Difficulty:** Medium | **Domains:** D3.3 (Path-Specific Rules)  
**Task Statement:** 3.3

Your codebase has distinct areas with different coding conventions: React components use functional style with hooks, API handlers use async/await with specific error handling, and database models follow a repository pattern. Test files are spread throughout the codebase alongside the code they test (e.g., `Button.test.tsx` next to `Button.tsx`), and you want all tests to follow the same conventions regardless of location. What's the most maintainable way to ensure Claude automatically applies the correct conventions when generating code?

**A.** Create rule files in `.claude/rules/` with YAML frontmatter specifying glob patterns to conditionally apply conventions based on file paths

**B.** Consolidate all conventions in the root `CLAUDE.md` file under headers for each area, relying on Claude to infer which section applies

**C.** Create skills in `.claude/skills/` for each code type that include the relevant conventions in their `SKILL.md` files

**D.** Place a separate `CLAUDE.md` file in each subdirectory containing that area's specific conventions

---

**Correct Answer: A**

**Explanation:**

Option A is correct because `.claude/rules/` with glob patterns (e.g., `**/*.test.tsx`) allows conventions to be **automatically applied based on file paths regardless of directory location**—essential for test files spread throughout the codebase.

**Why the other options fail:**

- **Option B** relies on inference rather than explicit matching, making it unreliable. Claude will guess which section applies, leading to inconsistency.
- **Option C** requires manual skill invocation or relies on Claude choosing to load them, contradicting the need for deterministic "automatic" application based on file paths. Skills are on-demand, not automatic.
- **Option D** can't easily handle files spread across many directories since `CLAUDE.md` files are directory-bound. A test in `src/components/` and another in `src/services/` would need separate rules, not separate `CLAUDE.md` files.

**Key Takeaway:** Use `.claude/rules/` with glob patterns for file-type conventions that span the codebase. Directory-level `CLAUDE.md` files are for directory-specific standards, not file-type standards.

**Exam Alignment:**
- **Task 3.3:** Apply path-specific rules for conditional convention loading
- **Your Study Note:** D3-03-Path-Specific-Rules.md

**Related Sample Scenario:** Code Generation with Claude Code (Scenario 2)

---

## Scenario 3: Multi-Agent Research System

**Context:** Building a multi-agent research system where a coordinator agent delegates to specialized subagents (web search, document analysis, synthesis, reporting). Goal: comprehensive, cited research reports with appropriate task decomposition and error handling.

---

### Question 7: Root-Cause Analysis—Coordinator Task Decomposition

**Difficulty:** Medium | **Domains:** D1.2 (Multi-Agent Orchestration)  
**Task Statement:** 1.2

After running the system on the topic "impact of AI on creative industries," you observe that each subagent completes successfully: the web search agent finds relevant articles, the document analysis agent summarizes papers correctly, and the synthesis agent produces coherent output. However, the final reports cover only visual arts, completely missing music, writing, and film production. When you examine the coordinator's logs, you see it decomposed the topic into three subtasks: "AI in digital art creation," "AI in graphic design," and "AI in photography." What is the most likely root cause?

**A.** The synthesis agent lacks instructions for identifying coverage gaps in the findings it receives from other agents.

**B.** The coordinator agent's task decomposition is too narrow, resulting in subagent assignments that don't cover all relevant domains of the topic.

**C.** The web search agent's queries are not comprehensive enough and need to be expanded to cover more creative industry sectors.

**D.** The document analysis agent is filtering out sources related to non-visual creative industries due to overly restrictive relevance criteria.

---

**Correct Answer: B**

**Explanation:**

The coordinator's logs reveal the root cause directly: it decomposed "creative industries" into **only visual arts subtasks** (digital art, graphic design, photography), completely omitting music, writing, and film. The subagents executed their assigned tasks correctly—**the problem is what they were assigned**.

**Why the other options are wrong:**

- **Option A** incorrectly blames synthesis for not detecting gaps when the coordinator created the gap by narrow decomposition.
- **Option C** incorrectly blames web search quality when the search agent was never asked to search music/writing/film topics.
- **Option D** incorrectly blames document analysis filters when no documents about music/writing/film were ever retrieved (because search was never asked for them).

**Key Takeaway:** When multi-agent output is incomplete, **diagnose the coordinator's task decomposition first**. Downstream agents are often victims of narrow task assignments, not perpetrators of narrow output.

**Exam Alignment:**
- **Task 1.2:** Orchestrate multi-agent systems with coordinator-subagent patterns
- **Your Study Note:** D1-02-Multi-Agent-Orchestration.md

**Related Sample Scenario:** Multi-Agent Research System (Scenario 3)

**Enhancement Needed:** Your D1-02 notes should include a section on "Diagnosing decomposition failures vs execution failures."

---

### Question 8: Error Propagation & Intelligent Recovery

**Difficulty:** Medium | **Domains:** D5.3 (Error Propagation), D2.2 (Structured Error Responses)  
**Task Statements:** 5.3, 2.2

The web search subagent times out while researching a complex topic. You need to design how this failure information flows back to the coordinator agent. Which error propagation approach best enables intelligent recovery?

**A.** Return structured error context to the coordinator including the failure type, the attempted query, any partial results, and potential alternative approaches.

**B.** Implement automatic retry logic with exponential backoff within the subagent, returning a generic "search unavailable" status only after all retries are exhausted.

**C.** Catch the timeout within the subagent and return an empty result set marked as successful.

**D.** Propagate the timeout exception directly to a top-level handler that terminates the entire research workflow.

---

**Correct Answer: A**

**Explanation:**

Structured error context gives the coordinator **the information it needs to make intelligent recovery decisions**—whether to retry with a modified query, try an alternative approach, or proceed with partial results.

**Why the other options fail:**

- **Option B** implements automatic retry within the subagent but returns a generic status that hides valuable context from the coordinator. Generic statuses prevent informed decisions.
- **Option C** suppresses the error by marking failure as success, which prevents any recovery and risks incomplete research output. The coordinator has no opportunity to adapt.
- **Option D** terminates the entire workflow unnecessarily when recovery strategies could succeed. One subagent timeout shouldn't kill other research threads.

**Key Takeaway:** For multi-agent error handling, **return structured context (failure type, what was attempted, partial results) to enable coordinator recovery decisions**. Don't suppress errors or terminate workflows prematurely.

**Exam Alignment:**
- **Task 5.3:** Implement error propagation strategies across multi-agent systems
- **Task 2.2:** Implement structured error responses for MCP tools
- **Your Study Note:** D5-04-Escalation-Patterns-Error-Propagation.md, D2-02-Structured-Error-Responses.md

**Related Sample Scenario:** Multi-Agent Research System (Scenario 3)

---

### Question 9: Tool Distribution & Scoped Access (80/20 Pattern)

**Difficulty:** High | **Domains:** D2.3 (Tool Distribution & Scoping), D1.2 (Multi-Agent Orchestration)  
**Task Statements:** 2.3, 1.2

During testing, you observe that the synthesis agent frequently needs to verify specific claims while combining findings. Currently, when verification is needed, the synthesis agent returns control to the coordinator, which invokes the web search agent, then re-invokes synthesis with results. This adds 2-3 round trips per task and increases latency by 40%. Your evaluation shows that 85% of these verifications are simple fact-checks (dates, names, statistics) while 15% require deeper investigation. What's the most effective approach to reduce overhead while maintaining system reliability?

**A.** Give the synthesis agent a scoped `verify_fact` tool for simple lookups, while complex verifications continue delegating to the web search agent through the coordinator.

**B.** Have the synthesis agent accumulate all verification needs and return them as a batch to the coordinator at the end of its pass, which then sends them all to the web search agent at once.

**C.** Give the synthesis agent access to all web search tools so it can handle any verification need directly without round-trips through the coordinator.

**D.** Have the web search agent proactively cache extra context around each source during initial research, anticipating what the synthesis agent might need to verify.

---

**Correct Answer: A**

**Explanation:**

Option A applies the **principle of least privilege** by giving the synthesis agent only what it needs for the **85% common case** (simple fact verification) while preserving the existing coordination pattern for complex cases. This is the "80/20 tool allocation" pattern.

**Why the other options fail:**

- **Option B** creates blocking dependencies. Synthesis steps may depend on earlier verified facts, so accumulating all needs until the end doesn't work—synthesis needs to verify incrementally.
- **Option C** over-provisions the synthesis agent, violating separation of concerns and increasing misuse risk. A synthesis agent with full search access is likely to search when fact-checking would suffice.
- **Option D** relies on speculative caching that cannot reliably predict what the synthesis agent will need to verify later. Anticipation fails when novel claims arise.

**Key Takeaway:** For high-frequency, low-complexity needs (85%), provide scoped tools. For rare, complex needs (15%), preserve existing coordination patterns. This balances latency and architectural cleanliness.

**Exam Alignment:**
- **Task 2.3:** Distribute tools appropriately across agents and configure tool_choice
- **Task 1.2:** Orchestrate multi-agent systems
- **Your Study Note:** D2-03-Tool-Distribution-Scoping.md

**Related Sample Scenario:** Multi-Agent Research System (Scenario 3)

**Enhancement Needed:** Your D2-03 notes should include the "80/20 tool allocation pattern" explicitly.

---

## Scenario 4: Claude Code for Continuous Integration

**Context:** Integrating Claude Code into CI/CD pipelines for automated code reviews, test generation, and PR feedback. Design prompts that provide actionable feedback while minimizing false positives and ensuring high reliability in blocking workflows.

---

### Question 10: Non-Interactive Pipeline Execution

**Difficulty:** Easy | **Domains:** D3.6 (CI/CD Integration)  
**Task Statement:** 3.6

Your pipeline script runs `claude "Analyze this pull request for security issues"` but the job hangs indefinitely. Logs indicate Claude Code is waiting for interactive input. What's the correct approach to run Claude Code in an automated pipeline?

**A.** Add the `-p` flag: `claude -p "Analyze this pull request for security issues"`

**B.** Set the environment variable `CLAUDE_HEADLESS=true` before running the command

**C.** Redirect stdin from `/dev/null`: `claude "Analyze this pull request for security issues" < /dev/null`

**D.** Add the `--batch` flag: `claude --batch "Analyze this pull request for security issues"`

---

**Correct Answer: A**

**Explanation:**

The `-p` (or `--print`) flag is the **documented way to run Claude Code in non-interactive mode**. It processes the prompt, outputs the result to stdout, and exits without waiting for user input—exactly what CI/CD pipelines require.

**Why the other options fail:**

- **Option B** references a non-existent environment variable. Claude Code doesn't have a `CLAUDE_HEADLESS` mode.
- **Option C** uses Unix workarounds that don't properly address Claude Code's command syntax. Redirecting stdin doesn't enable non-interactive mode in Claude Code.
- **Option D** references a non-existent `--batch` flag. Don't confuse this with the Message Batches API, which is a completely different concept.

**Key Takeaway:** For CI/CD pipelines, always use the `-p` flag to run Claude Code in non-interactive, print-to-stdout mode.

**Exam Alignment:**
- **Task 3.6:** Integrate Claude Code into CI/CD pipelines
- **Your Study Note:** D3-06-CI-CD-Integration.md

**Related Sample Scenario:** Claude Code for Continuous Integration (Scenario 5)

---

### Question 11: Batch API Appropriateness & Latency Tolerance

**Difficulty:** Medium | **Domains:** D4.5 (Batch Processing Strategies)  
**Task Statement:** 4.5

Your team wants to reduce API costs for automated analysis. Currently, real-time Claude calls power two workflows: (1) a **blocking pre-merge check** that must complete before developers can merge, and (2) a **technical debt report** generated overnight for review the next morning. Your manager proposes switching both to the Message Batches API for its 50% cost savings. How should you evaluate this proposal?

**A.** Use batch processing for the technical debt reports only; keep real-time calls for pre-merge checks.

**B.** Switch both workflows to batch processing with status polling to check for completion.

**C.** Keep real-time calls for both workflows to avoid batch result ordering issues.

**D.** Switch both to batch processing with a timeout fallback to real-time if batches take too long.

---

**Correct Answer: A**

**Explanation:**

The Message Batches API offers 50% cost savings but has processing times **up to 24 hours with no guaranteed latency SLA**. This makes it unsuitable for blocking pre-merge checks where developers wait for results, but **ideal for overnight batch jobs** like technical debt reports.

**Why the other options fail:**

- **Option B** is wrong because relying on "often faster" completion isn't acceptable for blocking workflows. You'd need a fallback, not just polling.
- **Option C** reflects a misconception—batch results can be correlated using `custom_id` fields. Result ordering isn't the limitation; latency is.
- **Option D** adds unnecessary complexity. The simpler solution is matching each API to its appropriate use case: real-time for blocking, batch for latency-tolerant.

**Key Takeaway:** **Match API to workflow:**
- **Real-time API:** Blocking workflows (developers waiting), strict latency SLAs
- **Batch API:** Non-blocking workflows (overnight jobs, weekly reports), latency-tolerant

**Exam Alignment:**
- **Task 4.5:** Design efficient batch processing strategies
- **Your Study Note:** D4-05-Batch-Processing-Patterns.md

**Related Sample Scenario:** Claude Code for Continuous Integration (Scenario 5)

---

### Question 12: Multi-Pass Review to Address Attention Dilution

**Difficulty:** High | **Domains:** D4.6 (Multi-Instance & Multi-Pass Review)  
**Task Statement:** 4.6

A pull request modifies 14 files across the stock tracking module. Your single-pass review analyzing all files together produces inconsistent results: detailed feedback for some files but superficial comments for others, obvious bugs missed, and contradictory feedback—flagging a pattern as problematic in one file while approving identical code elsewhere in the same PR. How should you restructure the review?

**A.** Split into focused passes: analyze each file individually for local issues, then run a separate integration-focused pass examining cross-file data flow.

**B.** Require developers to split large PRs into smaller submissions of 3-4 files before the automated review runs.

**C.** Switch to a higher-tier model with a larger context window to give all 14 files adequate attention in one pass.

**D.** Run three independent review passes on the full PR and only flag issues that appear in at least two of the three runs.

---

**Correct Answer: A**

**Explanation:**

Splitting reviews into focused passes directly addresses the **root cause: attention dilution** when processing many files at once. File-by-file analysis ensures consistent depth, while a separate integration pass catches cross-file issues.

**Why the other options fail:**

- **Option B** shifts burden to developers without improving the review system. You're externalizing the problem, not solving it.
- **Option C** misunderstands the root cause. Larger context windows don't solve attention quality issues—they might even worsen it (more tokens to process per file). The problem is attentional focus, not context size.
- **Option D** would actually suppress detection of real bugs by requiring consensus on issues that may only be caught intermittently. It trades false positives for false negatives.

**Key Takeaway:** For large multi-file reviews:
1. **Local analysis pass:** Deep review of each file individually
2. **Integration pass:** Cross-file data flow, dependency analysis, contradiction detection

This prevents attention dilution and ensures consistency.

**Exam Alignment:**
- **Task 4.6:** Design multi-instance and multi-pass review architectures
- **Your Study Note:** D4-06-Multi-Instance-Multi-Pass-Review.md

**Related Sample Scenario:** Claude Code for Continuous Integration (Scenario 5)

---

## 📊 Quick Reference Table

| Q# | Scenario | Domain | Task | Difficulty | Topic |
|---|----------|--------|------|-----------|-------|
| **1** | Customer Support | D1 | 1.1, 1.5 | Medium | Tool sequencing + hooks |
| **2** | Customer Support | D2 | 2.1 | Medium | Tool descriptions |
| **3** | Customer Support | D5 | 5.2 | Medium | Escalation criteria |
| **4** | Claude Code | D3 | 3.1, 3.2 | Easy | Command scope |
| **5** | Claude Code | D3 | 3.4 | Medium | Plan mode vs execution |
| **6** | Claude Code | D3 | 3.3 | Medium | Glob patterns + rules |
| **7** | Multi-Agent | D1 | 1.2 | Medium | Decomposition diagnosis |
| **8** | Multi-Agent | D5, D2 | 5.3, 2.2 | Medium | Error propagation |
| **9** | Multi-Agent | D2, D1 | 2.3, 1.2 | High | 80/20 tool allocation |
| **10** | CI/CD | D3 | 3.6 | Easy | -p flag |
| **11** | CI/CD | D4 | 4.5 | Medium | Batch API appropriateness |
| **12** | CI/CD | D4 | 4.6 | High | Multi-pass review |

---

## 🎯 Study Strategy

### By Difficulty Level

**Easy (2 questions):** Q4, Q10
- Foundational knowledge, command syntax
- Recommended for warm-up review

**Medium (7 questions):** Q1, Q2, Q3, Q5, Q6, Q7, Q8, Q11
- Conceptual understanding + design judgment
- Majority of exam

**High (3 questions):** Q9, Q12
- Complex tradeoffs, multi-dimensional analysis
- Require deep domain knowledge

### By Domain

**Domain 1 (Agentic Architecture):** Q1, Q5, Q7, Q9
- Focus on coordinator patterns and tool sequencing

**Domain 2 (Tool Design & MCP):** Q2, Q8, Q9
- Focus on descriptions, error handling, distribution

**Domain 3 (Claude Code Config):** Q4, Q5, Q6, Q10
- Focus on CLAUDE.md hierarchy, CLI, plan mode

**Domain 4 (Prompt Engineering):** Q11, Q12
- Focus on batch API and review architecture

**Domain 5 (Context Management & Reliability):** Q3, Q8
- Focus on escalation and error propagation

### Practice Recommendations

1. **First Pass:** Answer all 12 questions in 24 minutes (2 min/Q). Don't overthink.
2. **Review:** Read all explanations. Note which options you eliminated and why.
3. **Domain Drill:** Pick your weakest domain and re-answer only those questions.
4. **Timed Mock:** Answer Q1-6 in 12 minutes (blocking vs cognitive tasks don't follow linear timing).
5. **Final Review:** Re-read any Q where you weren't 100% certain of the answer.

---

## 📚 Cross-Reference to Study Materials

Each question links directly to relevant study notes:

- **Q1, Q7:** D1-02-Multi-Agent-Orchestration.md, D1-03-Hooks-Workflows-Sessions.md
- **Q2:** D2-01-Tool-Description-Best-Practices.md
- **Q3:** D5-02-Escalation-Patterns.md, D5-04-Escalation-Patterns-Error-Propagation.md
- **Q4:** D3-01-CLAUDE-md-Hierarchy.md, D3-02-Custom-Slash-Commands-Skills.md
- **Q5:** D3-04-Plan-Mode-vs-Direct-Execution.md
- **Q6:** D3-03-Path-Specific-Rules.md
- **Q8:** D5-04-Escalation-Patterns-Error-Propagation.md, D2-02-Structured-Error-Responses.md
- **Q9:** D2-03-Tool-Distribution-Scoping.md
- **Q10:** D3-06-CI-CD-Integration.md
- **Q11:** D4-05-Batch-Processing-Patterns.md
- **Q12:** D4-06-Multi-Instance-Multi-Pass-Review.md

---

**Source Document:** Claude Certified Architect – Foundations Exam Guide v1.0 (July 2026)  
**Prepared:** July 15, 2026  
**Status:** ✅ All 12 official sample questions extracted with detailed explanations
