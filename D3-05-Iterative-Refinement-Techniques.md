# Domain 3 | Topic 3.5: Iterative Refinement Techniques

## 1. Core Concepts

### The Five Iterative Refinement Techniques (Exam-Critical)

The exam tests five specific techniques for progressively improving Claude's output:

```
TECHNIQUE                          USE WHEN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Concrete input/output examples   Prose descriptions → inconsistent results
2. Test-driven iteration            Need verified correctness with ratchet
3. The interview pattern            Unfamiliar domain, unknown requirements
4. Interacting vs independent       Multiple issues to fix
5. Specific test cases              Edge cases not handled
```

---

### Technique 1: Concrete Input/Output Examples

**The most effective way to communicate expected transformations** when prose descriptions are interpreted inconsistently.

```
❌ PROSE (inconsistent):
"Clean up the customer data to a standard format"

✅ CONCRETE EXAMPLES (consistent):
Input:  {"name": "JOHN doe", "phone": "555.123.4567", "date": "3/15/22"}
Output: {"name": "John Doe", "phone": "+15551234567", "date": "2022-03-15"}

Input:  {"name": "jane SMITH", "phone": "(555) 987-6543", "date": "12-01-2021"}
Output: {"name": "Jane Smith", "phone": "+15559876543", "date": "2021-12-01"}
```

**Why 2-3 examples beat paragraphs of instructions:**
- No ambiguity about format expectations
- Implicitly teach casing, normalization, date format
- Claude generalizes the pattern to novel inputs

---

### Technique 2: Test-Driven Iteration

Write the test suite FIRST, then iterate by sharing specific failures:

```
WORKFLOW:
1. Write tests defining expected behavior (happy path + edge cases)
2. Ask Claude to implement
3. Run tests → some fail
4. Share SPECIFIC failure messages (not "some tests fail")
5. Claude fixes based on precise error information
6. Repeat until all green

EFFECTIVE FAILURE SHARING:
"Two tests failing:
- test_null_handling: AssertionError: Expected None, got '' for missing 'address'
- test_unicode: Expected 'José García', got 'Jos? Garc?a' (encoding lost)"
```

**Why this works:** Tests act as a **ratchet** — once a test passes, regressions are immediately caught. Each iteration adds information, never subtracts it.

---

### Technique 3: The Interview Pattern

Have Claude **ask questions to surface considerations you haven't anticipated** before implementing:

```
USER: "Add caching to our API"

WITHOUT INTERVIEW:
Claude immediately implements a basic in-memory cache (probably wrong)

WITH INTERVIEW:
Claude: "Before implementing, I have some questions:
1. What's your read/write ratio? (affects eviction strategy)
2. Single instance or multi-node? (affects cache coherence)
3. What's acceptable staleness? (affects TTL design)
4. Should we use stale-while-revalidate on miss?
5. Any data that must NEVER be cached? (compliance)"

USER: "Multi-node, 10:1 reads, 30s staleness OK, never cache PII"

Claude: [Now implements correctly the first time]
```

**When to use:**
- Unfamiliar domains (you don't know what you don't know)
- Complex features with hidden design decisions
- Ambiguous or underspecified requirements
- When tradeoffs are non-obvious

**When NOT to use:**
- Simple well-defined tasks ("fix this null pointer")
- Requirements already exhaustively specified
- Urgent fixes where exploration is costly

---

### Technique 4: Interacting vs Independent Issues

**Interacting issues** (fixes affect each other) → Report in a **SINGLE message**:

```
"Fix these together — they interact:
1. Retry logic uses fixed delay (needs exponential backoff)
2. Timeout too low for retried requests (depends on #1's delay)
3. Circuit breaker threshold ignores retry attempts (depends on both)

Changing retry timing (#1) affects timeout requirements (#2),
and both affect when the circuit breaker should trip (#3)."
```

**Independent issues** (fixes are isolated) → Fix **SEQUENTIALLY**, one per turn:

```
Turn 1: "Fix the date formatting bug in the export function"
  → Claude fixes
Turn 2: "Now add the missing null check in the validator"
  → Claude fixes (doesn't touch export)
Turn 3: "Now update the 404 error message"
  → Claude fixes (independent of both above)
```

**Decision rule:** "Would fixing A change how I need to fix B?" → YES = together, NO = separate.

---

### Technique 5: Specific Test Cases with Expected Output

When edge cases aren't handled, provide exact input → expected → actual:

```
"The migration script drops null values. Specific test case:

Input:  {"id": 42, "name": "Test", "address": null, "score": 0}
Expected: {"id": 42, "name": "Test", "address": null, "score": 0}
Actual:   {"id": 42, "name": "Test", "score": 0}  ← address DROPPED

Null fields must be preserved, not dropped."
```

This gives Claude everything needed for a one-iteration fix.

---

## 2. Design Patterns

### Pattern 1: Progressive Example Escalation

```
ROUND 1 — Happy path:
"March 15, 2024"  → "2024-03-15"
"Jan 1, 2023"    → "2023-01-01"

ROUND 2 — Edge cases (after basic works):
"15/03/2024"     → "2024-03-15" (European format)
"2024-03-15"     → "2024-03-15" (already correct)
""               → null
"sometime in Q1" → null (unparseable)
```

### Pattern 2: Test-First Development Loop

```
STEP 1: Write test suite
  → Happy path + edge cases + performance requirements

STEP 2: "Implement to pass these tests"
  → Claude writes implementation

STEP 3: Run tests, share failures
  → "test_concurrent fails: count 103, expected 100 (race condition)"

STEP 4: Claude fixes specifically
  → "Added atomic CAS for thread-safe increment"

STEP 5: All tests green → DONE
```

### Pattern 3: Interview → Implement → Iterate

```
Phase 1: Interview (surface requirements)
  → Claude asks 5-7 questions
  → You answer with constraints

Phase 2: Implement (first attempt with full context)
  → Usually much better than blind implementation

Phase 3: Test-driven refinement (if edge cases remain)
  → Share specific test failures
  → Fix in 1-2 iterations
```

### Pattern 4: Feedback Effectiveness

| Ineffective | Effective |
|-------------|-----------|
| "That's wrong" | "Expected 'Doe, John' but got 'John Doe' — last name first" |
| "Try again" | "Fails on European dates: '15/03/2024' → should be '2024-03-15'" |
| "Be more careful" | Specific: input X → expected Y → got Z |
| "Fix the bugs" | "test_null: got empty string, expected None for missing field" |

---

## 3. Implementation Details

### Technique Selection Guide

| Situation | Best Technique |
|-----------|---------------|
| Claude produces wrong format | Input/output examples (2-3) |
| Implementation has bugs | Test-driven iteration |
| Complex unfamiliar domain | Interview pattern first |
| Multiple related bugs | Single message (interacting) |
| Multiple unrelated bugs | Sequential turns (independent) |
| Edge cases missed | Specific test cases |
| Inconsistent output style | More examples covering variations |

### Iteration Efficiency

```
MOST EFFICIENT:
Interview (if unfamiliar domain)
  → Examples (define transformation)
    → Test-driven loop (verify)
      → Specific test cases (edge cases)

LEAST EFFICIENT (common mistake):
Vague description → "try again" → "try again" → frustration
```

---

## 4. Security & Reliability

- **Security code:** Write security tests FIRST (test-driven), never iterate blindly
- **Data migrations:** Full test suite before implementation, never partial iteration
- **Interview pattern for security:** Surfaces threat models you haven't considered

---

## 5. Exam Scenarios

### Scenario 1
**Q:** You ask Claude to "clean up the data format" for a pipeline. After 3 iterations, results are inconsistent — sometimes title-casing, sometimes not. What is MOST effective?

A) Add "be consistent" to the prompt  
B) Provide 2-3 concrete input/output examples showing exact expected transformation  
C) Switch to a larger model  
D) Break into separate prompts per field  

**A:** **B** — Concrete input/output examples are the most effective technique when prose produces inconsistent results.

### Scenario 2
**Q:** You're building a parser for an unfamiliar document format. What technique should you use BEFORE implementation?

A) Write a complete specification  
B) Write a full test suite  
C) Use the interview pattern — have Claude ask questions first  
D) Start implementing and fix bugs as they appear  

**A:** **C** — Interview pattern surfaces considerations you don't know about in unfamiliar domains. You can't write specs (A) or tests (B) for things you haven't discovered yet.

### Scenario 3
**Q:** Three bugs found: (1) wrong timezone in date formatter, (2) missing null check in email validator, (3) timezone test that fails in CI. How to report?

A) All three together  
B) Timezone issues (#1 and #3) together, then #2 separately  
C) All three sequentially  
D) Create a bug document and ask Claude to fix them in order  

**A:** **B** — #1 and #3 interact (fixing the formatter affects the test). #2 is independent. Group interacting issues; separate independent ones.

### Scenario 4
**Q:** Claude generates a migration script that drops null values. What is the most efficient feedback?

A) "Handle nulls correctly"  
B) "The script has a null handling bug"  
C) "Input: {id: 42, address: null} → Expected: {id: 42, address: null} → Actual: {id: 42}. Preserve nulls."  
D) "Run the tests and fix what fails"  

**A:** **C** — Specific test case with input/expected/actual gives Claude everything for a one-iteration fix.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| "Try again" without specifics | No direction for improvement | Specific expected vs actual |
| Vague prose for format requirements | Inconsistent interpretation | 2-3 concrete examples |
| Implementing in unfamiliar domain without questions | Missing requirements found late | Interview pattern first |
| Grouping unrelated issues | Unnecessary coupling, scope confusion | Fix independently in sequence |
| Separating related issues | Fix A breaks fix B | Group interacting issues together |
| Blind retry loops | No progress per iteration | Each iteration adds information |
| Skipping tests before implementation | No ratchet against regressions | Test-driven: tests first |

---

## 7. Quick Reference / Cheat Sheet

```
THE 5 TECHNIQUES
━━━━━━━━━━━━━━━━
1. INPUT/OUTPUT EXAMPLES → When format/transformation is inconsistent
2. TEST-DRIVEN ITERATION → When correctness needs verification
3. INTERVIEW PATTERN     → When domain is unfamiliar (ask before build)
4. INTERACTING/INDEPENDENT → How to report multiple issues
5. SPECIFIC TEST CASES   → When edge cases need fixing

INTERVIEW PATTERN
━━━━━━━━━━━━━━━━━
Claude asks questions BEFORE implementing
Surfaces considerations you haven't thought of
Use for: caching, auth, pipelines, unfamiliar domains
Skip for: simple tasks, exhaustive specs already exist

INTERACTING vs INDEPENDENT
━━━━━━━━━━━━━━━━━━━━━━━━━━
"Would fixing A change how I fix B?"
  YES → Single message (interacting)
  NO  → Sequential turns (independent)

EFFECTIVE FEEDBACK FORMULA
━━━━━━━━━━━━━━━━━━━━━━━━━━
"Input: [X] → Expected: [Y] → Actual: [Z]. The issue is [specific cause]."

NEVER: "Try again" / "That's wrong" / "Be more careful"
ALWAYS: Concrete input, expected output, actual output
```
