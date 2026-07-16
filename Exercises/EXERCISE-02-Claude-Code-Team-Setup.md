# Exercise 2: Configure Claude Code for Team Development Workflow

**Source:** Claude Certified Architect – Foundations Exam Guide v1.0, Section 8  
**Objective:** Practice configuring CLAUDE.md hierarchies, custom slash commands, path-specific rules, and MCP server integration for a multi-developer project.  
**Domains Reinforced:** D3 (Claude Code Configuration & Workflows), D2 (Tool Design & MCP Integration)  
**Estimated Duration:** 2-3 hours  
**Difficulty:** Medium

---

## Overview

You will configure Claude Code for a team project with multiple coding standards across different areas. This exercise reinforces:

- CLAUDE.md hierarchy (user, project, directory levels)
- Custom slash commands for team workflows
- Path-specific rules with glob patterns
- MCP server configuration (project and personal)
- Plan mode vs direct execution decisions

---

## Part 1: Create Project-Level CLAUDE.md

### Step 1.1: Establish Universal Coding Standards

Create a root-level `CLAUDE.md` file with standards shared across your entire team:

```markdown
# Claude Code Configuration — Team Development Standards

## Universal Standards (Applied Everywhere)

### Language & Tone
- Communicate clearly, avoid jargon
- Explain *why*, not just *what*
- Link to relevant task statements or PRs when referencing decisions

### Code Organization
- One responsibility per function/module
- Functions under 50 lines (except data initialization)
- Classes under 200 lines unless domain-specific (e.g., data containers)
- Avoid clever abstractions; prefer explicit code

### Testing Standards
- All production code has corresponding tests
- Test file naming: `Component.test.tsx`, `utils.test.ts`, etc.
- Tests in same directory as code (co-located)
- Each test file covers: happy path, edge cases, error cases
- No fixtures in main production code; use test factories

### Documentation
- No multi-line comments; one-line comments only
- Comments explain *why*, not *what*
- Public APIs have JSDoc stubs (purpose, parameters, return type)
- Internal functions don't need JSDoc

### Git & PR Practices
- Commit messages: imperative mood ("Add feature" not "Added feature")
- One logical change per commit
- PR descriptions include context (why, not what)
- No force-pushes to main or stable branches

### Error Handling
- Validate at system boundaries (user input, external APIs)
- Trust internal code and framework guarantees
- No defensive error handling for "can't happen" cases
- Errors include context: what failed and why

### Performance & Dependencies
- No premature optimization
- Dependencies reviewed: size, maintenance, alternatives
- Build time target: <10 seconds for incremental builds

---

## API Conventions (for src/api/**/* files — see .claude/rules/)

Handled by path-specific rules. See .claude/rules/api-conventions.md

---

## React Conventions (for src/components/**/* files — see .claude/rules/)

Handled by path-specific rules. See .claude/rules/react-conventions.md

---

## Database Conventions (for src/db/**/* files — see .claude/rules/)

Handled by path-specific rules. See .claude/rules/database-conventions.md

---

## CLI Tool Flags

When running Claude Code with structured output:
- Use `-p` flag in CI/CD pipelines to run non-interactively
- Use `--output-format json` with `--json-schema` for machine-readable results

```

### Step 1.2: Verify Project-Level Application

**Test:** Add the root `CLAUDE.md` to your project. Have a teammate:
1. Clone the repository
2. Open a file for editing
3. Confirm Claude Code loads the project-level standards

**Assertion:** All developers should see the universal standards without setting up personal configurations.

---

## Part 2: Create Path-Specific Rules

### Step 2.1: API Conventions Rule

Create `.claude/rules/api-conventions.md` with YAML frontmatter for path scoping:

```markdown
---
name: API-Conventions
description: Coding standards specific to API handlers and middleware
type: rule
paths:
  - "src/api/**/*"
  - "src/handlers/**/*"
  - "!**/*.test.ts"
---

# API Handler Conventions

## Request Validation
- Validate all user input at the handler boundary
- Use schema validation (Zod, Joi, or equivalent)
- Return 400 Bad Request with specific error messages (not generic)
- Log validation failures with request context (no sensitive data)

## Response Format
All responses follow this structure:
```json
{
  "success": boolean,
  "data": object | array | null,
  "error": { "code": string, "message": string } | null,
  "timestamp": ISO8601 string
}
```

## Error Handling
- 400: Bad request (validation failure)
- 401: Unauthorized (auth required but missing)
- 403: Forbidden (auth valid but insufficient permission)
- 404: Not found (resource doesn't exist)
- 500: Server error (internal failure, log stack trace)
- 503: Service unavailable (external dependency down)

## Async/Await Style
- All handlers must be async
- Use try/catch, not .catch()
- Errors propagate up; middleware handles global catch

## Rate Limiting & Caching
- Document cache headers in response
- Rate limiting: 1000 req/min per IP for public endpoints
- Apply rate limiting in middleware, not individual handlers

## Example Handler
```typescript
async function getUser(req: Request, res: Response) {
  try {
    const { userId } = validateRequest(req, userIdSchema);
    const user = await db.users.findById(userId);
    
    if (!user) {
      return res.status(404).json({
        success: false,
        error: { code: "USER_NOT_FOUND", message: "User does not exist" }
      });
    }
    
    return res.json({
      success: true,
      data: user,
      timestamp: new Date().toISOString()
    });
  } catch (error) {
    logger.error("getUser failed", { userId: req.params.userId, error });
    throw error; // Middleware will catch
  }
}
```
```

### Step 2.2: React Conventions Rule

Create `.claude/rules/react-conventions.md`:

```markdown
---
name: React-Conventions
description: Coding standards for React components
type: rule
paths:
  - "src/components/**/*.tsx"
  - "src/components/**/*.ts"
  - "!**/*.test.tsx"
---

# React Component Conventions

## Functional Components Only
- All components are functional (no class components)
- Hooks preferred over HOCs
- Custom hooks for reusable logic (extract at 2x reuse, not 3x)

## Component Structure
```typescript
// Props interface first
interface Props {
  title: string;
  onClose: () => void;
  items?: Item[];
}

// Component
export function Modal({ title, onClose, items = [] }: Props) {
  const [isOpen, setIsOpen] = useState(false);
  
  // Logic here
  
  return (
    <div>
      {/* JSX here */}
    </div>
  );
}
```

## Props & State Management
- Keep state at the lowest component that needs it
- Lift state only when shared by siblings
- No prop drilling >2 levels; use context if deeper
- Props are immutable; don't modify directly

## Hooks
- Use built-in hooks: useState, useEffect, useContext, useCallback, useMemo
- useEffect cleanup functions required for subscriptions/timers
- Dependencies array required; no empty deps unless intentional (comment why)

## Performance
- Memoize components that receive non-primitive props (React.memo)
- Memoize callbacks passed to children (useCallback)
- Memoize expensive computations (useMemo)

## Testing
- Test user interactions, not implementation details
- Use `screen.getByRole`, not `querySelector`
- Mock external dependencies (API calls, storage)
- Test: happy path, edge cases, error states

## Example Component
```typescript
interface Props {
  onSubmit: (data: FormData) => Promise<void>;
  isLoading: boolean;
}

export function LoginForm({ onSubmit, isLoading }: Props) {
  const [email, setEmail] = useState("");
  const handleSubmit = useCallback(async (e: React.FormEvent) => {
    e.preventDefault();
    await onSubmit({ email });
  }, [email, onSubmit]);
  
  return (
    <form onSubmit={handleSubmit}>
      <input value={email} onChange={(e) => setEmail(e.target.value)} />
      <button disabled={isLoading}>Sign In</button>
    </form>
  );
}
```
```

### Step 2.3: Database Conventions Rule

Create `.claude/rules/database-conventions.md`:

```markdown
---
name: Database-Conventions
description: Coding standards for database models and queries
type: rule
paths:
  - "src/db/**/*"
  - "!**/*.test.ts"
---

# Database Conventions

## Repository Pattern
- One repository per entity (UserRepository, OrderRepository)
- Repository exposes: find, findById, create, update, delete, query
- Business logic stays in service layer (not repository)

## Query Building
- Use query builder or ORM (avoid raw SQL except for complex queries)
- Parameterized queries always (prevent SQL injection)
- Include database indexes on frequently queried fields

## Migrations
- One migration per logical change (not per table)
- Reversible migrations with both up/down
- Test migrations on test data first

## Connection Pooling
- Enable connection pooling in all environments
- Pool size: 5-10 connections
- Idle timeout: 30 seconds

## Example Repository
```typescript
export class UserRepository {
  async findById(id: string): Promise<User | null> {
    return db.query("SELECT * FROM users WHERE id = ?", [id]);
  }
  
  async create(user: Omit<User, "id">): Promise<User> {
    const result = await db.query(
      "INSERT INTO users (name, email) VALUES (?, ?)",
      [user.name, user.email]
    );
    return result;
  }
}
```
```

### Step 2.4: Test Conventions Rule

Create `.claude/rules/test-conventions.md`:

```markdown
---
name: Test-Conventions
description: Testing standards applied to all **/*.test.ts files
type: rule
paths:
  - "**/*.test.ts"
  - "**/*.test.tsx"
---

# Testing Conventions

## Test Structure
Each test file covers: happy path, edge cases, error cases
```
✅ happy path (normal flow)
✅ edge case 1 (boundary)
✅ edge case 2 (boundary)
✅ error case (what happens when things fail)
```

## Test File Naming
- `Component.test.tsx` (for `Component.tsx`)
- `utils.test.ts` (for `utils.ts`)
- Co-located with code (same directory)

## Mocking & Fixtures
- Mock external dependencies (API calls, database, storage)
- Use test factories for complex fixtures, not shared setup
- Mock at the boundary (network requests, not internal functions)

## Assertion Style
```typescript
// Good: Clear, specific
expect(result).toBe(42);
expect(array).toHaveLength(3);
expect(element).toHaveTextContent("Hello");

// Avoid: Vague
expect(result).toBeTruthy();
expect(array.length > 0).toBe(true);
```

## Performance
- Tests should run in <100ms each
- Avoid sleep() calls; use fake timers (jest.useFakeTimers())
- Parallel execution encouraged (no shared state)

## Example Test
```typescript
describe("UserService", () => {
  it("returns user by ID", async () => {
    const user = await service.findById("123");
    expect(user.id).toBe("123");
  });
  
  it("throws when user not found", async () => {
    await expect(service.findById("nonexistent")).rejects.toThrow();
  });
});
```
```

### Step 2.5: Verify Path-Specific Rules Load

**Test:**
1. Open a file matching `src/api/**/*.ts` → API conventions should appear
2. Open a file matching `src/components/**/*.tsx` → React conventions should appear
3. Open a file matching `**/*.test.tsx` → Test conventions should appear
4. Open a file NOT matching any pattern → Only root CLAUDE.md standards apply

**Assertion:** Each developer sees context-appropriate standards without manual selection.

---

## Part 3: Create Custom Slash Commands & Skills

### Step 3.1: Project-Scoped Slash Command

Create `.claude/commands/review`:

```markdown
---
name: review
description: Run team code review checklist
usage: /review
---

You are performing a code review according to team standards.

Review the changed code against these criteria:

**Correctness:**
- ✅ Does the code do what it claims to do?
- ✅ Are there off-by-one errors, null checks, edge cases?
- ✅ Does it handle errors appropriately?

**Architecture:**
- ✅ Follows conventions (testing, error handling, organization)
- ✅ Functions under 50 lines; classes under 200 lines
- ✅ One responsibility per module
- ✅ No unnecessary complexity or abstractions

**Testing:**
- ✅ Are there tests? (Required for all production code)
- ✅ Do tests cover happy path, edge cases, errors?
- ✅ Are tests testing behavior, not implementation?

**Performance:**
- ✅ No premature optimization
- ✅ No obvious performance cliffs
- ✅ No unnecessary loops or duplicate queries

**Dependencies:**
- ✅ Any new dependencies? Are they reviewed and approved?

Report findings with:
- **Issue**: Specific code location and problem
- **Severity**: Bug (breaks things) / Convention (team standard) / Enhancement (nice to have)
- **Suggestion**: What to change and why
```

### Step 3.2: Project-Scoped Skill with context: fork

Create `.claude/skills/analyze-codebase/SKILL.md`:

```markdown
---
name: analyze-codebase
description: Analyze codebase structure, dependencies, and patterns
context: fork
allowed-tools:
  - Glob
  - Grep
  - Read
argument-hint: "What aspect would you like analyzed? (structure, dependencies, patterns)"
---

You are a codebase analyzer. Your job is to explore and understand the structure.

Instructions:
1. Start with file count and directory structure
2. Identify major modules and responsibilities
3. Find key entry points
4. Map dependencies between major modules
5. Identify common patterns
6. Report findings clearly

Use Glob and Grep to explore efficiently:
- Start with glob patterns to find files
- Use grep to search for specific patterns, imports, or exports
- Build understanding incrementally

Report structure, not prose. Use markdown for clarity.
```

### Step 3.3: Personal MCP Server (User-Level)

Create `~/.claude/commands/personal-quick-fix`:

```markdown
---
name: personal-quick-fix
description: Personal quick fixes (don't use in team PRs)
usage: /personal-quick-fix
---

Quick one-off fixes for personal exploration. Not for team PRs.
```

### Step 3.4: Verify Skills Load Correctly

**Test:**
1. Type `/review` in a code file → Should run team code review
2. Type `/analyze-codebase` → Should spawn isolated subagent
3. Verify that `/analyze-codebase` output doesn't pollute main conversation

**Assertion:** Commands are available, skills run in fork mode (isolated), no context pollution.

---

## Part 4: Configure MCP Servers

### Step 4.1: Project-Scoped MCP Configuration

Create `.mcp.json` in project root:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["mcp-github", "--token", "${GITHUB_TOKEN}"],
      "description": "GitHub API for issues, PRs, repos"
    },
    "notion": {
      "command": "npx",
      "args": ["mcp-notion", "--token", "${NOTION_API_KEY}"],
      "description": "Notion API for documentation"
    }
  }
}
```

**Environment Variables:**
- Create `.env.local` (NOT committed):
  ```
  GITHUB_TOKEN=ghp_xxx...
  NOTION_API_KEY=secret_xxx...
  ```
- Update `.gitignore` to exclude `.env.local`

**Verification:**
```bash
# Test that MCP servers load
claude --version  # Should list configured servers
```

### Step 4.2: User-Level MCP Configuration

Create `~/.claude.json` (in home directory) for personal/experimental servers:

```json
{
  "mcpServers": {
    "my-custom-tool": {
      "command": "node",
      "args": ["/path/to/my-custom-mcp-server.js"],
      "description": "Personal custom tool (not shared with team)"
    }
  }
}
```

### Step 4.3: Verify Both Load Simultaneously

**Test:**
1. Run `claude` and check available tools
2. Should include both project servers (GitHub, Notion) and personal servers
3. Both should be available to agents in the same session

**Assertion:** Project and user MCP servers are loaded together without conflict.

---

## Part 5: Plan Mode vs Direct Execution Decisions

### Step 5.1: Assess Task Complexity

For each scenario, decide: **Plan Mode** or **Direct Execution**?

**Scenario A: Add single field validation**
- Complexity: Low (single function, <20 lines)
- Changes: One file
- Rework risk: None
- **Decision:** ✅ Direct Execution
- Rationale: Scope is clear, low rework risk, changes isolated

**Scenario B: Library migration (affects 45+ files)**
- Complexity: High (large codebase)
- Changes: Many files, dependency order matters
- Rework risk: High (wrong order breaks build)
- Valid approaches: v1 → v2 gradual, or atomic cutover
- **Decision:** ✅ Plan Mode
- Rationale: Architectural implications, multiple valid approaches, high rework cost if wrong

**Scenario C: New feature with multiple valid implementations**
- Complexity: Medium-High (unclear best approach)
- Changes: New module + integration points
- Rework risk: Medium (wrong architecture requires refactoring)
- Tradeoffs: Monolithic vs distributed, cache strategy, API design
- **Decision:** ✅ Plan Mode
- Rationale: Architectural tradeoffs need exploration before commitment

**Scenario D: Bug fix with clear stack trace**
- Complexity: Low (problem is identified)
- Changes: Single function, root cause known
- Rework risk: None (fix is narrow)
- **Decision:** ✅ Direct Execution
- Rationale: Root cause known, fix scope is narrow

### Step 5.2: Create Decision Tree

Document in project wiki or CLAUDE.md:

```
Use PLAN MODE when:
❓ Large-scale changes (50+ files OR multiple modules)
❓ Multiple valid architectural approaches exist
❓ Tradeoffs between approaches need evaluation
❓ High rework cost if wrong (e.g., database migrations)

Use DIRECT EXECUTION when:
✅ Scope is clear and narrow (<20 lines OR single file)
✅ Root cause is identified (bug fix, single feature)
✅ No architectural decisions needed
✅ Change is independent of other work

Examples:
PLAN MODE:
- Monolith → microservices restructuring
- Library migrations affecting many modules
- API design refactors with multiple approaches

DIRECT EXECUTION:
- Single-file bug fix
- Add validation to one function
- Fix formatting in one file
- Add new utility function in isolated module
```

---

## Part 6: Test Multi-Developer Workflow

### Step 6.1: Simulate Team Member Onboarding

1. **Developer A:** Pushes initial setup (root CLAUDE.md + .claude/rules/ + .mcp.json)
2. **Developer B:** Clones repository
3. **Developer B:** Opens a file → Sees project standards (not personal setup required)
4. **Developer B:** Runs `/review` → Gets team code review checklist
5. **Developer B:** Uses GitHub MCP tool → Works without configuration

**Assertion:** New team member gets full configured experience from one clone, no manual setup.

### Step 6.2: Test Consistency Across Team

- Developer A edits React component → Sees React conventions
- Developer B edits API handler → Sees API conventions
- Both see same standards without coordination

**Assertion:** Standards are applied consistently regardless of who's using Claude Code.

---

## Validation Checklist

### CLAUDE.md Hierarchy ✅
- [ ] Root `CLAUDE.md` has universal standards
- [ ] All developers see root standards without setup
- [ ] Standards are practical and non-arbitrary

### Path-Specific Rules ✅
- [ ] `.claude/rules/` files have YAML frontmatter with glob patterns
- [ ] API rule loads for `src/api/**/*` only
- [ ] React rule loads for `src/components/**/*.tsx` only
- [ ] Test rule loads for `**/*.test.ts` only
- [ ] Non-matching files see only root standards

### Custom Commands & Skills ✅
- [ ] `.claude/commands/review` is available to all developers
- [ ] Skills use `context: fork` to isolate verbose output
- [ ] Skills have `allowed-tools` restricted appropriately
- [ ] Personal commands in `~/.claude/commands/` don't affect team

### MCP Configuration ✅
- [ ] Project `.mcp.json` configures shared team tools (GitHub, Notion)
- [ ] User `~/.claude.json` configures personal experimental tools
- [ ] Both load simultaneously in same session
- [ ] Tools available to agents via MCP

### Plan Mode Decisions ✅
- [ ] Large-scale changes trigger plan mode
- [ ] Single-file changes trigger direct execution
- [ ] Team understands when to use which mode
- [ ] Decision tree documented for reference

---

## 🎯 Success Criteria

Your configuration is complete when:

1. **New team member** clones repo and gets full configured experience without setup
2. **Path-specific rules** load correctly (API, React, Test conventions appear in context)
3. **Team commands** (`/review`, `/analyze-codebase`) work consistently
4. **MCP servers** from both `.mcp.json` and `~/.claude.json` load together
5. **Plan mode decisions** are documented and understood by team
6. **All developers** see consistent standards without coordination

---

## 📚 Related Study Materials

- D3-01: CLAUDE.md Hierarchy & Management
- D3-02: Custom Slash Commands & Skills
- D3-03: Path-Specific Rule Implementation
- D3-04: Plan Mode vs Direct Execution
- D2-04: MCP Server Configuration & Protocol

---

**Prepared:** July 15, 2026  
**Difficulty:** Medium  
**Estimated Time:** 2-3 hours  
**Status:** Ready for implementation
