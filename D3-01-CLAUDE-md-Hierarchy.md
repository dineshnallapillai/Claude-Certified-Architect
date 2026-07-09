# Domain 3 | Topic 3.1: CLAUDE.md Hierarchy & Management

## 1. Core Concepts

### What is CLAUDE.md?

`CLAUDE.md` is a special file that provides persistent instructions to Claude Code. It's loaded automatically into the system prompt at the start of every conversation, giving Claude project-specific context, conventions, and rules.

### CLAUDE.md Hierarchy

```
LOADING ORDER (all are loaded, not overriding):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

~/.claude/CLAUDE.md                    ← User global (all projects)
    │
    ├── /project/CLAUDE.md             ← Project root (shared via git)
    │       │
    │       ├── /project/src/CLAUDE.md ← Directory-specific
    │       │
    │       └── /project/tests/CLAUDE.md ← Directory-specific
    │
    └── /project/.claude/CLAUDE.md     ← Project local (not committed)
```

### Hierarchy Behavior

| Level | Location | Shared? | When Loaded |
|-------|----------|---------|-------------|
| **User** | `~/.claude/CLAUDE.md` | No | Always |
| **Project Root** | `./CLAUDE.md` | Yes (git) | When in this project |
| **Subdirectory** | `./src/CLAUDE.md` | Yes (git) | When working in that directory |
| **Local** | `./.claude/CLAUDE.md` | No | When in this project |

### Key Principles

- **Additive:** All levels are combined, not overridden
- **Context-aware:** Subdirectory CLAUDE.md only loads when Claude operates in that directory
- **Persistent:** Survives session changes and context compaction
- **Authoritative:** Treated as instructions from the user/project owner

---

## 2. Design Patterns

### Pattern 1: Project Root CLAUDE.md

```markdown
# Project: E-Commerce API

## Tech Stack
- TypeScript, Node.js 20, Express
- PostgreSQL 15 with Prisma ORM
- Jest for testing, ESLint + Prettier

## Conventions
- Use functional approach, avoid classes
- All API responses follow: { data, error, meta }
- Database queries go through service layer, never in controllers
- Error codes: E001-E099 (auth), E100-E199 (validation), E200+ (business logic)

## Testing
- Unit tests: `src/**/*.test.ts`
- Integration tests: `tests/integration/`
- Run: `npm test` (unit), `npm run test:integration`
- Always mock external services, never hit real APIs in tests

## Git
- Branch naming: feature/TICKET-123-description
- Commit format: "type(scope): message" (conventional commits)
- Never force push to main
```

### Pattern 2: Subdirectory CLAUDE.md for Specialized Rules

```markdown
# /src/database/CLAUDE.md

## Database Conventions
- All migrations in `migrations/` directory
- Migration naming: YYYYMMDD_HHMMSS_description.sql
- Always include rollback (down) migration
- Never modify existing migrations — create new ones
- Use transactions for multi-step migrations
- Test migrations against a copy of production schema
```

```markdown
# /src/api/CLAUDE.md

## API Design Rules
- All endpoints versioned: /api/v1/...
- Authentication required by default (use @public decorator to opt out)
- Rate limiting: 100 req/min for authenticated, 20 for anonymous
- Response format: { "data": ..., "meta": { "page", "total" } }
- Error format: { "error": { "code": "E001", "message": "..." } }
```

### Pattern 3: User-Level CLAUDE.md for Personal Preferences

```markdown
# ~/.claude/CLAUDE.md

## My Preferences
- I prefer concise responses — skip explanations I didn't ask for
- When showing code, include the file path as a comment at the top
- Use early returns instead of nested conditionals
- I'm experienced with TypeScript and Go — don't explain basic concepts
- For Python, I prefer type hints on all function signatures
```

### Pattern 4: Local CLAUDE.md for Machine-Specific Config

```markdown
# .claude/CLAUDE.md (not committed)

## Local Environment
- Database runs on localhost:5433 (non-standard port)
- Use `npm run dev:local` not `npm run dev` (I have custom env)
- My test database is `ecommerce_test_local`
- Skip integration tests that require Redis (I don't run it locally)
```

### Pattern 5: Modular CLAUDE.md with @import

Large CLAUDE.md files become unwieldy. The **@import** directive allows splitting into focused files:

```markdown
# ./CLAUDE.md (Project Root)

# Project: E-Commerce API

@import .claude/rules/tech-stack.md
@import .claude/rules/conventions.md
@import .claude/rules/testing.md
@import .claude/rules/git-workflow.md

## Additional Context
- First-time setup: see onboarding guide
- Common tasks: see /docs/howtos/
```

**@import files to create:**

```markdown
# ./.claude/rules/tech-stack.md
## Tech Stack
- TypeScript, Node.js 20, Express
- PostgreSQL 15 with Prisma ORM
- Jest for testing, ESLint + Prettier
```

```markdown
# ./.claude/rules/conventions.md
## Code Conventions
- Use functional approach, avoid classes
- All API responses: { data, error, meta }
- Database queries through service layer only
```

```markdown
# ./.claude/rules/testing.md
## Testing Standards
- Unit tests: `src/**/*.test.ts`
- Integration tests: `tests/integration/`
- Mock external services, never hit real APIs
```

#### @import Syntax Rules

| Element | Syntax | Example |
|---------|--------|---------|
| **Relative path** | `@import path/to/file.md` | `@import .claude/rules/testing.md` |
| **Absolute path** | Not supported; always relative to importing file | Use relative paths only |
| **Directory imports** | Not supported; must name specific files | Don't: `@import .claude/rules/` |
| **Recursive imports** | NOT allowed (A imports B, B cannot import A) | Creates circular dependency error |
| **Comments** | `# Comment here` before/after @import | Lines before @import are ignored |

#### @import Resolution

When Claude Code loads `./CLAUDE.md`:
1. Reads the file
2. Finds `@import .claude/rules/testing.md`
3. Loads `./.claude/rules/testing.md` relative to current file location
4. Inserts content inline where @import was
5. Continues processing remaining lines

Result: All imported files are merged into one logical CLAUDE.md content.

#### @import Use Cases

**Use case 1: By area (recommended)**
```
./CLAUDE.md
├── @import .claude/rules/api-conventions.md      (API module rules)
├── @import .claude/rules/database-conventions.md (DB module rules)
└── @import .claude/rules/frontend-rules.md       (Frontend module rules)
```

**Use case 2: By type**
```
./CLAUDE.md
├── @import .claude/standards/coding.md       (All coding standards)
├── @import .claude/standards/testing.md      (All testing standards)
├── @import .claude/standards/deployment.md   (All deploy standards)
└── @import .claude/standards/git.md          (All git standards)
```

**Use case 3: Selective import per subdirectory**
```
# ./src/api/CLAUDE.md
@import ../../.claude/rules/api-conventions.md
@import ../../.claude/standards/testing.md

## API-Specific Rules
- Rate limiting: 100 req/min
```

```
# ./src/database/CLAUDE.md
@import ../../.claude/rules/database-conventions.md
@import ../../.claude/standards/testing.md

## Database-Specific Rules
- Migrations always include rollback
```

#### When NOT to Use @import

❌ **Don't use @import for:**
- Machine-specific config (use .claude/CLAUDE.md local instead)
- Personal preferences (use ~/.claude/CLAUDE.md)
- Secrets or credentials (never commit to git)
- Large binary or non-text files

**Reason:** @import is for shared, version-controlled, text-based standards.

---

## 3. Implementation Details

### What Goes in CLAUDE.md vs. Other Files

| Information | Put In | Why |
|------------|--------|-----|
| Tech stack & conventions | Project CLAUDE.md | Everyone needs to know |
| Team-specific rules | Project CLAUDE.md | Consistency across team |
| Personal preferences | User ~/.claude/CLAUDE.md | Only for you |
| Machine-specific config | .claude/CLAUDE.md (local) | Varies per machine |
| Directory-specific rules | Subdirectory CLAUDE.md | Context-relevant only |
| Ephemeral task details | Memory/Tasks (not CLAUDE.md) | Not for every session |

### CLAUDE.md Loading Mechanics

1. At session start, Claude Code reads all applicable CLAUDE.md files
2. Contents are injected into the system prompt
3. They persist through context compaction (unlike message history)
4. Subdirectory files only activate when Claude reads/writes files in that directory

### Size Considerations

| Size | Impact |
|------|--------|
| < 500 tokens | Minimal overhead, ideal |
| 500-2000 tokens | Acceptable, good detail level |
| 2000-5000 tokens | Getting large, every call costs more |
| 5000+ tokens | Consider splitting into subdirectory files |

### File Discovery Rules

Claude Code looks for CLAUDE.md files:
- In the current working directory and all parent directories up to project root
- In `~/.claude/` for user-level instructions
- In subdirectories when Claude accesses files there
- The `.claude/` directory variant is for local-only instructions

---

## 4. Security & Reliability

### Trust Model

```
WHO CAN MODIFY → TRUST LEVEL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
~/.claude/CLAUDE.md       → Only you (highest trust)
.claude/CLAUDE.md (local) → Only you (high trust)
./CLAUDE.md (git)         → Any project contributor (review in PRs)
./subdir/CLAUDE.md (git)  → Any project contributor (review in PRs)
```

### Security Considerations

| Risk | Scenario | Mitigation |
|------|----------|-----------|
| Malicious instructions in PR | Someone adds "ignore all security rules" | Review CLAUDE.md changes in PRs |
| Overly permissive rules | "Always allow rm -rf" | Principle of least privilege |
| Conflicting instructions | Root says X, subdir says NOT X | Be explicit about precedence |
| Stale instructions | Old tech stack documented | Regular audits of CLAUDE.md |
| Sensitive info | API keys or passwords in CLAUDE.md | Never put secrets in CLAUDE.md |

### Best Practices

1. **Review CLAUDE.md changes in PRs** like any other code
2. **No secrets** — CLAUDE.md is git-tracked (use env vars instead)
3. **Keep current** — outdated instructions cause confusion
4. **Be specific** — vague instructions are interpreted variably
5. **Test impact** — changes affect every team member's Claude sessions

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A project has TypeScript in `/src` and Python in `/scripts`. They have different conventions (TypeScript uses camelCase, Python uses snake_case). What is the BEST CLAUDE.md architecture?

**A:** Use **subdirectory CLAUDE.md files**:
- `/src/CLAUDE.md` — TypeScript conventions (camelCase, ESLint rules, Jest patterns)
- `/scripts/CLAUDE.md` — Python conventions (snake_case, type hints, pytest patterns)
- `/CLAUDE.md` (root) — Shared project info (repo structure, git conventions, CI)

This way, Claude applies the right conventions based on which directory it's working in.

### Scenario 2
**Q:** A team member's CLAUDE.md instructs Claude to "skip all tests when committing." This creates risk for the team. Where should testing rules be enforced?

**A:** 
1. **Project-level CLAUDE.md** should state: "Always run tests before committing"
2. **PreToolUse hooks** should enforce test execution (hard block, not just instructions)
3. **CI pipeline** should reject untested commits
4. Defense in depth: instructions (CLAUDE.md) + enforcement (hooks) + verification (CI)

Note: A user's personal CLAUDE.md can't override project-level safety if hooks enforce it.

### Scenario 3
**Q:** A CLAUDE.md file is 8000 tokens long. What problems does this cause and how should it be fixed?

**A:** Problems:
- 8000 tokens consumed on EVERY API call (expensive in agentic loops)
- May crowd out actual conversation context
- Information overload — Claude may miss key instructions

Fix:
- Split into subdirectory CLAUDE.md files (only loaded when relevant)
- Keep root CLAUDE.md to ~1000 tokens (essentials only)
- Move detailed module-specific rules to their directory's CLAUDE.md
- Remove anything derivable from the code itself

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| 10,000+ token CLAUDE.md | Token waste on every call | Split into subdirectory files |
| Duplicating code conventions already in .eslintrc | Redundant, can conflict | Reference config files, don't duplicate |
| Secrets in CLAUDE.md | Committed to git, exposed | Use environment variables |
| Contradicting rules at different levels | Claude confused | Clear precedence, no conflicts |
| Instructions that belong in hooks | Instructions can be ignored | Hooks for hard enforcement |
| Never updating CLAUDE.md | Stale, misleading instructions | Regular maintenance |

---

## 7. Quick Reference / Cheat Sheet

```
CLAUDE.md HIERARCHY
━━━━━━━━━━━━━━━━━━━
~/.claude/CLAUDE.md         → Personal (all projects)
./CLAUDE.md                 → Project root (git-tracked, team-shared)
./subdir/CLAUDE.md          → Directory-specific (git-tracked)
./.claude/CLAUDE.md         → Project local (NOT committed)

BEHAVIOR
━━━━━━━━
• All levels are ADDITIVE (combined, not overridden)
• Loaded into system prompt at session start
• Survives context compaction
• Subdirectory files load on-demand

WHAT TO INCLUDE
━━━━━━━━━━━━━━━
✓ Tech stack and versions
✓ Coding conventions and patterns
✓ Testing requirements and commands
✓ Git workflow and branch naming
✓ Architecture decisions and constraints

WHAT NOT TO INCLUDE
━━━━━━━━━━━━━━━━━━━
✗ Secrets, API keys, passwords
✗ Things derivable from code/config files
✗ Ephemeral task information
✗ Very long documentation (split it up)
✗ Anything that belongs in hooks (enforcement)

SIZE GUIDELINES
━━━━━━━━━━━━━━
Root CLAUDE.md: < 1500 tokens ideal
Subdirectory: < 500 tokens each
Total loaded: < 3000 tokens target
```
