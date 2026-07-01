# Domain 3 | Topic 3.5: Permissions & Settings Configuration

## 1. Core Concepts

### Permission Model

Claude Code uses a **layered permission system** that controls what tools Claude can use without asking:

```
PERMISSION FLOW
━━━━━━━━━━━━━━━
Claude wants to use a tool
  │
  ├─ Is it in "allow" list? → YES → Execute silently
  │
  ├─ Is it in "deny" list? → YES → Block (never ask)
  │
  └─ Neither? → Prompt user for approval
```

### Settings Hierarchy

```
PRIORITY (highest to lowest):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. .claude/settings.local.json   ← Local overrides (not committed)
2. .claude/settings.json         ← Project settings (git-tracked)
3. ~/.claude/settings.json       ← User global settings

Higher priority settings OVERRIDE lower priority for same keys.
```

### Permission Types

| Permission | Meaning | Example |
|-----------|---------|---------|
| `allow` | Auto-execute without asking | `"allow": ["Read", "Glob"]` |
| `deny` | Block completely (never allow) | `"deny": ["Bash(rm *)"]` |
| (unlisted) | Ask user each time | Default for most tools |

### Tool Permission Patterns

```json
{
  "permissions": {
    "allow": [
      "Read",                      // All read operations
      "Glob",                      // File pattern matching
      "Grep",                      // Content search
      "Bash(npm test *)",          // npm test with any args
      "Bash(npx prettier *)",      // prettier formatting
      "Bash(git status)",          // git status only
      "Bash(git diff *)",          // git diff with any args
      "Write(src/**)",             // Write to src directory
      "Edit(src/**)"              // Edit in src directory
    ],
    "deny": [
      "Bash(rm -rf *)",           // Never allow recursive delete
      "Bash(git push --force *)", // Never force push
      "Write(.env*)",             // Never write env files
      "Bash(curl *)"              // Never make network requests
    ]
  }
}
```

---

## 2. Design Patterns

### Pattern 1: Conservative Default (Recommended)

```json
// .claude/settings.json — project-level, shared
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(npm test *)",
      "Bash(npm run lint *)",
      "Bash(npx tsc --noEmit)"
    ],
    "deny": [
      "Bash(rm *)",
      "Bash(git push *)",
      "Bash(npm publish *)",
      "Write(.env*)",
      "Write(*.pem)",
      "Write(*.key)"
    ]
  }
}
```

### Pattern 2: Developer-Friendly (More Permissive)

```json
// .claude/settings.local.json — personal, NOT committed
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Write",
      "Edit",
      "Bash(npm *)",
      "Bash(git add *)",
      "Bash(git commit *)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Bash(git reset --hard *)"
    ]
  }
}
```

### Pattern 3: CI/Automation (Most Permissive)

```json
// For automated pipelines (non-interactive)
{
  "permissions": {
    "allow": [
      "Read",
      "Write",
      "Edit",
      "Glob",
      "Grep",
      "Bash(*)"
    ],
    "deny": [
      "Bash(rm -rf /)",
      "Bash(git push --force *)"
    ]
  }
}
```

### Pattern 4: Graduated Trust by Tool Type

```json
{
  "permissions": {
    "allow": [
      // Level 1: Always safe
      "Read", "Glob", "Grep",
      
      // Level 2: Safe commands
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(npm test *)",
      "Bash(npm run lint *)",
      
      // Level 3: Writes within scope
      "Write(src/**)",
      "Write(tests/**)",
      "Edit(src/**)",
      "Edit(tests/**)"
    ],
    "deny": [
      // Destructive operations
      "Bash(rm *)",
      "Bash(git push --force *)",
      "Bash(git reset --hard *)",
      
      // Sensitive files
      "Write(.env*)",
      "Write(infrastructure/**)",
      "Edit(infrastructure/**)"
    ]
  }
}
```

---

## 3. Implementation Details

### Settings File Format

```json
// Full settings.json structure
{
  "permissions": {
    "allow": ["..."],
    "deny": ["..."]
  },
  "hooks": {
    "PreToolUse": [...],
    "PostToolUse": [...],
    "Stop": [...]
  },
  "mcpServers": {
    "server-name": {...}
  },
  "env": {
    "MY_VAR": "value"
  }
}
```

### Pattern Matching Syntax

| Pattern | Matches | Does NOT Match |
|---------|---------|----------------|
| `"Read"` | All Read tool calls | — |
| `"Bash(npm *)"` | `npm test`, `npm install` | `npx`, `node` |
| `"Bash(git status)"` | Exactly `git status` | `git status -s` |
| `"Bash(git diff *)"` | `git diff HEAD`, `git diff main` | `git status` |
| `"Write(src/**)"` | `src/app.ts`, `src/lib/util.ts` | `tests/app.test.ts` |
| `"Write(*.ts)"` | Any `.ts` file | `.js` files |
| `"Bash(*)"` | Any bash command | — |

### Environment Variables in Settings

```json
{
  "env": {
    "NODE_ENV": "development",
    "DEBUG": "true",
    "DATABASE_URL": "postgresql://localhost/dev"
  }
}
```

These are set in the environment for all tool executions.

### Merge Behavior Across Levels

```
User global:    allow: ["Read", "Glob"]
Project shared: allow: ["Bash(npm test *)"], deny: ["Bash(rm *)"]
Project local:  allow: ["Write(src/**)"]

Effective (merged):
  allow: ["Read", "Glob", "Bash(npm test *)", "Write(src/**)"]
  deny: ["Bash(rm *)"]
  
Rule: Deny rules are CUMULATIVE — a deny at ANY level blocks the action
      regardless of allow rules at other levels. Local allow CANNOT
      override project deny. This is by security design.
      
      For hard enforcement beyond allow/deny lists, use hooks (PreToolUse).
```

---

## 4. Security & Reliability

### Security Best Practices

| Practice | Why |
|----------|-----|
| Default deny for destructive commands | Prevents accidental damage |
| Project-level deny in shared settings | Team-wide safety guardrails |
| Never allow `Bash(*)` in shared settings | Too permissive for all team members |
| Deny access to sensitive file patterns | Protect secrets, keys, credentials |
| Review settings changes in PRs | Prevent permission escalation |

### Common Security Mistakes

```json
// DANGEROUS: Don't do this in shared settings
{
  "permissions": {
    "allow": ["Bash(*)"]  // ← Allows ANYTHING including rm -rf /
  }
}

// DANGEROUS: Overly broad write access
{
  "permissions": {
    "allow": ["Write"]  // ← Can write to ANY file including .env
  }
}

// BETTER: Scoped write access
{
  "permissions": {
    "allow": ["Write(src/**)", "Write(tests/**)"],
    "deny": ["Write(.env*)", "Write(*.key)", "Write(*.pem)"]
  }
}
```

### Defense in Depth

```
Layer 1: Permission settings (allow/deny lists)
Layer 2: PreToolUse hooks (programmatic validation)
Layer 3: User approval prompts (for unlisted operations)
Layer 4: Claude's own judgment (safety instructions in system prompt)
Layer 5: OS-level permissions (filesystem/network)
```

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A team wants Claude to freely run tests and linting but NEVER push code or delete files. Where should this be configured?

**A:** In **project-level settings** (`.claude/settings.json`, committed to git):
```json
{
  "permissions": {
    "allow": [
      "Bash(npm test *)",
      "Bash(npm run lint *)",
      "Bash(npx jest *)"
    ],
    "deny": [
      "Bash(git push *)",
      "Bash(rm *)",
      "Bash(del *)"
    ]
  }
}
```

Project-level because it's a team-wide policy. Committed so all members get it. Deny rules are cumulative across all levels, so even if a user adds a matching pattern to their local allow, the project deny still blocks the action.

### Scenario 2
**Q:** A developer finds that Claude asks permission for every `git diff` and `git log` command. How do they reduce permission prompts without compromising security?

**A:** Add read-only git commands to their **local settings** (`.claude/settings.local.json`):
```json
{
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git show *)",
      "Bash(git branch *)"
    ]
  }
}
```

These are all read-only operations with no security risk. Local settings because it's personal preference — some team members may prefer more prompts.

### Scenario 3
**Q:** Settings conflict: project settings deny `"Bash(rm *)"` but a developer's local settings allow `"Bash(rm temp/*)"`. Which wins?

**A:** **The deny rule wins** — deny rules are enforced cumulatively across all levels. A deny at ANY level (project, user global, or local) blocks the action regardless of allow rules at other levels. The local allow for `Bash(rm temp/*)` will NOT override the project deny for `Bash(rm *)`. This is by security design: deny rules are absolute guards.

For truly hard enforcement with programmatic logic (e.g., allowing specific rm patterns while blocking others), use PreToolUse hooks. Hooks provide the strongest enforcement because they execute external code that cannot be overridden by any settings file.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| `"allow": ["Bash(*)"]` in shared settings | Allows everything including rm | Specific patterns only |
| No deny list | Relying entirely on Claude's judgment | Explicit deny for dangerous commands |
| Secrets in settings files | Credentials exposed | Use env vars or vault |
| Over-restrictive shared settings | Team frustrated, overrides locally | Balance safety and productivity |
| No project settings at all | Each person configures independently | Establish team baseline |
| Relying only on settings for security | Deny rules block but hooks provide programmable logic | Use hooks for complex enforcement beyond pattern matching |

---

## 7. Quick Reference / Cheat Sheet

```
SETTINGS HIERARCHY (priority order)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. .claude/settings.local.json  (highest, personal)
2. .claude/settings.json        (project, git-tracked)
3. ~/.claude/settings.json      (global, personal)

PERMISSION PATTERNS
━━━━━━━━━━━━━━━━━━━
"Read"              → all Read operations
"Bash(npm *)"       → npm with any args
"Bash(git status)"  → exactly git status
"Write(src/**)"     → write to anything in src/
"Write(*.ts)"       → write to any .ts file
"Bash(*)"           → any bash command (⚠️ dangerous)

SETTINGS STRUCTURE
━━━━━━━━━━━━━━━━━━
{
  "permissions": {"allow": [...], "deny": [...]},
  "hooks": {"PreToolUse": [...], "PostToolUse": [...]},
  "mcpServers": {"name": {...}},
  "env": {"KEY": "VALUE"}
}

SAFE DEFAULTS TO ALLOW
━━━━━━━━━━━━━━━━━━━━━━
• Read, Glob, Grep (always safe)
• Bash(git status/diff/log) (read-only)
• Bash(npm test) (non-destructive)
• Bash(npx prettier) (formatters)
• Bash(npx tsc --noEmit) (type check)

ALWAYS DENY
━━━━━━━━━━━
• Bash(rm -rf *)
• Bash(git push --force *)
• Bash(git reset --hard *)
• Write(.env*), Write(*.key), Write(*.pem)
• Bash(npm publish *)
```
