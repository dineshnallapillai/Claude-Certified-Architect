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
  
Rule: If same pattern appears in both allow and deny at different levels,
      higher-priority file (local > project > global) wins.
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

Project-level because it's a team-wide policy. Committed so all members get it. Deny list ensures even if a user adds it to their local allow, the project deny takes precedence.

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

**A:** **Local settings have higher priority**, so `Bash(rm temp/*)` would be allowed. However, this is a security concern — the project-wide deny was intentionally protective.

Best practice: Project should use hooks (PreToolUse) for hard enforcement of safety-critical rules, since settings can be overridden locally. Settings deny is a guard, not a guarantee.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| `"allow": ["Bash(*)"]` in shared settings | Allows everything including rm | Specific patterns only |
| No deny list | Relying entirely on Claude's judgment | Explicit deny for dangerous commands |
| Secrets in settings files | Credentials exposed | Use env vars or vault |
| Over-restrictive shared settings | Team frustrated, overrides locally | Balance safety and productivity |
| No project settings at all | Each person configures independently | Establish team baseline |
| Relying only on settings for security | Local override bypasses project deny | Use hooks for hard enforcement |

---

## 7. Claude Code CLI Flags for Automation

When running Claude Code in CI/CD pipelines or automated contexts, specific flags enable non-interactive operation and structured output.

### Essential CLI Flags

#### -p / --print (Non-Interactive Mode)

```bash
# Interactive (default)
claude "Review this code for bugs"
# → Hangs waiting for user input in CI

# Non-interactive
claude -p "Review this code for bugs"
# → Processes prompt, outputs result, exits
```

**Use in CI:**
```yaml
# GitHub Actions example
- name: Code Review with Claude
  run: claude -p "Review for security issues" < pr_diff.txt
```

**Key behavior:**
- `-p` flag makes Claude exit after producing output
- No prompts for permission (uses settings.json allow/deny)
- Output goes to stdout (can pipe to files or tools)
- Exit code indicates success/failure

#### --output-format json

Produce structured JSON output for programmatic parsing:

```bash
# Default: prose output
claude -p "List all functions in this file"
# Output: "I found the following functions: function1(), function2()..."

# JSON output
claude -p --output-format json "List all functions in this file"
# Output: {"functions": ["function1", "function2"], "count": 2}
```

**Use in CI for parsing:**
```bash
result=$(claude -p --output-format json "Analyze this code")
num_issues=$(echo "$result" | jq '.issues | length')
echo "Found $num_issues issues"
```

#### --json-schema

Enforce structured output with a JSON schema:

```bash
claude -p --output-format json --json-schema '{
  "type": "object",
  "properties": {
    "issues": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "severity": {"type": "string", "enum": ["high", "medium", "low"]},
          "message": {"type": "string"},
          "line": {"type": "integer"}
        }
      }
    }
  }
}' "Review this code: $(cat file.js)"
```

**Benefits:**
- Guarantees valid JSON structure
- Eliminates parsing errors
- Can directly post findings to PR comments
- Schema validation prevents hallucinated fields

### CI/CD Integration Example

```bash
#!/bin/bash
# ci_review.sh — automated code review in CI pipeline

# 1. Run review in non-interactive mode with JSON output
REVIEW_JSON=$(claude -p \
  --output-format json \
  --json-schema '{
    "type": "object",
    "properties": {
      "findings": {
        "type": "array",
        "items": {
          "severity": {"type": "string", "enum": ["high", "medium", "low"]},
          "file": {"type": "string"},
          "line": {"type": "integer"},
          "issue": {"type": "string"},
          "suggestion": {"type": "string"}
        }
      },
      "pass": {"type": "boolean"}
    }
  }' \
  "Review this PR for security issues. $(cat pr_files.txt)")

# 2. Parse JSON and check for high-severity issues
HIGH_SEVERITY=$(echo "$REVIEW_JSON" | jq '.findings[] | select(.severity=="high") | length')

if [ "$HIGH_SEVERITY" -gt 0 ]; then
  echo "❌ Found $HIGH_SEVERITY high-severity issues"
  exit 1
fi

# 3. Output review findings as GitHub markdown
echo "## Code Review" >> $GITHUB_STEP_SUMMARY
echo "$REVIEW_JSON" | jq '.findings[] | "- [\(.severity)] \(.file):\(.line) - \(.issue)"' >> $GITHUB_STEP_SUMMARY

exit 0
```

### Flag Combinations

| Use Case | Flags |
|----------|-------|
| Interactive dev work | (none) |
| CI review with exit code | `-p` |
| CI with JSON parsing | `-p --output-format json` |
| CI with validated structure | `-p --output-format json --json-schema '...'` |
| Pipeline test generation | `-p --output-format json --json-schema test_schema.json` |
| Automated extraction | `-p --output-format json` (with few-shot examples in prompt) |

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

CLI FLAGS FOR AUTOMATION
━━━━━━━━━━━━━━━━━━━━━━
-p / --print              Exit after output (non-interactive)
--output-format json      Produce JSON instead of prose
--json-schema '...'       Enforce schema validation
-p --output-format json   Combined: non-interactive + structured output

TYPICAL CI COMMAND
━━━━━━━━━━━━━━━━━
claude -p --output-format json --json-schema SCHEMA.json "TASK"

ENVIRONMENT
━━━━━━━━━━━
CLAUDE_API_KEY            API key for authentication
ANTHROPIC_API_KEY         Alternative env var for API key
CLAUDE_CONFIG_DIR         ~/.claude by default
```
