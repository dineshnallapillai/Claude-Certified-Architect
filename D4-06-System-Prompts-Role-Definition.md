# Domain 4 | Topic 4.6: Multi-Instance & Multi-Pass Review Architectures

## 1. Core Concepts

### The Self-Review Limitation

When a model generates code and then reviews its own work in the same session, it retains the **reasoning context** from generation. This causes:
- Bias toward justifying design decisions already made
- Reluctance to question fundamental approaches
- Overlooking subtle bugs that were buried in generation reasoning
- False confidence in edge case handling

**Research finding:** Independent reviewers (without prior reasoning context) catch 30-40% more subtle bugs than self-review in the same session.

### Multi-Instance Architecture

Deploy **two independent Claude instances** for generation and review:

```
Instance 1 (Generator)          Instance 2 (Reviewer)
├─ Receives requirements        ├─ Receives ONLY code + tests
├─ Reasons about design         ├─ No access to generator's reasoning
├─ Generates code               ├─ Reviews from fresh perspective
└─ Returns code                 └─ Returns independent findings
```

**Key advantage:** Reviewer has zero knowledge of generator's reasoning. Blank slate = better coverage.

### Multi-Pass vs Single-Pass Review

| Approach | Coverage | Cost | Use Case |
|----------|----------|------|----------|
| **Single-pass** | 70% (attention dilution) | 1× | Small files (<500 lines) |
| **Multi-pass (per-file)** | 85% (focused analysis) | 1.5× | Medium PRs (5-10 files) |
| **Multi-pass (per-file + integration)** | 92% (local + cross-file) | 2-3× | Large PRs (20+ files, complex dependencies) |
| **Multi-instance + multi-pass** | 95%+ (independent + focused) | 3-4× | Critical reviews, security-sensitive code |

### Review Pass Strategies

#### Strategy 1: Per-File Local Analysis
Analyze each file independently for:
- Syntax/semantic correctness
- Function-level logic errors
- Local variable scope issues
- Style consistency within file

#### Strategy 2: Cross-File Integration Pass
Analyze interactions between files:
- Data flow across module boundaries
- Shared state mutations
- API contract violations
- Dependency inversion violations

#### Strategy 3: Security-Focused Pass
Specific review for:
- Input validation gaps
- OWASP Top 10 vulnerabilities
- Authentication/authorization errors
- Cryptographic mistakes

---

## 2. Design Patterns

### Pattern 1: Independent Instance Architecture (Generator + Reviewer)

```python
import anthropic

def generate_code(requirements: str) -> str:
    """Instance 1: Generate code with full context and reasoning."""
    generator = anthropic.Anthropic()
    
    response = generator.messages.create(
        model="claude-sonnet-4-6-20250514",
        max_tokens=4096,
        system="You are an expert Python engineer. Write clean, well-tested code.",
        messages=[{
            "role": "user",
            "content": f"Implement: {requirements}"
        }]
    )
    return response.content[0].text

def review_code_independent(code: str, tests: str) -> list:
    """Instance 2: Review ONLY the code, without generator's reasoning."""
    reviewer = anthropic.Anthropic()
    
    response = reviewer.messages.create(
        model="claude-sonnet-4-6-20250514",
        max_tokens=2048,
        system="""You are a security-focused code reviewer. 
        Analyze code from scratch without reference to generation intent.
        Assume adversarial input. Check OWASP Top 10.
        Report findings as JSON: [{"severity": "high|medium|low", "location": "line X", "issue": "...", "fix": "..."}]""",
        messages=[{
            "role": "user",
            "content": f"Review this code:\n```python\n{code}\n```\n\nTest suite:\n```\n{tests}\n```"
        }]
    )
    
    import json
    try:
        return json.loads(response.content[0].text)
    except:
        return []

# Usage
code = generate_code("Build a function that validates email addresses")
tests = "# test_email_validator.py\ndef test_valid_emails(): ...\ndef test_invalid_emails(): ..."
findings = review_code_independent(code, tests)
print(f"Independent review found {len(findings)} issues")
```

**Why this works:**
- Generator: Rich context (requirements, design decisions, tradeoffs)
- Reviewer: Blank slate (only sees code + tests, no reasoning)
- Result: Reviewer catches bugs generator missed

### Pattern 2: Multi-Pass Review (Per-File + Integration)

```python
def multi_pass_review(pr_files: list[str], all_code: str) -> dict:
    """
    Pass 1: Per-file local analysis
    Pass 2: Cross-file integration analysis
    """
    reviewer = anthropic.Anthropic()
    findings_by_file = {}
    
    # PASS 1: Local analysis
    for filepath in pr_files:
        local_response = reviewer.messages.create(
            model="claude-sonnet-4-6-20250514",
            max_tokens=1024,
            system="""Analyze this file for bugs, logic errors, and style issues.
            Focus on: correctness, variable scope, function contracts.
            Ignore cross-file concerns for now.""",
            messages=[{
                "role": "user",
                "content": f"File: {filepath}\n\n{load_file(filepath)}"
            }]
        )
        findings_by_file[filepath] = local_response.content[0].text
    
    # PASS 2: Integration analysis
    integration_response = reviewer.messages.create(
        model="claude-sonnet-4-6-20250514",
        max_tokens=2048,
        system="""Analyze cross-file interactions:
        - Data flow across modules
        - Shared state mutations
        - API contract violations
        - Dependency inversions
        Report ONLY integration-level issues, not local bugs.""",
        messages=[{
            "role": "user",
            "content": f"Full PR code:\n\n{all_code}\n\nLocal analysis:\n{json.dumps(findings_by_file)}"
        }]
    )
    
    return {
        "per_file": findings_by_file,
        "integration": integration_response.content[0].text
    }
```

**Pass distribution:**
- Pass 1 (file-level): 1 call per file → catches local bugs
- Pass 2 (integration): 1 call for all → catches cross-file issues
- Cost: ~2-3× single pass, Coverage: +20-25% more issues

### Pattern 3: Confidence-Calibrated Review

```python
def review_with_confidence(code: str) -> list:
    """Review with field-level confidence scores for routing."""
    reviewer = anthropic.Anthropic()
    
    response = reviewer.messages.create(
        model="claude-sonnet-4-6-20250514",
        max_tokens=2048,
        system="""Review code and return JSON with confidence calibration:
        [
          {
            "issue": "...",
            "severity": "high|medium|low",
            "confidence": 0.95,  # 0-1 scale
            "location": "line X",
            "explanation": "..."
          }
        ]
        
        CALIBRATION GUIDE:
        - 0.95+: Certain bugs (syntax, type errors)
        - 0.75-0.95: High confidence (logic errors with clear proof)
        - 0.50-0.75: Medium confidence (likely issues, needs verification)
        - <0.50: Speculative (interesting but unverified)""",
        messages=[{
            "role": "user",
            "content": f"Review:\n{code}"
        }]
    )
    
    import json
    findings = json.loads(response.content[0].text)
    
    # Route to human review based on confidence
    for finding in findings:
        if finding["confidence"] < 0.75:
            finding["route_to"] = "human_review"
        else:
            finding["route_to"] = "auto_comment"
    
    return findings
```

**Confidence-based routing:**
- High confidence (0.8+): Auto-comment on PR
- Medium confidence (0.5-0.8): Flag for human review
- Low confidence (<0.5): Suppress (avoid noise)

### Pattern 4: Consensus Review (Multi-Reviewer)

```python
def consensus_review(code: str, num_reviewers: int = 3) -> dict:
    """Multiple independent instances vote on findings."""
    client = anthropic.Anthropic()
    
    reviews = []
    for i in range(num_reviewers):
        response = client.messages.create(
            model="claude-sonnet-4-6-20250514",
            max_tokens=1024,
            system=f"""You are code reviewer #{i+1}. 
            Review independently without reference to other reviewers.
            Return JSON: [{{"issue": "...", "severity": "high|medium|low"}}]""",
            messages=[{
                "role": "user",
                "content": f"Review:\n{code}"
            }]
        )
        reviews.append(json.loads(response.content[0].text))
    
    # Consensus: issue must appear in 2+ reviewers to be reported
    issue_counts = {}
    for review in reviews:
        for issue in review:
            key = (issue["issue"], issue["severity"])
            issue_counts[key] = issue_counts.get(key, 0) + 1
    
    consensus_findings = []
    for (issue, severity), count in issue_counts.items():
        if count >= 2:  # Majority vote
            consensus_findings.append({
                "issue": issue,
                "severity": severity,
                "consensus": f"{count}/{num_reviewers} reviewers"
            })
    
    return {
        "all_reviews": reviews,
        "consensus": consensus_findings
    }
```

**Consensus benefits:**
- Requires 2-3 independent instances to agree
- Filters false positives significantly
- Cost: 2-3× but higher accuracy
- Best for: critical code, security-sensitive paths

---

## 3. Implementation Details

### Cost-Coverage Tradeoff

```
Approach                        Cost Multiplier    Coverage    Best For
────────────────────────────────────────────────────────────────────────
Single-pass full PR             1×                 70%         Trivial changes
Multi-pass (per-file only)      1.5×               82%         Small PRs (<10 files)
Multi-pass (file + integration) 2-3×               92%         Standard PR review
Independent instance + multi    3-4×               95%         Security-critical code
Consensus (3 instances)         3×                 88%         When accuracy >> cost
```

**Decision matrix:**
- Standard CI review → Multi-pass (file + integration)
- Pre-merge check → Multi-pass or Independent + Consensus
- Security audit → Independent instance + Consensus
- 1000-line single file → Single pass sufficient

### Attention Dilution Problem

When reviewing 14 files in one pass:

```
File 1:   ████████████ (excellent detail)
File 2:   ████████     (good detail)
File 3:   ███████      (degrading)
File 4:   ██████       (superficial)
File 5:   █████        (very thin)
Files 6-14: ████ to ██  (almost no attention)
```

**Solution:** Separate passes guarantee each file full attention:

```
Pass 1 (File-by-file):
  File 1: ████████████
  File 2: ████████████
  File 3: ████████████
  ...

Pass 2 (Integration):
  All files analyzed for cross-file issues
```

### Streaming Multi-Instance Review

```python
def streaming_multi_review(code: str):
    """Non-blocking multi-instance review with streaming."""
    import asyncio
    from anthropic import AsyncAnthropic
    
    async def review_instance(instance_id: int):
        client = AsyncAnthropic()
        stream = await client.messages.stream(
            model="claude-sonnet-4-6-20250514",
            max_tokens=1024,
            system=f"You are reviewer #{instance_id}. Review this code for bugs.",
            messages=[{"role": "user", "content": f"Code:\n{code}"}]
        )
        async for text in stream.text_stream:
            yield f"[Reviewer {instance_id}] {text}"
    
    async def main():
        # Fire all 3 instances in parallel
        async for item in asyncio.gather(
            review_instance(1),
            review_instance(2),
            review_instance(3)
        ):
            print(item)
    
    asyncio.run(main())
```

### Context Window Management in Reviews

When reviewing large files, preserve context efficiently:

```python
def review_large_file(filepath: str, max_context: int = 8000):
    """Chunk large files for review, preserving imports/structure."""
    
    # Extract critical context
    imports = extract_section(filepath, "imports")
    class_defs = extract_section(filepath, "class_definitions")
    function_sigs = extract_section(filepath, "function_signatures")
    
    # Chunk main content
    content = read_file(filepath)
    chunks = [content[i:i+4000] for i in range(0, len(content), 4000)]
    
    findings = []
    for i, chunk in enumerate(chunks):
        # Each review includes: imports + signatures + current chunk
        review_context = f"""
Imports: {imports}
Classes defined: {class_defs}
Functions: {function_sigs}

Reviewing chunk {i+1}/{len(chunks)}:
{chunk}
"""
        findings.extend(review_chunk(review_context))
    
    return findings
```

---

## 4. Reliability & Cost Optimization

### False Positive Management

High false positives (issues that aren't real) destroy developer trust. Solutions:

```python
# Strategy 1: Filter by confidence
findings = review_code(code)
high_confidence = [f for f in findings if f["confidence"] >= 0.8]
# Only report 0.8+ confidence to avoid noise

# Strategy 2: Temporary disable high-FP categories
# If security findings have 30% false positive rate, disable them while improving
if false_positive_rate("security") > 0.25:
    print("Security review disabled temporarily for prompt tuning")

# Strategy 3: Track findings type
for finding in findings:
    finding["type"] = categorize(finding)  # "correctness" | "security" | "style" | etc.
    finding["dismissed_count"] = 0  # Track dismissals

# Later: analyze which types are dismissed most
dismissal_rate_by_type = calculate_dismissals(findings_history)
# Focus improvement on high-dismissal categories
```

### Session Context Isolation for Review

When running review in a CI pipeline, create isolated sessions:

```python
def isolated_review_session(code: str, pr_context: str):
    """Start fresh session for review—no prior context bias."""
    client = anthropic.Anthropic()
    
    # NEW session (not resuming prior session)
    response = client.messages.create(
        model="claude-sonnet-4-6-20250514",
        max_tokens=2048,
        system="You are reviewing code fresh. No assumptions about prior context.",
        messages=[{
            "role": "user",
            "content": f"""
Review this code from first principles:
{pr_context}

Code:
{code}

Report ONLY issues you can verify from this code alone.
"""
        }]
    )
    
    return response
```

**Why isolation matters:**
- PR generator's session has "I designed it this way for reason X"
- Review session starts blank: "Is this design correct?"
- Without isolation, reviewer inherits generator's biases

---

## 5. Exam Scenarios

### Scenario 1: Single-Pass Review Consistency Issue
**Q:** A PR modifies 12 files across the payment module. A single-pass review finds deep issues in files 1-3, but files 10-12 receive only superficial feedback with obvious bugs missed. What is the ROOT CAUSE and BEST FIX?

**A:** **Root cause:** Attention dilution. With 12 files in one pass, the model reaches diminishing focus by file 8+.

**Best fix:** Multi-pass review:
- Pass 1: Per-file analysis (each file gets full attention)
- Pass 2: Integration analysis (cross-file data flow)

Single pass coverage: ~70%. Multi-pass coverage: ~92%.

### Scenario 2: Independent Review Instance
**Q:** Your code generation pipeline produces code and immediately reviews it in the same session. Developers report that ~30% of bugs found by external code auditors were missed. Why?

**A:** **Self-review bias.** When reviewing in the same session where code was generated, the model retains reasoning context ("I designed it this way because X"). This causes:
- Reluctance to question design decisions
- Overlooking edge cases that were discussed during generation
- Blind spots around the generator's assumptions

**Fix:** Deploy an independent review instance with:
- Fresh session (no prior context)
- Only code + tests as input (no generation reasoning)
- Different system prompt (security-focused, adversarial mindset)

Result: Independent reviewers find 30-40% more subtle bugs.

### Scenario 3: False Positive Rates
**Q:** Your security review findings show 97% overall accuracy, but developers dismiss 40% of "security" findings while only dismissing 5% of "correctness" findings. What's happening?

**A:** **High false positive rate in security category.** Aggregate accuracy (97%) masks category-specific problems. Developers lose trust in security findings due to dismissal noise.

**Fix:**
- Stratify accuracy by category (not just overall)
- Disable high-FP categories temporarily while improving
- Add confidence scoring to security findings
- Route low-confidence findings to human review
- Track dismissal patterns per finding type

### Scenario 4: Consensus vs Cost
**Q:** A critical payment feature needs review. You can do: (A) single-pass independent review ($X cost), or (B) consensus review with 3 independent instances ($3X cost). When is each appropriate?

**A:**
- **(A) Single independent review ($X):** Non-critical features, standard code, time-sensitive merges
- **(B) Consensus review ($3X):** Security-critical paths, payment logic, cryptographic code

**Threshold:** If bug severity > cost of review × 3, use consensus.

For payment code: a single missed bug could cost 10-100× the review cost. Use consensus.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Single-pass 20-file review | Attention dilution, 70% coverage | Multi-pass: file-level + integration |
| Same-session self-review | Generator bias, 30-40% bugs missed | Independent instance without prior context |
| No confidence scoring | Can't distinguish certain from speculative | Field-level confidence + threshold routing |
| Ignoring false positive rates | High dismissal → lost trust | Track by category, stratify accuracy |
| One-size-fits-all for all PRs | Overkill for trivial, insufficient for critical | Adjust approach by code risk |
| No error recovery path | Crashes break review pipeline | Graceful fallback: return findings or escalate |

---

## 7. Quick Reference / Cheat Sheet

```
MULTI-INSTANCE ARCHITECTURE
━━━━━━━━━━━━━━━━━━━━━━━━━━
Instance 1 (Generator)       Instance 2 (Reviewer)
├─ Full context              ├─ ONLY code + tests
├─ Rich reasoning            ├─ Fresh perspective
└─ Generates code            └─ Independent analysis

Result: 30-40% more bugs caught vs. self-review

MULTI-PASS STRATEGY
━━━━━━━━━━━━━━━━━━
Pass 1: Per-file analysis
  → Local correctness, variable scope, function contracts
  → 1 call per file
  → Catches ~85% of local bugs

Pass 2: Integration analysis
  → Cross-file data flow, API contracts, dependencies
  → 1 call for all files
  → Catches cross-file bugs (missed by file-level review)

Total: 1.5-2× cost, +25% coverage improvement

CONFIDENCE-BASED ROUTING
━━━━━━━━━━━━━━━━━━━━━━━
0.9-1.0 → Auto-comment (high certainty)
0.7-0.9 → Flag for human review (medium confidence)
<0.7   → Suppress (speculative, noisy)

COST DECISION MATRIX
━━━━━━━━━━━━━━━━━━━
Trivial changes              → Single pass (1×)
Standard features           → Multi-pass (2×)
Security-critical paths     → Independent + confidence (2-3×)
Mission-critical code       → Consensus 3-reviewer (3×)

Decision: If severity × 5 > review_cost, use higher tier

ANTI-PATTERNS TO AVOID
━━━━━━━━━━━━━━━━━━━━━━
✗ Single pass for 10+ files (dilution)
✗ Same-session self-review (bias)
✗ Aggregate accuracy only (hides category problems)
✗ No confidence scoring (can't triage)
✗ Ignoring false positive patterns (trust erosion)
```
