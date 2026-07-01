# Domain 3 | Topic 3.2: Custom Slash Commands & Skills

## 1. Core Concepts

### What Are Custom Slash Commands?

Custom slash commands are user-defined shortcuts that execute predefined prompts or workflows in Claude Code. They're invoked with `/<name>` and can accept arguments.

### Where They Live

```
.claude/commands/              ← Project-level (shared via git)
├── review.md                  ← /project:review
├── deploy-check.md            ← /project:deploy-check
└── generate-tests.md          ← /project:generate-tests

~/.claude/commands/            ← User-level (personal)
├── explain.md                 ← /user:explain
├── refactor.md                ← /user:refactor
└── quick-test.md              ← /user:quick-test
```

### Command File Format

A command file is a Markdown file where the content becomes the prompt sent to Claude:

```markdown
<!-- .claude/commands/review.md -->
Review the current git diff for:
1. Potential bugs or logic errors
2. Security vulnerabilities (OWASP top 10)
3. Performance issues
4. Missing error handling
5. Test coverage gaps

Focus on critical issues first. Format as a numbered list with severity (HIGH/MEDIUM/LOW).
```

### Command Arguments

Commands can accept arguments using `$ARGUMENTS` placeholder:

```markdown
<!-- .claude/commands/explain.md -->
Explain the following code/concept in detail, suitable for a senior engineer:

$ARGUMENTS

Include:
- What it does
- Why it's implemented this way
- Potential improvements
- Edge cases to be aware of
```

Usage: `/user:explain the useEffect cleanup pattern in React`

### Skills (Built-in Commands)

**Skills** are Anthropic-maintained commands with specialized behaviors:

**Code Quality & Review:**
- `/code-review` — Review the current diff for correctness bugs and cleanups (supports `--comment` and `--fix` flags)
- `/simplify` — Review changed code for reuse, simplification, and efficiency, then apply fixes
- `/security-review` — Complete security review of pending changes on the current branch

**Research & Verification:**
- `/deep-research` — Fan-out web searches, fetch sources, adversarially verify claims, synthesize a cited report
- `/verify` — Verify that a code change actually works by running the app and observing behavior
- `/run` — Launch and drive the project's app to see a change working

**Project & PR Management:**
- `/init` — Initialize a new CLAUDE.md file with codebase documentation
- `/review` — Review a GitHub pull request (for remote PRs; use `/code-review` for local diff)
- `/loop` — Run a prompt or slash command on a recurring interval (e.g., `/loop 5m /foo`)

---

## 2. Design Patterns

### Pattern 1: Code Review Command

```markdown
<!-- .claude/commands/review.md -->
Perform a thorough code review of the current changes (git diff HEAD).

Check for:
- [ ] Logic errors and edge cases
- [ ] Security issues (injection, auth, data exposure)
- [ ] Performance concerns (N+1 queries, unnecessary allocations)
- [ ] Error handling completeness
- [ ] Naming clarity and code readability
- [ ] Test coverage for new functionality

For each issue found, provide:
- Severity: CRITICAL / HIGH / MEDIUM / LOW
- File and line reference
- Specific concern
- Suggested fix

If no issues found, confirm the code looks good.
```

### Pattern 2: Test Generation Command

```markdown
<!-- .claude/commands/generate-tests.md -->
Generate comprehensive tests for: $ARGUMENTS

Requirements:
- Follow existing test patterns in this project
- Include happy path, edge cases, and error cases
- Use the project's testing framework and conventions
- Mock external dependencies
- Include descriptive test names that explain the scenario
- Add comments for non-obvious test cases

Place tests in the appropriate test file following project structure.
```

### Pattern 3: Documentation Command

```markdown
<!-- .claude/commands/document.md -->
Generate documentation for: $ARGUMENTS

Include:
- Purpose and responsibility
- Public API with parameter descriptions
- Usage examples
- Error cases and how they're handled
- Dependencies and requirements

Format as JSDoc/docstring comments in the source file.
```

### Pattern 4: Pre-Deployment Check

```markdown
<!-- .claude/commands/deploy-check.md -->
Run pre-deployment verification:

1. Check for uncommitted changes (git status)
2. Run the full test suite
3. Check for TODO/FIXME/HACK comments in changed files
4. Verify no console.log/print statements in production code
5. Check that environment variables are properly configured
6. Verify database migrations are up to date

Report: READY TO DEPLOY or BLOCKERS FOUND (with details).
```

---

## 3. Implementation Details

### Command Discovery

```
/project:<name>    → .claude/commands/<name>.md (project commands)
/user:<name>       → ~/.claude/commands/<name>.md (personal commands)
```

- Tab completion available for discovering commands
- Commands are automatically discovered from the filesystem
- No registration needed — just create the file
- Commands can be nested in subdirectories using colon separators:

```
.claude/commands/db/migrate.md     → /project:db:migrate
.claude/commands/db/seed.md        → /project:db:seed
.claude/commands/api/generate.md   → /project:api:generate
~/.claude/commands/git/squash.md   → /user:git:squash
```

### Naming Conventions

| Convention | Example | Use |
|-----------|---------|-----|
| Verb form | `review.md`, `deploy.md` | Action commands |
| Noun form | `status.md`, `checklist.md` | Information commands |
| Hyphenated | `deploy-check.md`, `pre-commit.md` | Multi-word names |

### Arguments Handling

```markdown
<!-- With $ARGUMENTS -->
Refactor $ARGUMENTS to improve readability and reduce complexity.

<!-- Usage -->
/project:refactor the authentication middleware

<!-- Claude receives -->
Refactor the authentication middleware to improve readability and reduce complexity.
```

If `$ARGUMENTS` is not in the template but the user provides args, they're appended after the template content.

### Command vs. Hook vs. CLAUDE.md

| Mechanism | Purpose | Trigger |
|-----------|---------|---------|
| CLAUDE.md | Persistent context/rules | Every session, automatic |
| Slash Commands | On-demand workflows | User explicitly types `/command` |
| Hooks | Automatic enforcement | Tool events (PreToolUse, etc.) |

---

## 4. Security & Reliability

### Command Security

| Risk | Mitigation |
|------|-----------|
| Malicious commands in shared repo | Review `.claude/commands/` in PRs |
| Commands that execute destructive ops | Commands are prompts, not scripts — Claude still needs permission |
| Overly broad commands | Scope commands narrowly |
| Outdated commands referencing wrong tools | Regular command maintenance |

### Important: Commands Are Prompts, Not Scripts

- Commands inject text into Claude's prompt
- Claude still makes decisions based on the prompt
- Tool permissions still apply
- Claude can refuse if the command asks for something unsafe

### Best Practices

1. **Keep commands focused** — one purpose per command
2. **Include success criteria** — what does "done" look like?
3. **Specify output format** — structured output is easier to verify
4. **Version control project commands** — they're team knowledge
5. **Document purpose** — add a comment at the top of each command file

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A team wants every PR to go through the same 5-point review checklist. Where should this be configured and how?

**A:** Create a **project-level slash command**:

File: `.claude/commands/pr-review.md`
```markdown
Review the current PR changes against our checklist:
1. All tests pass (run test suite)
2. No security vulnerabilities introduced
3. Error handling is complete
4. Documentation updated for API changes
5. No hardcoded secrets or credentials

Report pass/fail for each item with details.
```

- Committed to git → all team members have it
- Invoked explicitly → `/project:pr-review`
- Consistent → same checklist every time

### Scenario 2
**Q:** A developer wants a personal shortcut that generates commit messages in their preferred style. Should this be a project command or user command?

**A:** **User command** (`~/.claude/commands/commit-msg.md`) because:
- It's a personal preference, not a team standard
- Other team members may have different styles
- It shouldn't be committed to the project repo

```markdown
Generate a commit message for the current staged changes.
Style: conventional commits format (type(scope): description)
Keep the subject line under 50 characters.
Add a body with bullet points if changes are complex.
```

### Scenario 3
**Q:** What is the difference between putting review criteria in CLAUDE.md vs. in a slash command?

**A:**
- **CLAUDE.md:** Review criteria are ALWAYS active. Claude applies them automatically whenever reviewing code, even if not explicitly asked.
- **Slash command:** Review criteria are applied ON DEMAND. Only triggered when user types `/project:review`.

Use CLAUDE.md for rules that should ALWAYS be followed (conventions, constraints). Use commands for workflows that are explicitly triggered (reviews, deployments, generations).

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Huge command files (1000+ words) | Overwhelming, buried instructions | Keep focused and concise |
| Commands duplicating CLAUDE.md rules | Redundancy, potential conflict | Reference CLAUDE.md, don't repeat |
| Personal preferences in project commands | Imposed on team | Use user-level commands |
| No arguments support when useful | Less flexible | Include $ARGUMENTS for parameterized commands |
| Commands without clear output format | Inconsistent results | Specify expected format |
| Never updating outdated commands | Wrong instructions applied | Regular maintenance |

---

## 7. Quick Reference / Cheat Sheet

```
COMMAND LOCATIONS
━━━━━━━━━━━━━━━━
Project: .claude/commands/<name>.md → /project:<name>
User:    ~/.claude/commands/<name>.md → /user:<name>

FILE FORMAT
━━━━━━━━━━━
Just markdown text — becomes Claude's prompt
Use $ARGUMENTS for parameterized input
No frontmatter needed

BUILT-IN SKILLS
━━━━━━━━━━━━━━━
Code Quality:     /code-review, /simplify, /security-review
Research/Verify:  /deep-research, /verify, /run
Project/PR:       /init, /review, /loop

SUBDIRECTORY COMMANDS
━━━━━━━━━━━━━━━━━━━━━
.claude/commands/db/migrate.md → /project:db:migrate
~/.claude/commands/git/squash.md → /user:git:squash

WHEN TO USE WHAT
━━━━━━━━━━━━━━━━
CLAUDE.md  → Always-on rules and context
Commands   → On-demand workflows (user triggers)
Hooks      → Automated enforcement (event triggers)

COMMAND DESIGN CHECKLIST
━━━━━━━━━━━━━━━━━━━━━━━━
✓ Single clear purpose
✓ Specific output format requested
✓ Success criteria defined
✓ Scope appropriate (project vs user)
✓ Reviewed in PRs (if project-level)
✓ $ARGUMENTS used where flexibility needed
```
