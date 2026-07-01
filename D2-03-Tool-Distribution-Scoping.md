# Domain 2 | Topic 2.3: Tool Distribution & Scoping

## 1. Core Concepts

### What is Tool Distribution?

**Tool distribution** is the strategy of deciding which tools are available to which agents, at which times, and with what constraints. It's about right-sizing the toolset for each context.

### Why Scope Tools?

| Reason | Explanation |
|--------|-------------|
| **Security** | Fewer tools = smaller attack surface |
| **Accuracy** | Fewer tools = less confusion for Claude |
| **Cost** | Fewer tool descriptions = fewer tokens per call |
| **Safety** | Dangerous tools only where needed |
| **Performance** | Less time reasoning about irrelevant options |

### Scoping Dimensions

```
WHO: Which agent/role gets which tools?
├── Research agent → read-only tools
├── Code agent → read + write tools
└── Admin agent → all tools including destructive

WHEN: When are tools available?
├── Always available (core tools)
├── Conditionally loaded (based on context)
└── Temporarily enabled (after permission grant)

WHERE: What scope can the tool operate on?
├── Directory-scoped (only /src)
├── Resource-scoped (only this database)
└── Operation-scoped (only SELECT, no DELETE)

HOW MUCH: What are the limits?
├── Rate limits (10 calls/minute)
├── Result limits (max 50 results)
└── Size limits (files under 1MB)
```

### Tool Categories by Risk Level

```
┌─────────────────────────────────────────────┐
│  RISK LEVEL 1: Read-Only (Low Risk)         │
│  search, read_file, list_files, get_status  │
├─────────────────────────────────────────────┤
│  RISK LEVEL 2: Create/Modify (Medium Risk)  │
│  write_file, edit_file, create_record       │
├─────────────────────────────────────────────┤
│  RISK LEVEL 3: Destructive (High Risk)      │
│  delete_file, drop_table, remove_user       │
├─────────────────────────────────────────────┤
│  RISK LEVEL 4: External Effects (Critical)  │
│  send_email, deploy, publish, push_to_prod  │
└─────────────────────────────────────────────┘
```

---

## 2. Design Patterns

### Pattern 1: Role-Based Tool Assignment

```python
TOOL_SETS = {
    "researcher": {
        "tools": ["web_search", "read_document", "query_database"],
        "constraints": {"query_database": {"read_only": True}}
    },
    "developer": {
        "tools": ["read_file", "write_file", "edit_file", "run_tests", "search_code"],
        "constraints": {"write_file": {"allowed_paths": ["/src", "/tests"]}}
    },
    "reviewer": {
        "tools": ["read_file", "search_code", "run_linter", "add_comment"],
        "constraints": {}  # All read-only anyway
    },
    "deployer": {
        "tools": ["read_file", "run_tests", "deploy", "rollback"],
        "constraints": {"deploy": {"requires_approval": True}}
    }
}

def get_tools_for_role(role):
    config = TOOL_SETS[role]
    tools = load_tools(config["tools"])
    apply_constraints(tools, config["constraints"])
    return tools
```

### Pattern 2: Progressive Tool Disclosure

```python
def get_tools_for_phase(task_phase):
    """Start with minimal tools, expand as needed."""
    
    if task_phase == "analysis":
        return ["read_file", "search_code", "list_files"]
    
    elif task_phase == "planning":
        return ["read_file", "search_code", "list_files", "create_plan"]
    
    elif task_phase == "implementation":
        return ["read_file", "write_file", "edit_file", "run_tests", "search_code"]
    
    elif task_phase == "deployment":
        return ["run_tests", "build", "deploy", "health_check", "rollback"]
```

### Pattern 3: Directory-Scoped File Tools

```python
def create_scoped_file_tool(allowed_directories):
    """Create a file read tool that only works within allowed directories."""
    
    return {
        "name": "read_file",
        "description": f"Read files within: {', '.join(allowed_directories)}. "
                       f"Cannot access files outside these directories.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": f"Path within {allowed_directories}"
                }
            },
            "required": ["path"]
        },
        "validator": lambda input: any(
            input["path"].startswith(d) for d in allowed_directories
        )
    }
```

### Pattern 4: Contextual Tool Loading

```python
def load_tools_for_query(user_message, available_tools):
    """Lightweight classification to load only relevant tools."""
    
    # Use a fast model to classify the request
    classification = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=50,
        messages=[{"role": "user", "content": f"Classify this request into categories (code, data, communication, admin): {user_message}"}]
    )
    
    category = parse_category(classification)
    
    CATEGORY_TOOLS = {
        "code": ["read_file", "write_file", "edit_file", "run_tests", "search_code"],
        "data": ["query_database", "export_csv", "create_chart"],
        "communication": ["send_email", "post_slack", "create_ticket"],
        "admin": ["manage_users", "update_config", "view_logs"]
    }
    
    return [t for t in available_tools if t["name"] in CATEGORY_TOOLS.get(category, [])]
```

---

## 3. Implementation Details

### Claude Code Permission Levels

```
PERMISSION MODES (from most to least restrictive):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Ask Every Time — prompt for every tool use
2. Auto-Allow Safe — read tools allowed, write tools ask
3. Allowlist — specific commands pre-approved
4. YOLO/Trust — all tools allowed (⚠️ dangerous)
```

### Settings-Based Tool Permissions

```json
// .claude/settings.json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob", 
      "Grep",
      "Bash(npm test *)",
      "Bash(npx prettier *)"
    ],
    "deny": [
      "Bash(rm *)",
      "Bash(git push *)"
    ]
  }
}
```

### MCP Server Scoping

```json
{
  "mcpServers": {
    "readonly-db": {
      "command": "npx",
      "args": ["-y", "mcp-server-postgres", "--read-only"],
      "env": {
        "DATABASE_URL": "postgresql://readonly_user:***@host/db"
      }
    },
    "project-files": {
      "command": "npx",
      "args": ["-y", "mcp-server-filesystem", "/project/src"]
    }
  }
}
```

### Dynamic Tool Discovery via MCP

MCP supports dynamic tool discovery through the `tools/list` method, which can be re-queried mid-session. This enables **progressive tool availability** — servers can add, remove, or modify tools during a conversation based on changing context:

```python
# Server-side: tools change based on authentication state
@server.list_tools()
async def list_tools():
    base_tools = [
        Tool(name="login", description="Authenticate with credentials", ...)
    ]
    
    if session.is_authenticated:
        base_tools.extend([
            Tool(name="query_data", description="Query the data warehouse", ...),
            Tool(name="export_report", description="Export a report to CSV", ...),
        ])
        # Notify client that tool list has changed
        await server.notify("notifications/tools/list_changed")
    
    return base_tools
```

**How it works:**
- The server declares `tools: { listChanged: true }` in its capabilities during initialization
- When the tool list changes (e.g., after user authenticates, or a plugin is loaded), the server sends a `notifications/tools/list_changed` notification
- The client re-queries `tools/list` to get the updated set of available tools
- This allows tool sets to evolve without restarting the server or reconnecting

**Use cases:**
- **Authentication gating**: Show only public tools until the user logs in, then reveal privileged tools
- **Workflow progression**: Unlock deployment tools only after tests pass
- **Feature flags**: Enable experimental tools for specific sessions
- **Resource-dependent tools**: Only show database tools when a database connection is active

### Tool Count Impact on Performance

| Tool Count | Impact | Recommendation |
|-----------|--------|----------------|
| 1-5 | Excellent selection accuracy | Ideal range |
| 6-15 | Good, with proper descriptions | Use clear disambiguation |
| 16-30 | Degraded accuracy possible | Contextual loading recommended |
| 30+ | Significant confusion risk | Must use tool categories/routing |

---

## 4. Security & Reliability

### Principle of Least Privilege for Tools

```
GOOD: Agent gets minimum tools for its specific task
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Research Agent: [search, read]  ← Can't modify anything
Code Agent: [read, write, edit] in /src only ← Can't access secrets
Deploy Agent: [deploy, rollback] ← Can't write code

BAD: Every agent gets all tools
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
All Agents: [search, read, write, delete, deploy, manage_users]
↑ One compromised agent = total system access
```

### Defense in Depth for Tool Scoping

```
Layer 1: Tool availability — only load relevant tools
Layer 2: Tool descriptions — warn about constraints
Layer 3: Input validation — reject out-of-scope inputs
Layer 4: Runtime checks — verify permissions at execution
Layer 5: Audit logging — track all tool usage for review
```

### Common Scoping Vulnerabilities

| Vulnerability | Example | Fix |
|--------------|---------|-----|
| Path traversal | `read_file("../../etc/passwd")` | Normalize + validate paths |
| Scope escape via symlinks | Symlink in /src → /etc | Resolve real path before access |
| Tool parameter manipulation | Using query tool to write | Separate read/write tools entirely |
| Permission confusion | Same tool, different agents, different access | Per-agent tool instances |

---

## 5. Exam Scenarios

### Scenario 1
**Q:** You have 40 tools available. Claude frequently selects wrong tools and makes multiple attempts before finding the right one. What is the BEST architectural fix?

**A:** Implement **two-phase tool routing:**
1. Phase 1: Present Claude with 5-6 "meta-tools" (categories): `use_code_tools`, `use_data_tools`, `use_communication_tools`
2. Phase 2: When Claude selects a category, load only that category's specific tools
3. This reduces per-turn confusion while maintaining full capability

Alternative: Use a lightweight classifier (Haiku) to pre-select the tool category based on the user's message.

### Scenario 2
**Q:** A code review agent accidentally deletes a production file because it had access to `delete_file`. How should the tool set be restructured?

**A:** Apply **role-based scoping:**
- Review agent should ONLY have: `read_file`, `search_code`, `run_linter`, `add_comment`
- Remove ALL write/delete tools from the review agent's toolset
- If the agent needs to suggest deletions, it should recommend them via `add_comment`, not execute them
- Destructive tools should only be available to agents with explicit "write" roles AND require human approval

### Scenario 3
**Q:** An MCP server provides both `query_database` (read) and `modify_database` (write). Only the research agent should have query access. How do you scope this?

**A:** 
1. Create the MCP connection with **read-only database credentials**
2. Only expose `query_database` to the research agent (don't include `modify_database` in its tool list)
3. For agents that need write access, use a separate MCP server connection with write credentials
4. This provides both API-level scoping (tool availability) and infrastructure-level scoping (credentials)

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| All tools to all agents | Security risk, selection confusion | Role-based tool sets |
| Scoping only in descriptions | Claude can ignore descriptions | Enforce at execution layer |
| No path validation | Directory traversal attacks | Resolve + validate all paths |
| Static tool set regardless of task | Wasted tokens, poor accuracy | Contextual loading |
| Shared credentials across tools | Blast radius of compromise | Per-tool credentials |
| No audit trail | Can't investigate incidents | Log all tool invocations |

---

## 7. Quick Reference / Cheat Sheet

```
SCOPING DIMENSIONS
━━━━━━━━━━━━━━━━━━
• WHO: Role-based tool sets
• WHEN: Phase-based tool loading
• WHERE: Directory/resource constraints
• HOW MUCH: Rate + result + size limits

RISK LEVELS
━━━━━━━━━━━
Level 1 (Low): Read-only — always safe to allow
Level 2 (Medium): Create/Modify — scope to directories
Level 3 (High): Delete/Destroy — require confirmation
Level 4 (Critical): External effects — require approval + logging

PERMISSION ENFORCEMENT LAYERS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Tool availability (don't even show it)
2. Description warnings (soft)
3. Input validation (rejects bad params)
4. Runtime permission check (hard enforcement)
5. Audit logging (accountability)

TOOL COUNT GUIDELINES
━━━━━━━━━━━━━━━━━━━━━
• 1-5: Best accuracy, ideal for focused agents
• 6-15: Good with clear descriptions
• 16-30: Use contextual loading
• 30+: Must use routing/categories

CLAUDE CODE PERMISSIONS
━━━━━━━━━━━━━━━━━━━━━━━
• allow: [tool patterns that auto-execute]
• deny: [tool patterns that always block]
• Settings hierarchy: local > project > global
• Pattern matching: "Bash(npm *)" allows npm commands
```
