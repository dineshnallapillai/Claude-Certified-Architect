# Domain 1 | Topic 1.4: Tool Design Principles & MCP Integration

## 1. Core Concepts

### What Are Tools in Claude's Architecture?

Tools (a.k.a. function calling) let Claude interact with external systems. Claude doesn't execute tools itself — it **requests** tool use, and your application executes the tool and returns results.

```
User Message → Claude Reasoning → Tool Use Request → Your Code Executes → Tool Result → Claude Continues
```

### Tool Definition Anatomy

```json
{
  "name": "get_weather",
  "description": "Get current weather for a city. Use this when the user asks about weather conditions, temperature, or forecasts for a specific location.",
  "input_schema": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string",
        "description": "City name (e.g., 'San Francisco, CA')"
      },
      "units": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "description": "Temperature unit preference"
      }
    },
    "required": ["city"]
  }
}
```

### Three Pillars of Tool Design

| Pillar | Principle |
|--------|-----------|
| **Name** | Verb_noun format, clear action (`search_documents`, `create_ticket`) |
| **Description** | WHEN to use it (not just what it does), with examples |
| **Schema** | Precise types, enums for constrained values, helpful property descriptions |

### Model Context Protocol (MCP)

**MCP** is an open protocol that standardizes how AI applications connect to external data sources and tools. Think of it as a "USB-C for AI" — one standard interface for many integrations.

```
┌─────────────┐     MCP Protocol     ┌─────────────────┐
│  Claude Code │ ◄──────────────────► │  MCP Server     │
│  (MCP Host)  │     (JSON-RPC)       │  (Tool Provider)│
└─────────────┘                       └─────────────────┘
                                          │
                                          ├── Database
                                          ├── API Service
                                          └── File System
```

#### MCP Architecture

| Component | Role |
|-----------|------|
| **Host** | The AI application (Claude Code, Claude Desktop) |
| **Client** | Protocol handler within the host |
| **Server** | Provides tools, resources, and prompts |
| **Transport** | Communication layer (stdio, HTTP/SSE) |

#### MCP Capabilities

| Capability | Description |
|-----------|-------------|
| **Tools** | Functions the AI can invoke |
| **Resources** | Data sources the AI can read (like file:// URIs) |
| **Prompts** | Pre-built prompt templates |
| **Sampling** | Server can request LLM completions (advanced) |

---

## 2. Design Patterns

### Pattern 1: Action-Oriented Tool Design

```json
// GOOD: Clear action, specific purpose
{
  "name": "search_knowledge_base",
  "description": "Search the internal knowledge base for relevant articles. Use when the user asks a question that might be answered by existing documentation. Returns top 5 matching articles with relevance scores.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Natural language search query"
      },
      "category": {
        "type": "string",
        "enum": ["engineering", "product", "hr", "finance"],
        "description": "Optional category filter to narrow results"
      },
      "max_results": {
        "type": "integer",
        "default": 5,
        "description": "Maximum number of results to return (1-20)"
      }
    },
    "required": ["query"]
  }
}
```

### Pattern 2: Composite Tools (Reduce Round Trips)

```json
// Instead of separate search + read + summarize
{
  "name": "research_topic",
  "description": "Search for information on a topic, read the top results, and return a structured summary. Combines search, retrieval, and summarization in one call.",
  "input_schema": {
    "type": "object",
    "properties": {
      "topic": {"type": "string"},
      "depth": {"type": "string", "enum": ["shallow", "medium", "deep"]}
    },
    "required": ["topic"]
  }
}
```

### Pattern 3: MCP Server Configuration

```json
// In .claude/settings.json or claude_desktop_config.json
{
  "mcpServers": {
    "database": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost:5432/mydb"
      }
    },
    "github": {
      "command": "npx", 
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"]
    }
  }
}
```

### Pattern 4: Tool Scoping by Agent Role

```python
# Research agent: read-only tools
research_tools = [search_web, read_document, query_database]

# Editor agent: write tools (scoped to specific directories)
editor_tools = [read_file, write_file, edit_file]  

# Admin agent: dangerous tools (requires elevated permissions)
admin_tools = [delete_resource, modify_permissions, deploy]
```

---

## 3. Implementation Details

### Tool Description Best Practices

| Practice | Example |
|----------|---------|
| State WHEN to use | "Use when the user asks about order status" |
| State WHEN NOT to use | "Do NOT use for historical orders older than 90 days" |
| Describe output format | "Returns JSON with fields: id, status, date" |
| Include examples | "e.g., query: 'shipping status for order #123'" |
| Specify limits | "Maximum 10 results per query" |

### Structured Error Responses from Tools

```python
# Return structured errors so Claude can reason about them
def execute_tool(name, input):
    try:
        result = tool_functions[name](**input)
        return {"status": "success", "data": result}
    except NotFoundError as e:
        return {"status": "error", "error_type": "not_found", 
                "message": str(e), "suggestion": "Try a broader search query"}
    except PermissionError as e:
        return {"status": "error", "error_type": "permission_denied",
                "message": str(e), "suggestion": "This requires admin access"}
    except Exception as e:
        return {"status": "error", "error_type": "internal",
                "message": "An unexpected error occurred"}
```

### MCP Server Implementation (TypeScript)

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server({
  name: "my-custom-server",
  version: "1.0.0"
}, {
  capabilities: {
    tools: {},
    resources: {}
  }
});

// Register a tool
server.setRequestHandler("tools/list", async () => ({
  tools: [{
    name: "query_data",
    description: "Query the analytics database",
    inputSchema: {
      type: "object",
      properties: {
        sql: { type: "string", description: "SQL query to execute" }
      },
      required: ["sql"]
    }
  }]
}));

// Handle tool execution
server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "query_data") {
    const result = await executeQuery(request.params.arguments.sql);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  }
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

### MCP Transport Types

| Transport | Use Case | Communication | Status |
|-----------|----------|---------------|--------|
| **stdio** | Local processes | stdin/stdout | Recommended (local) |
| **Streamable HTTP** | Remote servers | Single HTTP endpoint | Recommended (remote) |
| **HTTP+SSE** | Remote servers | HTTP POST + Server-Sent Events | Legacy (deprecated) |

---

## 4. Security & Reliability

### Tool Security Principles

| Principle | Implementation |
|-----------|---------------|
| Input validation | Validate all tool inputs before execution |
| Output sanitization | Strip sensitive data from tool results |
| Rate limiting | Prevent tool abuse (max calls per minute) |
| Scoping | Filesystem tools limited to specific directories |
| Authentication | MCP servers require proper credentials |
| Least privilege | Each tool does ONE thing with minimal permissions |

### MCP Security Model

```
TRUST LEVELS
━━━━━━━━━━━━
1. Host (Claude Code) — fully trusted
2. MCP Client — trusted, manages connections
3. MCP Server — semi-trusted, validated tools
4. External APIs — untrusted, validate all responses
```

- **Never expose credentials in tool results** returned to Claude
- **Validate MCP server identity** before connecting
- **Sandbox MCP servers** — they shouldn't have unrestricted system access
- **Audit tool usage** — log all tool invocations for review

### Tool Reliability Patterns

| Pattern | Purpose |
|---------|---------|
| Retry with backoff | Handle transient failures |
| Circuit breaker | Stop calling broken tools |
| Timeout | Don't wait forever |
| Fallback tools | Alternative when primary fails |
| Idempotency | Safe to retry without side effects |

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A tool description says: "Searches the database." Claude is using this tool when it should be using the API search instead. What is the MOST likely cause and fix?

**A:** The tool description lacks **WHEN to use** guidance. Fix: Add explicit usage criteria:
```
"Search the internal PostgreSQL database for user records. 
Use ONLY when looking up user profiles or account data. 
Do NOT use for product searches (use search_products instead) 
or for analytics queries (use analytics_query instead)."
```

The key principle: tool descriptions should include both positive (when to use) and negative (when NOT to use) guidance.

### Scenario 2
**Q:** An MCP server provides a `run_sql` tool with unrestricted query access. What is the PRIMARY security concern and recommended mitigation?

**A:** **SQL injection and data exfiltration risk.** Claude could be manipulated (via prompt injection in user content) to run destructive or data-leaking queries.

Mitigations:
1. **Read-only access** — MCP server connects with read-only DB credentials
2. **Query allowlist** — only permit specific query patterns
3. **Row limits** — cap results to prevent bulk data extraction
4. **Parameterized queries** — prevent SQL injection
5. **Audit logging** — track all queries for review

### Scenario 3
**Q:** You need Claude to interact with three external services: GitHub, Jira, and Slack. What's the BEST architectural approach?

**A:** Use **separate MCP servers** for each service:
- `@modelcontextprotocol/server-github` — repo tools
- Custom Jira MCP server — ticket tools
- Custom Slack MCP server — messaging tools

**Why separate servers:**
- Independent credentials per service
- Principle of least privilege (can disable one without affecting others)
- Independent scaling and updating
- Clear security boundaries

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Vague tool descriptions | Claude uses wrong tool | Specific WHEN/WHEN-NOT guidance |
| Too many tools | Claude confused, picks poorly | Group related tools, use composites |
| Raw error strings | Claude can't reason about failures | Structured error objects with types |
| One mega-MCP server | Security risk, hard to maintain | One server per service/concern |
| Secrets in tool results | Leaked to conversation/logs | Strip sensitive data before returning |
| No input validation | Injection attacks, crashes | Validate all inputs server-side |
| Tools without rate limits | Runaway agent hammers API | Per-tool and global rate limits |

---

## 7. Quick Reference / Cheat Sheet

```
TOOL DEFINITION CHECKLIST
━━━━━━━━━━━━━━━━━━━━━━━━
✓ name: verb_noun (snake_case)
✓ description: WHEN to use + WHEN NOT + output format
✓ input_schema: JSON Schema with property descriptions
✓ required: only truly mandatory fields
✓ enums: for constrained value sets
✓ defaults: for optional params with sensible defaults

MCP ARCHITECTURE
━━━━━━━━━━━━━━━━
Host → Client → Server → External Systems
      (protocol)  (tools, resources, prompts)

MCP TRANSPORTS
━━━━━━━━━━━━━━
• stdio: local processes (most common for CLI)
• Streamable HTTP: remote servers (recommended)
• HTTP+SSE: remote servers (legacy/deprecated)

MCP SERVER CONFIG
━━━━━━━━━━━━━━━━━
Location: .claude/settings.json or claude_desktop_config.json
Fields: command, args, env
Env vars: use ${VAR} for secrets from environment

TOOL RESULT FORMAT
━━━━━━━━━━━━━━━━━━
Success: {"status": "success", "data": {...}}
Error: {"status": "error", "error_type": "...", "message": "...", "suggestion": "..."}

SECURITY PRIORITIES
━━━━━━━━━━━━━━━━━━
1. Validate inputs (prevent injection)
2. Sanitize outputs (prevent data leaks)
3. Scope access (least privilege)
4. Rate limit (prevent abuse)
5. Audit log (track usage)
```
