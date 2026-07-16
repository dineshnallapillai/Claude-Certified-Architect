# Exercise 4: Design and Debug a Multi-Agent Research Pipeline

**Source:** Claude Certified Architect – Foundations Exam Guide v1.0, Section 8  
**Objective:** Practice orchestrating subagents, managing context passing, implementing error propagation, and handling synthesis with provenance tracking.  
**Domains Reinforced:** D1 (Agentic Architecture & Orchestration), D2 (Tool Design & MCP Integration), D5 (Context Management & Reliability)  
**Estimated Duration:** 3-4 hours  
**Difficulty:** High

---

## Overview

You will build a multi-agent research system where a coordinator agent delegates to specialized subagents for web search, document analysis, synthesis, and reporting. This exercise reinforces:

- Coordinator-subagent orchestration patterns
- Subagent context passing (no automatic inheritance)
- Parallel subagent execution
- Structured output with provenance (source attribution)
- Error propagation and recovery
- Handling conflicting information from credible sources

---

## Part 1: Design Agent Architecture

### Step 1.1: Agent Roles & Responsibilities

**Coordinator Agent:**
- Receives user research query
- Decomposes into subtasks
- Spawns and coordinates subagents
- Handles errors and partial results
- Synthesizes final report

**Web Search Subagent:**
- Receives search queries (from coordinator)
- Searches web for relevant articles
- Returns structured findings with URLs and publication dates

**Document Analysis Subagent:**
- Receives documents or URLs (from coordinator)
- Analyzes and extracts key claims
- Returns structured findings with source references

**Synthesis Subagent:**
- Receives all findings (from coordinator)
- Combines into cohesive report
- Preserves source attribution
- Handles conflicting information

### Step 1.2: Data Flow Architecture

```
┌─────────────────────┐
│  User Research      │
│  Query              │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────────┐
│  COORDINATOR AGENT                  │
│  ├─ Analyze query                   │
│  ├─ Decompose into subtasks         │
│  ├─ Determine which subagents needed│
│  └─ Orchestrate & synthesize        │
└──┬──────────────────┬──────────────┬┘
   │                  │              │
   ▼                  ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ WEB      │  │ DOCUMENT │  │SYNTHESIS │
│ SEARCH   │  │ ANALYSIS │  │ SUBAGENT │
│SUBAGENT  │  │SUBAGENT  │  │          │
└───┬──────┘  └────┬─────┘  └────┬─────┘
    │ [urls,  │ [claims,  │ [report
    │  dates] │  sources] │  structure]
    └─────────┴────┬──────┴──────┘
           Results │
                   │
           ┌───────▼────────┐
           │ COORDINATOR    │
           │ Aggregates &   │
           │ Routes to      │
           │ Synthesis      │
           └─────────┬──────┘
                     │
              ┌──────▼──────────┐
              │ Final Report    │
              │ with Attribution│
              └─────────────────┘
```

---

## Part 2: Implement Coordinator Agent

### Step 2.1: Agent Definition

```python
from anthropic.sdk.beta.agents import Agent, AgentDefinition

COORDINATOR_DEFINITION = AgentDefinition(
    name="research_coordinator",
    description="Coordinates research subagents to investigate topics comprehensively",
    instructions="""You are a research coordinator. Your goal is to investigate topics comprehensively by delegating to specialized subagents.

WORKFLOW:
1. Analyze the research query to understand scope
2. Determine which subagents you need (web search, document analysis, synthesis)
3. Decompose the topic into subtasks for each subagent
4. Invoke subagents in PARALLEL (emit multiple tool calls in one response)
5. Aggregate findings and route to synthesis
6. Present final report

KEY PRINCIPLES:
- Each subagent operates with isolated context; pass findings explicitly in prompts
- Subagents do NOT automatically inherit prior conversation history
- Ensure no duplication: assign distinct subtopics to search and analysis subagents
- For large topics, run multiple search/analysis passes with refinement

TASK TOOL USAGE:
You have access to the Task tool for spawning subagents. Each Task call spawns one subagent.
To run subagents in parallel, emit multiple Task calls in a single response.

Example:
Task 1: Spawn web search for Topic A
Task 2: Spawn web search for Topic B  
Task 3: Spawn document analysis on Topic A findings
[All run in parallel; you'll get all results before proceeding]

DECOMPOSITION STRATEGY:
For broad topics like "impact of AI on creative industries":
- Identify subtopics: visual arts, music, writing, film, design
- Assign subagents distinctly: don't have all search on "creative industries"
- Search specific subtopics: "AI impact on music production"
- Analysis specific angles: "AI in film generation vs film distribution"
- Synthesis combines across all subtopics

ERROR HANDLING:
If a subagent returns error_propagation (transient error):
- Try with modified query or alternative approach
- Include "attempted X, failed with Y" in context for refinement
If subagent returns permission error:
- Propagate to user; this is not your failure

SYNTHESIS ROUTING:
After collecting findings, route to synthesis subagent with:
- All findings (with source attribution preserved)
- Research goals (what was the question?)
- Coverage gaps (which topics have gaps?)
""",
    tools=["Task"]  # Can spawn subagents
)

coordinator = Agent(definition=COORDINATOR_DEFINITION)
```

### Step 2.2: Coordinator Prompt for Query Analysis

```python
def analyze_and_decompose(query: str):
    """Coordinator analyzes query and decides on subagent decomposition"""
    
    analysis_prompt = f"""
Research Query: {query}

Analyze this query and decide:
1. What subtopics or dimensions need investigation?
2. Which subagents should I invoke? (web_search, document_analysis, synthesis)
3. What specific queries should I send to each subagent?
4. Are there overlaps I should avoid?

Example decomposition for "Impact of AI on creative industries":
Subtopic 1: Visual arts (digital art, graphic design, photography)
  → Send to web search: "AI impact on digital art creation 2023-2024"
  
Subtopic 2: Music production
  → Send to web search: "AI in music production and composition"
  
Subtopic 3: Writing and content
  → Send to web search: "AI tools for writing and content generation"
  
Subtopic 4: Film and video
  → Send to web search: "AI in film generation and video production"

Subtopic 5: Cross-cutting analysis
  → Send to document analysis: [findings from searches]
  → Send to synthesis: "Synthesize findings on AI's creative industry impact"

Now analyze the actual query and create your decomposition plan.
"""
    
    return analysis_prompt
```

---

## Part 3: Implement Subagents

### Step 3.1: Web Search Subagent

```python
WEB_SEARCH_DEFINITION = AgentDefinition(
    name="research_web_search",
    description="Searches web for research findings on specific topics",
    instructions="""You are a web research agent. Your goal is to find relevant sources on the assigned topic.

TASK:
You will receive a specific research query from the coordinator.
Search the web for relevant articles, reports, and sources.
Return structured findings.

OUTPUT FORMAT:
For each finding, provide:
- Claim/Finding: What did you learn?
- Evidence: Quote or paraphrase from source
- Source URL: Where did this come from?
- Publication Date: When was this published?
- Relevance: How relevant to the research query (high/medium/low)?

GUIDELINES:
- Focus on credible sources: academic, news, industry reports
- Include publication dates (temporal context is important)
- Return 5-10 findings max (avoid information overload)
- If a source is paywalled, note it but still include URL
- For conflicting information, include both with attribution

IMPORTANT:
- You are NOT the final answer; coordinator will synthesize
- Include complete source information so coordinator can verify
- Be specific: avoid generic statements
- Include dates so temporal context is preserved
""",
    tools=["WebSearch", "WebFetch"]  # Can search and fetch
)
```

### Step 3.2: Document Analysis Subagent

```python
DOCUMENT_ANALYSIS_DEFINITION = AgentDefinition(
    name="research_document_analysis",
    description="Analyzes documents and papers for research findings",
    instructions="""You are a document analysis agent. Your goal is to extract key findings from documents.

TASK:
You will receive documents, URLs, or prior search findings from the coordinator.
Analyze and extract key claims, methodology, and conclusions.
Return structured findings.

OUTPUT FORMAT:
For each document/finding, provide:
- Document Title: Source name
- Key Claims: Main arguments or findings
- Evidence/Support: How are claims supported?
- Source Details: Author, publication date, type
- Limitations: What are known limitations?
- Relevance to Research Query: How does this fit?

GUIDELINES:
- Focus on claims explicitly stated (not inferred)
- Preserve original source characterizations
- Note methodology context (survey vs case study vs opinion)
- If sources conflict, note both with attribution
- Include publication/collection dates

IMPORTANT:
- You are NOT creating a synthesis; just analyzing each source
- Preserve full source information for later attribution
- Be thorough: this feeds the synthesis step
""",
    tools=["Read", "WebFetch"]  # Can read documents and fetch URLs
)
```

### Step 3.3: Synthesis Subagent

```python
SYNTHESIS_DEFINITION = AgentDefinition(
    name="research_synthesis",
    description="Synthesizes research findings into cohesive reports",
    instructions="""You are a research synthesis agent. Your goal is to combine findings into a comprehensive report.

INPUT:
You will receive:
- Research query (what were we investigating?)
- Web search findings (with source URLs and dates)
- Document analysis findings (with source details)
- Coverage gaps (which topics have gaps?)

OUTPUT:
Produce a comprehensive report that:
1. Answers the research query
2. Preserves source attribution (claims linked to sources)
3. Distinguishes well-established from contested findings
4. Handles conflicting information with attribution
5. Notes coverage gaps

REPORT STRUCTURE:
- Executive Summary (key findings, main takeaway)
- Findings by Topic (organized for clarity)
- Conflicting Information (where credible sources differ)
- Coverage Gaps (what couldn't we research?)
- Methodology (what sources and approaches were used?)
- Sources (full citations)

HANDLING CONFLICTS:
When credible sources conflict:
- Include both viewpoints with attribution
- Note why they might differ (different metrics, time periods, definitions)
- Don't arbitrarily pick one
- Example: "Source A reports X found Y in 2023, while Source B found Z in 2024 (different methodology)"

HANDLING GAPS:
Clearly note where we don't have complete information:
- Topic couldn't be researched (no sources found)
- Sources are outdated (needs recent data)
- Area requires specialist knowledge not available

IMPORTANCE OF PROVENANCE:
- Every claim in the report should be traceable to a source
- Use citations: "[Source: URL, Date]"
- Preserve original source characterizations
- Include publication/collection dates for temporal context

IMPORTANT:
- You are NOT writing an opinion piece; synthesize findings
- Preserve all source information
- Be comprehensive but organized
- Make clear what is well-supported vs contested vs speculative
""",
    tools=["Write"]  # Can write report
)
```

---

## Part 4: Implement Parallel Subagent Execution

### Step 4.1: Spawn Subagents in Parallel

```python
def spawn_subagents_parallel(
    search_queries: List[str],
    document_queries: List[str]
):
    """
    Spawn multiple subagents in parallel
    Emit multiple Task calls in one response to coordinator
    """
    
    # In actual coordinator implementation, this would be:
    task_calls = []
    
    # Spawn web search subagents (one per query)
    for i, query in enumerate(search_queries):
        task_calls.append({
            "type": "task",
            "subagent": "research_web_search",
            "prompt": f"""
Search the web for: {query}

Return findings with source attribution.
Include URLs and publication dates.
"""
        })
    
    # Spawn document analysis subagents (one per document set)
    for i, doc_query in enumerate(document_queries):
        task_calls.append({
            "type": "task",
            "subagent": "research_document_analysis",
            "prompt": f"""
Analyze documents on: {doc_query}

Extract key claims with source attribution.
Include document titles, authors, publication dates.
"""
        })
    
    # All emitted in single coordinator response = parallel execution
    return task_calls
```

### Step 4.2: Measure Parallel Execution

```python
import time

def measure_parallel_vs_sequential(research_query: str):
    """Compare parallel vs sequential execution latency"""
    
    # Sequential: Run one subagent, wait, run next
    start_seq = time.time()
    
    search_result_1 = spawn_subagent("research_web_search", "Query 1")
    search_result_2 = spawn_subagent("research_web_search", "Query 2")
    search_result_3 = spawn_subagent("research_web_search", "Query 3")
    
    seq_time = time.time() - start_seq
    
    # Parallel: Emit multiple task calls, wait for all
    start_par = time.time()
    
    results = spawn_subagents_parallel(
        ["Query 1", "Query 2", "Query 3"],
        []
    )
    
    par_time = time.time() - start_par
    
    # Parallel should be ~1/N of sequential time
    print(f"Sequential: {seq_time:.2f}s")
    print(f"Parallel: {par_time:.2f}s")
    print(f"Speedup: {seq_time/par_time:.1f}x")
```

---

## Part 5: Implement Error Propagation

### Step 5.1: Structured Error Context

```python
from typing import Optional

class SubagentError:
    def __init__(self, failure_type: str, attempted_query: str, 
                 partial_results: Optional[list] = None, 
                 alternative_approaches: Optional[list] = None):
        self.failure_type = failure_type  # transient/validation/permission/access
        self.attempted_query = attempted_query
        self.partial_results = partial_results or []
        self.alternative_approaches = alternative_approaches or []
    
    def to_dict(self):
        return {
            "error": True,
            "failure_type": self.failure_type,
            "attempted_query": self.attempted_query,
            "partial_results": self.partial_results,
            "alternative_approaches": self.alternative_approaches,
            "coordinator_action": self._recommended_action()
        }
    
    def _recommended_action(self) -> str:
        if self.failure_type == "transient":
            return "Retry with exponential backoff"
        elif self.failure_type == "validation":
            return "Invalid query format; modify and retry"
        elif self.failure_type == "permission":
            return "Escalate to user; resource access denied"
        elif self.failure_type == "access":
            return "Try alternative approach or source"
```

### Step 5.2: Subagent Error Handling

```python
def research_web_search_with_error_handling(query: str):
    """Web search with structured error propagation"""
    
    try:
        results = web_search(query)
        if not results:
            # Valid empty result (no matches found)
            return {
                "success": True,
                "findings": [],
                "message": f"No results found for: {query}"
            }
        return {"success": True, "findings": results}
    
    except TimeoutError:
        # Transient error
        error = SubagentError(
            failure_type="transient",
            attempted_query=query,
            partial_results=[],
            alternative_approaches=[
                "Retry with shorter query",
                "Try alternative search engine",
                "Break into smaller queries"
            ]
        )
        return error.to_dict()
    
    except PermissionError:
        # Permission error
        error = SubagentError(
            failure_type="permission",
            attempted_query=query,
            alternative_approaches=["Request elevated permissions"]
        )
        return error.to_dict()
    
    except Exception as e:
        # Transient error (assume)
        error = SubagentError(
            failure_type="transient",
            attempted_query=query,
            alternative_approaches=["Retry with modified query format"]
        )
        return error.to_dict()
```

### Step 5.3: Coordinator Error Recovery

```python
def coordinator_handle_subagent_error(error_dict: dict, context: dict):
    """Coordinator decides how to recover from subagent error"""
    
    failure_type = error_dict.get("failure_type")
    attempted_query = error_dict.get("attempted_query")
    alternatives = error_dict.get("alternative_approaches", [])
    partial_results = error_dict.get("partial_results", [])
    
    if failure_type == "transient":
        # Retry with alternative approach
        for approach in alternatives:
            print(f"Attempting: {approach}")
            # Retry logic here
    
    elif failure_type == "validation":
        # Modify query and retry
        modified_query = simplify_query(attempted_query)
        print(f"Retrying with: {modified_query}")
    
    elif failure_type == "permission":
        # Propagate to user
        print(f"Cannot access: {attempted_query}. Manual action needed.")
    
    elif failure_type == "access":
        # Proceed with partial results
        print(f"Proceeding with partial results: {partial_results}")
        return {"partial": True, "results": partial_results}
```

---

## Part 6: Provenance Tracking & Synthesis

### Step 6.1: Structured Claim-Source Mappings

```python
from dataclasses import dataclass
from typing import List

@dataclass
class Finding:
    claim: str
    evidence: str  # Quote or paraphrase
    source_url: str
    source_title: str
    publication_date: str
    author: Optional[str] = None
    credibility: str = "high"  # high/medium/low
    
    def to_dict(self):
        return {
            "claim": self.claim,
            "evidence": self.evidence,
            "source": {
                "url": self.source_url,
                "title": self.source_title,
                "date": self.publication_date,
                "author": self.author,
                "credibility": self.credibility
            }
        }

@dataclass
class ResearchReport:
    query: str
    findings: List[Finding]
    conflicting_findings: List[tuple]  # (Finding, Finding) pairs that conflict
    coverage_gaps: List[str]
    methodology: str
    
    def render(self) -> str:
        """Render report preserving provenance"""
        
        report = f"# Research Report: {self.query}\n\n"
        
        # Well-established findings
        report += "## Established Findings\n"
        for finding in self.findings:
            if finding.credibility == "high":
                report += f"\n- {finding.claim}\n"
                report += f"  Source: {finding.source_title} ({finding.publication_date})\n"
                report += f"  URL: {finding.source_url}\n"
        
        # Conflicting findings
        if self.conflicting_findings:
            report += "\n## Conflicting Information\n"
            for finding1, finding2 in self.conflicting_findings:
                report += f"\n- Source A ({finding1.publication_date}): {finding1.claim}\n"
                report += f"  URL: {finding1.source_url}\n"
                report += f"- Source B ({finding2.publication_date}): {finding2.claim}\n"
                report += f"  URL: {finding2.source_url}\n"
        
        # Coverage gaps
        if self.coverage_gaps:
            report += "\n## Coverage Gaps\n"
            for gap in self.coverage_gaps:
                report += f"- {gap}\n"
        
        return report
```

### Step 6.2: Handle Conflicting Sources

```python
def handle_conflicting_sources(findings: List[Finding]) -> tuple[List, List]:
    """Separate well-supported from conflicting findings"""
    
    established = []
    conflicting = []
    
    # Group findings by claim similarity
    claim_groups = {}
    for finding in findings:
        # Simple similarity: exact match or key phrase match
        key = finding.claim[:50]  # First 50 chars as similarity key
        if key not in claim_groups:
            claim_groups[key] = []
        claim_groups[key].append(finding)
    
    # Identify conflicts
    for key, group in claim_groups.items():
        if len(group) > 1:
            # Check if sources conflict
            if not all_sources_agree(group):
                conflicting.append(tuple(group[:2]))  # Add conflicting pair
            else:
                established.extend(group)  # Agreement = established
        else:
            established.extend(group)
    
    return established, conflicting

def all_sources_agree(findings: List[Finding]) -> bool:
    """Simple agreement check: do temporal contexts align?"""
    
    dates = [f.publication_date for f in findings]
    dates_recent = all(d >= "2023" for d in dates)
    
    # If all recent sources, more likely to agree
    if dates_recent:
        return True
    
    # If mix of old/new, likely disagreement
    return False
```

---

## Part 7: End-to-End Research Workflow

### Step 7.1: Complete Research Pipeline

```python
def run_research_investigation(query: str) -> ResearchReport:
    """End-to-end research pipeline"""
    
    print(f"Starting research on: {query}\n")
    
    # Step 1: Coordinator analyzes and decomposes
    coordinator = Agent(definition=COORDINATOR_DEFINITION)
    decomposition = coordinator.process(
        analyze_and_decompose(query)
    )
    print(f"Decomposition:\n{decomposition}\n")
    
    # Step 2: Spawn subagents in parallel
    search_queries = extract_search_queries(decomposition)
    doc_queries = extract_doc_queries(decomposition)
    
    print(f"Spawning {len(search_queries)} search + {len(doc_queries)} analysis subagents\n")
    
    search_results = spawn_subagents_parallel(search_queries, doc_queries)
    
    # Step 3: Handle any errors
    findings = []
    for result in search_results:
        if result.get("error"):
            print(f"Handling error: {result['failure_type']}")
            # Retry logic
            result = retry_subagent(result)
        findings.extend(result.get("findings", []))
    
    print(f"Collected {len(findings)} findings\n")
    
    # Step 4: Identify conflicts and gaps
    established, conflicting = handle_conflicting_sources(findings)
    gaps = identify_coverage_gaps(query, findings)
    
    # Step 5: Synthesis
    synthesis_agent = Agent(definition=SYNTHESIS_DEFINITION)
    report = synthesis_agent.process(
        f"""
Research Query: {query}

Findings ({len(findings)} total):
{format_findings(findings)}

Established Finding ({len(established)}):
{format_findings(established)}

Conflicting Information ({len(conflicting)} pairs):
{format_conflicts(conflicting)}

Coverage Gaps:
{format_gaps(gaps)}

Synthesize into a comprehensive report.
"""
    )
    
    return ResearchReport(
        query=query,
        findings=established,
        conflicting_findings=conflicting,
        coverage_gaps=gaps,
        methodology="Multi-agent research: web search + document analysis + synthesis"
    )
```

---

## Validation Checklist

### Coordinator Orchestration ✅
- [ ] Coordinator analyzes query and decomposes into subtasks
- [ ] Each subagent receives its findings directly in prompt
- [ ] Subagents don't automatically inherit prior context
- [ ] Coordinator decides which subagents to invoke dynamically

### Parallel Execution ✅
- [ ] Multiple Task calls emitted in single coordinator response
- [ ] All subagents run concurrently (not sequentially)
- [ ] Latency improvement measured (speedup ~1/N)

### Error Propagation ✅
- [ ] Subagent errors include structured context (type, query, alternatives)
- [ ] Transient errors trigger retry with modified approach
- [ ] Permission errors escalate to coordinator
- [ ] Access errors allow partial results to proceed

### Structured Output ✅
- [ ] Findings include source URL, publication date, credibility
- [ ] Claim-source mappings preserved through synthesis
- [ ] Subagent outputs structured (not prose)

### Provenance Tracking ✅
- [ ] Final report links every claim to source
- [ ] Conflicting sources both included with attribution
- [ ] Publication dates preserved for temporal context
- [ ] Coverage gaps explicitly noted

---

## 🎯 Success Criteria

Your pipeline is complete when:

1. **Coordinator** decomposes query, spawns appropriate subagents
2. **Parallel execution** runs multiple subagents concurrently
3. **Subagents** receive explicit context (no automatic inheritance)
4. **Error propagation** includes structured recovery information
5. **Synthesis** preserves source attribution throughout
6. **Conflicting sources** both included with clear attribution
7. **Coverage gaps** explicitly identified and reported
8. **End-to-end test** produces comprehensive research report with full provenance

---

## 📚 Related Study Materials

- D1-02: Multi-Agent Orchestration (Hub-and-Spoke)
- D1-03: Subagent Invocation, Context Passing
- D2-02: Structured Error Responses
- D5-03: Error Propagation Strategies
- D5-06: Information Provenance Tracking

---

**Prepared:** July 15, 2026  
**Difficulty:** High  
**Estimated Time:** 3-4 hours  
**Status:** Ready for implementation
