# Domain 4 | Topic 4.2: Few-Shot Prompting Techniques

## 1. Core Concepts

### What is Few-Shot Prompting?

**Few-shot prompting** provides Claude with examples of desired input-output pairs before asking it to handle new inputs. Examples demonstrate the pattern better than descriptions alone.

```
ZERO-SHOT: "Classify this ticket as bug/feature/question"
ONE-SHOT:  "Example: 'App crashes on login' → bug. Now classify: ..."
FEW-SHOT:  "[3-5 examples covering edge cases]. Now classify: ..."
```

### Why Few-Shot Works

| Benefit | Explanation |
|---------|-------------|
| Pattern demonstration | Shows exact format expected |
| Edge case coverage | Examples handle ambiguous cases |
| Reduced ambiguity | Concrete > abstract instructions |
| Consistent output | Claude mimics example format exactly |
| Implicit rules | Patterns in examples teach without explicit rules |

### Zero-Shot vs Few-Shot vs Many-Shot

| Approach | Examples | Best For |
|----------|----------|----------|
| Zero-shot | 0 | Simple, well-defined tasks |
| One-shot | 1 | Format demonstration |
| Few-shot | 2-5 | Pattern learning, classification |
| Many-shot | 10-50+ | Complex nuanced patterns |

---

## 2. Design Patterns

### Pattern 1: Classification Few-Shot

```xml
<examples>
<example>
<input>The login page shows a blank screen on Safari 17</input>
<output>{"category": "bug", "priority": "high", "component": "auth"}</output>
</example>

<example>
<input>Can we add dark mode to the dashboard?</input>
<output>{"category": "feature", "priority": "medium", "component": "ui"}</output>
</example>

<example>
<input>How do I reset my password?</input>
<output>{"category": "question", "priority": "low", "component": "auth"}</output>
</example>

<example>
<input>The app is slightly slow when loading 1000+ items</input>
<output>{"category": "bug", "priority": "medium", "component": "performance"}</output>
</example>
</examples>

Now classify this ticket:
<input>Users are getting 500 errors when uploading files larger than 10MB</input>
```

### Pattern 2: Code Transformation Few-Shot

```
Convert these JavaScript functions to TypeScript with proper types:

Example 1:
Input:
```js
function add(a, b) { return a + b; }
```
Output:
```ts
function add(a: number, b: number): number { return a + b; }
```

Example 2:
Input:
```js
function fetchUser(id) { return fetch(`/api/users/${id}`).then(r => r.json()); }
```
Output:
```ts
interface User { id: string; name: string; email: string; }
async function fetchUser(id: string): Promise<User> { 
  const response = await fetch(`/api/users/${id}`);
  return response.json() as User;
}
```

Now convert:
```js
function processItems(items, callback) { return items.map(callback).filter(Boolean); }
```
```

### Pattern 3: Structured Output Few-Shot

```
Generate API documentation in this exact format:

Example:
Function: createUser(email, password, role?)
Documentation:
---
### POST /api/v1/users
Creates a new user account.

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| email | string | Yes | Valid email address |
| password | string | Yes | Min 8 chars, mixed case |
| role | string | No | Default: "user" |

**Responses:**
- `201`: User created successfully
- `400`: Validation error
- `409`: Email already exists
---

Now document:
Function: updateProfile(userId, updates)
```

### Pattern 4: Chain-of-Thought Few-Shot

```
Analyze this code for security issues. Think step by step.

Example:
Code: `query = f"SELECT * FROM users WHERE id = {user_input}"`
Analysis:
1. The code constructs a SQL query using string formatting
2. `user_input` is directly interpolated without sanitization
3. An attacker could inject: `1 OR 1=1` to dump all users
4. This is a SQL injection vulnerability (CRITICAL)
Verdict: CRITICAL - SQL Injection
Fix: Use parameterized query: `cursor.execute("SELECT * FROM users WHERE id = %s", (user_input,))`

Now analyze:
Code: `return render_template_string(f"Hello {request.args.get('name')}")`
```

---

## 3. Implementation Details

### Example Selection Strategy

| Strategy | When to Use |
|----------|------------|
| Diverse examples | Cover different categories/cases |
| Boundary examples | Show edge cases and ambiguous cases |
| Difficulty progression | Simple → complex |
| Error examples | Show what NOT to do (negative examples) |

### Optimal Example Count

| Task Complexity | Recommended Examples |
|----------------|---------------------|
| Simple format conversion | 1-2 |
| Classification (3-5 classes) | 3-5 (at least 1 per class) |
| Complex transformation | 3-5 with progressive difficulty |
| Nuanced judgment | 5-10 covering edge cases |
| Highly ambiguous | 10-20 (many-shot) |

### Token Efficiency

```
PROBLEM: 5 examples × 500 tokens each = 2,500 tokens per call
IN AGENTIC LOOP: 2,500 × 20 iterations = 50,000 tokens just for examples

SOLUTIONS:
1. Cache examples in system prompt (only sent once with prompt caching)
2. Use concise examples (show minimum needed)
3. Dynamic example selection (only relevant examples per sub-task)
4. Move examples to tool descriptions (when tool-specific)
```

### Few-Shot in Tool Descriptions

```json
{
  "name": "categorize_feedback",
  "description": "Categorize user feedback. Examples: 'App crashed' → bug/critical; 'Would love dark mode' → feature/medium; 'How do I export?' → question/low. Always return JSON: {category, priority, summary}.",
  "input_schema": { ... }
}
```

### XML Tags for Clear Separation

```xml
<system>You are a code review agent.</system>

<examples>
<example>
  <code>if password == "admin123":</code>
  <review>CRITICAL: Hardcoded password. Use environment variable or secrets manager.</review>
</example>
<example>
  <code>data = json.loads(request.body)</code>
  <review>MEDIUM: Missing try/except for malformed JSON. Add error handling.</review>
</example>
</examples>

<task>Review the following code:</task>
<code>{{user_code}}</code>
```

---

## 4. Security & Reliability

### Few-Shot Injection Risks

| Risk | Example | Mitigation |
|------|---------|-----------|
| Example pollution | User-provided "example" contains instructions | Separate example source from user input |
| Overfitting to examples | Claude copies example too literally | Include diverse examples |
| Negative example confusion | Claude mimics the "bad" example | Clearly label positive vs negative |

### Reliable Few-Shot Patterns

1. **Consistent format** — all examples must follow identical structure
2. **Representative** — examples cover the real distribution of inputs
3. **Clear separation** — examples clearly separated from the actual task
4. **No ambiguity** — each example has ONE correct answer
5. **Up-to-date** — examples reflect current requirements

---

## 5. Exam Scenarios

### Scenario 1
**Q:** Claude is inconsistently formatting JSON responses — sometimes camelCase, sometimes snake_case, sometimes with extra fields. What is the MOST effective prompting fix?

**A:** Add **few-shot examples** showing the exact format expected:
```
Always respond in this exact JSON format:

Example 1: {"user_id": "u123", "status": "active", "created_at": "2024-01-15"}
Example 2: {"user_id": "u456", "status": "suspended", "created_at": "2024-03-20"}

Rules derived from examples:
- snake_case keys
- Only these fields: user_id, status, created_at
- Dates in YYYY-MM-DD format
```

Examples are more powerful than rules alone because Claude pattern-matches the format.

### Scenario 2
**Q:** You need Claude to classify support tickets into 5 categories. With zero-shot it achieves 70% accuracy. How do you improve to 90%+?

**A:** Add few-shot examples covering:
1. **Clear cases** — one obvious example per category
2. **Ambiguous cases** — tickets that could go either way, with the correct answer
3. **Edge cases** — unusual tickets that shouldn't be misclassified

5 clear + 5 ambiguous examples typically reaches 90%+ accuracy. Focus examples on the boundary cases where zero-shot fails.

### Scenario 3
**Q:** In an agentic loop running 30 iterations, few-shot examples consume 3000 tokens per call. How do you reduce this cost?

**A:** 
1. **Prompt caching** — with Anthropic's caching, examples in the system prompt are cached and only billed once at full price, then 90% discount on subsequent calls
2. **Move examples to system prompt** (not per-message) so caching applies
3. **Reduce to minimum effective examples** — test with 2-3 instead of 5+
4. **Dynamic loading** — only include examples relevant to the current sub-task

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| All examples from same category | Claude biases toward that category | Balanced representation |
| Examples that contradict each other | Confusing signal | Consistent logic across examples |
| Too many examples (50+) in every call | Token waste | Minimize, cache, or use many-shot only where needed |
| Examples without edge cases | Fails on ambiguous inputs | Include boundary cases |
| Using only positive examples | Claude doesn't know what to avoid | Add negative examples with corrections |
| User content embedded in examples | Injection risk | Keep examples separate from user input |

---

## 7. Quick Reference / Cheat Sheet

```
FEW-SHOT STRUCTURE
━━━━━━━━━━━━━━━━━━
<examples>
  <example>
    <input>...</input>
    <output>...</output>
  </example>
  (repeat 2-5 times)
</examples>
<task>Now handle: [actual input]</task>

EXAMPLE SELECTION RULES
━━━━━━━━━━━━━━━━━━━━━━━
• One per category (minimum coverage)
• Include boundary/ambiguous cases
• Progress from simple → complex
• Include negative examples (what NOT to do)
• All must use identical format

QUANTITY GUIDELINES
━━━━━━━━━━━━━━━━━━
Format demonstration: 1-2 examples
Classification: 1 per category + ambiguous cases
Complex transformation: 3-5 progressive examples
Nuanced judgment: 5-10+ with edge cases

TOKEN OPTIMIZATION
━━━━━━━━━━━━━━━━━━
• Put examples in system prompt (enables caching)
• Use concise examples (minimum viable length)
• Dynamic selection (only relevant examples)
• Test minimum effective count (often fewer is fine)

KEY PRINCIPLES
━━━━━━━━━━━━━━
• Show, don't tell (examples > rules)
• Consistent format across all examples
• Cover the distribution of real inputs
• Separate examples from user content (security)
• Examples in system prompt → cached in agentic loops
```
