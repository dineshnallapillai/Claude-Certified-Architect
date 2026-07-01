# Domain 2 | Topic 2.2: Structured Error Responses

## 1. Core Concepts

### Why Structure Errors?

When tools fail, Claude needs to **reason about the failure** and decide what to do next. Unstructured error strings ("Something went wrong") give Claude nothing to work with. Structured errors enable:
- **Root cause identification** — what type of error?
- **Actionable recovery** — what should Claude try instead?
- **User communication** — what should the user know?
- **Conditional logic** — different error types → different strategies

### Error Response Schema

```json
{
  "status": "error",
  "error_type": "not_found | validation | permission | timeout | rate_limit | conflict | internal",
  "message": "Human-readable explanation of what went wrong",
  "details": {
    "field": "email",
    "constraint": "must be a valid email format",
    "received": "not-an-email"
  },
  "suggestion": "What Claude should try instead",
  "retryable": true | false
}
```

### Error Types Taxonomy

| Error Type | Meaning | Claude's Action |
|-----------|---------|-----------------|
| `not_found` | Resource doesn't exist | Try different ID/query |
| `validation` | Input failed validation | Fix input and retry |
| `permission` | Insufficient access | Inform user, don't retry |
| `timeout` | Operation took too long | Retry once, then simplify |
| `rate_limit` | Too many requests | Wait and retry |
| `conflict` | Conflicting state (e.g., duplicate) | Resolve conflict or inform user |
| `internal` | Unexpected server error | Retry once, then escalate |

### The `is_error` Flag

```python
# In the tool_result sent back to Claude:
{
    "type": "tool_result",
    "tool_use_id": "toolu_abc123",
    "content": '{"status":"error", "error_type":"not_found", ...}',
    "is_error": True   # ← Key signal to Claude
}
```

- `is_error: True` explicitly tells Claude the tool failed
- Without it, Claude might interpret error JSON as "successful data about an error"
- Claude Code and SDK use this to adjust reasoning

---

## 2. Design Patterns

### Pattern 1: Typed Error Factory

```python
class ToolError:
    @staticmethod
    def not_found(resource_type, identifier, suggestion=None):
        return {
            "status": "error",
            "error_type": "not_found",
            "message": f"{resource_type} with identifier '{identifier}' was not found",
            "suggestion": suggestion or f"Verify the {resource_type} ID or try searching with different criteria",
            "retryable": False
        }
    
    @staticmethod
    def validation(field, constraint, received):
        return {
            "status": "error",
            "error_type": "validation",
            "message": f"Invalid value for '{field}': {constraint}",
            "details": {"field": field, "constraint": constraint, "received": received},
            "suggestion": f"Please provide a valid value for '{field}' that satisfies: {constraint}",
            "retryable": True
        }
    
    @staticmethod
    def rate_limited(retry_after_seconds):
        return {
            "status": "error",
            "error_type": "rate_limit",
            "message": f"Rate limit exceeded. Retry after {retry_after_seconds}s",
            "retryable": True,
            "retry_after": retry_after_seconds
        }
    
    @staticmethod
    def permission_denied(required_permission, current_role):
        return {
            "status": "error",
            "error_type": "permission",
            "message": f"Access denied. Required: '{required_permission}', current role: '{current_role}'",
            "suggestion": "Inform the user that elevated permissions are needed for this action",
            "retryable": False
        }
```

### Pattern 2: Error with Context

```python
def search_users(query):
    results = db.search(query)
    
    if not results:
        return ToolError.not_found(
            resource_type="user",
            identifier=query,
            suggestion=f"No users match '{query}'. Try: "
                       f"1) A partial name match, "
                       f"2) Search by email domain, "
                       f"3) Check if the user was recently deleted"
        )
    
    return {"status": "success", "data": results}
```

### Pattern 3: Partial Success Responses

```python
def batch_update(items):
    results = {"succeeded": [], "failed": []}
    
    for item in items:
        try:
            update(item)
            results["succeeded"].append(item["id"])
        except Exception as e:
            results["failed"].append({
                "id": item["id"],
                "error": str(e)
            })
    
    if results["failed"] and not results["succeeded"]:
        return {"status": "error", "error_type": "batch_failure", 
                "message": "All items failed", "details": results}
    elif results["failed"]:
        return {"status": "partial_success", 
                "message": f"{len(results['succeeded'])} succeeded, {len(results['failed'])} failed",
                "data": results}
    else:
        return {"status": "success", "data": results}
```

### Pattern 4: Error Chain (Preserving Context)

```python
def complex_operation(params):
    # Step 1
    user = get_user(params["user_id"])
    if user["status"] == "error":
        return {**user, "context": "Failed while looking up user for operation"}
    
    # Step 2
    result = process(user["data"])
    if result["status"] == "error":
        return {
            **result,
            "context": "User found but processing failed",
            "chain": [
                {"step": "get_user", "status": "success"},
                {"step": "process", "status": "error", "error": result["message"]}
            ]
        }
    
    return {"status": "success", "data": result}
```

---

## 3. Implementation Details

### Response Format Comparison

```python
# BAD: Unstructured string
"Error: something went wrong with the query"

# POOR: Has type but no guidance
{"error": "NotFoundError", "message": "User not found"}

# GOOD: Typed, actionable, with guidance
{
    "status": "error",
    "error_type": "not_found",
    "message": "User with email 'john@example.com' not found in active users",
    "suggestion": "Try searching by partial name or check if user was deactivated",
    "retryable": False,
    "details": {
        "searched_field": "email",
        "searched_value": "john@example.com",
        "searched_scope": "active_users_only"
    }
}
```

### Error Handling in Tool Execution Wrapper

```python
def execute_tool(name, input_params, tool_use_id):
    """Universal tool execution wrapper with structured error handling."""
    
    if name not in TOOL_REGISTRY:
        return tool_result(tool_use_id, {
            "status": "error",
            "error_type": "not_found",
            "message": f"Tool '{name}' does not exist. Available: {list(TOOL_REGISTRY.keys())}",
            "retryable": False
        }, is_error=True)
    
    tool_fn = TOOL_REGISTRY[name]
    
    try:
        # Validate input against schema
        validate_input(tool_fn.schema, input_params)
    except ValidationError as e:
        return tool_result(tool_use_id, {
            "status": "error",
            "error_type": "validation",
            "message": f"Invalid input: {e.message}",
            "details": {"field": e.field, "expected": e.expected},
            "suggestion": f"Fix the '{e.field}' parameter: {e.expected}",
            "retryable": True
        }, is_error=True)
    
    try:
        result = tool_fn(**input_params)
        return tool_result(tool_use_id, {
            "status": "success",
            "data": result
        })
    except NotFoundException as e:
        return tool_result(tool_use_id, ToolError.not_found(e.resource, e.id), is_error=True)
    except PermissionError as e:
        return tool_result(tool_use_id, ToolError.permission_denied(e.required, e.current), is_error=True)
    except TimeoutError:
        return tool_result(tool_use_id, {
            "status": "error",
            "error_type": "timeout",
            "message": "Operation timed out after 30 seconds",
            "suggestion": "Try a more specific query or break into smaller operations",
            "retryable": True
        }, is_error=True)
    except Exception as e:
        # Catch-all: don't expose internals
        logger.error(f"Unexpected error in {name}: {e}")
        return tool_result(tool_use_id, {
            "status": "error",
            "error_type": "internal",
            "message": "An unexpected error occurred. The operation could not be completed.",
            "retryable": True
        }, is_error=True)
```

---

## 4. Security & Reliability

### What NOT to Include in Error Responses

| Don't Include | Why | Include Instead |
|---------------|-----|-----------------|
| Stack traces | Exposes internals | Generic error type |
| Database queries | SQL injection hints | "Query failed" |
| File system paths | Information disclosure | "Resource not found" |
| API keys/tokens | Credential leakage | "Authentication failed" |
| Internal IPs | Network mapping | "Service unavailable" |
| Library versions | Vulnerability targeting | "Internal error" |

### Error Response Security Checklist

```python
def sanitize_error(raw_error):
    """Ensure errors are safe to return to Claude/user."""
    
    # Remove sensitive patterns
    sensitive_patterns = [
        r'(?:password|secret|token|key)\s*[:=]\s*\S+',
        r'\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b',  # IPs
        r'/(?:home|var|etc|usr)/\S+',  # File paths
        r'(?:postgres|mysql|mongodb)://\S+',  # Connection strings
    ]
    
    message = raw_error.get("message", "")
    for pattern in sensitive_patterns:
        message = re.sub(pattern, "[REDACTED]", message)
    
    raw_error["message"] = message
    return raw_error
```

---

## 5. Exam Scenarios

### Scenario 1
**Q:** Claude calls `get_user(id="usr_999")` and the tool returns the string "Error: user not found". Claude then tells the user "The user usr_999 has an error status." What caused this misunderstanding?

**A:** The unstructured error string was interpreted as **data about the user** rather than a **tool failure**. Fix:
1. Return structured error: `{"status": "error", "error_type": "not_found", ...}`
2. Set `is_error: True` in the tool_result wrapper
3. With both signals, Claude correctly understands the tool failed and won't misinterpret the message as user data.

### Scenario 2
**Q:** A tool returns `{"status": "error", "error_type": "validation", "retryable": true}` but no message or suggestion. Is this sufficient?

**A:** **No.** Without `message` and `suggestion`, Claude knows it failed and can retry, but doesn't know:
- What was wrong (which parameter?)
- How to fix it (what's the correct format?)
- This leads to trial-and-error retries that waste tokens

Minimum effective error: type + message + suggestion.

### Scenario 3
**Q:** An agent receives a `rate_limit` error. What is the correct implementation pattern?

**A:**
```python
if error["error_type"] == "rate_limit":
    wait_time = error.get("retry_after", 5)
    time.sleep(wait_time)
    # Retry the same operation
```

Key: rate limits are **transient and retryable**. The error should include `retry_after` timing. The system should back off automatically without asking the user.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Plain string errors | Claude can't reason about them | Structured JSON with type |
| Missing `is_error` flag | Claude treats error as data | Always set `is_error: True` for failures |
| Error without suggestion | Claude retries blindly | Include actionable suggestion |
| Exposing raw exceptions | Security risk | Sanitized, categorized errors |
| Same error for all failures | No differentiation possible | Typed errors (not_found, validation, etc.) |
| Non-retryable marked retryable | Infinite retry loops | Correctly classify retryability |
| Error messages for Claude AND user | Mixing audiences | Separate internal error from user-facing message |

---

## 7. Quick Reference / Cheat Sheet

```
STRUCTURED ERROR TEMPLATE
━━━━━━━━━━━━━━━━━━━━━━━━━
{
  "status": "error",
  "error_type": "<type>",       ← Required: categorizes the error
  "message": "<explanation>",   ← Required: what went wrong
  "suggestion": "<next step>",  ← Required: what to try instead
  "retryable": true/false,      ← Required: can Claude retry?
  "details": {...}              ← Optional: structured metadata
}

ERROR TYPES → CLAUDE'S ACTION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
not_found    → Different query/ID
validation   → Fix input, retry
permission   → Tell user, stop
timeout      → Retry once, simplify
rate_limit   → Wait, retry
conflict     → Resolve or inform user
internal     → Retry once, then escalate

TOOL_RESULT WRAPPER
━━━━━━━━━━━━━━━━━━━
{
  "type": "tool_result",
  "tool_use_id": "<id>",
  "content": "<JSON string of error>",
  "is_error": true   ← CRITICAL: signals failure to Claude
}

SECURITY RULES
━━━━━━━━━━━━━━
✗ No stack traces
✗ No file paths
✗ No connection strings
✗ No API keys/tokens
✗ No internal IPs
✓ Sanitize all error messages before returning
```
