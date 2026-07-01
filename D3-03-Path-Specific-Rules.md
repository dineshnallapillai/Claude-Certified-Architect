# Domain 3 | Topic 3.3: Path-Specific Rule Implementation

## 1. Core Concepts

### What Are Path-Specific Rules?

Path-specific rules allow different Claude behaviors based on which files or directories are being accessed. They enable:
- Different coding conventions per module
- Different permission levels per directory
- Different testing requirements per area
- Language-specific rules in polyglot repos

### Implementation Mechanisms

| Mechanism | How It Works |
|-----------|-------------|
| **Subdirectory CLAUDE.md** | Loaded when Claude works in that directory |
| **Hooks with path matching** | Execute different hooks based on file paths |
| **Permission patterns** | Allow/deny tools for specific paths |
| **Settings matchers** | Configure tools by file glob patterns |

### Subdirectory CLAUDE.md Activation

```
/project/
├── CLAUDE.md                     ← Always loaded
├── src/
│   ├── CLAUDE.md                 ← Loaded when working in /src/**
│   ├── api/
│   │   └── CLAUDE.md             ← Loaded when working in /src/api/**
│   └── database/
│       └── CLAUDE.md             ← Loaded when working in /src/database/**
└── tests/
    └── CLAUDE.md                 ← Loaded when working in /tests/**
```

**Activation rule:** A subdirectory CLAUDE.md is loaded when Claude reads, writes, or edits ANY file under that directory path.

---

## 2. Design Patterns

### Pattern 1: Language-Specific Rules in Monorepo

```markdown
<!-- /services/go-api/CLAUDE.md -->
## Go Service Rules
- Use standard library where possible
- Error handling: always wrap with fmt.Errorf("context: %w", err)
- Logging: use structured logging with slog
- Tests: table-driven tests preferred
- Run: `go test ./...` before committing
```

```markdown
<!-- /services/react-app/CLAUDE.md -->
## React Frontend Rules
- Functional components only (no class components)
- State management: Zustand (not Redux)
- Styling: Tailwind CSS classes
- Tests: React Testing Library (not Enzyme)
- Run: `npm test` before committing
```

### Pattern 2: Sensitive Directory Protection

```markdown
<!-- /infrastructure/CLAUDE.md -->
## Infrastructure Rules — CRITICAL
- NEVER modify production configs without explicit confirmation
- All Terraform changes must include a plan output
- Database migrations require both up AND down
- Secrets: NEVER hardcode — use vault references
- Always run `terraform validate` and `terraform plan` before apply
```

### Pattern 3: Hook-Based Path Enforcement

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "command": "python scripts/check_path_rules.py \"$FILE_PATH\""
      }
    ]
  }
}
```

```python
# scripts/check_path_rules.py
import sys
import os

file_path = sys.argv[1]

PROTECTED_PATHS = [
    "/infrastructure/production/",
    "/secrets/",
    "/.env"
]

for protected in PROTECTED_PATHS:
    if protected in file_path:
        print(f"BLOCKED: Cannot modify files in {protected}")
        print("Reason: This path requires manual modification.")
        sys.exit(1)  # Non-zero = block

sys.exit(0)  # Allow
```

### Pattern 4: Test Requirements by Directory

```markdown
<!-- /src/core/CLAUDE.md -->
## Core Module Rules
- This is the most critical module — changes require extra care
- Every function must have unit tests (100% coverage)
- No external dependencies allowed (pure functions only)
- Performance-critical: avoid allocations in hot paths
- Run benchmarks: `go test -bench=. ./core/...`
```

```markdown
<!-- /src/utils/CLAUDE.md -->
## Utilities
- Helpers and shared code
- Tests required but 80% coverage sufficient
- Can import from /core but not from other modules
- Document all exported functions
```

---

## 3. Implementation Details

### How Path Rules Interact

```
When Claude edits /src/api/routes/auth.ts:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Loaded rules (ALL apply):
1. ~/.claude/CLAUDE.md          (user preferences)
2. /project/CLAUDE.md           (project-wide rules)
3. /project/src/CLAUDE.md       (src rules)
4. /project/src/api/CLAUDE.md   (api rules)

All are ADDITIVE — Claude follows all of them.
```

### Permission Patterns for Paths

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "Read",
      "Bash(npm test *)",
      "Write(src/**)",
      "Edit(src/**)"
    ],
    "deny": [
      "Write(infrastructure/production/**)",
      "Bash(rm *)",
      "Write(.env*)"
    ]
  }
}
```

### Path Matching in Hooks

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "bash -c 'if [[ \"$FILE_PATH\" == *.ts ]]; then npx prettier --write \"$FILE_PATH\"; fi'"
      },
      {
        "matcher": "Write",
        "command": "bash -c 'if [[ \"$FILE_PATH\" == *.py ]]; then black \"$FILE_PATH\"; fi'"
      }
    ]
  }
}
```

### Priority When Rules Conflict

CLAUDE.md rules are additive, not overriding. If they conflict:
- Claude attempts to satisfy all applicable rules
- More specific (deeper) rules are treated as more authoritative
- If truly contradictory, Claude may ask for clarification

---

## 4. Security & Reliability

### Path-Based Security Model

```
SECURITY ZONES
━━━━━━━━━━━━━━
Zone 1 (Open): /src, /tests
  → Full read/write for Claude
  → Auto-format, auto-test

Zone 2 (Guarded): /config, /scripts
  → Read allowed, write needs confirmation
  → Changes logged

Zone 3 (Protected): /infrastructure, /secrets, /.env
  → Read restricted, write blocked by hooks
  → Requires explicit human approval
```

### Common Vulnerabilities

| Risk | Example | Mitigation |
|------|---------|-----------|
| Path traversal in rules | `../../etc/passwd` bypasses directory scope | Normalize paths, use realpath |
| Symlink escape | Symlink in /src → /secrets | Resolve symlinks before checking |
| Rule bypass via move | Edit in /tmp, move to protected dir | Check final destination, not working dir |
| Stale rules after restructure | Rules reference old paths | Audit rules when moving files |

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A monorepo has `/frontend` (React), `/backend` (Python), and `/infra` (Terraform). Each needs different formatters and test commands. What's the MOST maintainable configuration?

**A:** Use subdirectory CLAUDE.md files + path-aware PostToolUse hooks:

```
/frontend/CLAUDE.md → React conventions, `npm test`
/backend/CLAUDE.md → Python conventions, `pytest`
/infra/CLAUDE.md → Terraform conventions, `terraform validate`
```

Hooks:
```json
{
  "hooks": {
    "PostToolUse": [
      {"matcher": "Write", "command": "scripts/format_by_path.sh \"$FILE_PATH\""}
    ]
  }
}
```

The formatter script inspects the path and runs the appropriate tool (prettier, black, terraform fmt).

### Scenario 2
**Q:** A developer accidentally uses Claude to modify a production Terraform file. How could path-specific rules have prevented this?

**A:** Three layers of protection:
1. `/infra/production/CLAUDE.md` stating "NEVER modify without explicit human confirmation"
2. PreToolUse hook blocking writes to `/infra/production/**`
3. Permission deny rule: `"deny": ["Write(infra/production/**)", "Edit(infra/production/**))"]`

The hook (layer 2) provides the strongest guarantee since it's a hard block, not just an instruction.

### Scenario 3
**Q:** Why would you use a subdirectory CLAUDE.md instead of putting all rules in the root CLAUDE.md?

**A:**
1. **Token efficiency** — subdir rules only load when relevant (not every API call)
2. **Clarity** — rules are co-located with the code they govern
3. **Maintainability** — module owners can maintain their own rules
4. **Avoid root bloat** — root CLAUDE.md stays concise and readable
5. **Context relevance** — Claude only sees rules applicable to current work

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| All rules in root CLAUDE.md | Bloated, always loaded, expensive | Split into subdirectories |
| Trusting CLAUDE.md alone for security | Instructions can be ignored | Use hooks for enforcement |
| No rules for sensitive directories | Unprotected critical paths | Explicit protection rules |
| Path rules that don't match restructured repo | Rules apply to wrong paths | Audit on restructure |
| Conflicting rules across levels | Confusing for Claude | Ensure consistency, test combinations |
| Hardcoded absolute paths in rules | Break on different machines | Use relative paths from project root |

---

## 7. Quick Reference / Cheat Sheet

```
PATH RULE MECHANISMS
━━━━━━━━━━━━━━━━━━━━
1. Subdirectory CLAUDE.md → conventions per area
2. Hook path matching → automated enforcement
3. Permission patterns → allow/deny by glob
4. Settings configuration → tool behavior per path

ACTIVATION RULES
━━━━━━━━━━━━━━━━
• Subdir CLAUDE.md loads when Claude accesses files in that directory
• ALL ancestor CLAUDE.md files are loaded (additive)
• More specific (deeper) = more authoritative on conflicts

SECURITY ZONES
━━━━━━━━━━━━━━
Open: src, tests → full access
Guarded: config → read yes, write with caution
Protected: infra/prod, secrets → blocked by hooks

HOOK PATH MATCHING
━━━━━━━━━━━━━━━━━━
$FILE_PATH available in hook commands
Use bash conditionals: if [[ "$FILE_PATH" == *.ts ]]; then ...
Or dedicated script that inspects the path

BEST PRACTICES
━━━━━━━━━━━━━━
✓ Co-locate rules with code they govern
✓ Use hooks for hard enforcement (not just CLAUDE.md)
✓ Keep root CLAUDE.md minimal
✓ Protect sensitive directories at multiple layers
✓ Audit rules when restructuring code
```
