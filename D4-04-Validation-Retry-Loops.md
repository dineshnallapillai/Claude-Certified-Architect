# Domain 4 | Topic 4.4: Validation-Retry Loops

## 1. Core Concepts

### What is a Validation-Retry Loop?

A pattern where output is **validated against criteria**, and if it fails, fed back with error details for correction — repeating until success or max attempts.

```
Generate → Validate → [Pass?] → Done
                       ↓ No
              Feed errors back → Regenerate → Validate → ...
```

### Why Validation-Retry Matters

- Claude's first attempt isn't always perfect
- Complex outputs (code, structured data) often need refinement
- Automated validation catches issues Claude's reasoning missed
- Converges toward correct output without human intervention

### Loop Components

| Component | Role | Example |
|-----------|------|---------|
| **Generator** | Produces the output | Claude writes code |
| **Validator** | Checks output against criteria | Linter, tests, schema check |
| **Error formatter** | Structures feedback for retry | "Line 42: type error: expected string" |
| **Termination condition** | When to stop | Pass OR max_retries reached |
| **Escalation** | What to do on persistent failure | Ask user, try different approach |

---

## 2. Design Patterns

### Pattern 1: Code Generation with Test Validation

```python
def generate_with_validation(spec, max_retries=3):
    messages = [{"role": "user", "content": f"Write code that: {spec}"}]
    
    for attempt in range(max_retries):
        # Generate
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            messages=messages
        )
        code = extract_code(response)
        
        # Validate
        lint_result = run_linter(code)
        test_result = run_tests(code)
        
        if lint_result.passed and test_result.passed:
            return code  # Success!
        
        # Format errors for retry
        error_feedback = format_errors(lint_result, test_result)
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": f"""
Your code has the following issues. Please fix them:

{error_feedback}

Provide the complete corrected code.
"""})
    
    # Escalate on persistent failure
    return escalate("Code generation failed after {max_retries} attempts", messages)
```

### Pattern 2: JSON Schema Validation Loop

```python
import jsonschema

EXPECTED_SCHEMA = {
    "type": "object",
    "properties": {
        "summary": {"type": "string", "minLength": 10},
        "score": {"type": "integer", "minimum": 0, "maximum": 100},
        "tags": {"type": "array", "items": {"type": "string"}, "minItems": 1}
    },
    "required": ["summary", "score", "tags"]
}

def generate_structured_output(prompt, schema=EXPECTED_SCHEMA, max_retries=3):
    messages = [{"role": "user", "content": f"{prompt}\n\nRespond with JSON matching this schema: {json.dumps(schema)}"}]
    
    for attempt in range(max_retries):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            messages=messages
        )
        
        try:
            output = json.loads(extract_json(response.content[0].text))
            jsonschema.validate(output, schema)
            return output  # Valid!
        except json.JSONDecodeError as e:
            error = f"Invalid JSON: {e}. Ensure your response is valid JSON."
        except jsonschema.ValidationError as e:
            error = f"Schema validation failed: {e.message}. Path: {e.json_path}"
        
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": f"Error: {error}\nPlease fix and respond with valid JSON only."})
    
    raise ValidationFailure("Could not generate valid structured output")
```

### Pattern 3: Multi-Stage Validation

```python
def multi_stage_validate(code):
    """Validate in order of cheapness (fast checks first)."""
    
    # Stage 1: Syntax check (instant)
    syntax = check_syntax(code)
    if not syntax.valid:
        return ValidationResult(stage="syntax", errors=syntax.errors)
    
    # Stage 2: Type check (fast)
    types = check_types(code)
    if not types.valid:
        return ValidationResult(stage="types", errors=types.errors)
    
    # Stage 3: Lint (medium)
    lint = run_linter(code)
    if not lint.valid:
        return ValidationResult(stage="lint", errors=lint.errors)
    
    # Stage 4: Tests (slow, expensive)
    tests = run_tests(code)
    if not tests.valid:
        return ValidationResult(stage="tests", errors=tests.errors)
    
    return ValidationResult(stage="all", valid=True)
```

### Pattern 4: Bounded Retry with Backoff Strategy

```python
def smart_retry(prompt, validators, max_retries=3):
    """Each retry gives more specific guidance."""
    
    strategies = [
        "Please fix the errors below.",                    # Attempt 1: minimal
        "Focus specifically on: {errors}. Ignore else.",  # Attempt 2: focused
        "Here is a working example: {example}. Match it." # Attempt 3: prescriptive
    ]
    
    messages = [{"role": "user", "content": prompt}]
    
    for attempt, strategy in enumerate(strategies[:max_retries]):
        response = generate(messages)
        result = validate(response, validators)
        
        if result.valid:
            return response
        
        # Escalating specificity in feedback
        feedback = strategy.format(
            errors=result.errors,
            example=get_working_example()
        )
        messages.extend([
            {"role": "assistant", "content": response},
            {"role": "user", "content": feedback}
        ])
    
    return None  # Failed after all strategies
```

---

## 3. Implementation Details

### Error Feedback Formatting

```python
def format_errors_for_claude(validation_result):
    """Structure errors so Claude can efficiently fix them."""
    
    feedback = "The following issues were found:\n\n"
    
    for error in validation_result.errors:
        feedback += f"- **{error.severity}** at {error.location}: {error.message}\n"
        if error.suggestion:
            feedback += f"  Suggestion: {error.suggestion}\n"
    
    feedback += "\nPlease fix ALL issues and provide the complete corrected output."
    
    return feedback
```

### When to Stop Retrying

| Signal | Action |
|--------|--------|
| All validations pass | Return result (success) |
| Max retries reached | Escalate or return best attempt |
| Same error repeated 2+ times | Agent is stuck — change strategy |
| Error count increasing | Getting worse — revert to attempt 1 |
| Critical error (security) | Stop immediately, don't keep trying |

### Cost Control

```python
class RetryBudget:
    def __init__(self, max_tokens=50000, max_retries=5):
        self.tokens_used = 0
        self.max_tokens = max_tokens
        self.attempts = 0
        self.max_retries = max_retries
    
    def can_retry(self):
        return (self.attempts < self.max_retries and 
                self.tokens_used < self.max_tokens)
    
    def record_attempt(self, tokens):
        self.attempts += 1
        self.tokens_used += tokens
```

---

## 4. Security & Reliability

### Validation as Security Layer

```python
def secure_code_validation(code):
    """Validate generated code for security before execution."""
    
    DANGEROUS_PATTERNS = [
        r'eval\(', r'exec\(', r'subprocess\.call\(',
        r'os\.system\(', r'__import__\(',
        r'rm\s+-rf', r'DROP\s+TABLE'
    ]
    
    for pattern in DANGEROUS_PATTERNS:
        if re.search(pattern, code):
            return ValidationResult(
                valid=False,
                errors=[f"Security: dangerous pattern '{pattern}' detected"],
                block=True  # Don't retry, escalate
            )
    
    return ValidationResult(valid=True)
```

### Infinite Loop Prevention

```python
MAX_RETRIES = 5
SAME_ERROR_THRESHOLD = 2

error_history = []

for attempt in range(MAX_RETRIES):
    result = generate_and_validate()
    
    if result.valid:
        break
    
    # Detect stuck loops
    if result.error_signature in error_history[-SAME_ERROR_THRESHOLD:]:
        # Same error repeated — change approach, don't just retry
        raise StuckLoopError("Agent making same error repeatedly")
    
    error_history.append(result.error_signature)
```

---

## 5. Exam Scenarios

### Scenario 1
**Q:** Claude generates code that passes linting but fails 2 out of 10 tests. After retry with test errors, it fixes those 2 but breaks 1 other test. What pattern prevents this regression?

**A:** **Regression-aware validation:**
1. Track which tests pass on each attempt
2. On retry, include BOTH the new failures AND a warning: "Tests X, Y must CONTINUE to pass"
3. Consider: accept the version with the fewest failures (greedy best)
4. Or: include all test expectations in a single validation pass

```python
if new_failures and fixed_previous:
    feedback = f"You fixed {fixed_previous}, but broke {new_failures}. "
    feedback += f"ALL of these must pass simultaneously: {all_tests}"
```

### Scenario 2
**Q:** A validation-retry loop for JSON generation reaches max retries (3) with Claude still producing invalid output. What should the system do?

**A:** Escalation options (choose based on context):
1. **Return best attempt** with validation errors flagged
2. **Try different prompt strategy** (more prescriptive, include example)
3. **Simplify the schema** (break into smaller pieces)
4. **Escalate to human** with the failed attempts and errors
5. **Use forced tool_choice** with the schema as input_schema (most reliable for structured output)

### Scenario 3
**Q:** What is the MOST token-efficient way to handle validation-retry for structured output?

**A:** Use **tool_choice with forced tool use** instead of text-based JSON:
```python
response = client.messages.create(
    tools=[{"name": "output", "input_schema": expected_schema}],
    tool_choice={"type": "tool", "name": "output"},
    messages=[{"role": "user", "content": prompt}]
)
# Claude's tool input IS the structured output — schema-validated by design
```
This eliminates JSON parsing errors and often succeeds on first attempt (no retry needed).

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Unlimited retries | Token explosion | Bounded (3-5 max) |
| Same feedback each retry | Claude repeats same mistake | Escalating specificity |
| Validating only final output | Late failure discovery | Validate incrementally |
| No error context in retry prompt | Claude guesses what to fix | Include specific errors + locations |
| Retrying security violations | Dangerous | Block immediately, don't retry |
| Ignoring regression between retries | Fix one, break another | Track all validations across attempts |

---

## 7. Quick Reference / Cheat Sheet

```
VALIDATION-RETRY PATTERN
━━━━━━━━━━━━━━━━━━━━━━━━
Generate → Validate → Pass? → Done
                      Fail? → Format errors → Retry (max N)

VALIDATION ORDER (cheapest first)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Syntax check (instant)
2. Schema validation (fast)
3. Type checking (fast)
4. Linting (medium)
5. Unit tests (slow)
6. Integration tests (slowest)

RETRY BUDGET
━━━━━━━━━━━━
• Max retries: 3-5 typical
• Max tokens: set a budget
• Same error 2x: change strategy (don't just retry)
• Error count increasing: revert, try different approach

ERROR FEEDBACK FORMAT
━━━━━━━━━━━━━━━━━━━━━
"Issues found:
- [SEVERITY] at [LOCATION]: [MESSAGE]
  Suggestion: [FIX]
Fix ALL issues. Provide complete corrected output."

TERMINATION CONDITIONS
━━━━━━━━━━━━━━━━━━━━━━
✓ All validations pass → success
✓ Max retries reached → escalate
✓ Same error repeating → change approach
✓ Security violation → stop immediately
✓ Budget exhausted → return best attempt
```
