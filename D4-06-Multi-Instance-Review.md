# Domain 4 | Topic 4.6: Multi-Instance & Multi-Pass Review Architectures

## 1. Core Concepts

### Self-Review Limitations (Key Exam Concept)

A model that GENERATED code retains its reasoning context, making it less likely to question its own decisions:

```
SAME SESSION (self-review = weak):
┌─────────────────────────────────────────────┐
│ Claude generates code with reasoning R      │
│ Claude reviews own code → still "sees" R    │
│ R biases review → misses issues it created  │
└─────────────────────────────────────────────┘

INDEPENDENT INSTANCE (strong review):
┌─────────────────┐    ┌──────────────────────┐
│ Instance A:     │    │ Instance B:          │
│ Generates code  │    │ Reviews code ONLY    │
│ (has context R) │    │ (no access to R)     │
└─────────────────┘    └──────────────────────┘
                       ↑ Catches issues A missed
```

**Why independent instances catch more bugs:**
- No reasoning bias from generation
- Fresh perspective on code logic
- Can't be "anchored" by original intent
- Questions assumptions the generator took for granted

### Multi-Pass vs Single-Pass Review

| Approach | What It Does | When to Use |
|----------|-------------|-------------|
| **Single-pass** | One review of all files together | Small PRs (1-3 files) |
| **Multi-pass** | Per-file local pass + cross-file integration pass | Large PRs (5+ files) |

**Why single-pass fails on large PRs:**
- Attention dilution across many files
- Contradictory feedback (flags pattern in file A, approves identical pattern in file B)
- Misses cross-file data flow issues
- Superficial comments on later files as context fills

---

## 2. Design Patterns

### Pattern 1: Independent Review Instance

```python
# GENERATION phase (Instance A)
gen_response = client.messages.create(
    model="claude-sonnet-4-6",
    system="You are a code generator...",
    messages=[{"role": "user", "content": "Implement feature X"}]
)
generated_code = extract_code(gen_response)

# REVIEW phase (Instance B — completely independent, no generation context)
review_response = client.messages.create(
    model="claude-sonnet-4-6",
    system="You are a code reviewer. Review this code for bugs, security issues, and logic errors.",
    messages=[{"role": "user", "content": f"Review this code:\n\n{generated_code}"}]
)
# Instance B has ZERO context about WHY the code was written this way
# → more likely to catch subtle issues
```

### Pattern 2: Multi-Pass Review (Per-File + Integration)

```
PASS 1: Local analysis (per-file)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
File A → Review for local bugs, style, logic
File B → Review for local bugs, style, logic
File C → Review for local bugs, style, logic
(Each file reviewed independently, full attention)

PASS 2: Cross-file integration
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
All findings from Pass 1 + file relationships →
"Now analyze how A, B, C interact:
- Data flow between modules
- Consistent error handling across boundaries
- Interface contract violations
- Shared state mutations"
```

**Why this works:** Per-file passes ensure consistent depth. Integration pass catches cross-cutting concerns that individual file reviews miss.

### Pattern 3: Confidence-Calibrated Review

```python
review_schema = {
    "type": "object",
    "properties": {
        "findings": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "file": {"type": "string"},
                    "line": {"type": "integer"},
                    "issue": {"type": "string"},
                    "severity": {"type": "string", "enum": ["critical", "high", "medium", "low"]},
                    "confidence": {"type": "number", "minimum": 0, "maximum": 1}
                }
            }
        }
    }
}

# Route based on confidence:
# confidence > 0.9 → auto-comment on PR
# confidence 0.5-0.9 → human reviewer checks first
# confidence < 0.5 → suppress (likely false positive)
```

### Pattern 4: Session Context Isolation for Review

```python
# ❌ WRONG: Same session generates AND reviews
messages = [
    {"role": "user", "content": "Write auth middleware"},
    {"role": "assistant", "content": "Here's the middleware: ..."},
    {"role": "user", "content": "Now review what you just wrote"}
    # Claude still has reasoning context → weak review
]

# ✅ CORRECT: Fresh session for review
# Session 1: Generate
gen_messages = [{"role": "user", "content": "Write auth middleware"}]
code = generate(gen_messages)

# Session 2: Review (no history from generation)
review_messages = [{"role": "user", "content": f"Review this code:\n{code}"}]
findings = review(review_messages)
```

---

## 3. Implementation Details

### Cost-Coverage Tradeoff

| Architecture | Cost | Coverage |
|-------------|------|----------|
| Self-review (same session) | 1x | Low (biased) |
| Independent instance | 2x | High (fresh perspective) |
| Multi-pass (per-file + integration) | N+1x | Highest for large PRs |
| Consensus (3 independent instances) | 3x | Highest confidence |

### When Consensus Review (3x cost) is Worth It

- Security-critical code changes
- Financial transaction logic
- Data migration scripts
- Public API contract changes
- Changes touching authentication/authorization

### Multi-Pass Split Strategy for CI

```bash
# Split large PR into focused review passes
FILES=$(git diff --name-only main)

# Pass 1: Review each file individually
for file in $FILES; do
  claude -p "Review this file for local issues: $(cat $file)" >> findings.json
done

# Pass 2: Cross-file integration review
claude -p "Given these per-file findings and the full diff, identify cross-file issues:
$(cat findings.json)
$(git diff main)" >> integration_findings.json
```

---

## 4. Security & Reliability

### False Positive Management

High false positive rates destroy developer trust. Strategies:
- Temporarily disable high-FP categories while improving prompts
- Confidence-based routing (only auto-post high-confidence findings)
- Track which findings developers dismiss → improve prompts for those patterns

### Attention Dilution in Large Reviews

```
14-file PR, single pass:
- Files 1-3: Detailed, insightful feedback
- Files 4-8: Moderate feedback
- Files 9-14: Superficial, "looks fine" ← ATTENTION DILUTED

14-file PR, multi-pass:
- Each file: Full attention, consistent depth
- Integration pass: Focused on relationships only
```

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A single-pass review of 14 files produces contradictory feedback — flagging a pattern as problematic in file A but approving identical code in file B. How should you restructure?

A) Use a larger model with more context  
B) Split into per-file local passes, then a separate cross-file integration pass  
C) Run three independent reviews and flag issues that appear in 2+ runs  
D) Require developers to split large PRs into 3-4 file submissions  

**A:** **B** — Multi-pass review directly addresses attention dilution. Per-file ensures consistent depth; integration pass catches cross-file issues. Option A misunderstands that larger context doesn't fix attention quality. Option C suppresses real bugs. Option D shifts burden to developers.

### Scenario 2
**Q:** Your CI pipeline uses Claude to both generate test code and review it in the same session. Reviews consistently approve the generated tests even when they contain logical gaps. What's the root cause?

A) The model isn't capable enough for review tasks  
B) The review prompt needs more specific criteria  
C) The same session retains reasoning context from generation, biasing the review  
D) Test code is harder to review than application code  

**A:** **C** — Self-review limitation. The model retains its generation reasoning and is biased toward confirming its own decisions. An independent review instance (separate session, no generation context) would catch issues the generator missed.

### Scenario 3
**Q:** You want Claude to review generated code but have a limited token budget. What's the minimum effective architecture?

A) Self-review in the same session with "be critical" instructions  
B) One independent instance that receives ONLY the code, no generation context  
C) Three independent instances with majority voting  
D) Self-review with extended thinking enabled  

**A:** **B** — One independent instance (no generation context) is the minimum effective architecture. It eliminates reasoning bias at only 2x cost. Options A and D still have self-review bias. Option C is effective but overkill for limited budgets.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Same session generates + reviews | Reasoning bias → misses own bugs | Independent instance for review |
| Single-pass review of 10+ files | Attention dilution, contradictions | Multi-pass: per-file + integration |
| "Be critical" in self-review | Doesn't overcome reasoning bias | Separate session without context |
| Auto-posting all findings | False positives erode trust | Confidence routing: auto/human/suppress |
| All files reviewed with equal priority | Security files get same depth as docs | Priority routing by file risk |

---

## 7. Quick Reference / Cheat Sheet

```
SELF-REVIEW LIMITATION
━━━━━━━━━━━━━━━━━━━━━━
Same session = retains reasoning context = biased review
Independent instance = no generation context = catches more bugs
MINIMUM: Separate API call with ONLY the code (no conversation history)

MULTI-PASS REVIEW
━━━━━━━━━━━━━━━━━
Pass 1: Per-file local analysis (each file gets full attention)
Pass 2: Cross-file integration (data flow, contracts, consistency)
Use for: PRs with 5+ files, contradictory single-pass results

CONFIDENCE ROUTING
━━━━━━━━━━━━━━━━━━
> 0.9 → Auto-comment on PR
0.5–0.9 → Human reviewer checks first
< 0.5 → Suppress (likely false positive)

COST vs COVERAGE
━━━━━━━━━━━━━━━━
Self-review: 1x cost, LOW coverage (biased)
Independent: 2x cost, HIGH coverage (fresh eyes)
Multi-pass: N+1x cost, HIGHEST for large PRs
Consensus (3x): 3x cost, HIGHEST confidence

KEY EXAM ANSWERS
━━━━━━━━━━━━━━━━
• Contradictory feedback on large PRs → Multi-pass (per-file + integration)
• Same-session review misses bugs → Independent instance (no generation context)
• False positives eroding trust → Confidence routing + disable high-FP categories
• Large review cost concern → Independent instance is minimum effective (2x)
```
