# Domain 3 | Topic 3.6: Hooks & Automation

## 1. Core Concepts

### Hooks as Automation Layer

Hooks provide **automated, event-driven enforcement** that runs independently of Claude's decision-making. Unlike CLAUDE.md instructions (which Claude may interpret flexibly), hooks are hard-coded behaviors that execute every time.

### Hook Architecture

```
┌─────────────────────────────────────────────────────┐
│  HOOK EXECUTION FLOW                                │
│                                                     │
│  Claude decides to use a tool                       │
│       │                                             │
│       ▼                                             │
│  [PreToolUse Hook] ──── Exit 0 → Proceed            │
│       │                                             │
│       │ Exit non-zero → BLOCKED (tool not executed) │
│       │ Stdout → fed back to Claude as context      │
│       │                                             │
│       ▼                                             │
│  Tool Executes                                      │
│       │                                             │
│       ▼                                             │
│  [PostToolUse Hook] ──── Runs (result available)    │
│       │                                             │
│       │ Stdout → fed back to Claude as context      │
│       │                                             │
│       ▼                                             │
│  Claude sees tool result + hook output              │
└─────────────────────────────────────────────────────┘
```

### Hook Types Summary

| Hook | Trigger | Can Block? | Access To |
|------|---------|-----------|-----------|
| `PreToolUse` | Before tool execution | YES (non-zero exit) | Tool name, input |
| `PostToolUse` | After tool execution | No (already ran) | Tool name, input, output |
| `Notification` | Claude sends notification | No | Notification content |
| `Stop` | Claude finishes response | No | Final response |

### Hook Configuration Structure

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "<tool_pattern>",
        "command": "<shell_command>",
        "timeout": 10000
      }
    ],
    "PostToolUse": [
      {
        "matcher": "<tool_pattern>",
        "command": "<shell_command>"
      }
    ],
    "Stop": [
      {
        "command": "<shell_command>"
      }
    ]
  }
}
```

---

## 2. Design Patterns

### Pattern 1: Security Gate (Block Dangerous Commands)

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "python .claude/hooks/security_check.py"
      }
    ]
  }
}
```

```python
# .claude/hooks/security_check.py
import sys, json

# Hooks receive data via STDIN as a JSON object
input_data = json.loads(sys.stdin.read())
tool_name = input_data.get("tool_name", "")
tool_input = json.dumps(input_data.get("tool_input", {}))

BLOCKED_PATTERNS = [
    "rm -rf",
    "rm -r /",
    "DROP TABLE",
    "DROP DATABASE",
    "> /dev/sda",
    "chmod 777",
    "curl.*| bash",
    "wget.*| sh"
]

for pattern in BLOCKED_PATTERNS:
    if pattern.lower() in tool_input.lower():
        print(f"BLOCKED: Command contains dangerous pattern '{pattern}'")
        print("This operation is not allowed by project security policy.")
        sys.exit(1)

sys.exit(0)
```

### Pattern 2: Auto-Format on Write

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "bash .claude/hooks/auto_format.sh"
      },
      {
        "matcher": "Edit",
        "command": "bash .claude/hooks/auto_format.sh"
      }
    ]
  }
}
```

```bash
#!/bin/bash
# .claude/hooks/auto_format.sh
# Hooks receive JSON on STDIN — parse with jq
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

if [ -z "$FILE_PATH" ]; then
  exit 0
fi

case "$FILE_PATH" in
  *.ts|*.tsx|*.js|*.jsx)
    npx prettier --write "$FILE_PATH" 2>/dev/null
    echo "Formatted with Prettier"
    ;;
  *.py)
    black "$FILE_PATH" 2>/dev/null
    echo "Formatted with Black"
    ;;
  *.go)
    gofmt -w "$FILE_PATH" 2>/dev/null
    echo "Formatted with gofmt"
    ;;
esac
```

### Pattern 3: Notification Routing

```json
{
  "hooks": {
    "Notification": [
      {
        "command": "bash .claude/hooks/notify.sh"
      }
    ]
  }
}
```

```bash
#!/bin/bash
# Route notifications to system tray / Slack
# Hooks receive JSON on STDIN
INPUT=$(cat)
MESSAGE=$(echo "$INPUT" | jq -r '.message // empty')
# macOS notification
osascript -e "display notification \"$MESSAGE\" with title \"Claude Code\""
# Or send to Slack webhook
# curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"$MESSAGE\"}" "$SLACK_WEBHOOK"
```

### Pattern 4: Auto-Test on Stop

```json
{
  "hooks": {
    "Stop": [
      {
        "command": "bash .claude/hooks/auto_verify.sh"
      }
    ]
  }
}
```

```bash
#!/bin/bash
# Run quick checks after Claude finishes a task
echo "Running verification..."
npm test --silent 2>&1 | tail -5
if [ $? -ne 0 ]; then
    echo "⚠️ Tests failed after Claude's changes"
fi
```

### Pattern 5: Secrets Detection

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "command": "python .claude/hooks/check_secrets.py"
      },
      {
        "matcher": "Edit",
        "command": "python .claude/hooks/check_secrets.py"
      }
    ]
  }
}
```

```python
# .claude/hooks/check_secrets.py
import sys, json, re

# Read JSON from STDIN
input_data = json.loads(sys.stdin.read())
tool_input = json.dumps(input_data.get("tool_input", {}))

SECRET_PATTERNS = [
    r'(?:api[_-]?key|apikey)\s*[:=]\s*["\']?[A-Za-z0-9+/=]{20,}',
    r'(?:secret|token|password)\s*[:=]\s*["\']?[^\s"\']{8,}',
    r'-----BEGIN (?:RSA |EC )?PRIVATE KEY-----',
    r'sk-[A-Za-z0-9]{32,}',
    r'ghp_[A-Za-z0-9]{36}',
    r'AKIA[A-Z0-9]{16}',
]

for pattern in SECRET_PATTERNS:
    if re.search(pattern, tool_input, re.IGNORECASE):
        print("BLOCKED: Potential secret/credential detected in file content")
        print("Please use environment variables instead of hardcoding secrets.")
        sys.exit(1)

sys.exit(0)
```

---

## 3. Implementation Details

### Data Available to Hooks (via STDIN)

Hooks receive a **JSON object on STDIN** (not environment variables). The hook script should read and parse this JSON:

| Field | Available In | Contains |
|-------|-------------|----------|
| `tool_name` | PreToolUse, PostToolUse | Name of tool (e.g., "Bash", "Write") |
| `tool_input` | PreToolUse, PostToolUse | Object with tool input parameters |
| `tool_output` | PostToolUse only | Result from tool execution |

Example STDIN JSON for a PreToolUse hook:
```json
{
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test",
    "timeout": 30000
  }
}
```

### Hook Execution Details

- Hooks run **synchronously** — Claude waits for completion
- Default timeout: ~10 seconds (configurable)
- Hook stdout is captured and shown to Claude
- Hook stderr is typically not shown
- Multiple hooks for same event run in order
- First blocking hook (non-zero exit in PreToolUse) stops execution

### Hook vs Settings Permissions

| Mechanism | Enforcement | Override? |
|-----------|------------|-----------|
| Settings `deny` | Static pattern match | Deny is cumulative; cannot be overridden by allow at any level |
| PreToolUse hook | Dynamic, programmatic | Cannot be overridden by settings |
| CLAUDE.md instructions | Claude interprets (soft) | Claude could deviate |

Hooks are the **strongest enforcement** mechanism because:
- They execute external code (not Claude's interpretation)
- Non-zero exit = hard block (Claude cannot proceed)
- Cannot be overridden by conversation or local settings

### Matcher Syntax

```
"Bash"           → matches all Bash tool calls
"Write"          → matches all Write tool calls
"Edit"           → matches all Edit tool calls
"Read"           → matches all Read tool calls
"*"              → matches ALL tool calls
"Bash(npm *)"    → matches Bash calls starting with "npm"
"Write(src/**)"  → matches writes to src directory
```

---

## 4. Security & Reliability

### Hook Security Considerations

| Risk | Mitigation |
|------|-----------|
| Malicious hook in shared settings | Review .claude/settings.json changes in PRs |
| Hook script vulnerabilities | Input sanitization, no eval/exec |
| Hook timeout hanging Claude | Always set reasonable timeouts |
| Hook stdout injection | Sanitize hook output |
| Environment variable leakage | Don't expose secrets in hook env |

### Reliability Patterns

```python
# Robust hook script template
import sys, json

try:
    # Read JSON from STDIN
    raw_input = sys.stdin.read()
    input_data = json.loads(raw_input) if raw_input else {}
    
    tool_name = input_data.get("tool_name", "")
    tool_input = input_data.get("tool_input", {})
    
    # Your validation logic here
    # ...
    
    sys.exit(0)  # Allow

except Exception as e:
    # Fail OPEN or CLOSED depending on security requirement
    # Fail open: sys.exit(0)  — don't block on hook errors
    # Fail closed: sys.exit(1) — block if hook can't validate
    print(f"Hook error: {e}")
    sys.exit(0)  # Fail open (less disruptive)
```

### Fail-Open vs Fail-Closed

| Strategy | When Hook Errors | Use When |
|----------|-----------------|----------|
| Fail-open | Allow the operation | Non-critical checks (formatting) |
| Fail-closed | Block the operation | Security-critical checks (secrets) |

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A team wants to ensure ALL code written by Claude is automatically formatted AND passes linting before Claude sees the result. Which hook configuration achieves this?

**A:** Use `PostToolUse` hooks on Write and Edit:
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "bash .claude/hooks/format_and_lint.sh"
      },
      {
        "matcher": "Edit",
        "command": "bash .claude/hooks/format_and_lint.sh"
      }
    ]
  }
}
```

```bash
#!/bin/bash
# .claude/hooks/format_and_lint.sh
# Parse file path from STDIN JSON
FILE_PATH=$(cat | jq -r '.tool_input.file_path // empty')
if [ -n "$FILE_PATH" ]; then
  npx prettier --write "$FILE_PATH" && npx eslint --fix "$FILE_PATH"
fi
```

PostToolUse is correct because:
- We want formatting AFTER the file is written (not before)
- Claude sees the formatted/linted result in subsequent reads
- Ensures code is always in standard format

### Scenario 2
**Q:** What happens if a PreToolUse hook exits with code 1 and prints "Permission denied: cannot modify production files"?

**A:**
1. The tool call is **BLOCKED** (not executed)
2. Claude receives the stdout message: "Permission denied: cannot modify production files"
3. Claude can reason about the block and try a different approach
4. The original tool (e.g., Write) was never executed — the file is untouched

### Scenario 3
**Q:** A hook checks for secrets in code. Should it fail-open or fail-closed if the hook script itself crashes?

**A:** **Fail-closed** (exit non-zero on crash) because:
- This is a security-critical check (preventing credential leaks)
- A hook crash means we can't verify the content is safe
- Better to block and investigate than to allow potentially dangerous content
- False positives (unnecessary blocks) are preferable to false negatives (leaked secrets)

For non-security hooks (like formatting), fail-open is appropriate.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Hooks on every tool (`"*"`) | Massive performance hit | Target specific risky tools |
| No timeout on hooks | Stuck hook blocks Claude forever | Always set timeout |
| Complex logic in inline commands | Hard to maintain, error-prone | Separate script files |
| Hooks that modify tool input | Unexpected behavior | Hooks should validate, not transform |
| No error handling in hook scripts | Crashes cause silent failures | Try/catch with explicit fail strategy |
| Security hooks that fail-open | Defeats the purpose | Security = fail-closed |

---

## 7. Quick Reference / Cheat Sheet

```
HOOK TYPES & PURPOSE
━━━━━━━━━━━━━━━━━━━━
PreToolUse:  Validate/block before execution
PostToolUse: React after execution (format, log, verify)
Notification: Custom notification routing
Stop:         Post-completion actions

EXIT CODES (PreToolUse only)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Exit 0     → Allow (tool proceeds)
Exit non-0 → Block (tool NOT executed)
Stdout     → Shown to Claude as context

STDIN JSON DATA
━━━━━━━━━━━━━━━
Hooks receive a JSON object on STDIN:
{
  "tool_name": "Bash" | "Write" | "Edit" | "Read",
  "tool_input": { ... tool parameters ... },
  "tool_output": "..." (PostToolUse only)
}
Parse with: jq (bash) or json.loads(sys.stdin.read()) (Python)

ENFORCEMENT STRENGTH
━━━━━━━━━━━━━━━━━━━━
Strongest: PreToolUse hook (hard block, code-enforced, programmable)
Medium:    Settings deny (pattern-based, cumulative across all levels)
Weakest:   CLAUDE.md instruction (Claude interprets, may deviate)

COMMON HOOK USE CASES
━━━━━━━━━━━━━━━━━━━━━
• Block dangerous commands (security)
• Auto-format code (quality)
• Detect secrets (security)
• Run tests after changes (quality)
• Custom notifications (workflow)
• Audit logging (compliance)

FAIL STRATEGY
━━━━━━━━━━━━━
Security checks → fail-CLOSED (block on error)
Quality checks  → fail-OPEN (allow on error)
```
