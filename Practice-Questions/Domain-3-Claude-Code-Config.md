# Domain 3: Claude Code Configuration — Practice Quiz

## Instructions
- 25 scenario-based questions
- Select the BEST answer for each
- Answers with explanations at the bottom
- Time target: 30 minutes

---

## Questions

### Q1
A project has CLAUDE.md files at: `~/.claude/CLAUDE.md`, `./CLAUDE.md`, `./src/CLAUDE.md`, and `./.claude/CLAUDE.md`. When Claude edits a file in `/src/`, which CLAUDE.md files are loaded?

A) Only `./src/CLAUDE.md`  
B) All four files (they are additive)  
C) `./CLAUDE.md` and `./src/CLAUDE.md` only  
D) The most specific one wins: `./src/CLAUDE.md`  

---

### Q2
A team's root CLAUDE.md is 8,000 tokens. In a 20-iteration agentic loop, how many tokens are consumed JUST by CLAUDE.md content?

A) 8,000 (loaded once)  
B) 160,000 (loaded every iteration)  
C) 8,000 per call but cached automatically  
D) It depends on whether prompt caching is enabled  

---

### Q3
A developer adds `"allow": ["Bash(*)"]` to the project's `.claude/settings.json`. What is the security implication?

A) Only their machine is affected  
B) ALL team members can now run ANY bash command without permission prompts, including `rm -rf /`  
C) It only allows read-only bash commands  
D) It's overridden by the deny list  

---

### Q4
A PreToolUse hook exits with code 2 and prints "Blocked: potential secret detected". What happens next?

A) The tool executes anyway, but the message is logged  
B) The tool execution is BLOCKED, and Claude receives the hook's stdout as context  
C) The entire session terminates  
D) The user is prompted to approve or deny  

---

### Q5
Settings exist at all three levels: `~/.claude/settings.json` allows `Bash(git push *)`, `.claude/settings.json` denies `Bash(git push *)`, `.claude/settings.local.json` allows `Bash(git push *)`. Which takes effect?

A) Denied — project settings are authoritative for team safety  
B) Allowed — local settings have highest priority and override project deny  
C) Conflict error — contradictory rules cause a failure  
D) User is prompted each time due to conflicting rules  

---

### Q6
A slash command file `.claude/commands/deploy.md` contains: `"Run the deployment pipeline for $ARGUMENTS environment"`. A user types `/project:deploy staging`. What prompt does Claude receive?

A) "Run the deployment pipeline for staging environment"  
B) "deploy staging"  
C) The entire file contents plus "staging" appended  
D) An error because $ARGUMENTS must be quoted  

---

### Q7
A project has TypeScript in `/frontend` and Go in `/backend`. Each has different test commands, linting rules, and naming conventions. What is the MOST maintainable configuration?

A) Put everything in the root CLAUDE.md with conditional statements  
B) Subdirectory CLAUDE.md files: `/frontend/CLAUDE.md` with TS rules, `/backend/CLAUDE.md` with Go rules  
C) Two separate projects with their own `.claude/` directories  
D) A single comprehensive CLAUDE.md covering both languages  

---

### Q8
Claude is asked to refactor a payment processing module (12 files, handles real transactions). Should Claude use plan mode or direct execution?

A) Direct execution — experienced developers trust Claude  
B) Plan mode — high-risk domain, multi-file changes, multiple approaches possible  
C) It doesn't matter — both produce the same result  
D) Direct execution with extra safety checks enabled  

---

### Q9
A PostToolUse hook runs `npx prettier --write "$FILE_PATH"` after every Write operation. Claude writes a Python file. What happens?

A) Prettier formats the Python file (may break it)  
B) Prettier ignores non-JS files automatically  
C) The hook should check the file extension before running  
D) Nothing — PostToolUse hooks can't modify files  

---

### Q10
Which information should go in CLAUDE.md vs. memory files?

A) CLAUDE.md: project conventions; Memory: user preferences  
B) CLAUDE.md: everything about the project; Memory: nothing  
C) CLAUDE.md: ephemeral task details; Memory: permanent rules  
D) Both contain the same information for redundancy  

---

### Q11
A developer's CLAUDE.md says "always use single quotes in JavaScript." The project's `.eslintrc` enforces double quotes. Claude writes code with single quotes that fails linting. What's the fix?

A) Update CLAUDE.md to match .eslintrc (double quotes)  
B) Update .eslintrc to match CLAUDE.md (single quotes)  
C) Add a PostToolUse hook to auto-fix lint errors  
D) Remove the conflicting rule from CLAUDE.md  

---

### Q12
A critical security rule ("never commit .env files") is ONLY documented in a user message from 50 messages ago. After context compaction, what happens?

A) The rule is preserved in the compaction summary  
B) The rule may be lost — critical rules should be in CLAUDE.md or system prompt  
C) Claude always remembers security rules regardless of compaction  
D) Settings deny rules handle this case  

---

### Q13
What is the STRONGEST enforcement mechanism in Claude Code?

A) CLAUDE.md instructions  
B) System prompt constraints  
C) PreToolUse hooks (non-zero exit blocks execution)  
D) Settings deny list  

---

### Q14
A team member creates `/infrastructure/production/CLAUDE.md` with the rule: "NEVER modify files in this directory without explicit human confirmation." Is this sufficient protection?

A) Yes — CLAUDE.md instructions are always followed  
B) No — CLAUDE.md is an instruction Claude interprets; use PreToolUse hooks for hard enforcement  
C) Yes — but only if it's the most specific CLAUDE.md  
D) No — CLAUDE.md in subdirectories are never loaded  

---

### Q15
Plan mode is active. Claude has analyzed the codebase and produced a 10-step plan. The user says "step 4 seems risky, can you try a different approach?" What should Claude do?

A) Execute steps 1-3 and skip step 4  
B) Revise the plan with an alternative for step 4, present the updated plan for approval  
C) Exit plan mode and execute directly  
D) Explain why step 4 is necessary and proceed  

---

### Q16
A permission pattern `"Write(src/**)"` is in the allow list. Claude tries to write to `src/../secrets/api.key`. Should this be allowed?

A) Yes — `src/..` starts with `src/`  
B) No — path traversal via `..` exits the allowed directory; paths should be normalized before matching  
C) It depends on the operating system  
D) The Write tool doesn't accept relative paths  

---

### Q17
A custom slash command `/project:review` includes detailed review criteria. How is this different from putting the same criteria in CLAUDE.md?

A) No difference — both are always active  
B) Slash command: applied ON DEMAND when user types `/project:review`; CLAUDE.md: ALWAYS active in every interaction  
C) Slash commands override CLAUDE.md rules  
D) CLAUDE.md criteria only apply during reviews  

---

### Q18
A hook command `"bash validate.sh"` takes 45 seconds to run (it runs a full test suite). Claude appears frozen. What's the problem and fix?

A) The hook is too slow — set a timeout and use PostToolUse for heavy validation  
B) Hooks always run asynchronously, so this shouldn't happen  
C) Move the test suite to a Stop hook instead  
D) This is expected behavior — hooks are important  

---

### Q19
`.claude/settings.json` contains `"env": {"NODE_ENV": "development"}`. What does this do?

A) Sets NODE_ENV for the entire system  
B) Sets NODE_ENV in the environment for all tool executions within Claude Code  
C) Exports NODE_ENV to the user's shell  
D) Passes NODE_ENV to MCP servers only  

---

### Q20
A project has 5 CLAUDE.md files totaling 6,000 tokens. To reduce token cost, which is the BEST approach?

A) Combine all into one root CLAUDE.md  
B) Delete all subdirectory CLAUDE.md files  
C) Keep subdirectory files (they load on-demand) and trim root to essentials only  
D) Move all content to settings.json  

---

### Q21
Claude is working on a simple typo fix in README.md. Should it enter plan mode?

A) Yes — always plan before any change  
B) No — trivial change, zero risk, direct execution appropriate  
C) Only if the user explicitly requests it  
D) It depends on the file type  

---

### Q22
A hook `"matcher": "Bash(npm *)"` is configured. Claude runs `npx prettier --write file.ts`. Does the hook fire?

A) Yes — npx is an npm command  
B) No — the matcher is `npm *` which doesn't match `npx prettier --write file.ts`  
C) Yes — all Node.js commands trigger npm matchers  
D) It depends on the shell alias configuration  

---

### Q23
A team wants to enforce that Claude never pushes to `main` branch. Which configuration is MOST reliable?

A) Add "never push to main" in CLAUDE.md  
B) `"deny": ["Bash(git push * main*)", "Bash(git push origin main*)"]` in project settings  
C) PreToolUse hook that parses git push commands and blocks pushes to main  
D) Both B and C together (defense in depth)  

---

### Q24
`.claude/settings.local.json` exists but is empty (`{}`). What effect does this have?

A) It overrides all project settings with empty values  
B) No effect — empty settings don't override anything  
C) It blocks all tool permissions  
D) It causes a configuration error  

---

### Q25
A plan includes modifying a database migration file that already exists in production. The plan says "risk: medium." Should the human approve based on the plan?

A) Yes — the agent assessed medium risk, which is acceptable  
B) No — the human should independently verify: is this migration reversible? Has it been applied to production? What data is affected?  
C) Yes — if the plan is well-structured, it's safe to proceed  
D) No — modifying existing migrations is always forbidden  

---

### Q26 (NEW: Domain 3.6 - CLI Flags)
Your CI/CD pipeline runs:
```bash
claude "Analyze this pull request for security issues"
```
The job hangs indefinitely, and the log shows "Waiting for user input." What is the issue and the fix?

A) Claude Code isn't installed in the CI environment  
B) The -p flag is missing; Claude needs non-interactive mode: `claude -p "Analyze..."`  
C) The pipeline needs to set CLAUDE_INTERACTIVE=false environment variable  
D) The review needs to be in plan mode with --plan flag  

---

### Q27 (NEW: Domain 3.6 - JSON Schema Output)
Your CI pipeline needs to parse code review findings and post them as GitHub comments. You run:
```bash
claude -p "Review this code for bugs" > findings.txt
```
The output is prose like "I found 3 bugs: the first is...". Parsing this is fragile. What should you use instead?

A) `--format json` flag to produce structured JSON  
B) `--output-format json --json-schema SCHEMA` to enforce schema validation  
C) `--parse json` to automatically convert output to JSON  
D) The output cannot be structured; parse the prose using regex  

---

### Q28 (NEW: Domain 3.6 - JSON Schema Enforcement)
You're extracting security findings with a JSON schema. The schema has required fields: `[severity, file, line, issue]`. Claude's response doesn't include a `line` field for one finding. What happens?

A) Claude fabricates a line number  
B) The tool_use response fails schema validation; Claude retries with correction guidance  
C) The field is omitted from the output (null)  
D) The entire batch fails and must be resubmitted  

---

### Q29 (NEW: Domain 3.6 - CI Session Isolation)
Your CI pipeline runs code review twice: once when code is committed (generator context), once when it's merged (reviewer context). The reviewer instance occasionally misses bugs the generator instance caught. Why?

A) The CI environment is non-deterministic  
B) Session context isolation: the reviewer shouldn't see the generator's reasoning—they need an independent perspective  
C) The reviewer is using a cheaper model  
D) The database cache isn't invalidated between runs  

---

### Q30 (NEW: Domain 3.6 - Structured Output in CI)
Your pipeline needs to:
1. Run review with --json-schema
2. Parse findings
3. Post to GitHub API

What is the CORRECT command structure?

A) `claude -p --output-format json "Review..." | jq '.findings[]'`  
B) `claude -p --output-format json --json-schema 'findings_schema.json' "Review..."` then parse JSON  
C) `claude --print --json-schema --output-format text "Review..."`  
D) `claude --batch "Review..."` (batches handle formatting automatically)  

---

---

## Answers

### A1: **B**
CLAUDE.md files are ADDITIVE — all applicable files are loaded simultaneously. When editing in `/src/`, all four files apply: global user preferences, project-wide rules, src-specific rules, and local project overrides. They combine, not override.

### A2: **D**
Without prompt caching, CLAUDE.md content (part of system prompt) is sent every iteration: 8,000 × 20 = 160,000 tokens. WITH prompt caching enabled: first call is full price, subsequent 19 calls at 10% = 8,000 + 19 × 800 = ~23,200 effective tokens. The answer depends on caching configuration.

### A3: **B**
`.claude/settings.json` is git-tracked and shared with ALL team members. `"Bash(*)"` allows any shell command without prompts, including destructive ones. This is a dangerous team-wide configuration. Should never be in shared settings.

### A4: **B**
PreToolUse hooks with non-zero exit BLOCK tool execution. The hook's stdout ("Blocked: potential secret detected") is fed back to Claude as context, so Claude understands WHY the tool was blocked and can try a different approach.

### A5: **B**
Settings priority: local > project > global. `.claude/settings.local.json` has the highest priority and overrides the project deny. This is a security concern — project-level deny should be enforced via hooks (which can't be locally overridden) for critical rules.

### A6: **A**
`$ARGUMENTS` is replaced with whatever the user types after the command name. `/project:deploy staging` → "Run the deployment pipeline for staging environment." The substitution is direct and literal.

### A7: **B**
Subdirectory CLAUDE.md files are the correct approach: each directory gets rules relevant to its language/framework. They only load when Claude works in that directory (token-efficient), and each module's rules are co-located with its code (maintainable).

### A8: **B**
Plan mode is essential here: 12 files = high complexity, payment processing = high risk (bugs = money), and there are multiple valid refactoring approaches. Plan mode lets the human review the scope, approach, and risks before any changes happen.

### A9: **A/C**
Prettier will attempt to format the Python file, likely producing invalid Python. The hook SHOULD check the file extension before running language-specific formatters. Correct hook: `if [[ "$FILE_PATH" == *.ts || "$FILE_PATH" == *.js ]]; then npx prettier --write "$FILE_PATH"; fi`

### A10: **A**
CLAUDE.md: project conventions, tech stack, coding rules (needed every session, loaded automatically). Memory: user preferences, role, working style (personal, carried across projects). They serve different purposes — CLAUDE.md is project knowledge, memory is user knowledge.

### A11: **A (or D)**
CLAUDE.md should not contradict the project's actual tooling configuration. If .eslintrc enforces double quotes, CLAUDE.md should match (or not mention quotes at all, since the linter handles it). Remove conflicting/redundant rules.

### A12: **B**
Context compaction may summarize or drop messages from 50 turns ago. Critical rules that must ALWAYS apply should be in CLAUDE.md (always loaded) or system prompt. Relying on message history for security-critical rules is an anti-pattern.

### A13: **C**
PreToolUse hooks are the strongest: they execute external code (not Claude's interpretation), and a non-zero exit physically blocks the tool from executing. Settings deny (D) can be locally overridden. CLAUDE.md/system prompt (A/B) are instructions Claude interprets (soft).

### A14: **B**
CLAUDE.md is an instruction that Claude generally follows, but it's not a hard guarantee. For critical infrastructure protection, use a PreToolUse hook that checks `$FILE_PATH` and blocks writes to `/infrastructure/production/**`. Hooks are code-enforced; CLAUDE.md is Claude-interpreted.

### A15: **B**
In plan mode, the user's feedback refines the plan before execution begins. Claude should present an updated plan with an alternative approach for step 4, await approval, then execute the revised plan. This is the core value of plan mode — alignment before action.

### A16: **B**
Path traversal (`src/../secrets/`) escapes the intended directory scope. Paths must be normalized (resolve `..`) before matching against allow patterns. Without normalization, permission patterns are bypassable. This is a security vulnerability.

### A17: **B**
Key distinction: CLAUDE.md criteria are ALWAYS active (every message, every action). Slash commands are ON-DEMAND — only triggered when the user explicitly types `/project:review`. Use CLAUDE.md for rules that should always apply; commands for workflows triggered explicitly.

### A18: **A**
Hooks run synchronously — Claude waits for completion. A 45-second hook makes Claude appear frozen for 45 seconds on every tool call. Fix: set a timeout (e.g., 10s), and move heavy validation to Stop hooks (run after Claude finishes) or make the hook more targeted.

### A19: **B**
`"env"` in settings.json sets environment variables available to all tool executions (Bash commands, MCP servers) within Claude Code. It doesn't affect the user's system-wide shell environment.

### A20: **C**
Subdirectory CLAUDE.md files only load when Claude accesses those directories — they're already on-demand (token-efficient). The biggest win is trimming the ROOT CLAUDE.md (always loaded) to essentials. Subdirectory files add minimal cost since they only appear in relevant contexts.

### A21: **B**
Plan mode is for complex, risky, or ambiguous tasks. A typo fix in README is: single file, zero risk, one obvious approach, trivially reversible. Direct execution is appropriate — plan mode would just add unnecessary friction.

### A22: **B**
The matcher `"Bash(npm *)"` matches Bash calls where the command starts with "npm" followed by anything. `npx prettier --write file.ts` starts with "npx", not "npm", so it does NOT match. You'd need `"Bash(npx *)"` separately.

### A23: **D**
Defense in depth: settings deny (B) provides the first line of defense (pattern matching). PreToolUse hook (C) provides programmable enforcement that can parse complex git commands, check branch names, and handle edge cases. Both together give layered protection.

### A24: **B**
An empty JSON object `{}` has no settings defined — it doesn't override anything. Settings merge: keys present in higher-priority files override lower-priority; absent keys pass through from lower-priority files.

### A25: **B**
The human should NOT blindly trust the agent's risk assessment. For database migrations: Is this migration already applied in production? (Modifying applied migrations is dangerous). Is it reversible? What data would be affected? How many rows? The agent provides information; the human makes the judgment call.

### A26: **B**
Claude Code defaults to interactive mode, waiting for user permission prompts. In CI/CD, there's no user to respond. The `-p` (or `--print`) flag runs in non-interactive mode: processes the prompt, outputs results, exits. This is the documented way to run Claude in pipelines.

### A27: **B**
`--output-format json` alone doesn't guarantee schema compliance (Claude could return invalid JSON). The `--json-schema` flag enforces the schema, eliminates JSON syntax errors, and guarantees structured output for programmatic parsing. Together: `--output-format json --json-schema SCHEMA` ensures both format and validation.

### A28: **B**
Strict JSON schema validation with tool_use ensures valid structure. If a required field is missing, Claude's response fails schema validation, and Claude retries with the validation error included in the prompt. Claude self-corrects and re-attempts until it produces valid output or the retry limit is hit.

### A29: **B**
Session context isolation is intentional: the reviewer instance shouldn't inherit the generator's reasoning context. A fresh instance (without prior reasoning) reviews with independent perspective and catches bugs the generator missed due to biases. This is why multi-instance review is more effective.

### A30: **B**
Complete command: `claude -p --output-format json --json-schema 'findings_schema.json' "Review this code..."` produces validated JSON output that can be piped to `jq` for parsing and GitHub API posting. The flags work together: `-p` (non-interactive), `--output-format json` (format), `--json-schema` (validation).
