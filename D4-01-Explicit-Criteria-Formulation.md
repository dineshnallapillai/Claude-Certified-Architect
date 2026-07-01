# Domain 4 | Topic 4.1: Explicit Criteria Formulation

## 1. Core Concepts

### What is Explicit Criteria Formulation?

Writing prompts with **clear, unambiguous, measurable criteria** so Claude knows exactly what "good" looks like. Instead of vague instructions, you define specific success conditions.

### Vague vs Explicit

```
VAGUE: "Write good code"
EXPLICIT: "Write TypeScript code that:
  - Passes strict mode (`--strict`)
  - Has no any types
  - Functions under 30 lines
  - All public functions have JSDoc
  - Error cases return typed Result<T, Error>"
```

### Why Criteria Matter for Agents

In agentic systems, Claude makes autonomous decisions. Without explicit criteria:
- Claude uses its own judgment (unpredictable)
- Results vary between runs (non-deterministic)
- Validation is impossible (no success definition)
- Quality drift over long sessions

### Criteria Types

| Type | Purpose | Example |
|------|---------|---------|
| **Functional** | What it must do | "Returns user profile with all required fields" |
| **Non-functional** | How well it must do it | "Response time < 200ms" |
| **Constraint** | What it must NOT do | "Never expose PII in logs" |
| **Format** | How output should look | "JSON with snake_case keys" |
| **Completion** | When to stop | "All tests pass, linter clean" |

---

## 2. Design Patterns

### Pattern 1: Criteria in System Prompts

```python
system_prompt = """You are a code review agent. When reviewing code, evaluate against these criteria:

MUST PASS (blocking):
- No SQL injection vulnerabilities
- No hardcoded secrets
- No unbounded loops without exit conditions
- All error paths handled (no swallowed exceptions)

SHOULD PASS (advisory):
- Functions under 50 lines
- Cyclomatic complexity under 10
- Variable names are descriptive (not single letters except loop counters)
- Public APIs have documentation

REPORT FORMAT:
For each issue, provide:
- Severity: CRITICAL / HIGH / MEDIUM / LOW
- Location: file:line
- Issue: one-sentence description
- Fix: concrete suggestion
"""
```

### Pattern 2: Output Schema as Criteria

```json
{
  "name": "analyze_code",
  "description": "Analyze code quality against project standards",
  "input_schema": {
    "type": "object",
    "properties": {
      "file_path": {"type": "string"}
    }
  },
  "output_criteria": {
    "type": "object",
    "properties": {
      "passes": {"type": "boolean"},
      "score": {"type": "integer", "minimum": 0, "maximum": 100},
      "issues": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "severity": {"enum": ["critical", "high", "medium", "low"]},
            "line": {"type": "integer"},
            "message": {"type": "string"},
            "fix": {"type": "string"}
          }
        }
      }
    }
  }
}
```

### Pattern 3: Grading Rubric

```markdown
## Code Quality Rubric

### A (Excellent) - Score 90-100
- Zero critical/high issues
- Clean architecture
- Comprehensive error handling
- Well-tested (>80% coverage for new code)

### B (Good) - Score 70-89
- No critical issues
- Minor style issues acceptable
- Error handling covers main paths
- Key paths tested

### C (Acceptable) - Score 50-69
- No critical issues
- Some missing error handling
- Works for happy path
- Some tests exist

### D (Needs Work) - Score below 50
- Has critical issues OR
- Missing error handling for common cases OR
- No tests for new functionality
```

### Pattern 4: Completion Criteria for Agentic Tasks

```markdown
## Task: Implement User Registration

### Done When:
1. [ ] POST /api/v1/register endpoint exists
2. [ ] Validates: email format, password strength (8+ chars, mixed case)
3. [ ] Returns 201 with user ID on success
4. [ ] Returns 400 with specific validation errors
5. [ ] Returns 409 if email already exists
6. [ ] Passwords hashed with bcrypt (cost 12)
7. [ ] Unit tests cover all 5 response scenarios
8. [ ] Integration test with real database
9. [ ] No sensitive data in response body (no password hash)
10. [ ] Rate limited to 5 attempts per IP per minute
```

---

## 3. Implementation Details

### Criteria in CLAUDE.md

```markdown
# Code Quality Criteria

## All Code Must:
- TypeScript strict mode compliant
- No `any` types (use `unknown` + type guards)
- No `console.log` in production code (use logger)
- All async functions have error handling
- No nested callbacks deeper than 2 levels

## Tests Must:
- Cover happy path + at least 2 error cases
- Use descriptive names: `test_[function]_[scenario]_[expected]`
- Mock external services (never hit real APIs)
- Run in under 5 seconds each
- Be deterministic (no random, no time-dependent)
```

### Measurable vs Unmeasurable Criteria

| Measurable (GOOD) | Unmeasurable (BAD) |
|-------------------|-------------------|
| "Under 30 lines" | "Short functions" |
| "Responds in < 200ms" | "Fast response" |
| "Zero critical vulnerabilities" | "Secure code" |
| "100% of error paths return typed errors" | "Good error handling" |
| "All public functions have JSDoc" | "Well documented" |

### Criteria Hierarchy for Conflicts

```
When criteria conflict, prioritize in this order:
1. Security (never compromise)
2. Correctness (must work correctly)
3. Reliability (must handle errors)
4. Performance (should be efficient)
5. Readability (should be clear)
6. Style (nice to have)
```

---

## 4. Security & Reliability

### Security Criteria Template

```markdown
## Security Criteria (Non-Negotiable)

### Input Handling:
- All user inputs validated and sanitized
- SQL queries use parameterized statements
- No string interpolation in queries
- File paths normalized, no traversal possible

### Authentication:
- Passwords hashed (bcrypt, cost ≥ 12)
- Tokens have expiration (max 24h for access, 30d refresh)
- Failed attempts rate-limited

### Data Protection:
- PII never in logs
- Sensitive data encrypted at rest
- API responses exclude internal fields
- Error messages don't expose stack traces
```

---

## 5. Exam Scenarios

### Scenario 1
**Q:** An agent is producing inconsistent code quality across different runs. Some outputs are excellent, others miss error handling. What is the ROOT CAUSE and best fix?

**A:** Root cause: **Implicit criteria** — the agent has no explicit definition of what "good" means. Fix: Add explicit criteria to the system prompt:
```
Every function you write must:
1. Handle all error cases with typed errors
2. Validate inputs at the boundary
3. Include at minimum a happy-path test
4. Be under 40 lines
```

Explicit criteria make output deterministic and verifiable.

### Scenario 2
**Q:** How should completion criteria differ between a "plan mode" task and a "direct execution" task?

**A:**
- **Plan mode:** Criteria define what the PLAN should contain (steps, risks, verification approach)
- **Direct execution:** Criteria define what the RESULT should look like (working code, passing tests, specific behavior)

Example plan criteria: "Plan must include: affected files, execution order, rollback approach, and verification steps"
Example execution criteria: "All tests pass, no lint errors, endpoint returns expected 200 response"

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| "Write good code" | Undefined "good" | Explicit measurable criteria |
| Criteria without priority | Conflicting rules can't be resolved | Numbered priority order |
| Too many criteria (50+) | Claude overwhelmed, misses some | 5-10 critical criteria |
| Only positive criteria | Doesn't know what to avoid | Include constraints (must NOT) |
| Criteria without format spec | Output varies | Define exact output structure |

---

## 7. Quick Reference / Cheat Sheet

```
CRITERIA FORMULA
━━━━━━━━━━━━━━━
[WHAT] must [VERB] [MEASURABLE CONDITION]

CRITERIA TYPES
━━━━━━━━━━━━━━
• MUST: Non-negotiable (blocking)
• SHOULD: Important but flexible (advisory)
• MUST NOT: Constraints (blocking)
• FORMAT: Output structure
• DONE WHEN: Completion conditions

GOOD CRITERIA ARE
━━━━━━━━━━━━━━━━━
✓ Measurable (can verify programmatically)
✓ Specific (no ambiguity)
✓ Prioritized (what matters most)
✓ Complete (covers all dimensions)
✓ Testable (clear pass/fail)

PRIORITY ORDER
━━━━━━━━━━━━━━
1. Security → 2. Correctness → 3. Reliability
→ 4. Performance → 5. Readability → 6. Style
```
