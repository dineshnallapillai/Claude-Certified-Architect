# Domain 3 | Topic 3.4: Plan Mode vs Direct Execution

## 1. Core Concepts

### What is Plan Mode?

**Plan Mode** is a Claude Code operational mode where Claude creates a structured plan before executing any changes. It separates the "thinking" phase from the "doing" phase.

```
PLAN MODE                          DIRECT EXECUTION
━━━━━━━━━━━━━━━━                  ━━━━━━━━━━━━━━━━━━
1. Analyze request                 1. Analyze request
2. Create structured plan          2. Start executing immediately
3. Present plan to user            3. Mix thinking and doing
4. Get approval                    4. Report results
5. Execute plan
6. Report results
```

### When to Use Plan Mode

| Use Plan Mode When... | Use Direct Execution When... |
|----------------------|------------------------------|
| Large, multi-file changes | Simple, single-file edits |
| Architectural decisions needed | Well-understood task |
| Multiple valid approaches | Only one obvious approach |
| High-risk changes | Low-risk, reversible changes |
| Need user buy-in first | User trusts Claude to proceed |
| Complex refactoring | Bug fix with clear cause |
| New feature design | Adding a test case |

### Plan Mode Mechanics

```
┌──────────────────────────────────────────────────────┐
│  PLAN MODE WORKFLOW                                  │
│                                                      │
│  1. User enters plan mode (Shift+Tab to toggle)      │
│  2. Claude reads code, analyzes task                 │
│  3. Claude produces plan (no tool execution)         │
│  4. Plan presented:                                  │
│     - What files to change                           │
│     - What changes in each file                      │
│     - Order of operations                            │
│     - Risks and considerations                       │
│  5. User reviews:                                    │
│     - Approve → execute                              │
│     - Modify → revise plan                           │
│     - Reject → abandon                               │
│  6. Execution with progress tracking                 │
└──────────────────────────────────────────────────────┘

Entering Plan Mode:
- Interactive (TUI): Press Shift+Tab to toggle between Plan and Act modes
- Headless/CI: Use the --plan CLI flag (e.g., `claude --plan "refactor auth"`)
```

### Plan Structure

A good plan includes:
- **Goal:** What we're trying to achieve
- **Approach:** Strategy chosen and why
- **Steps:** Ordered list of concrete actions
- **Files affected:** Which files change and how
- **Risks:** What could go wrong
- **Verification:** How to confirm success

---

## 2. Design Patterns

### Pattern 1: Feature Implementation Plan

```markdown
## Plan: Add User Profile API Endpoint

### Goal
Add a GET /api/v1/users/:id/profile endpoint returning user profile data.

### Approach
Follow existing endpoint patterns in /src/api/routes. Use the UserService 
for data access, add validation middleware.

### Steps
1. Create route handler in `src/api/routes/userProfile.ts`
2. Add `getProfile()` method to `src/services/UserService.ts`
3. Create response DTO in `src/api/dto/UserProfileResponse.ts`
4. Add route to Express router in `src/api/index.ts`
5. Add request validation schema
6. Write unit tests for service method
7. Write integration test for endpoint

### Files Affected
- `src/api/routes/userProfile.ts` (NEW)
- `src/services/UserService.ts` (MODIFY - add method)
- `src/api/dto/UserProfileResponse.ts` (NEW)
- `src/api/index.ts` (MODIFY - add route)
- `tests/unit/UserService.test.ts` (MODIFY - add tests)
- `tests/integration/userProfile.test.ts` (NEW)

### Risks
- Need to check if user privacy settings affect profile visibility
- Rate limiting may need adjustment for this endpoint

### Verification
- Unit tests pass
- Integration test confirms 200 response with expected shape
- 404 for non-existent user
- 403 for private profiles
```

### Pattern 2: Refactoring Plan

```markdown
## Plan: Extract Authentication Middleware

### Goal
Move inline auth logic from routes into reusable middleware.

### Approach
Extract-then-replace: create middleware first, verify it works, 
then update routes one by one.

### Steps
1. [READ] Analyze all current auth patterns across routes
2. [CREATE] `src/middleware/auth.ts` with extracted logic
3. [TEST] Write tests for new middleware in isolation
4. [MODIFY] Update route A to use middleware
5. [VERIFY] Run route A tests
6. [MODIFY] Update route B to use middleware
7. [VERIFY] Run route B tests
8. [REPEAT] Continue for remaining routes
9. [CLEANUP] Remove old inline auth code

### Rollback Plan
Each route update is independent — if one fails, 
revert just that route's change.
```

### Pattern 3: Risk-Based Plan Ordering

```markdown
## Execution Order (by risk)

### Phase 1: Safe Changes (no-risk)
- Add new files (no existing code affected)
- Add new tests (existing tests unaffected)

### Phase 2: Low-Risk Changes
- Modify test files
- Update documentation

### Phase 3: Medium-Risk Changes
- Modify implementation files
- Update configurations

### Phase 4: High-Risk Changes (do last)
- Database migrations
- API contract changes
- Infrastructure modifications

Each phase verified before moving to next.
```

---

## 3. Implementation Details

### Entering/Exiting Plan Mode

```
User: Shift+Tab                → Toggles plan mode on/off in the TUI
User: "implement feature X"   → Claude creates plan (no execution)
User: "looks good, proceed"   → Claude exits plan mode, executes
User: "change step 3"         → Claude revises plan
User: "cancel"                → Abandon plan

Headless/CLI usage:
$ claude --plan "implement feature X"   → Runs in plan mode non-interactively
```

### Plan Mode Constraints

In plan mode, Claude:
- **CAN:** Read files, search code, analyze dependencies
- **CANNOT:** Write files, edit files, run commands that modify state
- **PRODUCES:** A structured plan document
- **AWAITS:** User approval before any modifications

### Direct Execution Default Behavior

Without plan mode, Claude:
- Reads requirements
- Immediately starts executing (read → edit → test)
- Reports progress as it goes
- May ask for clarification mid-execution if needed

### Hybrid Approach: Plan for Complex, Execute for Simple

```python
# Mental model for when Claude should self-plan:
if task.files_affected > 3 or task.risk == "high":
    enter_plan_mode()
elif task.is_ambiguous:
    ask_clarification()
else:
    execute_directly()
```

---

## 4. Security & Reliability

### Why Plan Mode Improves Safety

| Benefit | Explanation |
|---------|-------------|
| **Human review** | User sees what will change before it happens |
| **Reduced surprises** | No unexpected file modifications |
| **Scope containment** | Plan defines boundaries, prevents scope creep |
| **Reversibility awareness** | Risks are identified upfront |
| **Alignment check** | Catch misunderstandings before code is written |

### Plan Mode for High-Stakes Operations

```
ALWAYS plan for:
- Database migrations
- API breaking changes
- Security-related modifications
- Multi-service changes
- Production configuration updates
- Large refactoring (10+ files)

OKAY to execute directly for:
- Bug fixes with clear cause
- Adding test cases
- Updating comments/docs
- Simple style changes
- Single-file modifications
```

### Risks of Skipping Plan Mode

| Risk | Scenario |
|------|----------|
| Wrong approach chosen | Claude picks a suboptimal architecture |
| Scope explosion | Simple request → 20 files modified |
| Misunderstood requirements | Claude interprets differently than intended |
| Irreversible changes | Database column dropped before plan review |
| Wasted work | Claude writes 500 lines, user wanted different approach |

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A user asks Claude to "refactor the payment processing module." The module spans 15 files and handles real financial transactions. Should Claude use plan mode or direct execution?

**A:** **Plan mode is essential** here because:
1. 15 files = high complexity, many potential interactions
2. Financial transactions = high-risk domain (bugs = money loss)
3. Refactoring has multiple valid approaches (user should choose)
4. Need to identify which changes can be made safely vs. need extra caution
5. A plan lets the user verify the scope won't accidentally break payment flows

### Scenario 2
**Q:** A user says "fix the typo in the README." Should Claude use plan mode?

**A:** **No — direct execution** is appropriate because:
- Single file change
- Zero risk (typo fix can't break anything)
- Only one obvious approach
- Trivially reversible
- Plan mode adds unnecessary friction

### Scenario 3
**Q:** Claude is in plan mode and discovers mid-analysis that the task is simpler than expected (just a one-line config change). What should it do?

**A:** Claude should:
1. Present a minimal plan: "This only requires changing one line in config.yaml"
2. Explain why it's simpler than expected
3. Ask user if they want to proceed directly given the simplicity
4. Exit plan mode and execute if user agrees

The key insight: plan mode is a tool for managing complexity. When complexity is low, acknowledge it and offer to proceed directly.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Plan mode for trivial tasks | Wasted time and tokens | Direct execution for simple changes |
| Direct execution for complex refactors | Risky, may go wrong | Plan first, execute after approval |
| Plans without verification steps | Can't tell if execution succeeded | Include "how to verify" in every plan |
| Overly detailed plans | Analysis paralysis | Right level of detail for the task |
| Plans that don't mention risks | User can't make informed decisions | Always include risk assessment |
| Executing without completing the plan | Partial implementation | Finish plan, get approval, then execute ALL |

---

## 7. Quick Reference / Cheat Sheet

```
PLAN MODE
━━━━━━━━━
• Separates thinking from doing
• Enter: Shift+Tab (toggles Plan/Act in TUI) or --plan flag (CLI/headless)
• Claude CAN: read, search, analyze
• Claude CANNOT: write, edit, execute
• Produces: structured plan document
• Exit: user approves → execution begins (Shift+Tab back to Act mode)

DIRECT EXECUTION
━━━━━━━━━━━━━━━━━
• Think and do simultaneously
• Default mode for Claude Code
• Reads, edits, tests in sequence
• Good for simple, well-understood tasks
• Reports progress as it goes

DECISION MATRIX
━━━━━━━━━━━━━━━
Plan Mode if:
✓ Multi-file changes (>3 files)
✓ Multiple valid approaches
✓ High-risk domain (security, finance, data)
✓ Architectural decisions needed
✓ Large refactoring
✓ User wants alignment first

Direct Execution if:
✓ Single file, obvious fix
✓ Low risk, easily reversible
✓ Well-understood task
✓ User said "just do it"
✓ Bug with clear root cause

GOOD PLAN STRUCTURE
━━━━━━━━━━━━━━━━━━━
1. Goal (what)
2. Approach (how + why this approach)
3. Steps (ordered, concrete)
4. Files affected (with NEW/MODIFY/DELETE)
5. Risks (what could go wrong)
6. Verification (how to confirm success)
```
