# Domain 2 | Topic 2.1: Tool Description Best Practices

## 1. Core Concepts

### Why Tool Descriptions Matter

Claude selects tools based on their **descriptions**, not their names. A well-written description:
- Tells Claude WHEN to use the tool (trigger conditions)
- Tells Claude WHEN NOT to use it (disambiguation)
- Describes the expected output format
- Includes boundary conditions and limitations

### Anatomy of an Effective Tool Description

```
[WHAT] One sentence describing the tool's function
[WHEN] Clear trigger conditions — when Claude should reach for this tool
[WHEN NOT] Disambiguation — when to use a different tool instead
[OUTPUT] What the response looks like
[LIMITATIONS] Constraints, rate limits, data freshness
[EXAMPLES] Optional: example queries that should trigger this tool
```

### Description Quality Spectrum

```
BAD:    "Searches stuff"
POOR:   "Searches the database"
OK:     "Searches the customer database for user records"
GOOD:   "Search the customer database for user records by name, email, or ID.
         Use when the user asks about a specific customer's account details.
         Do NOT use for aggregate queries (use analytics_query instead).
         Returns: {id, name, email, status, created_at}"
```

---

## 2. Design Patterns

### Pattern 1: The Complete Description Template

```json
{
  "name": "search_orders",
  "description": "Search for customer orders by order ID, customer email, or date range. Use this tool when the user asks about order status, tracking, delivery, or purchase history. Do NOT use for refund processing (use process_refund) or inventory checks (use check_inventory). Returns up to 10 orders sorted by date, each containing: order_id, status, items, total, tracking_url. Note: only returns orders from the last 12 months.",
  "input_schema": { ... }
}
```

### Pattern 2: Disambiguation Between Similar Tools

```json
// Tool 1
{
  "name": "search_documents",
  "description": "Full-text search across all internal documentation (wiki, runbooks, design docs). Use when the user needs to find information that may exist in written documentation. NOT for code search (use search_code) or ticket search (use search_jira)."
}

// Tool 2  
{
  "name": "search_code",
  "description": "Search across all code repositories for functions, classes, variables, or patterns. Use when looking for code implementations, definitions, or usage examples. NOT for documentation (use search_documents) or issues (use search_jira)."
}

// Tool 3
{
  "name": "search_jira",
  "description": "Search Jira tickets by keyword, assignee, or status. Use when looking for bugs, feature requests, or task status. NOT for code (use search_code) or docs (use search_documents)."
}
```

### Pattern 3: Parameter-Level Descriptions

```json
{
  "name": "query_metrics",
  "description": "Query time-series metrics from the monitoring system...",
  "input_schema": {
    "type": "object",
    "properties": {
      "metric_name": {
        "type": "string",
        "description": "The metric identifier. Common metrics: 'http_requests_total', 'response_time_p99', 'error_rate', 'cpu_usage'. Use dot notation for namespaced metrics (e.g., 'service.auth.latency')"
      },
      "time_range": {
        "type": "string",
        "description": "Relative time range. Format: number + unit (m=minutes, h=hours, d=days). Examples: '30m', '24h', '7d'. Maximum: '90d'",
        "pattern": "^\\d+[mhd]$"
      },
      "aggregation": {
        "type": "string",
        "enum": ["avg", "sum", "max", "min", "p50", "p95", "p99"],
        "description": "How to aggregate data points. Use 'avg' for typical values, 'p99' for worst-case latency, 'sum' for total counts"
      }
    }
  }
}
```

### Pattern 4: Action Verbs in Names

| Good Names | Bad Names | Why |
|-----------|-----------|-----|
| `search_documents` | `documents` | Verb makes action clear |
| `create_ticket` | `ticket_creator` | Standard verb_noun pattern |
| `get_user_profile` | `user_data` | Indicates read operation |
| `update_config` | `config_manager` | Specific action |
| `delete_cache_entry` | `cache` | Makes danger level clear |

---

## 3. Implementation Details

### Description Length Guidelines

| Scenario | Recommended Length | Rationale |
|----------|-------------------|-----------|
| Few tools (1-5) | Medium (2-3 sentences) | Less confusion, less needed |
| Many tools (10+) | Detailed (4-6 sentences) | Critical for disambiguation |
| Similar tools | Extra detailed | Must clearly differentiate |
| Dangerous tools | Verbose with warnings | Safety requires clarity |

### Token Cost of Descriptions

Tool descriptions are included in EVERY API call. Optimization matters:

```
10 tools × 100 tokens each = 1,000 tokens per call
10 tools × 300 tokens each = 3,000 tokens per call

In a 20-iteration agentic loop:
1,000 × 20 = 20,000 tokens (just for descriptions)
3,000 × 20 = 60,000 tokens (just for descriptions)
```

**Balance:** Descriptive enough for correct selection, concise enough for token efficiency.

### Dynamic Tool Descriptions

```python
# Adapt descriptions based on context
def get_tools(user_role):
    base_tools = load_tools()
    
    if user_role == "admin":
        # Include dangerous tools with extra warnings
        base_tools.append({
            "name": "delete_user",
            "description": "PERMANENTLY delete a user account and ALL associated data. This action is IRREVERSIBLE. Use ONLY when explicitly asked to delete a user. Always confirm the user ID before executing."
        })
    
    return base_tools
```

### Tool Annotations (MCP Metadata Hints)

The MCP specification defines an `annotations` field on tool definitions that provides structured metadata about a tool's behavior. These hints help clients (and hosts like Claude) understand what a tool does without executing it, enabling smarter UI decisions and safety checks:

```json
{
  "name": "delete_user",
  "description": "Permanently delete a user account and all associated data.",
  "inputSchema": { ... },
  "annotations": {
    "readOnlyHint": false,
    "destructiveHint": true,
    "idempotentHint": false,
    "openWorldHint": false
  }
}
```

| Annotation | Type | Default | Meaning |
|-----------|------|---------|---------|
| `readOnlyHint` | boolean | `false` | Tool does not modify any state (safe to call without side effects) |
| `destructiveHint` | boolean | `true` | Tool may perform destructive/irreversible operations (e.g., delete) |
| `idempotentHint` | boolean | `false` | Calling the tool multiple times with the same input has the same effect as calling it once |
| `openWorldHint` | boolean | `true` | Tool interacts with external entities beyond the server's control (e.g., third-party APIs, the internet) |

**How clients use annotations:**
- A client may auto-approve tools with `readOnlyHint: true` without prompting the user
- Tools with `destructiveHint: true` may trigger extra confirmation dialogs
- `idempotentHint: true` tells the client it is safe to retry on failure
- `openWorldHint: false` indicates the tool only interacts with local/controlled resources

**Important:** These are *hints*, not guarantees. Clients should not rely on them for security enforcement but can use them for UX optimization and risk signaling. Always combine annotations with proper server-side authorization.

```python
# Example: MCP server defining tools with annotations
@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="read_config",
            description="Read the current application configuration.",
            inputSchema={"type": "object", "properties": {}},
            annotations={
                "readOnlyHint": True,
                "destructiveHint": False,
                "idempotentHint": True,
                "openWorldHint": False
            }
        ),
        Tool(
            name="send_notification",
            description="Send a push notification to a user.",
            inputSchema={...},
            annotations={
                "readOnlyHint": False,
                "destructiveHint": False,
                "idempotentHint": False,
                "openWorldHint": True
            }
        )
    ]
```

### Testing Tool Descriptions

```python
# Test that Claude selects the right tool for given prompts
test_cases = [
    ("What's the status of order #123?", "search_orders"),
    ("Show me the code for the auth module", "search_code"),
    ("Is there a bug ticket for the login issue?", "search_jira"),
    ("How do I configure the database?", "search_documents"),
]

for query, expected_tool in test_cases:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        tools=tools,
        messages=[{"role": "user", "content": query}]
    )
    actual_tool = get_tool_name(response)
    assert actual_tool == expected_tool, f"Expected {expected_tool}, got {actual_tool}"
```

---

## 4. Security & Reliability

### Description-Based Security

| Technique | Example |
|-----------|---------|
| Explicit scope limits | "Only searches files in /app directory" |
| Data sensitivity warnings | "Returns PII — do not include in logs" |
| Permission requirements | "Requires admin role — verify authorization first" |
| Irreversibility warnings | "IRREVERSIBLE: deletes data permanently" |
| Rate limit documentation | "Limited to 10 calls per minute" |

### Misleading Descriptions (Attack Vector)

If tool descriptions are user-provided (e.g., from MCP servers), a malicious server could:
- Describe a "read_file" tool that actually deletes files
- Describe a "search" tool that exfiltrates data

**Mitigation:** Only connect to trusted MCP servers; audit tool descriptions.

---

## 5. Exam Scenarios

### Scenario 1
**Q:** Claude consistently picks `search_all` instead of the more specific `search_users` tool. Both tool names mention "search." What is the MOST effective fix?

**A:** Update `search_users` description to include explicit trigger conditions:
```
"Search for user accounts by name, email, or ID. USE THIS TOOL (not search_all) 
when the query is specifically about user profiles, accounts, or login information. 
Returns structured user data: {id, name, email, role, last_login}."
```

And add exclusion to `search_all`:
```
"General-purpose search across all data types. Use ONLY when the query doesn't 
match a specific search tool. Do NOT use for user lookups (use search_users), 
order lookups (use search_orders), or code search (use search_code)."
```

### Scenario 2
**Q:** You have 25 tools defined. Claude is using 3,000+ tokens just for tool descriptions per API call. What optimization MOST reduces cost without hurting accuracy?

**A:** **Contextual tool loading** — only include tools relevant to the current task:
1. Categorize tools by domain
2. Use a lightweight router (or first turn without tools) to identify needed category
3. Load only that category's tools for subsequent turns

Secondary: Tighten descriptions to essentials (WHEN to use + output format) and remove verbose examples.

### Scenario 3
**Q:** A tool's description says "Creates a new user" but the implementation also sends a welcome email. Claude never mentions the email to users. What should change?

**A:** Update the description to reflect the full behavior:
```
"Creates a new user account AND sends a welcome email to the provided address. 
Use when registering new users. Returns: {user_id, email_sent: boolean}. 
Note: the welcome email is sent immediately upon user creation."
```

Tool descriptions must accurately reflect observable behavior so Claude can inform users correctly.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Name-only, no description | Claude guesses based on name alone | Always write detailed descriptions |
| Description says WHAT, not WHEN | Claude doesn't know trigger conditions | Lead with trigger conditions |
| Copy-paste similar descriptions | Claude can't disambiguate | Explicit mutual exclusion clauses |
| Hidden side effects | Claude unaware of consequences | Document ALL observable effects |
| Novel jargon in descriptions | Claude may misinterpret | Use standard terminology |
| Descriptions reference other tools by internal ID | Fragile, breaks on rename | Reference by purpose/function |

---

## 7. Quick Reference / Cheat Sheet

```
DESCRIPTION TEMPLATE
━━━━━━━━━━━━━━━━━━━━
"[WHAT it does]. Use when [TRIGGER conditions].
Do NOT use for [DISAMBIGUATION — use X instead].
Returns: [OUTPUT format/fields].
[LIMITATIONS if any]."

NAMING RULES
━━━━━━━━━━━━
• verb_noun format (snake_case)
• Action verbs: search, create, get, update, delete, list, validate
• Be specific: search_users > search > find_stuff

PROPERTY DESCRIPTIONS MUST INCLUDE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• Expected format ("ISO 8601 date", "email address")
• Valid range ("1-100", "max 500 characters")
• Examples for ambiguous fields
• Enum values when options are finite

TOKEN OPTIMIZATION
━━━━━━━━━━━━━━━━━━
• Average good description: 50-150 tokens
• N tools × M tokens × K iterations = total cost
• Contextual tool loading for large tool sets
• Essential info only (WHEN + output format minimum)

DISAMBIGUATION CHECKLIST
━━━━━━━━━━━━━━━━━━━━━━━━
✓ Each similar tool explicitly says what it's NOT for
✓ Mutual exclusion: "Use X, not Y, when Z"
✓ Different return types clearly stated
✓ Scope boundaries explicit (which data, which systems)
```
