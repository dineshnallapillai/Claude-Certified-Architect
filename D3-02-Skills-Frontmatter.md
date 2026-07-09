# Domain 3 | Topic 3.2 (Part 2): Custom Skills with SKILL.md Frontmatter

## 1. Core Concepts

### Skills vs Slash Commands vs CLAUDE.md

| Mechanism | Location | Trigger | Isolation | Tool Control |
|-----------|----------|---------|-----------|-------------|
| **CLAUDE.md** | Root/subdirs | Always loaded | None | None |
| **Slash Commands** | `.claude/commands/` | User types `/name` | None | None |
| **Skills** | `.claude/skills/` | User invokes | `context: fork` | `allowed-tools` |

### What Makes Skills Special

Skills support **frontmatter configuration** that commands don't:

```yaml
---
context: fork              # Run in isolated sub-agent (verbose output hidden)
allowed-tools:             # Hard restriction on available tools
  - Read
  - Grep
  - Glob
argument-hint: "target file or module name"  # Prompt when no args given
---

# Skill instructions below frontmatter
Analyze the provided target for code quality issues...
```

### SKILL.md File Structure

```
.claude/skills/                    ← Project-level (shared via git)
├── analyze/
│   └── SKILL.md
├── generate-migration/
│   └── SKILL.md
└── security-audit/
    └── SKILL.md

~/.claude/skills/                  ← User-level (personal, NOT shared)
├── my-quick-review/
│   └── SKILL.md
└── explore-code/
    └── SKILL.md
```

---

## 2. Frontmatter Options (Exam-Critical)

### context: fork

Runs the skill in an **isolated sub-agent context**. Verbose intermediate output (file reads, grep results, analysis chains) stays in the fork — only the final result returns to the main conversation.

```yaml
---
context: fork
---
```

**Without fork:** All tool calls and intermediate reasoning accumulate in the main conversation context → context pollution for subsequent questions.

**With fork:** Skill executes in isolation → only the summary/result returns → main context stays clean.

```
WITHOUT FORK:                      WITH FORK:
┌─────────────────────┐           ┌─────────────────────┐
│ Main Context        │           │ Main Context        │
│ [30 file reads]     │           │                     │
│ [15 grep results]   │           │ "Summary: ..."      │ ← Only this returns
│ [analysis chains]   │           │                     │
│ "Summary: ..."      │           └─────────────────────┘
│ ← Context is FULL → │                    ▲
└─────────────────────┘           ┌────────┴────────────┐
                                  │ Fork (isolated)     │
                                  │ [30 file reads]     │
                                  │ [15 grep results]   │
                                  │ [analysis chains]   │
                                  └─────────────────────┘
```

**Use `context: fork` for:**
- Codebase analysis (reads many files)
- Exploratory skills (verbose discovery)
- Brainstorming/alternatives (don't want to pollute main)
- Any skill that produces verbose intermediate output

### allowed-tools

**Hard restriction** on which tools the skill can access. Not prompt-based — the skill literally cannot call tools not in the list.

```yaml
---
allowed-tools:
  - Read
  - Grep
  - Glob
---
```

Common configurations:

| Profile | Tools | Use Case |
|---------|-------|----------|
| Read-only | `[Read, Grep, Glob]` | Analysis, exploration |
| Read + Write | `[Read, Write, Edit, Grep, Glob]` | Code generation |
| Full minus network | `[Read, Write, Edit, Bash, Grep, Glob]` | Local operations |
| Omit field entirely | All tools available | Full-power skills |

**Security value:** Prevents skills from:
- Running destructive Bash commands (if Bash excluded)
- Making network requests (if Bash excluded)
- Modifying files (if Write/Edit excluded)
- Making git commits or pushes (if Bash excluded)

### argument-hint

Prompts the user for required input when they invoke the skill without arguments:

```yaml
---
argument-hint: "file path, module name, or PR number to analyze"
---
```

**Behavior:**
1. User types: `/security-audit` (no arguments)
2. Claude displays: "Please provide: file path, module name, or PR number to analyze"
3. User provides: `src/auth/`
4. Skill executes with that target

---

## 3. Design Patterns

### Pattern 1: Isolated Analysis Skill

```yaml
# .claude/skills/analyze-module/SKILL.md
---
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
argument-hint: "module or directory to analyze"
---

Analyze the specified module comprehensively:
1. Map file structure (Glob)
2. Find entry points (Grep for exports)
3. Trace dependencies (Read imports)
4. Identify patterns and potential issues

Return a structured summary:
- Architecture overview
- Key files and responsibilities
- Dependencies
- Potential issues
```

### Pattern 2: Safe Code Generation Skill

```yaml
# .claude/skills/generate-tests/SKILL.md
---
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
argument-hint: "component or function to generate tests for"
---

Generate unit tests for the specified target.
Follow existing test patterns in this project.
Cover: happy path, edge cases, error cases.
```

**No `context: fork`** — output (test files) needs to be written to the main workspace.
**No Bash** — skill shouldn't run tests or make commits, only generate files.

### Pattern 3: Personal Skill Variant

```yaml
# ~/.claude/skills/quick-review/SKILL.md   ← DIFFERENT name from team's /review
---
allowed-tools:
  - Read
  - Grep
  - Glob
---

Quick review focused ONLY on correctness bugs.
Skip: style, naming, minor suggestions.
Only report issues that would cause incorrect runtime behavior.
```

**Important:** Uses a **different name** (`quick-review` vs `review`) to avoid conflicting with team skills.

---

## 4. Security & Reliability

### Tool Restriction as Security Boundary

```yaml
# DANGEROUS — no restrictions:
---
argument-hint: "files to process"
---
# ← Skill could run `rm -rf`, `git push --force`, etc.

# SAFE — restricted to read-only:
---
allowed-tools:
  - Read
  - Grep
  - Glob
argument-hint: "files to analyze"
---
# ← Cannot modify anything
```

### context: fork as Safety

- Fork prevents skill from seeing/modifying main conversation state
- If skill fails or produces garbage, main context is unaffected
- Tool permissions still apply within the fork

### Personal Skills Risk

- Personal skills (`~/.claude/skills/`) are not code-reviewed
- They still respect tool permissions (deny lists still block)
- Best practice: always include `allowed-tools` even in personal skills

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A codebase analysis skill reads 50+ files, producing extensive output that clutters the conversation. Subsequent questions get worse answers. What fix?

A) Add "be concise" to the skill  
B) Add `context: fork` to the SKILL.md frontmatter  
C) Move the skill logic to CLAUDE.md  
D) Split into 5 smaller skills  

**A:** **B** — `context: fork` isolates the verbose exploration in a sub-agent. Only the final summary returns to the main conversation, keeping context clean.

### Scenario 2
**Q:** A test generation skill should read source files and write test files but MUST NOT run commands, commit, or access the network. How to configure?

A) Add "do not run bash commands" to the skill instructions  
B) Set `allowed-tools: [Read, Write, Edit, Grep, Glob]` in SKILL.md frontmatter  
C) Configure deny rules in settings.json  
D) Use `context: fork` which auto-restricts tools  

**A:** **B** — `allowed-tools` provides a hard restriction. Only listed tools are available. Prompt instructions (A) are probabilistic. Settings deny (C) is global, not skill-specific. Fork (D) isolates context but doesn't restrict tools.

### Scenario 3
**Q:** A skill requires a target path but developers keep invoking it without arguments, causing it to analyze the wrong thing. What frontmatter fixes this?

A) `required-args: true`  
B) `argument-hint: "target file or directory path"`  
C) `prompt-for-input: true`  
D) `default-arg: "."`  

**A:** **B** — `argument-hint` is the correct frontmatter field. When the skill is invoked without arguments, the hint text is displayed, prompting the user to provide the required input.

### Scenario 4
**Q:** A developer wants a personal skill with more relaxed review criteria than the team standard. The team skill is at `.claude/skills/review/SKILL.md`. Correct approach?

A) Edit the team skill locally  
B) Create `~/.claude/skills/relaxed-review/SKILL.md` with a different name  
C) Override with `~/.claude/skills/review/SKILL.md` (same name)  
D) Add personal exceptions to CLAUDE.md  

**A:** **B** — Personal skills go in `~/.claude/skills/` with a **different name** to avoid affecting teammates or creating name conflicts.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Verbose skills without `context: fork` | Pollutes main context | Add `context: fork` |
| No `allowed-tools` on powerful skills | Skill could run destructive commands | Restrict to minimum needed |
| Same name for personal and team skills | Conflicts, confusion | Different names for personal variants |
| Using skills for always-on rules | Skills are on-demand only | CLAUDE.md for always-active |
| Prompt-based tool restriction | Probabilistic, ignorable | `allowed-tools` for hard enforcement |
| Omitting `argument-hint` | Skill runs without needed input | Always add for parameterized skills |
| Using CLAUDE.md for complex workflows | Bloats always-loaded context | Skills for on-demand workflows |

---

## 7. Quick Reference / Cheat Sheet

```
SKILL FILE LOCATION
━━━━━━━━━━━━━━━━━━━
.claude/skills/<name>/SKILL.md   → Project (shared via git)
~/.claude/skills/<name>/SKILL.md → Personal (not shared)

FRONTMATTER (all three are independent, use any combo)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
---
context: fork              # Isolate to sub-agent
allowed-tools:             # Hard tool restriction
  - Read
  - Grep
  - Glob
argument-hint: "what to provide"  # Prompt for args
---

CONTEXT: FORK
━━━━━━━━━━━━━
• Verbose output stays in fork
• Only final result returns to main context
• Use for: analysis, exploration, brainstorming

ALLOWED-TOOLS
━━━━━━━━━━━━━
• HARD restriction (not prompt-based)
• Only listed tools available
• Omit field = all tools
• Read-only: [Read, Grep, Glob]
• Read+Write: [Read, Write, Edit, Grep, Glob]

ARGUMENT-HINT
━━━━━━━━━━━━━
• Shows when user invokes without args
• Tells user what input is expected
• Example: "file path or module name"

DECISION: SKILL vs COMMAND vs CLAUDE.MD
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Always-on rules → CLAUDE.md
Simple prompt template → Command (.claude/commands/)
Complex workflow needing isolation/restriction → Skill (.claude/skills/)

PERSONAL SKILLS
━━━━━━━━━━━━━━━
• ~/.claude/skills/ (NOT in git)
• Use DIFFERENT NAME from team skills
• Same frontmatter options available
```
