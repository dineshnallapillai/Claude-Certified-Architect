# Domain 5 | Topic 5.3: Information Provenance Tracking

## 1. Core Concepts

### What is Information Provenance?

**Information provenance** is tracking WHERE information came from, WHEN it was obtained, and HOW reliable it is. In agentic systems, Claude acts on information from multiple sources — knowing the source helps assess trustworthiness.

```
PROVENANCE CHAIN
━━━━━━━━━━━━━━━━
Source → Retrieval Method → Transformation → Current Use
  │           │                    │              │
  └ Trust     └ Freshness         └ Fidelity     └ Context
    level       guarantee            guarantee      appropriateness
```

### Why Provenance Matters

| Without Provenance | With Provenance |
|--------------------|-----------------|
| "The API supports pagination" | "The API docs (read 5 min ago) state pagination via ?page=N" |
| "Tests pass" | "npm test (run at 14:32) returned 0 failures" |
| "File doesn't exist" | "Glob for *.config.ts found no matches in /src" |
| "The user wants X" | "User stated in message #3: 'I want X'" |

### Trust Hierarchy

```
TRUST LEVELS (highest to lowest)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Direct observation (read file, ran test) — HIGH
2. User statement (user explicitly told us) — HIGH  
3. Tool result (tool returned data) — MEDIUM-HIGH
4. Inference (derived from other facts) — MEDIUM
5. External source (web, docs, API) — MEDIUM
6. Memory/cached info (from prior session) — LOW
7. Assumption (Claude's general knowledge) — LOWEST
```

---

## 2. Design Patterns

### Pattern 1: Source Attribution in Agent Responses

```python
# Good: Claude tracks provenance in reasoning
"""
Based on my analysis:
- The auth module uses JWT tokens (source: read src/auth/token.ts, line 15)
- Tokens expire after 24h (source: .env configuration, JWT_EXPIRY=86400)
- The refresh endpoint is at /api/refresh (source: src/routes/index.ts, line 42)
- No rate limiting exists on refresh (source: grep found no rate-limit middleware)
"""

# Bad: No provenance
"""
The auth module uses JWT tokens that expire after 24 hours.
You can refresh them at /api/refresh.
"""
```

### Pattern 2: Structured Provenance Metadata

```python
class Finding:
    """A fact with provenance metadata."""
    
    def __init__(self, fact, source, method, timestamp, confidence):
        self.fact = fact              # What was found
        self.source = source          # Where it came from
        self.method = method          # How it was obtained
        self.timestamp = timestamp    # When (for freshness)
        self.confidence = confidence  # How reliable (0-1)
    
    def is_stale(self, max_age_seconds=300):
        return (time.time() - self.timestamp) > max_age_seconds

# Example:
finding = Finding(
    fact="Users table has 50,000 rows",
    source="production_db via query_database tool",
    method="SELECT COUNT(*) FROM users",
    timestamp=time.time(),
    confidence=0.99  # Direct observation
)
```

### Pattern 3: Provenance-Aware Decision Making

```
DECISION FRAMEWORK:
━━━━━━━━━━━━━━━━━━
1. What do I know? (facts with sources)
2. How do I know it? (method of acquisition)
3. How fresh is it? (when was it obtained)
4. How reliable is it? (confidence level)
5. What am I assuming? (gaps without evidence)
6. Should I verify? (re-check stale or low-confidence info)
```

### Pattern 4: Tool Result Provenance

```python
def format_tool_result_with_provenance(tool_name, tool_input, result, timestamp):
    """Wrap tool results with provenance metadata."""
    return {
        "status": "success",
        "data": result,
        "provenance": {
            "tool": tool_name,
            "input": tool_input,
            "obtained_at": timestamp,
            "reliability": "high"  # Direct tool execution
        }
    }
```

---

## 3. Implementation Details

### When to Verify vs Trust

| Condition | Action |
|-----------|--------|
| Information from current session's tool call | Trust it |
| Information from user's latest message | Trust it |
| Information from memory (prior session) | Verify if acting on it |
| Information from external source (web) | Cross-reference if critical |
| Information derived by inference | State confidence level |
| Information older than 5 minutes (in active system) | Consider refreshing |

### Provenance in Multi-Agent Systems

```
ORCHESTRATOR must track:
━━━━━━━━━━━━━━━━━━━━━━━
• Which agent produced each finding
• What tools that agent used
• When the finding was produced
• Whether agents' findings conflict (same source, different conclusions?)

CONFLICT RESOLUTION:
━━━━━━━━━━━━━━━━━━━
• Direct observation > inference
• More recent > older
• Specialized agent > general agent
• Multiple confirmations > single source
```

### Expressing Uncertainty

```python
# Claude should communicate confidence levels:
CONFIDENCE_EXPRESSIONS = {
    "certain": "Based on direct observation (read file/ran test):",
    "high": "Based on tool results:",
    "medium": "Based on my analysis of the code:",
    "low": "I believe (but haven't verified):",
    "speculation": "I'm guessing, but this needs verification:"
}
```

---

## 4. Security & Reliability

### Provenance-Based Security

| Threat | Defense |
|--------|---------|
| Injected false information in tool results | Track source, verify unexpected claims |
| Stale cached data leading to wrong decisions | Timestamp everything, re-verify before critical actions |
| Conflicting sources (which to trust?) | Trust hierarchy + recency |
| Agent hallucinating provenance | Cross-check: does the cited source actually say that? |

### Reliability Through Provenance

1. **Distinguish facts from assumptions** — Claude should explicitly flag what it assumes vs. observed
2. **Re-verify before destructive actions** — re-read the file before deleting/modifying
3. **Track information age** — code changes fast; a file read 20 minutes ago may be stale
4. **Source diversity** — multiple sources confirming = higher confidence

---

## 5. Exam Scenarios

### Scenario 1
**Q:** An agent reads a config file, then 20 tool calls later modifies it based on its earlier reading. What provenance risk exists?

**A:** **Stale information risk** — the file may have changed between the read and the modification (especially in multi-user environments or if another agent modifies it).

Fix: Re-read the file immediately before modification to verify the state hasn't changed. This is analogous to an "optimistic lock" pattern in databases.

### Scenario 2
**Q:** Claude states "The API supports rate limiting at 1000 req/min" but no tool was used to verify this. How should this be communicated?

**A:** Claude should flag this as unverified:
- "Based on general knowledge (NOT verified against current codebase), the API may support rate limiting at 1000 req/min. Let me verify by checking the configuration."
- Then use a tool to confirm before acting on this assumption
- Never present assumptions as facts, especially for architectural decisions

### Scenario 3
**Q:** Two sub-agents return conflicting information about the same system. How should the orchestrator resolve this?

**A:** Apply provenance-based resolution:
1. Check which agent used **more direct observation** (tool calls vs inference)
2. Check which information is **more recent**
3. Check if one agent had **better access** to the relevant data
4. If still ambiguous: verify independently (orchestrator checks itself) or escalate to human

---

## 6. Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|-----------------|
| Stating facts without source | Can't assess reliability | Always attribute source |
| Acting on stale information | May be outdated | Re-verify before critical actions |
| Treating all sources equally | Some are more reliable | Trust hierarchy |
| No confidence signaling | User can't judge reliability | Express uncertainty levels |
| Assuming memory is current | Memory is a point-in-time snapshot | Verify against live state |
| Not tracking when info was obtained | Can't assess freshness | Timestamp all findings |

---

## 7. Quick Reference / Cheat Sheet

```
PROVENANCE = SOURCE + METHOD + TIME + CONFIDENCE

TRUST HIERARCHY
━━━━━━━━━━━━━━━
1. Direct observation (tool call this session)
2. User's explicit statement
3. Tool result from verified source
4. Logical inference from known facts
5. External source (docs, web)
6. Prior session memory
7. General knowledge (assumption)

VERIFICATION RULES
━━━━━━━━━━━━━━━━━━
• Before destructive actions: ALWAYS re-verify
• Memory claims: verify against current state
• Assumptions: flag explicitly, verify if critical
• Conflicting info: prefer more direct, more recent

COMMUNICATION PATTERNS
━━━━━━━━━━━━━━━━━━━━━━
Verified: "Based on [tool] at [time]: [fact]"
Inferred: "Based on my analysis: [inference]"
Uncertain: "I believe [claim] but haven't verified"
Assumed: "Assuming [X] (needs verification)"

CONFLICT RESOLUTION
━━━━━━━━━━━━━━━━━━━
Direct > Inferred
Recent > Old
Specialized > General
Multiple sources > Single source
Verified > Assumed
```
