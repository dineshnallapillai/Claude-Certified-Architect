# Claude Certified Architect — Foundations Study Plan

## Exam Overview

| Detail | Info |
|--------|------|
| **Certification** | Claude Certified Architect (Foundations) |
| **Provider** | Anthropic (claudecertifications.com) |
| **Format** | Multiple-choice, scenario-based |
| **Domains** | 5 core domains |
| **Preparation** | ~84 hours recommended |
| **Reference** | https://claudecertifications.com/claude-certified-architect/study-guide |

---

## 5 Exam Domains & Topics

### Domain 1: Agentic Architecture & Multi-Agent Systems
| # | Topic | Status |
|---|-------|--------|
| 1.1 | Agentic Loops & Core API Functionality | ⬜ |
| 1.2 | Multi-Agent Orchestration (Hub-and-Spoke) | ⬜ |
| 1.3 | Hooks, Workflows & Session Management | ⬜ |
| 1.4 | Tool Design Principles & MCP Integration | ⬜ |
| 1.5 | Agent Delegation & Task Decomposition | ⬜ |
| 1.6 | Error Handling in Agentic Systems | ⬜ |

### Domain 2: Tool Design & Integration
| # | Topic | Status |
|---|-------|--------|
| 2.1 | Tool Description Best Practices | ⬜ |
| 2.2 | Structured Error Responses | ⬜ |
| 2.3 | Tool Distribution & Scoping | ⬜ |
| 2.4 | MCP Server Configuration & Protocol | ⬜ |
| 2.5 | JSON Schema Design for Tools | ⬜ |
| 2.6 | Tool Chaining & Composition Patterns | ⬜ |

### Domain 3: Claude Code Configuration
| # | Topic | Status |
|---|-------|--------|
| 3.1 | CLAUDE.md Hierarchy & Management | ⬜ |
| 3.2 | Custom Slash Commands & Skills | ⬜ |
| 3.3 | Path-Specific Rule Implementation | ⬜ |
| 3.4 | Plan Mode vs Direct Execution | ⬜ |
| 3.5 | Permissions & Settings Configuration | ⬜ |
| 3.6 | Hooks & Automation | ⬜ |

### Domain 4: Prompt Engineering & Validation
| # | Topic | Status |
|---|-------|--------|
| 4.1 | Explicit Criteria Formulation | ⬜ |
| 4.2 | Few-Shot Prompting Techniques | ⬜ |
| 4.3 | Tool Use with JSON Schemas | ⬜ |
| 4.4 | Validation-Retry Loops | ⬜ |
| 4.5 | Batch Processing Patterns | ⬜ |
| 4.6 | System Prompts & Role Definition | ⬜ |

### Domain 5: Context Management & Reliability
| # | Topic | Status |
|---|-------|--------|
| 5.1 | Progressive Summarization & Compaction | ⬜ |
| 5.2 | Context Window Degradation Handling | ⬜ |
| 5.3 | Information Provenance Tracking | ⬜ |
| 5.4 | Escalation Patterns & Error Propagation | ⬜ |
| 5.5 | Human Review & Synthesis Strategies | ⬜ |
| 5.6 | Caching & Token Optimization | ⬜ |

---

## Study Approach

**Per-Topic Study Note Structure:**
1. **Core Concepts** — Key definitions, architecture diagrams, and mental models
2. **Design Patterns** — Best practices, recommended architectures, anti-patterns
3. **Implementation Details** — API calls, configuration, code examples
4. **Security & Reliability** — Safety considerations, failure modes, mitigations
5. **Exam Scenarios** — Situational questions with explained answers
6. **Anti-Patterns** — What NOT to do and why
7. **Quick Reference / Cheat Sheet** — Key facts for rapid review

---

## Progress Tracker & Updates

**Update Log (July 9, 2026):**
- ✅ D1-03: Added comprehensive Subagent Invocation section (Task tool, allowedTools, fork_session)
- ✅ D1-03: Added --resume CLI flag documentation with session resumption patterns
- ✅ D1-03: Added fork_session exploration pattern (divergent approaches)
- ✅ D1-03: Added Pattern 5 for coordinator-subagent orchestration
- ✅ D3-05: Added CLI Flags section (-p, --output-format json, --json-schema)
- ✅ D3-05: CI/CD integration examples with structured output
- ✅ D4-06: **COMPLETELY REWRITTEN** from "System Prompts" to "Multi-Instance & Multi-Pass Review"
- ✅ D4-06: Added independent instance architecture, multi-pass patterns, consensus review
- ✅ D4-06: Added cost-coverage tradeoff analysis and false positive management

**Current Status:**
- **Total Topics:** 30
- **Comprehensively Covered:** 28/30 (93% alignment with exam guide)
- **Remaining Minor Gaps:** 2 topics with thin coverage (see below)
- **Critical Gaps Resolved:** 0 (all addressed)

**Minor Gaps Remaining (Optional Enhancement):**
- D1-04: Handoff protocols could be expanded (currently in D1-03, could be dedicated section)
- D3-01: @import syntax documentation could be more detailed (basic coverage exists)

---

## File Naming Convention

```
claude-architect-cert/
├── 00-Study-Plan.md          (this file)
├── D1-01-Agentic-Loops.md
├── D1-02-Multi-Agent-Orchestration.md
├── D1-03-Hooks-Workflows-Sessions.md
├── ...
├── D5-06-Caching-Token-Optimization.md
└── Practice-Questions/
    ├── Domain-1-Questions.md
    ├── Domain-2-Questions.md
    └── ...
```
