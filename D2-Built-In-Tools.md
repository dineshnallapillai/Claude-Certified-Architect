# Domain 2 | Topic 2.5: Built-In Tools (Read, Write, Edit, Bash, Grep, Glob)

## 1. Core Concepts

### What are Built-In Tools?

Claude Code provides a set of **built-in tools** for common development tasks. These are distinct from MCP tools—they come with Claude Code and are always available.

**Six built-in tools:**
- **Read** — Load file contents
- **Write** — Create new files or overwrite existing files
- **Edit** — Modify specific sections of existing files
- **Bash** — Execute shell commands
- **Grep** — Search file contents for patterns
- **Glob** — Find files matching name patterns

### Key Principle: Right Tool for the Right Task

Each tool is optimized for specific tasks. Mismatching task to tool leads to inefficiency or errors:

| Goal | Right Tool | Why Not Others |
|------|-----------|-----------------|
| Load a file | Read | Write overwrites; Edit requires unique text |
| Create a file | Write | Read is for loading; Edit fails if file absent |
| Change part of file | Edit | Requires unique matching text; Write overwrites whole file |
| Find files by name | Glob | Grep searches content, not paths |
| Search code for "function foo" | Grep | Read requires knowing file paths first |
| Run npm test | Bash | Other tools don't execute commands |

---

## 2. Design Patterns

### Read: Load Complete File Contents

**Purpose:** Load entire file into memory for analysis

**Schema:**
```json
{
  "name": "read",
  "description": "Read a file and return its contents",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": {"type": "string", "description": "Absolute path to file"}
    },
    "required": ["path"]
  }
}
```

**When to use Read:**
- Load full file for analysis (parsing, understanding structure)
- Get file contents before making edits
- Read configuration files (package.json, tsconfig.json)
- Load test files to understand existing patterns

**Example:**
```python
# Good: Read to understand before editing
response = client.messages.create(
    messages=[
        {"role": "user", "content": "Fix the bug in /src/auth.ts"}
    ],
    tools=[read_tool]
)

# Claude reads /src/auth.ts, analyzes, then uses Edit to fix specific section
```

**Constraints:**
- Binary files: Can read but may return garbage (use Bash to convert)
- Large files (>100MB): Read succeeds but may exceed context limits
- Permissions: Can only read files the user can access

**Avoid:**
```python
# WRONG: Reading just to write
with open("file.txt") as f:
    content = f.read()
write(path, content.replace("old", "new"))  # Use Edit instead!

# WRONG: Reading config file you should hard-code
read("/etc/passwd")  # Security issue; should be in CLAUDE.md
```

---

### Write: Create or Overwrite Entire Files

**Purpose:** Create new files or completely replace existing files

**Schema:**
```json
{
  "name": "write",
  "description": "Write content to a file (creates or overwrites)",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": {"type": "string", "description": "Absolute path"},
      "content": {"type": "string", "description": "File contents"}
    },
    "required": ["path", "content"]
  }
}
```

**When to use Write:**
- Create new files (scaffolding, boilerplate)
- Complete file rewrites (refactoring)
- Generate code from scratch
- Replace entire file content

**Example:**
```python
# Good: Create new test file
write(
    path="/tests/auth.test.ts",
    content="""
import { login } from '../src/auth';

describe('login', () => {
  test('returns token on valid credentials', () => {
    // ...
  });
});
"""
)

# Good: Complete rewrite of configuration
write(
    path="/tsconfig.json",
    content=json.dumps(new_config, indent=2)
)
```

**Constraints:**
- **Overwrites:** Write replaces entire file (cannot preserve parts)
- **Permissions:** Can only write to paths user has write access to
- **Backup:** No automatic backup (use git for recovery)

**Avoid:**
```python
# WRONG: Using Write to modify one line
content = read("/src/main.ts")
new_content = content.replace("oldLine", "newLine")
write("/src/main.ts", new_content)
# Use Edit instead for targeted changes!

# WRONG: Using Write for tiny changes across many files
# This is inefficient; use Bash script or Edit loops
```

---

### Edit: Modify Specific Sections of Files

**Purpose:** Replace specific text sections within existing files

**Schema:**
```json
{
  "name": "edit",
  "description": "Replace text in a file (preserves rest of file)",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": {"type": "string", "description": "Absolute path"},
      "old_string": {"type": "string", "description": "Exact text to replace"},
      "new_string": {"type": "string", "description": "Replacement text"}
    },
    "required": ["path", "old_string", "new_string"]
  }
}
```

**When to use Edit:**
- Targeted modifications (add validation, fix bug)
- Preserve file structure except changed sections
- Multiple independent changes (call Edit multiple times)
- Ensure precise replacements using unique text anchors

**Example:**
```python
# Good: Fix a specific function
edit(
    path="/src/auth.ts",
    old_string="function validate(token) {\n  return token.length > 0;\n}",
    new_string="function validate(token) {\n  if (!token) return false;\n  return token.startsWith('sk_');\n}"
)

# Good: Add import statement
edit(
    path="/src/main.ts",
    old_string="import { Router } from 'express';\nimport { logger } from './logger';",
    new_string="import { Router } from 'express';\nimport { logger } from './logger';\nimport { auth } from './middleware';"
)
```

**When Edit Fails:**
- **Non-unique text:** old_string matches multiple places → Use Read + Write as fallback
- **File not found:** Edit requires existing file → Use Write to create
- **Whitespace mismatch:** Indentation doesn't match exactly → Use Read first to check exact spacing

**Fallback pattern:**
```python
def edit_with_fallback(path, old, new):
    try:
        edit(path, old, new)
    except ValueError as e:
        if "not unique" in str(e):
            # Fallback to Read + Write
            content = read(path)
            new_content = content.replace(old, new)
            write(path, new_content)
```

**Avoid:**
```python
# WRONG: Edit with whitespace mismatch
edit(
    path="/src/main.ts",
    old_string="function test() {",  # Missing indentation
    new_string="function test() { // Fixed"
)

# WRONG: Edit with overly generic text
edit(
    path="/src/main.ts",
    old_string="return;",  # Matches 50+ places
    new_string="return null;"
)
# Provide more context to make unique!
```

---

### Bash: Execute Shell Commands

**Purpose:** Run shell commands for testing, building, deployment

**Schema:**
```json
{
  "name": "bash",
  "description": "Execute a shell command",
  "input_schema": {
    "type": "object",
    "properties": {
      "command": {"type": "string", "description": "Shell command to execute"}
    },
    "required": ["command"]
  }
}
```

**When to use Bash:**
- Run tests (npm test, pytest)
- Build/compile code (npm run build)
- Install dependencies (npm install)
- Execute scripts
- System operations (ls, grep, find)

**Example:**
```python
# Good: Run tests
bash(command="npm test")

# Good: Install dependencies
bash(command="npm install express zod")

# Good: Build TypeScript
bash(command="npx tsc")

# Good: Check git status
bash(command="git status")
```

**Constraints:**
- **Permissions:** Some commands (rm -rf, systemctl) require explicit approval
- **Working directory:** Runs in current project directory
- **Timeouts:** Long-running commands may timeout
- **Interactive:** Cannot respond to prompts (use -y/--yes flags)

**Avoid:**
```python
# WRONG: Risky commands without warnings
bash(command="rm -rf /tmp/build")  # Dangerous!

# WRONG: Interactive commands without flags
bash(command="apt-get install npm")  # Will hang waiting for [Y/n]
# Use: bash(command="apt-get install -y npm")

# WRONG: Assuming specific tools available
bash(command="cargo build")  # May not be installed

# WRONG: Long-running without timeout
bash(command="find / -type f")  # Will take forever
```

---

### Grep: Search File Contents for Patterns

**Purpose:** Find patterns in file contents (not file paths)

**Schema:**
```json
{
  "name": "grep",
  "description": "Search file contents for regex patterns",
  "input_schema": {
    "type": "object",
    "properties": {
      "pattern": {"type": "string", "description": "Regex pattern to find"},
      "path": {"type": "string", "description": "Directory or file path"},
      "type": {"type": "string", "description": "File type filter (e.g., 'js', 'ts', 'py')"},
      "output_mode": {"type": "string", "enum": ["content", "files_with_matches", "count"]}
    },
    "required": ["pattern"]
  }
}
```

**When to use Grep:**
- Find all calls to a function: `grep(pattern="import.*auth")`
- Locate error messages: `grep(pattern="ERROR.*timeout")`
- Find TODOs/FIXMEs: `grep(pattern="TODO|FIXME")`
- Search for specific strings across codebase
- Count matches: `grep(pattern="async", output_mode="count")`

**Example:**
```python
# Good: Find all imports of a module
grep(pattern="from.*auth.*import", path="/src", type="ts")

# Good: Find all error handling
grep(pattern="catch|throw", path="/src", output_mode="files_with_matches")

# Good: Find deprecated function calls
grep(pattern="deprecated_function\\(", path="/src", type="js")

# Good: Count async/await usage
grep(pattern="async|await", path="/src", output_mode="count")
```

**Regex Basics:**
- `.` = any character
- `*` = 0+ of previous
- `+` = 1+ of previous
- `|` = OR
- `^` = start of line
- `$` = end of line
- `\w` = word character
- `\s` = whitespace

**Avoid:**
```python
# WRONG: Using Grep to find files (use Glob instead)
grep(pattern="test.ts")  # Finds content matching "test.ts"
# Use: glob(pattern="**/*.test.ts")

# WRONG: Complex logic in regex
grep(pattern="function.*\\{.*return.*\\}")  # Too complex
# Break into multiple searches or use Code reading

# WRONG: Searching for regex metacharacters without escaping
grep(pattern="cost = $5.00")  # $ is regex anchor!
# Use: grep(pattern="cost = \\$5\\.00")
```

---

### Glob: Find Files Matching Name Patterns

**Purpose:** Find file paths matching glob patterns (not file contents)

**Schema:**
```json
{
  "name": "glob",
  "description": "Find files matching glob patterns",
  "input_schema": {
    "type": "object",
    "properties": {
      "pattern": {"type": "string", "description": "Glob pattern to match"},
      "path": {"type": "string", "description": "Base directory (optional)"}
    },
    "required": ["pattern"]
  }
}
```

**When to use Glob:**
- Find all test files: `glob(pattern="**/*.test.ts")`
- Find all Python files: `glob(pattern="**/*.py")`
- Find files in specific directory: `glob(pattern="src/**/*.tsx")`
- Find configuration files: `glob(pattern="**/tsconfig.json")`

**Example:**
```python
# Good: Find all test files
glob(pattern="**/*.test.ts")
# Returns: ["src/auth.test.ts", "src/utils.test.ts", ...]

# Good: Find all Python files in src
glob(pattern="src/**/*.py")

# Good: Find all package.json files
glob(pattern="**/package.json")

# Good: Find CSS files
glob(pattern="**/*.{css,scss,less}")
```

**Glob Patterns:**
- `*` = matches anything in one level
- `**` = matches anything across multiple levels
- `*.ts` = all TypeScript files
- `src/**/*.ts` = all TypeScript files in src (recursively)
- `**/*.test.ts` = all test files anywhere
- `{css,scss}` = OR (CSS or SCSS)

**Avoid:**
```python
# WRONG: Using Glob to search content
glob(pattern="TODO")  # Matches filenames, not content
# Use: grep(pattern="TODO")

# WRONG: Forgetting ** for recursion
glob(pattern="*.test.ts")  # Only finds tests in root
# Use: glob(pattern="**/*.test.ts")

# WRONG: Overly complex patterns
glob(pattern="src/([a-z]+)/tests/*.test.ts")  # Too specific
# Use: glob(pattern="src/**/tests/*.test.ts")
```

---

## 3. Implementation Details

### Tool Selection Decision Tree

```
Task: "Find where the bug is"
  ├─ Know the file? 
  │   ├─ YES → Read(file) to load full context
  │   └─ NO → Grep(pattern) to search, then Read(file)
  │
  └─ Task: "Fix the bug in that file"
      ├─ Small change (1-2 lines)?
      │   └─ YES → Edit(old, new) with unique text
      │   
      ├─ Multiple changes?
      │   └─ YES → Multiple Edit() calls
      │   
      ├─ Can't find unique text for Edit?
      │   └─ YES → Read(file) + Write(file) fallback
      │   
      └─ Complete rewrite?
          └─ YES → Write(file) to replace entire
```

### Workflow: Incremental Code Search and Fix

```python
# Step 1: Find all files with pattern
test_files = glob(pattern="**/*.test.ts")  # Find tests

# Step 2: Search for specific issue in those files
grep(pattern="console.log", path="src", type="ts")  # Find debug logs

# Step 3: Read the file with the issue
content = read(path="/src/main.ts")

# Step 4: Edit specific section
edit(
    path="/src/main.ts",
    old_string="console.log(result);  // TODO: remove",
    new_string="// Debug logging removed"
)
```

### Built-In Tools vs MCP Tools

| Aspect | Built-In | MCP |
|--------|----------|-----|
| **Purpose** | General development tasks | Backend system integration |
| **Availability** | Always present | Configured per project |
| **Examples** | Read, Write, Grep | Database query, API call, GitHub API |
| **Configuration** | None | .mcp.json or settings.json |
| **Security** | Subject to hooks and permissions | Defined by MCP server |

**When to combine:**
```python
# Good: Use built-in for code, MCP for external systems
glob(pattern="**/*.ts")  # Built-in: find TypeScript files
client.query(query="SELECT * FROM users")  # MCP: database query
```

---

## 4. Security & Reliability

### Tool Safety Considerations

| Tool | Risk | Mitigation |
|------|------|-----------|
| **Bash** | Command injection, rm -rf | Hooks validation, explicit flags |
| **Write** | Overwrite important files | Git backup, careful path selection |
| **Edit** | Wrong line replaced (non-unique) | Use unique anchor text, preview |
| **Read** | Information disclosure | Check for secrets in output |
| **Grep** | Information leakage | Sensitive patterns may be visible |
| **Glob** | Finding hidden files | Explicit patterns, avoid `**` if not needed |

### Edit Safety Pattern

Always use unique, specific text to avoid accidental replacements:

```python
# WRONG: Generic text, matches multiple places
edit(
    path="/src/main.ts",
    old_string="return;",  # Could match 50+ locations
    new_string="return null;"
)

# RIGHT: Include context to make unique
edit(
    path="/src/main.ts",
    old_string="export function getUserById(id) {\n  if (!id) return;\n  return db.get(id);\n}",
    new_string="export function getUserById(id) {\n  if (!id) return null;\n  return db.get(id);\n}"
)
```

### Bash Safety Pattern

Validate commands before execution and use safety flags:

```python
# WRONG: Dangerous without confirmation
bash(command="rm -rf /src/*")

# RIGHT: Use safe flags
bash(command="rm -rf dist/")  # OK with explicit safe path
bash(command="git add --all && git commit -m 'changes'")  # Safe git ops

# RIGHT: Test before destructive operation
bash(command="ls dist/")  # Verify what exists
bash(command="rm -rf dist/")  # Then remove
```

---

## 5. Exam Scenarios

### Scenario 1
**Q:** You need to find all files containing "deprecated_api" calls and update them. What sequence of tools should you use?

**A:** 
1. **Grep** to find all files: `grep(pattern="deprecated_api", output_mode="files_with_matches")`
2. **Read** each file to understand context
3. **Edit** each file to replace calls

```python
# Step 1: Find files
files = grep(pattern="deprecated_api", output_mode="files_with_matches")

# Step 2: For each file, read and edit
for file in files:
    content = read(path=file)
    # Analyze content to find unique text
    edit(path=file, old_string="...", new_string="...")
```

### Scenario 2
**Q:** You need to create 10 new test files with the same boilerplate template. What's the most efficient approach?

**A:** Use **Write** to create each file with the template:

```python
template = """import { test } from 'vitest';

describe('Module', () => {
  test('basic test', () => {
    expect(true).toBe(true);
  });
});
"""

test_files = [
    "/tests/auth.test.ts",
    "/tests/api.test.ts",
    "/tests/utils.test.ts",
    # ... 10 files total
]

for file_path in test_files:
    write(path=file_path, content=template)
```

### Scenario 3
**Q:** Your Edit tool fails with "text not unique" error. What should you do?

**A:** Use **Read + Write** fallback:

```python
# Step 1: Read the file
content = read(path="/src/main.ts")

# Step 2: Perform replacement using string methods
new_content = content.replace("old_pattern", "new_pattern")

# Step 3: Write back
write(path="/src/main.ts", content=new_content)
```

---

## Quick Reference

| Tool | Input | Output | Use When |
|------|-------|--------|----------|
| **Read** | path | file contents | Load file for analysis |
| **Write** | path, content | success/error | Create or completely replace file |
| **Edit** | path, old, new | success/error | Modify specific section (unique text) |
| **Bash** | command | stdout + exit code | Run shell commands/tests |
| **Grep** | pattern, path | matching lines | Search file contents |
| **Glob** | pattern | file paths | Find files by name pattern |

---

**Ready for exam!** These six tools cover 95% of development automation tasks. Master the decision tree (when to use each) and the fallback patterns (Edit → Read+Write) and you're set.
