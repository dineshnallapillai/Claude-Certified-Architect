# Domain 2 | Topic 2.4: MCP Server Configuration & Protocol

## 1. Core Concepts

### MCP Protocol Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    MCP ECOSYSTEM                          │
│                                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────────────────┐  │
│  │  HOST   │    │ CLIENT  │    │      SERVER          │  │
│  │(Claude  │───▶│(Protocol│───▶│(Tool/Resource        │  │
│  │ Code)   │    │ Handler)│    │  Provider)           │  │
│  └─────────┘    └─────────┘    └─────────────────────┘  │
│                                         │                │
│                                    ┌────┴────┐           │
│                                    │         │           │
│                                 Tools   Resources        │
│                                Prompts  Sampling         │
└──────────────────────────────────────────────────────────┘
```

### Protocol Components

| Component | Description | Example |
|-----------|-------------|---------|
| **Host** | Application that embeds MCP | Claude Code, Claude Desktop |
| **Client** | Protocol implementation in the host | Handles JSON-RPC messages |
| **Server** | External process providing capabilities | Database connector, API wrapper |
| **Transport** | Communication layer | stdio (pipes), Streamable HTTP, HTTP+SSE (legacy) |

### Server Capabilities

| Capability | What It Provides | Example |
|-----------|-----------------|---------|
| **Tools** | Functions Claude can call | `query_database`, `send_email` |
| **Resources** | Data Claude can read (URI-based) | `file:///docs/readme.md` |
| **Prompts** | Pre-built prompt templates | "Summarize this document as..." |
| **Sampling** | Server requests LLM completion | Advanced: server asks Claude to process something |

### Transport Types

| Transport | Mechanism | Best For |
|-----------|-----------|----------|
| **stdio** | stdin/stdout pipes | Local CLI tools, same-machine servers |
| **Streamable HTTP** | Single HTTP endpoint for bidirectional communication; supports request-response and streaming patterns | Remote servers, web services (recommended) |
| **HTTP+SSE** (legacy) | HTTP POST (client→server) + Server-Sent Events (server→client) | Legacy remote servers; superseded by Streamable HTTP |

#### Streamable HTTP Transport (Recommended for Remote)

Streamable HTTP is the current recommended remote transport in the MCP specification, replacing the older HTTP+SSE approach. Key characteristics:

- **Single endpoint**: Uses one HTTP endpoint for both client-to-server and server-to-client communication (unlike HTTP+SSE which required separate endpoints for each direction)
- **Dual pattern support**: Supports both simple request-response (client sends POST, receives JSON-RPC response) and streaming (server can upgrade to SSE on the same endpoint for long-running operations)
- **Simpler deployment**: No need to maintain separate SSE connection endpoints; works naturally with standard HTTP infrastructure, load balancers, and proxies
- **Session management**: Supports optional session IDs via the `Mcp-Session-Id` header for stateful interactions
- **Graceful fallback**: Clients can detect whether the server supports streaming and fall back to simple request-response if not

```
Client                          Server (single /mcp endpoint)
  │                                │
  │── POST /mcp (JSON-RPC req) ──▶│  Simple request-response
  │◀── 200 OK (JSON-RPC res) ─────│
  │                                │
  │── POST /mcp (JSON-RPC req) ──▶│  Streaming response
  │◀── 200 OK (SSE stream) ───────│  (Content-Type: text/event-stream)
  │◀── data: {progress...}  ──────│
  │◀── data: {result...}    ──────│
```

#### HTTP+SSE Transport (Legacy)

The older HTTP+SSE transport used two separate mechanisms: HTTP POST for client-to-server messages and a dedicated SSE endpoint for server-to-client messages. While still supported by many implementations, new servers should use Streamable HTTP.

---

## 2. Design Patterns

### Pattern 1: Local stdio Server Configuration

```json
// .claude/settings.json or claude_desktop_config.json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/project"],
      "env": {}
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRESQL_URI": "postgresql://user:pass@localhost:5432/mydb"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

### Pattern 2: Remote Streamable HTTP Server

```json
{
  "mcpServers": {
    "remote-analytics": {
      "url": "https://mcp.mycompany.com/analytics/mcp",
      "headers": {
        "Authorization": "Bearer ${MCP_AUTH_TOKEN}"
      }
    }
  }
}
```

### Pattern 2b: Remote HTTP+SSE Server (Legacy)

```json
{
  "mcpServers": {
    "remote-analytics": {
      "url": "https://mcp.mycompany.com/analytics",
      "headers": {
        "Authorization": "Bearer ${MCP_AUTH_TOKEN}"
      }
    }
  }
}
```

### Pattern 3: Custom MCP Server (TypeScript)

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server(
  { name: "company-tools", version: "1.0.0" },
  { capabilities: { tools: {}, resources: {} } }
);

// List available tools
server.setRequestHandler("tools/list", async () => ({
  tools: [
    {
      name: "search_tickets",
      description: "Search Jira tickets by keyword or ID",
      inputSchema: {
        type: "object",
        properties: {
          query: { type: "string", description: "Search query or ticket ID" },
          status: { type: "string", enum: ["open", "closed", "all"] }
        },
        required: ["query"]
      }
    }
  ]
}));

// Handle tool calls
server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;
  
  if (name === "search_tickets") {
    const results = await jiraClient.search(args.query, args.status);
    return {
      content: [{ type: "text", text: JSON.stringify(results, null, 2) }]
    };
  }
  
  throw new Error(`Unknown tool: ${name}`);
});

// List available resources
server.setRequestHandler("resources/list", async () => ({
  resources: [
    {
      uri: "jira://board/sprint-current",
      name: "Current Sprint",
      description: "Current sprint board with all tickets",
      mimeType: "application/json"
    }
  ]
}));

// Read a resource
server.setRequestHandler("resources/read", async (request) => {
  const { uri } = request.params;
  
  if (uri === "jira://board/sprint-current") {
    const sprint = await jiraClient.getCurrentSprint();
    return {
      contents: [{ uri, mimeType: "application/json", text: JSON.stringify(sprint) }]
    };
  }
});

// Connect via stdio
const transport = new StdioServerTransport();
await server.connect(transport);
```

### Pattern 4: Custom MCP Server (Python)

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

server = Server("analytics-server")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="run_query",
            description="Run an analytics query against the data warehouse",
            inputSchema={
                "type": "object",
                "properties": {
                    "sql": {"type": "string", "description": "SQL query (read-only)"},
                    "limit": {"type": "integer", "default": 100}
                },
                "required": ["sql"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "run_query":
        # Enforce read-only
        sql = arguments["sql"].strip().upper()
        if not sql.startswith("SELECT"):
            return [TextContent(type="text", 
                text='{"error": "Only SELECT queries are allowed"}')]
        
        results = await db.execute(arguments["sql"], limit=arguments.get("limit", 100))
        return [TextContent(type="text", text=json.dumps(results))]

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream)
```

---

## 3. Implementation Details

### Configuration Locations

| Location | Scope | Shared? |
|----------|-------|---------|
| `~/.claude/settings.json` | All projects (user global) | No |
| `.claude/settings.json` | This project (all team members) | Yes (git) |
| `.claude/settings.local.json` | This project (you only) | No |
| `~/Library/Application Support/Claude/claude_desktop_config.json` | Claude Desktop | No |

### Environment Variable Handling

```json
{
  "mcpServers": {
    "myserver": {
      "command": "node",
      "args": ["server.js"],
      "env": {
        "API_KEY": "${MY_API_KEY}",
        "DB_HOST": "${DATABASE_HOST:-localhost}",
        "DEBUG": "false"
      }
    }
  }
}
```
- `${VAR}` — reads from shell environment
- `${VAR:-default}` — default if not set
- Bare strings — used as-is

### Server Lifecycle

```
1. Claude Code starts → reads settings
2. For each mcpServer:
   a. Spawn process with command + args
   b. Initialize protocol handshake
   c. Discover capabilities (tools/list, resources/list)
   d. Register tools/resources for Claude to use
3. During conversation:
   a. Claude requests tool → MCP client calls server
   b. Server executes → returns result
   c. Result fed back to Claude
4. Session ends → servers terminated
```

### MCP Initialization Handshake

Before any tool calls can happen, the client and server must complete an initialization handshake. This is the first exchange after transport connection is established:

```json
// Step 1: Client sends initialize request
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-03-26",
    "capabilities": {
      "roots": { "listChanged": true }
    },
    "clientInfo": {
      "name": "claude-code",
      "version": "1.0.0"
    }
  }
}

// Step 2: Server responds with its capabilities
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-03-26",
    "capabilities": {
      "tools": { "listChanged": true },
      "resources": { "subscribe": true },
      "prompts": { "listChanged": true }
    },
    "serverInfo": {
      "name": "company-tools",
      "version": "1.0.0"
    }
  }
}

// Step 3: Client sends initialized notification (no id — it's a notification)
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

**Key points:**
- The client proposes a `protocolVersion`; the server responds with the version it supports (must match or be compatible)
- Both sides declare their `capabilities` so each knows what features the other supports
- The `initialized` notification signals the server that the client is ready — only after this can normal requests begin
- If the server does not support the client's protocol version, it should respond with an error

### MCP Notifications

MCP defines notification messages (JSON-RPC messages without an `id` field) that do not require a response. These enable real-time feedback during operations:

#### Progress Reporting (`notifications/progress`)

Servers can report progress on long-running operations:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/progress",
  "params": {
    "progressToken": "op-42",
    "progress": 75,
    "total": 100,
    "message": "Processing records: 75/100 complete"
  }
}
```

- The `progressToken` is provided by the client in the original request's `_meta.progressToken` field
- `progress` and `total` are numeric; clients can calculate percentage
- `message` is an optional human-readable status string

#### Log Messages (`notifications/message`)

Servers can emit log messages for debugging and observability:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/message",
  "params": {
    "level": "warning",
    "logger": "database-connector",
    "data": "Connection pool nearing capacity (45/50 active connections)"
  }
}
```

- `level` follows standard severity: `debug`, `info`, `warning`, `error`
- `logger` identifies the source component
- `data` can be a string or structured object
- Clients can use these for diagnostics without interrupting the request flow

### JSON-RPC Message Format

```json
// Client → Server (tool call)
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_tickets",
    "arguments": {"query": "login bug", "status": "open"}
  }
}

// Server → Client (result)
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {"type": "text", "text": "[{\"id\": \"PROJ-123\", \"title\": \"Login fails on Safari\"}]"}
    ]
  }
}
```

---

## 4. Security & Reliability

### MCP Security Considerations

| Concern | Risk | Mitigation |
|---------|------|-----------|
| Credential exposure | Server env vars leaked | Use environment references, not hardcoded |
| Untrusted servers | Malicious tool behavior | Only use verified/audited MCP servers |
| Data exfiltration | Server sends data externally | Network isolation, audit logging |
| Command injection | Malicious server args | Validate all configuration values |
| Privilege escalation | Server gains host access | Run servers with minimal OS permissions |

### Best Practices for Production MCP

1. **Audit server code** before deploying
2. **Use read-only credentials** where possible
3. **Network isolation** — servers shouldn't have internet access unless needed
4. **Resource limits** — CPU, memory, and time limits on server processes
5. **Separate secrets** — use environment variables, not config files
6. **Version pinning** — pin exact versions of MCP servers (`npx -y package@1.2.3`)
7. **Monitor health** — detect and restart crashed servers

### Server Health Monitoring

```python
class MCPServerManager:
    def __init__(self):
        self.servers = {}
    
    async def health_check(self, server_name):
        server = self.servers[server_name]
        try:
            # Ping with short timeout
            await asyncio.wait_for(server.ping(), timeout=5.0)
            return True
        except (asyncio.TimeoutError, ConnectionError):
            await self.restart_server(server_name)
            return False
```

---

## 5. Exam Scenarios

### Scenario 1
**Q:** A team wants all developers to have access to a shared database query tool via MCP, but credentials should not be committed to git. What is the BEST configuration approach?

**A:** Use project-level settings with environment variable references:

```json
// .claude/settings.json (committed to git)
{
  "mcpServers": {
    "database": {
      "command": "npx",
      "args": ["-y", "@mcp/server-postgres", "--read-only"],
      "env": {
        "DATABASE_URL": "${TEAM_DB_URL}"
      }
    }
  }
}
```

Each developer sets `TEAM_DB_URL` in their shell environment or `.env` file (not committed). The server config is shared; secrets are local.

### Scenario 2
**Q:** An MCP server occasionally crashes, leaving Claude unable to use its tools. What architectural pattern ensures reliability?

**A:** Implement:
1. **Health check ping** — detect crashes within seconds
2. **Auto-restart** — respawn the server process on crash
3. **Graceful degradation** — if server is down, Claude gets a structured error saying the tool is temporarily unavailable (not a hard crash)
4. **Circuit breaker** — if server crashes 3+ times in 5 minutes, disable it and notify the user

### Scenario 3
**Q:** You need an MCP server that provides both read and write access to a database. Different agents should have different access levels. What's the correct architecture?

**A:** Run **two separate MCP server instances** with different credentials:

```json
{
  "mcpServers": {
    "db-read": {
      "command": "npx",
      "args": ["-y", "@mcp/server-postgres", "--read-only"],
      "env": {"DATABASE_URL": "${DB_READONLY_URL}"}
    },
    "db-write": {
      "command": "npx",
      "args": ["-y", "@mcp/server-postgres"],
      "env": {"DATABASE_URL": "${DB_READWRITE_URL}"}
    }
  }
}
```

Then scope tool access: research agents only see `db-read` tools; admin agents see `db-write` tools.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Hardcoded credentials in config | Leaked if committed to git | Use `${ENV_VAR}` references |
| Unpinned server versions | Breaking changes on update | Pin exact versions |
| Single server for all capabilities | Blast radius, hard to scope | One server per concern |
| No health monitoring | Silent failures | Health checks + auto-restart |
| Overly permissive servers | Security risk | Read-only by default, write only where needed |
| No error handling in server | Crashes propagate | Catch all exceptions, return structured errors |

---

## 7. Quick Reference / Cheat Sheet

```
MCP ARCHITECTURE
━━━━━━━━━━━━━━━━
Host (Claude Code) → Client (protocol) → Server (tools/resources)

TRANSPORT TYPES
━━━━━━━━━━━━━━━
• stdio: Local processes, stdin/stdout pipes
• Streamable HTTP: Remote servers, single endpoint (recommended)
• HTTP+SSE: Legacy remote transport (superseded)

CONFIGURATION FORMAT
━━━━━━━━━━━━━━━━━━━━
{
  "mcpServers": {
    "<name>": {
      "command": "<executable>",
      "args": ["<arg1>", "<arg2>"],
      "env": {"KEY": "${ENV_VAR}"}
    }
  }
}

SERVER CAPABILITIES
━━━━━━━━━━━━━━━━━━━
• tools: Functions Claude can call
• resources: Data Claude can read (URI-based)
• prompts: Template prompts
• sampling: Server requests LLM help (advanced)

CONFIG LOCATIONS (priority order)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. .claude/settings.local.json  (local, highest priority)
2. .claude/settings.json        (project, git-tracked)
3. ~/.claude/settings.json      (user global)

SECURITY CHECKLIST
━━━━━━━━━━━━━━━━━━
✓ No hardcoded credentials
✓ Environment variable references for secrets
✓ Read-only access by default
✓ Version pinning for servers
✓ Audit server code before use
✓ Health monitoring
✓ Network isolation where possible
```
