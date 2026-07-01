# Domain 2 | Topic 2.5: JSON Schema Design for Tools

## 1. Core Concepts

### Why JSON Schema for Tools?

Claude's tool inputs are validated against **JSON Schema** definitions. Well-designed schemas:
- Constrain inputs to valid values (preventing hallucination)
- Self-document expected formats
- Enable automatic validation before execution
- Reduce tool errors (less retry needed)

### JSON Schema Basics for Tool Definitions

```json
{
  "input_schema": {
    "type": "object",
    "properties": {
      "name": {
        "type": "string",
        "description": "User's full name",
        "minLength": 1,
        "maxLength": 100
      },
      "age": {
        "type": "integer",
        "minimum": 0,
        "maximum": 150
      },
      "role": {
        "type": "string",
        "enum": ["admin", "user", "viewer"]
      },
      "tags": {
        "type": "array",
        "items": {"type": "string"},
        "maxItems": 10
      }
    },
    "required": ["name", "role"]
  }
}
```

### Supported JSON Schema Types

| Type | Use For | Key Constraints |
|------|---------|----------------|
| `string` | Text, IDs, paths | `minLength`, `maxLength`, `pattern`, `enum` |
| `integer` | Counts, IDs | `minimum`, `maximum` |
| `number` | Decimals, scores | `minimum`, `maximum` |
| `boolean` | Flags, toggles | — |
| `array` | Lists, collections | `items`, `minItems`, `maxItems` |
| `object` | Nested structures | `properties`, `required` |
| `null` | Explicit null values | Often in `oneOf` |

### Key Schema Features

| Feature | Purpose | Example |
|---------|---------|---------|
| `enum` | Restrict to specific values | `"enum": ["asc", "desc"]` |
| `required` | Mark mandatory fields | `"required": ["query"]` |
| `default` | Suggest default value | `"default": 10` |
| `description` | Guide Claude on usage | `"description": "ISO 8601 date"` |
| `pattern` | Regex validation | `"pattern": "^[A-Z]{2}-\\d{4}$"` |
| `oneOf`/`anyOf` | Union types | Multiple valid formats |

---

## 2. Design Patterns

### Pattern 1: Comprehensive Property Descriptions

```json
{
  "name": "create_deployment",
  "input_schema": {
    "type": "object",
    "properties": {
      "service": {
        "type": "string",
        "description": "Service name as it appears in the service registry (e.g., 'auth-service', 'api-gateway')",
        "pattern": "^[a-z][a-z0-9-]*$"
      },
      "version": {
        "type": "string",
        "description": "Semantic version to deploy (e.g., '2.1.0'). Must be a valid tag in the repository.",
        "pattern": "^\\d+\\.\\d+\\.\\d+$"
      },
      "environment": {
        "type": "string",
        "enum": ["staging", "production"],
        "description": "Target environment. Use 'staging' for testing, 'production' for live deployment."
      },
      "rollback_on_failure": {
        "type": "boolean",
        "default": true,
        "description": "If true, automatically rolls back to previous version on health check failure"
      },
      "canary_percentage": {
        "type": "integer",
        "minimum": 0,
        "maximum": 100,
        "default": 10,
        "description": "Percentage of traffic routed to new version during canary deployment (0 = no canary)"
      }
    },
    "required": ["service", "version", "environment"]
  }
}
```

### Pattern 2: Nested Objects for Complex Inputs

```json
{
  "name": "configure_alert",
  "input_schema": {
    "type": "object",
    "properties": {
      "name": {"type": "string", "description": "Alert name"},
      "condition": {
        "type": "object",
        "description": "Alert trigger condition",
        "properties": {
          "metric": {"type": "string", "description": "Metric to monitor (e.g., 'cpu_usage', 'error_rate')"},
          "operator": {"type": "string", "enum": ["gt", "lt", "eq", "gte", "lte"]},
          "threshold": {"type": "number", "description": "Threshold value to trigger alert"},
          "duration_minutes": {"type": "integer", "minimum": 1, "default": 5, "description": "Minutes the condition must persist before alerting"}
        },
        "required": ["metric", "operator", "threshold"]
      },
      "notification": {
        "type": "object",
        "properties": {
          "channels": {
            "type": "array",
            "items": {"type": "string", "enum": ["email", "slack", "pagerduty"]},
            "minItems": 1
          },
          "severity": {"type": "string", "enum": ["info", "warning", "critical"]}
        },
        "required": ["channels", "severity"]
      }
    },
    "required": ["name", "condition", "notification"]
  }
}
```

### Pattern 3: Union Types with `oneOf`

```json
{
  "name": "search",
  "input_schema": {
    "type": "object",
    "properties": {
      "filter": {
        "description": "Search filter - either a simple text query or a structured filter",
        "oneOf": [
          {
            "type": "string",
            "description": "Simple text search query"
          },
          {
            "type": "object",
            "description": "Structured filter with field-specific criteria",
            "properties": {
              "field": {"type": "string"},
              "operator": {"type": "string", "enum": ["equals", "contains", "gt", "lt"]},
              "value": {"type": "string"}
            },
            "required": ["field", "operator", "value"]
          }
        ]
      }
    },
    "required": ["filter"]
  }
}
```

### Pattern 4: Minimal Required, Rich Optional

```json
{
  "name": "list_items",
  "input_schema": {
    "type": "object",
    "properties": {
      "collection": {
        "type": "string",
        "description": "Which collection to list (e.g., 'users', 'orders', 'products')"
      },
      "page": {"type": "integer", "minimum": 1, "default": 1},
      "page_size": {"type": "integer", "minimum": 1, "maximum": 100, "default": 20},
      "sort_by": {"type": "string", "default": "created_at"},
      "sort_order": {"type": "string", "enum": ["asc", "desc"], "default": "desc"},
      "filter_status": {"type": "string", "enum": ["active", "archived", "all"], "default": "active"}
    },
    "required": ["collection"]
  }
}
```

---

## 3. Implementation Details

### Schema Design Decisions

| Decision | Guidance |
|----------|----------|
| `required` vs optional | Only require what's truly mandatory for execution |
| `enum` vs free text | Use enum when options are finite and known |
| `pattern` regex | Use for IDs, versions, dates with specific formats |
| Nested objects | Use when properties naturally group together |
| `default` values | Include for all optional params with sensible defaults |
| Array constraints | Always set `maxItems` to prevent abuse |
| String length | Set `maxLength` for user-provided strings |

### How Claude Interacts with Schemas

```
1. Claude sees the schema in the tool definition
2. When calling the tool, Claude generates JSON matching the schema
3. Claude uses:
   - enum values (strongly adhered to)
   - descriptions (guides value selection)
   - required fields (always provided)
   - defaults (may omit if default is appropriate)
   - patterns (attempts to match, but may fail)
4. Your application validates the JSON before execution
```

### Validation Implementation

```python
import jsonschema

def validate_tool_input(tool_name, input_data, schema):
    """Validate Claude's tool input against the JSON schema."""
    try:
        jsonschema.validate(input_data, schema)
        return True, None
    except jsonschema.ValidationError as e:
        return False, {
            "status": "error",
            "error_type": "validation",
            "message": f"Invalid input for {tool_name}: {e.message}",
            "details": {
                "field": ".".join(str(p) for p in e.absolute_path),
                "constraint": e.validator,
                "expected": e.validator_value,
                "received": e.instance
            },
            "suggestion": f"Please fix the '{e.path[-1] if e.path else 'input'}' field"
        }
```

### Schema Complexity vs. Claude Accuracy

| Schema Complexity | Claude's Behavior |
|-------------------|-------------------|
| Simple (5 props, basic types) | Near-perfect adherence |
| Medium (10 props, nested) | Good, occasional errors on edge cases |
| Complex (15+ props, deep nesting, oneOf) | May hallucinate structure, needs good descriptions |
| Overly complex | Increased errors, consider splitting into multiple tools |

---

## 4. Security & Reliability

### Schema as Security Boundary

```json
// Prevent command injection via regex pattern
{
  "filename": {
    "type": "string",
    "pattern": "^[a-zA-Z0-9._-]+$",
    "maxLength": 255,
    "description": "Filename (alphanumeric, dots, hyphens, underscores only)"
  }
}

// Prevent path traversal
{
  "path": {
    "type": "string",
    "pattern": "^(?!.*\\.\\.)[a-zA-Z0-9/._-]+$",
    "description": "File path (no .. allowed)"
  }
}
```

### Input Validation Best Practices

| Validation | Purpose |
|-----------|---------|
| `maxLength` on all strings | Prevent memory/storage abuse |
| `maxItems` on all arrays | Prevent unbounded processing |
| `pattern` on IDs/paths | Prevent injection attacks |
| `enum` for known values | Eliminate invalid options entirely |
| `minimum`/`maximum` for numbers | Prevent overflow/underflow |
| Server-side re-validation | Schema is first line, not only line |

### What Schemas Can't Prevent

| Limitation | Example | Additional Defense |
|-----------|---------|-------------------|
| Semantic validity | Correct format but wrong user ID | Business logic validation |
| Authorization | Valid query for unauthorized data | Permission checks |
| Cross-field rules | start_date before end_date | Custom validation logic |
| Context-dependent rules | "Only on weekdays" | Runtime checks |

---

## 5. Exam Scenarios

### Scenario 1
**Q:** Claude keeps providing dates in "MM/DD/YYYY" format but your tool expects ISO 8601 ("YYYY-MM-DD"). What schema change fixes this MOST reliably?

**A:** Add both a `pattern` and a clear `description`:
```json
{
  "date": {
    "type": "string",
    "pattern": "^\\d{4}-\\d{2}-\\d{2}$",
    "description": "Date in ISO 8601 format: YYYY-MM-DD (e.g., '2024-03-15'). Do NOT use MM/DD/YYYY."
  }
}
```

The `pattern` provides hard validation; the `description` with an example guides Claude to generate the correct format. Both together give the highest success rate.

### Scenario 2
**Q:** A tool has 25 optional parameters. Claude often provides incorrect combinations. What is the BEST schema refactoring approach?

**A:** Split into multiple focused tools or use nested objects:

**Option A: Split tools**
- `create_basic_alert` (5 core params)
- `configure_alert_conditions` (condition params)
- `configure_alert_notifications` (notification params)

**Option B: Group with nested objects**
```json
{
  "properties": {
    "basic": {"type": "object", "properties": {...}, "required": [...]},
    "conditions": {"type": "object", "properties": {...}},
    "notifications": {"type": "object", "properties": {...}}
  },
  "required": ["basic"]
}
```

Grouping reduces cognitive load and error rates.

### Scenario 3
**Q:** Your tool accepts a `query` parameter that users could use for SQL injection if not validated. What schema-level and implementation-level defenses are appropriate?

**A:**
- **Schema level:** `"maxLength": 500` (limits complexity), `description` stating "Natural language query only, no SQL"
- **Implementation level:** Parameterized queries, query allowlist/parser, read-only database connection
- **Note:** Schema alone CANNOT prevent injection — it's a first filter, not a security boundary. Always validate server-side.

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| No `description` on properties | Claude guesses format | Always describe expected format + example |
| Free text where enum works | Claude invents invalid values | Use enum for known option sets |
| No `maxLength` / `maxItems` | Potential abuse/overflow | Always constrain sizes |
| Over-complex nested schemas | High error rate | Split into multiple simpler tools |
| Schema as only validation | Security bypass | Always re-validate server-side |
| No `required` specification | Claude may omit critical fields | Mark essential fields required |
| Using `additionalProperties: true` | Claude adds random fields | Omit or set to false |

---

## 7. Quick Reference / Cheat Sheet

```
SCHEMA STRUCTURE
━━━━━━━━━━━━━━━━
{
  "type": "object",
  "properties": {
    "<field>": {
      "type": "<type>",
      "description": "<when/how to fill>",
      "<constraints>": "<value>"
    }
  },
  "required": ["<mandatory_fields>"]
}

TYPE → CONSTRAINTS MAP
━━━━━━━━━━━━━━━━━━━━━━
string  → minLength, maxLength, pattern, enum
integer → minimum, maximum
number  → minimum, maximum
boolean → (none needed)
array   → items, minItems, maxItems
object  → properties, required

DESCRIPTION FORMULA
━━━━━━━━━━━━━━━━━━━
"[What this field is]. [Format: X]. (e.g., '[example]')"

ENUM USAGE
━━━━━━━━━━
✓ Use when: options are finite, known, stable
✗ Don't use when: values are dynamic or user-generated

SECURITY PATTERNS
━━━━━━━━━━━━━━━━━
• Path safety: "pattern": "^(?!.*\\.\\.)[a-zA-Z0-9/._-]+$"
• Filename safety: "pattern": "^[a-zA-Z0-9._-]+$"
• ID format: "pattern": "^[a-zA-Z0-9_-]{1,64}$"
• Always: maxLength, maxItems, server-side re-validation

COMPLEXITY GUIDELINES
━━━━━━━━━━━━━━━━━━━━━
• ≤5 properties: Simple, high accuracy
• 6-10 properties: Good with descriptions
• 11-15 properties: Consider grouping/nesting
• 15+ properties: Split into multiple tools
```
