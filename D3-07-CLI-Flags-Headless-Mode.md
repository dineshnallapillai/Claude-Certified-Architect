# Domain 3 | Topic 3.7: CLI Flags & Headless Mode

## 1. Core Concepts

### Claude Code Invocation Modes

Claude Code supports multiple invocation modes for different workflows:

```
INVOCATION MODES
━━━━━━━━━━━━━━━━
Interactive (TUI):   claude                     → Full terminal UI with conversation
Print (single-shot): claude -p "prompt"         → One response to stdout, then exit
Pipe:                echo "prompt" | claude -p  → Accepts stdin as the prompt
Continue:            claude --continue          → Resume the most recent conversation
Resume:              claude --resume <id>       → Resume a specific session by ID
```

### Interactive Mode (Default)

When you run `claude` with no flags, it opens the **Terminal User Interface (TUI)** — a full interactive session with tool use, multi-turn conversation, and permission prompting. This is the standard developer workflow.

### Print Mode (`--print` / `-p`)

Print mode is the foundation for automation. Key characteristics:
- Sends a single prompt, receives a single response, then exits
- No interactive TUI — output goes directly to stdout
- No conversation state maintained between invocations
- Non-zero exit code on errors
- Supports output format control (`--output-format`)

```bash
# Basic print mode
claude -p "Explain this error: $(cat error.log)"

# With model override
claude -p --model claude-sonnet-4-6 "Summarize this file" < main.py

# With output format
claude -p --output-format json "List all TODO comments in src/"
```

### Pipe Mode

Pipe mode combines shell pipelines with print mode. The prompt comes from stdin:

```bash
# Pipe file content as context
cat src/app.ts | claude -p "Find bugs in this code"

# Chain with other CLI tools
git diff HEAD~1 | claude -p "Write a commit message for this diff"

# Process command output
npm test 2>&1 | claude -p "Explain these test failures and suggest fixes"
```

### Continue Mode (`--continue` / `-c`)

Resumes the **most recent** conversation in the current project:

```bash
# Start a conversation
claude -p "Create a user authentication module"

# Continue it (Claude remembers prior context)
claude --continue "Now add password reset functionality"

# Short form
claude -c "Add rate limiting to the auth endpoints"
```

### Resume Mode (`--resume <session-id>`)

Resumes a **specific** conversation by session ID:

```bash
# Resume a specific past session
claude --resume abc123def "Continue where we left off"

# Useful when working on multiple features in parallel
claude --resume feature-auth-session "Add the JWT middleware"
```

---

## 2. Design Patterns

### Pattern 1: CI/CD Code Review Pipeline

```yaml
# .github/workflows/code-review.yml
name: Automated Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run Claude Code Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          git diff origin/main...HEAD | claude -p \
            --output-format json \
            --max-turns 1 \
            --dangerously-skip-permissions \
            "Review this diff for bugs, security issues, and style problems. 
             Output a JSON array of findings with file, line, severity, message."
```

### Pattern 2: Automated Test Generation

```bash
#!/bin/bash
# generate-tests.sh — Run in CI on new files
NEW_FILES=$(git diff --name-only origin/main...HEAD -- '*.ts' '*.tsx')

for file in $NEW_FILES; do
  claude -p \
    --max-turns 5 \
    --dangerously-skip-permissions \
    --output-format text \
    "Generate comprehensive unit tests for $file. 
     Write them to ${file%.ts}.test.ts. 
     Use Jest and follow existing test patterns in this repo."
done
```

### Pattern 3: Structured Output for Tooling

```bash
# Get structured JSON output for downstream processing
RESULT=$(claude -p --output-format json "Analyze the complexity of src/core/")

# Parse with jq
echo "$RESULT" | jq '.result.content[0].text'

# Stream processing with stream-json
claude -p --output-format stream-json "Refactor this large file" | \
  while IFS= read -r line; do
    TYPE=$(echo "$line" | jq -r '.type')
    if [ "$TYPE" = "content_block_delta" ]; then
      echo "$line" | jq -r '.delta.text' 
    fi
  done
```

### Pattern 4: Multi-Directory Monorepo Operations

```bash
# Work across multiple packages in a monorepo
claude -p \
  --add-dir ./packages/shared \
  --add-dir ./packages/api \
  "Find all places where the User type from @shared is used in @api 
   and ensure they handle the new 'role' field"
```

### Pattern 5: Controlled Agentic Execution

```bash
# Limit agent iterations to prevent runaway execution
claude -p \
  --max-turns 10 \
  --allowedTools "Read,Grep,Glob,Bash(npm test*)" \
  --dangerously-skip-permissions \
  "Find and fix the failing test in src/utils/parser.test.ts"
```

---

## 3. Implementation Details

### Complete CLI Flag Reference

| Flag | Short | Purpose | Mode |
|------|-------|---------|------|
| `--print` | `-p` | Single-shot mode, output to stdout | Print |
| `--continue` | `-c` | Resume last conversation | Interactive/Print |
| `--resume <id>` | | Resume specific session | Interactive/Print |
| `--model <id>` | | Override model selection | All |
| `--output-format <fmt>` | | Control output format (text/json/stream-json) | Print only |
| `--max-turns <n>` | | Limit agentic loop iterations | Print |
| `--system-prompt <text>` | | Override system prompt entirely | Print only |
| `--append-system-prompt <text>` | | Append text to system prompt | Print only |
| `--allowedTools <tools>` | | Comma-separated tools to allow | All |
| `--disallowedTools <tools>` | | Comma-separated tools to deny | All |
| `--add-dir <path>` | | Add additional directory context | All |
| `--verbose` | | Show detailed tool calls and reasoning | All |
| `--dangerously-skip-permissions` | | Skip ALL permission checks | Print/CI |

### Output Formats in Print Mode

**`text` (default):**
```
$ claude -p "What is 2+2?"
2 + 2 = 4
```

**`json`:**
```json
{
  "type": "result",
  "subtype": "success",
  "cost_usd": 0.003,
  "is_error": false,
  "duration_ms": 1200,
  "duration_api_ms": 980,
  "num_turns": 1,
  "result": {
    "role": "assistant",
    "content": [
      {
        "type": "text",
        "text": "2 + 2 = 4"
      }
    ]
  },
  "session_id": "abc123"
}
```

**`stream-json`:**
Newline-delimited JSON events emitted in real-time:
```jsonl
{"type":"message_start","message":{"id":"msg_01..."}}
{"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}
{"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"2 + 2"}}
{"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":" = 4"}}
{"type":"content_block_stop","index":0}
{"type":"message_stop"}
```

### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success — prompt completed normally |
| `1` | Error — invalid flags, API failure, or runtime error |

### System Prompt Control

```bash
# Override system prompt entirely (replaces default)
claude -p --system-prompt "You are a SQL expert. Only output valid SQL." \
  "Convert this to a query: find users older than 30"

# Append to system prompt (keeps default + adds yours)
claude -p --append-system-prompt "Always respond in bullet points" \
  "Explain the SOLID principles"
```

**Key distinction:** `--system-prompt` replaces the entire system prompt (use carefully — you lose default behaviors). `--append-system-prompt` adds to it (safer for most use cases).

### The `--dangerously-skip-permissions` Flag

This flag disables ALL permission checks and tool approval prompts:

```bash
# Claude can run any tool without user approval
claude -p --dangerously-skip-permissions "Delete all .tmp files and reformat src/"
```

**Critical rules for this flag:**
- ONLY use in CI/CD or fully automated environments
- NEVER use interactively on a developer machine
- Always combine with `--max-turns` to limit blast radius
- Requires explicit trust in the prompt (it can do anything)
- Named "dangerously" intentionally as a deterrent

---

## 4. Security & Reliability

### CI/CD Security Considerations

| Risk | Mitigation |
|------|-----------|
| API key exposure | Use secrets management (GitHub Secrets, Vault) |
| Runaway agents in CI | Always set `--max-turns` |
| Uncontrolled file modifications | Use `--allowedTools` to restrict tool access |
| Prompt injection via PR content | Sanitize inputs, limit scope |
| Cost overruns | Set `--max-turns`, use cheaper models for CI |
| Unauthorized code execution | Scope permissions with `--allowedTools` |

### Permission Control in Automation

```bash
# Restrictive: only allow read operations
claude -p \
  --allowedTools "Read,Grep,Glob" \
  --max-turns 5 \
  "Analyze the architecture of this project"

# Moderate: allow reads + controlled writes
claude -p \
  --allowedTools "Read,Grep,Glob,Write,Edit" \
  --disallowedTools "Bash" \
  --max-turns 10 \
  "Refactor the utils module to use TypeScript generics"

# Wide open (CI only, with guard rails)
claude -p \
  --dangerously-skip-permissions \
  --max-turns 15 \
  "Fix all failing tests and ensure the build passes"
```

### Authentication for CI

```bash
# Primary method: environment variable
export ANTHROPIC_API_KEY="sk-ant-..."

# The key is picked up automatically by claude
claude -p "Hello"

# Never hardcode in scripts — always use secrets management
# GitHub Actions: ${{ secrets.ANTHROPIC_API_KEY }}
# GitLab CI: $ANTHROPIC_API_KEY (masked variable)
```

### Rate Limiting and Cost Control

- `--max-turns` prevents infinite agentic loops (each turn = one tool use cycle)
- Use `--model` to select cost-appropriate models for CI tasks
- Monitor usage via the JSON output format (`cost_usd` field)
- Set budget alerts in your Anthropic account

---

## 5. Exam Scenarios

### Scenario 1: Choosing the Right Mode

**Q:** A developer wants to integrate Claude into a GitHub Actions workflow that automatically reviews PRs for security vulnerabilities. The workflow should output structured findings that can be parsed by another tool. Which combination of flags is most appropriate?

**A:** Use `claude -p --output-format json --max-turns 3 --dangerously-skip-permissions` with the diff piped in:
```bash
git diff origin/main...HEAD | claude -p \
  --output-format json \
  --max-turns 3 \
  --dangerously-skip-permissions \
  "Review this diff for security vulnerabilities"
```

Rationale:
- `-p` (print mode): non-interactive, outputs to stdout for CI capture
- `--output-format json`: structured output for downstream parsing
- `--max-turns 3`: prevents runaway execution (security review needs minimal tool use)
- `--dangerously-skip-permissions`: CI has no interactive user to approve tools
- Pipe input: passes the diff directly as context

### Scenario 2: System Prompt Override vs Append

**Q:** A team has a CI job that uses Claude to generate API documentation. They want Claude to always output valid Markdown with specific heading structures, but they also want Claude to retain its default tool-use capabilities. Should they use `--system-prompt` or `--append-system-prompt`?

**A:** Use `--append-system-prompt`. The `--system-prompt` flag **replaces** the entire default system prompt, which would remove Claude's instructions for tool use, safety behavior, and built-in capabilities. `--append-system-prompt` adds the documentation format requirements while preserving all default behavior:

```bash
claude -p --append-system-prompt "Format all output as Markdown. Use ## for sections." \
  "Generate API docs for src/routes/"
```

### Scenario 3: Continue vs Resume

**Q:** A developer has been working on two features in parallel using Claude — a "payments" feature (session ID: `pay-001`) and an "auth" feature (session ID: `auth-002`). They last worked on "payments." How do they resume work on the auth feature?

**A:** Use `claude --resume auth-002`. The `--continue` flag would resume the **most recent** session (payments), which is not what they want. `--resume <id>` targets a specific session regardless of recency:

```bash
# This would resume "payments" (most recent) — WRONG
claude --continue

# This resumes "auth" specifically — CORRECT
claude --resume auth-002 "Add the refresh token rotation logic"
```

### Scenario 4: Preventing Runaway Agents

**Q:** A CI pipeline uses Claude to fix linting errors automatically. In a recent incident, Claude entered a loop — fixing one lint error introduced another, consuming excessive API credits. What is the primary mechanism to prevent this?

**A:** Use `--max-turns` to cap the number of agentic loop iterations:

```bash
claude -p \
  --max-turns 5 \
  --dangerously-skip-permissions \
  "Fix all ESLint errors in src/"
```

`--max-turns 5` means Claude can use at most 5 tool calls before being forced to stop. This prevents infinite fix-break-fix loops. Additionally:
- Monitor the `cost_usd` field in JSON output for budget alerting
- Use `--allowedTools` to restrict which tools are available (e.g., only Edit, not Bash)
- The exit code will still be 0 if Claude stopped due to max-turns (it completed normally)

### Scenario 5: Combining Flags for Minimal-Privilege CI

**Q:** An organization wants Claude to perform automated code review in CI but with the principle of least privilege. Claude should only be able to read files and search code — never write, edit, or execute commands. What flag combination achieves this?

**A:** Combine `--allowedTools` with explicit read-only tools and omit `--dangerously-skip-permissions`:

```bash
claude -p \
  --allowedTools "Read,Grep,Glob" \
  --disallowedTools "Bash,Write,Edit" \
  --max-turns 10 \
  --output-format json \
  "Review src/ for code quality issues and report findings"
```

Key points:
- `--allowedTools "Read,Grep,Glob"` — only these tools run without prompting
- `--disallowedTools "Bash,Write,Edit"` — these are explicitly blocked
- Without `--dangerously-skip-permissions`, any unlisted tool would prompt (and in CI, prompting causes a hang/timeout — effectively blocking)
- This achieves read-only analysis with no ability to modify the codebase

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Using `--dangerously-skip-permissions` locally | Removes all safety guardrails on your machine | Only use in CI; use `--allowedTools` for local automation |
| No `--max-turns` in CI | Agent can loop indefinitely, consuming credits | Always set a reasonable cap (3-15 depending on task) |
| `--system-prompt` when you mean `--append-system-prompt` | Destroys default behaviors and tool instructions | Use `--append-system-prompt` unless you need full control |
| Hardcoding API keys in CI scripts | Key exposure in logs or version control | Use secrets management (env vars from secure store) |
| Using interactive mode in CI | Pipeline hangs waiting for user input | Always use `-p` (print mode) in automation |
| Ignoring exit codes | Silent failures in CI pipelines | Check `$?` or use `set -e` in scripts |
| No output format in CI | Unstructured text is hard to parse programmatically | Use `--output-format json` for machine consumption |
| Using `--continue` in parallel CI jobs | Race condition — multiple jobs resume same session | Each CI job should be a fresh invocation (no continue/resume) |
| Piping sensitive data without sanitization | Secrets in prompts may appear in logs or API calls | Sanitize inputs, redact credentials before piping |

---

## 7. Quick Reference / Cheat Sheet

```
CLI MODES
━━━━━━━━━
claude                          → Interactive TUI (default)
claude -p "prompt"              → Print mode (single-shot, stdout)
echo "x" | claude -p            → Pipe mode (stdin as prompt)
claude -c "prompt"              → Continue last conversation
claude --resume <id> "prompt"   → Resume specific session

KEY FLAGS
━━━━━━━━━
--print / -p                    → Non-interactive single response
--continue / -c                 → Resume last session
--resume <id>                   → Resume specific session
--model <id>                    → Override model
--output-format text|json|stream-json → Output control (print mode)
--max-turns <n>                 → Limit agentic iterations
--system-prompt <text>          → Replace system prompt (print only)
--append-system-prompt <text>   → Add to system prompt (print only)
--allowedTools <tools>          → Comma-separated allow list
--disallowedTools <tools>       → Comma-separated deny list
--add-dir <path>                → Add directory to context
--verbose                       → Show detailed reasoning
--dangerously-skip-permissions  → Skip ALL permission checks (CI only!)

OUTPUT FORMATS
━━━━━━━━━━━━━━
text         → Plain text response (default)
json         → Full structured result with metadata and cost
stream-json  → Newline-delimited JSON events (real-time streaming)

CI/CD CHECKLIST
━━━━━━━━━━━━━━━
[ ] Use -p (print mode) — never interactive in CI
[ ] Set ANTHROPIC_API_KEY via secrets management
[ ] Set --max-turns to prevent runaway agents
[ ] Use --output-format json for structured output
[ ] Use --allowedTools for least-privilege access
[ ] Use --dangerously-skip-permissions only in trusted CI
[ ] Check exit codes (0 = success, 1 = error)
[ ] Use appropriate model for cost (haiku for simple, sonnet for complex)

SESSION MANAGEMENT
━━━━━━━━━━━━━━━━━━
--continue    → Most recent session (same project)
--resume <id> → Specific session by ID
Sessions are project-scoped (based on working directory)
Sessions store full conversation history
Context compaction happens automatically within long sessions

COMMON CI PATTERNS
━━━━━━━━━━━━━━━━━━
# Code review
git diff | claude -p --output-format json "Review this diff"

# Test generation
claude -p --max-turns 10 --dangerously-skip-permissions "Write tests for src/new.ts"

# Documentation
claude -p --append-system-prompt "Output Markdown only" "Document src/api/"

# Quick analysis (cheap model)
claude -p --model claude-haiku-4-5 "Summarize this error log"

FLAG INTERACTIONS
━━━━━━━━━━━━━━━━
--system-prompt:          ONLY works with -p (print mode)
--append-system-prompt:   ONLY works with -p (print mode)
--output-format:          ONLY works with -p (print mode)
--max-turns:              ONLY works with -p (print mode)
--continue + --resume:    Cannot combine (mutually exclusive)
--allowedTools:           Works in all modes
--dangerously-skip-permissions: Intended for -p in CI only
```
