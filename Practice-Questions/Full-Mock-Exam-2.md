# Claude Certified Architect — Full Mock Exam 2

**Time:** 90 minutes | **Questions:** 50 | **Passing:** 80%

---

## Questions

### Q1
A team is building a real-time chat assistant that displays Claude's responses token-by-token. During streaming, they need to detect when Claude begins generating a new text block so they can initialize a UI container. Which streaming event should they listen for to accomplish this?

- A) `message_start` — it fires once per content block and includes the block type
- B) `content_block_delta` — the first delta for a block implicitly signals its start
- C) `content_block_start` — it indicates a new content block is beginning and includes the block type and index
- D) `message_delta` — it provides incremental updates including new block metadata

### Q2
An engineering team is implementing tool use with streaming. Claude calls a `get_weather` tool, and they need to reconstruct the full JSON input from the stream. How are tool input arguments delivered during streaming?

- A) As a single complete JSON string in the `content_block_start` event
- B) As incremental string fragments via `input_json_delta` events within `content_block_delta`, which must be concatenated and parsed after `content_block_stop`
- C) As pre-parsed key-value pairs in each `content_block_delta` event
- D) As a complete JSON object in the `message_delta` event's `tool_use` field

### Q3
A production system streaming Claude responses encounters a network interruption mid-response. The stream terminates without a `message_stop` event. What is the correct architectural approach for handling this scenario?

- A) Treat whatever content was received as the complete response since partial responses are always valid
- B) Detect the absence of `message_stop`, mark the response as incomplete, and implement retry logic that resends the original request
- C) Wait indefinitely for the `message_stop` event since the connection may resume
- D) Parse the last `message_delta` event to extract the error code and display it to the user

### Q4
A developer wants to enable extended thinking for Claude to reason through a complex multi-step math problem. On an older model (e.g., Sonnet 4.5), they set `thinking: {type: "enabled", budget_tokens: 10000}` but receive an error. Which configuration mistake most likely caused this?

- A) The `budget_tokens` value exceeds the model's maximum allowed thinking budget
- B) They set `temperature` to 0.7 — extended thinking with `budget_tokens` requires `temperature` to be exactly 1
- C) They forgot to include `stream: true` since extended thinking only works with streaming
- D) They used the `messages` endpoint instead of the dedicated `thinking` endpoint

### Q5
An architect is budgeting costs for a system that uses extended thinking with a budget of 50,000 tokens. The model uses 30,000 thinking tokens and produces 2,000 output tokens in the response. How are the thinking tokens billed?

- A) Thinking tokens are free — only the visible output tokens (2,000) are billed as output
- B) Thinking tokens are billed as input tokens at the input token rate
- C) Thinking tokens are billed as output tokens — total output billing is 32,000 tokens (30,000 thinking + 2,000 response)
- D) Thinking tokens are billed at a special reduced "reasoning" rate that is 50% of the output token rate

### Q6
A platform team is building a usage dashboard to track API costs. After receiving a response, they inspect the `usage` object. Which set of fields does the usage object provide for a request that hit the prompt cache?

- A) `input_tokens`, `output_tokens`, `total_tokens`
- B) `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`
- C) `prompt_tokens`, `completion_tokens`, `cached_tokens`
- D) `input_tokens`, `output_tokens`, `cache_hit_ratio`, `cache_savings`

### Q7
Before sending a large prompt to Claude, an architect wants to verify it will fit within the context window without making a billable completions request. Which approach correctly accomplishes pre-flight token estimation?

- A) Use the `count_tokens` API endpoint, passing the same model and messages to get the exact token count without generating a response
- B) Divide the character count by 4 to approximate the token count, as Claude uses roughly 4 characters per token
- C) Send the request with `max_tokens: 1` and read the `input_tokens` field from the usage object
- D) Use a local tiktoken library since Claude uses the same tokenizer as GPT models

### Q8
A startup is designing a document processing pipeline. Simple documents need metadata extraction (labels, dates), complex documents need detailed summaries with citations, and a small percentage need deep multi-step analysis with research. Which multi-model architecture is most cost-effective?

- A) Use Opus for all tasks to ensure consistent quality across the pipeline
- B) Use Haiku for triage/classification, Sonnet for standard summarization, and Opus for complex multi-step analysis
- C) Use Sonnet for everything since it balances cost and quality
- D) Use Haiku for all tasks and retry with Sonnet only when Haiku returns an error

### Q9
A team is choosing between Claude models for a real-time customer support chatbot that handles 10,000 requests per hour. Most requests are straightforward FAQ-style questions, but roughly 5% require nuanced reasoning about company policies. What is the optimal model strategy?

- A) Deploy Opus for all requests to handle the 5% complex cases well
- B) Deploy Haiku for all requests since speed and cost matter most at this volume
- C) Use Haiku for initial response generation, with automatic escalation to Sonnet when the model's confidence is low or the query is classified as complex
- D) Deploy Sonnet for all requests as a single-model compromise

### Q10
An architect is designing a system where users upload images for analysis. They need to support both direct uploads and images referenced by URL. Which content block structure correctly provides an image to Claude via the Messages API?

- A) `{type: "image", data: "<base64>", format: "png"}` for uploads and `{type: "image", url: "https://..."}` for URLs
- B) `{type: "image", source: {type: "base64", media_type: "image/png", data: "<base64>"}}` for uploads and `{type: "image", source: {type: "url", url: "https://..."}}` for URLs
- C) `{type: "file", content: "<base64>", mime: "image/png"}` for both methods
- D) Images must be converted to text descriptions before sending; Claude cannot process raw image data via the API

### Q11
A legal tech company needs Claude to analyze 50-page PDF contracts and extract key clauses. Which approach correctly handles PDF document input in the Messages API?

- A) Convert each page to an image and send as a sequence of image content blocks
- B) Use a `{type: "document", source: {type: "base64", media_type: "application/pdf", data: "<base64>"}}` content block to send the PDF directly
- C) Extract text from the PDF client-side and send as a plain text content block
- D) Upload the PDF to a file hosting service and pass the URL in the system prompt

### Q12
A developer is making their first API call to Claude and receives a 401 Unauthorized error. Their request includes `Content-Type: application/json` and `anthropic-version: 2023-06-01`. What is most likely missing?

- A) The `Authorization: Bearer sk-...` header required for authentication
- B) The `x-api-key` header containing their Anthropic API key
- C) The `anthropic-model` header specifying which model to use
- D) The `Accept: application/json` header for response format negotiation

### Q13
A team wants to use a beta feature (e.g., token counting) that is not yet in the stable API. How do they opt into beta functionality?

- A) Set `anthropic-version` to the beta version date string (e.g., `2024-01-01-beta`)
- B) Include the `anthropic-beta` header with the specific beta feature flag (e.g., `token-counting-2024-11-01`)
- C) Append `/beta` to the API endpoint URL path
- D) Set `beta: true` in the request body JSON

### Q14
An architect is designing an MCP server that will be deployed behind a corporate load balancer. They previously used HTTP+SSE transport but learn it has been superseded. What is the recommended transport and why?

- A) WebSocket transport — it provides full-duplex communication without the limitations of SSE
- B) Streamable HTTP — it replaces HTTP+SSE by allowing both streaming and non-streaming responses over standard HTTP, improving compatibility with existing infrastructure
- C) gRPC — it provides efficient binary serialization and bidirectional streaming
- D) Long polling — it is the most compatible transport for corporate proxy environments

### Q15
During MCP initialization, a client connects to a server and they exchange capabilities. What is the purpose of this handshake, and what happens if a client requests a capability the server does not support?

- A) The server crashes with an unsupported capability error
- B) The handshake establishes which protocol features each side supports; unsupported capabilities are simply omitted from the server's response, and the client must not use them
- C) The client automatically downgrades to a minimal protocol version that both sides support
- D) The server acknowledges all requested capabilities regardless of support and silently ignores unsupported ones at runtime

### Q16
An MCP server is performing a long-running database migration and wants to keep the client informed of progress. Which MCP mechanism should the server use, and how does it work?

- A) Return partial results via multiple response messages until the operation completes
- B) Send progress notifications and log message notifications to the client — these are fire-and-forget messages that do not require a response
- C) Open a separate WebSocket connection dedicated to progress updates
- D) Include a `progress_url` in the initial response that the client polls for status

### Q17
An MCP server exposes a `delete_user` tool and a `get_user_profile` tool. The server developer wants to help clients present appropriate confirmation UIs. Which tool annotations should they apply?

- A) `delete_user`: `{readOnlyHint: false, destructiveHint: true}` and `get_user_profile`: `{readOnlyHint: true, destructiveHint: false}`
- B) `delete_user`: `{requiresConfirmation: true}` and `get_user_profile`: `{requiresConfirmation: false}`
- C) `delete_user`: `{dangerLevel: "high"}` and `get_user_profile`: `{dangerLevel: "none"}`
- D) Both tools should use `{idempotentHint: true}` since they can be retried safely

### Q18
A client application receives an MCP tool with annotations `{readOnlyHint: true, openWorldHint: true}`. How should the client use these annotations to improve the user experience?

- A) The client must enforce these as access control rules and block any write operations
- B) The client can use these as UX hints — for example, auto-approving read-only tools without user confirmation, and indicating the tool accesses external resources beyond the server's control
- C) The client should ignore annotations entirely since they are only for server-side logic
- D) The client must display a security warning because `openWorldHint: true` means the tool is potentially unsafe

### Q19
An architect is building an agent that must always call a `validate_input` tool before processing any user request. How should they configure `tool_choice` to guarantee Claude calls this specific tool?

- A) `tool_choice: {type: "auto"}` with instructions in the system prompt to always call validate_input first
- B) `tool_choice: {type: "any"}` which forces Claude to call at least one tool
- C) `tool_choice: {type: "tool", name: "validate_input"}` which forces Claude to call that specific tool
- D) `tool_choice: {type: "required", tools: ["validate_input"]}` with a list of required tools

### Q20
A developer has an agent with three tools: `search_database`, `call_api`, and `format_response`. The tools have dependencies — `format_response` requires output from either of the other two. Claude keeps calling `search_database` and `format_response` in parallel, causing errors. What is the correct fix?

- A) Remove `format_response` from the tool list and handle formatting in post-processing
- B) Set `disable_parallel_tool_use: true` in the `tool_choice` object to force sequential tool calls
- C) Add `depends_on: ["search_database", "call_api"]` to the format_response tool definition
- D) Set `max_concurrent_tools: 1` in the request body

### Q21
An architect is implementing prompt caching for a system that sends a 3,000-token system prompt with every request. They add `cache_control: {type: "ephemeral"}` to the system message. After the first request, subsequent requests within 5 minutes show `cache_read_input_tokens: 3000`. What pricing impact does this have?

- A) Cached reads cost the same as regular input tokens — the benefit is only reduced latency
- B) Cached reads cost 90% less than regular input tokens, but the initial cache write costs 25% more than regular input tokens
- C) Cached reads are completely free, but the cache creation doubles the cost of those tokens
- D) Cached reads cost 50% less than regular input tokens with no write premium

### Q22
A developer adds a `cache_control` breakpoint to a 500-token block of content, but cache reads never appear in the usage response. What is the most likely cause?

- A) The API key does not have caching permissions enabled
- B) The cached content does not meet the minimum token threshold — cached blocks must be at least 1,024 tokens (Sonnet 4.5/4.1/4/3.7), 2,048 tokens (Fable 5/Sonnet 4.6/Haiku 3.5/3), or 4,096 tokens (Opus 4.8/4.7/4.6/4.5 & Haiku 4.5)
- C) The `anthropic-version` header does not support caching
- D) Cache breakpoints only work on system messages, not user messages

### Q23
An architect is optimizing a multi-turn conversation system for caching. The system has: a static system prompt (2,000 tokens), few-shot examples (3,000 tokens), tool definitions (1,500 tokens), and dynamic user messages. How should they order content for maximum cache efficiency?

- A) Put dynamic user messages first, then static content, to keep the conversation feeling natural
- B) Place static system prompt first with a cache breakpoint, then few-shot examples with a cache breakpoint, then tool definitions, then dynamic messages last — static content first maximizes prefix cache hits
- C) Interleave static and dynamic content to distribute cache breakpoints evenly
- D) Place tool definitions first since they change least frequently, followed by user messages, then the system prompt

### Q24
A Claude Code hook script needs to inspect which tool is about to be invoked and conditionally block dangerous operations. How does the hook receive information about the tool call?

- A) Via environment variables: `$TOOL_NAME` and `$TOOL_INPUT` are set before the hook runs
- B) Via command-line arguments: the tool name is passed as `$1` and input as `$2`
- C) Via stdin as a JSON object containing fields like `tool_name` and `tool_input`, which the script must parse
- D) Via a temporary file whose path is stored in the `$HOOK_DATA_FILE` environment variable

### Q25
A team writes a Claude Code hook that should block any file deletion outside the project directory. The hook reads stdin JSON and checks `tool_input`. Which implementation approach is correct?

- A) Parse the stdin JSON, extract the `tool_name` field, check if it is a file operation, then inspect `tool_input` for paths outside the project root — exit with non-zero status to block
- B) Read the `$TOOL_NAME` environment variable and use a shell `case` statement to match dangerous commands
- C) Write the hook in the `hooks.json` file inline as a JSON expression that pattern-matches tool names
- D) Set the hook to `type: "passive"` so it monitors but cannot block tool execution

---

## Answer Key (Q1-Q25)

**Q1: C**
The `content_block_start` event fires when Claude begins a new content block (text, tool_use, etc.) and includes the block's type and index. This is the correct signal to initialize a UI container for incoming content.

**Q2: B**
Tool input arguments arrive as incremental JSON string fragments in `input_json_delta` fields within `content_block_delta` events. The client must concatenate all fragments and parse the complete JSON only after receiving `content_block_stop` for that block.

**Q3: B**
The absence of a `message_stop` event indicates an incomplete response. The correct approach is to detect this condition (e.g., via timeout or connection close), mark the response as incomplete, and implement retry logic to resend the request.

**Q4: B**
On older models using `budget_tokens`, extended thinking requires the temperature parameter to be set to exactly 1. Any other temperature value causes an error. Note: On newer models (Opus 4.6+ with adaptive thinking), the `budget_tokens` approach is deprecated or removed — use `thinking: {type: "adaptive"}` with `output_config: {effort: ...}` instead. On Opus 4.7/4.8 and Fable 5, `temperature` is removed entirely.

**Q5: C**
Thinking tokens are billed as output tokens. The total output token billing includes both the thinking tokens (30,000) and the visible response tokens (2,000), for a total of 32,000 output tokens billed.

**Q6: B**
The usage object includes `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, and `cache_read_input_tokens`. The cache-specific fields appear when prompt caching is active, allowing precise cost tracking.

**Q7: A**
The `count_tokens` API endpoint accepts the same model and messages parameters as a completions request but returns only the token count without generating a response or incurring output token charges.

**Q8: B**
A multi-model architecture uses Haiku for fast, cheap triage and classification; Sonnet for standard processing tasks like summarization; and Opus for complex multi-step reasoning. This optimizes both cost and quality by matching model capability to task complexity.

**Q9: C**
Using Haiku for the majority of simple requests keeps costs low and latency minimal, while escalating complex queries to Sonnet ensures quality where it matters. This is more cost-effective than using a single powerful model for all traffic.

**Q10: B**
Image content blocks use `{type: "image", source: {...}}` with the source object specifying either `type: "base64"` with `media_type` and `data` fields, or `type: "url"` with a `url` field. This is the correct Messages API structure.

**Q11: B**
The Messages API supports PDF documents directly via a document content block with `type: "document"`, `source.type: "base64"`, `media_type: "application/pdf"`, and the base64-encoded PDF data. This preserves document structure and formatting.

**Q12: B**
Anthropic's API uses the `x-api-key` header for authentication, not the `Authorization: Bearer` pattern. A missing or invalid `x-api-key` header results in a 401 Unauthorized response.

**Q13: B**
Beta features are enabled by including the `anthropic-beta` header with the appropriate feature flag string. Multiple beta features can be enabled by comma-separating them in the header value.

**Q14: B**
Streamable HTTP is the successor to HTTP+SSE transport in MCP. It uses standard HTTP POST requests that can return either a direct JSON response or upgrade to a stream, making it more compatible with existing infrastructure like load balancers and proxies.

**Q15: B**
The initialization handshake lets client and server declare their supported capabilities. If the client requests a capability the server does not support, the server simply omits it from its response. The client must respect this and not use unsupported features.

**Q16: B**
MCP supports progress notifications and log message notifications that servers can send during long-running operations. These are one-way notifications (fire-and-forget) that inform the client of status without requiring a response.

**Q17: A**
Tool annotations use `readOnlyHint`, `destructiveHint`, `idempotentHint`, and `openWorldHint`. A delete operation should be marked `destructiveHint: true` (and `readOnlyHint: false`), while a read operation should be marked `readOnlyHint: true` to help clients show appropriate confirmation dialogs.

**Q18: B**
Tool annotations are advisory hints for client UX, not enforced access control. A client can auto-approve `readOnlyHint: true` tools without user confirmation, and use `openWorldHint` to inform users that the tool interacts with external systems beyond the server's direct control.

**Q19: C**
Setting `tool_choice: {type: "tool", name: "validate_input"}` forces Claude to call that specific tool. This guarantees the tool is invoked, unlike `auto` (which allows Claude to skip tools) or `any` (which forces a tool call but not a specific one).

**Q20: B**
Setting `disable_parallel_tool_use: true` within the `tool_choice` configuration forces Claude to make only one tool call per turn. This ensures sequential execution, preventing `format_response` from being called before its dependencies complete.

**Q21: B**
Prompt caching offers a 90% discount on cached reads compared to standard input token pricing. However, the initial cache write incurs a 25% premium over standard input pricing. After the first request, subsequent cache hits provide significant savings.

**Q22: B**
Cached content must meet minimum token thresholds to be eligible for caching: 1,024 tokens (Sonnet 4.5/4.1/4/3.7), 2,048 tokens (Fable 5/Sonnet 4.6/Haiku 3.5/3), or 4,096 tokens (Opus 4.8/4.7/4.6/4.5 & Haiku 4.5). A 500-token block falls below all thresholds and will never be cached.

**Q23: B**
Cache efficiency depends on prefix matching — content is cached from the beginning of the prompt up to cache breakpoints. Static content should be placed first (system prompt, few-shot examples) with breakpoints, and dynamic content last, so the static prefix remains cacheable across requests.

**Q24: C**
Claude Code hooks receive context via stdin as a JSON object. The script must read from stdin and parse the JSON to extract fields like `tool_name` and `tool_input`. Environment variables and command-line arguments are not used for this purpose.

**Q25: A**
The correct approach is to parse stdin JSON, extract `tool_name` to identify file operations, inspect `tool_input` for file paths, check if any path is outside the project root, and exit with a non-zero status code to block the operation. Non-zero exit signals the hook rejection.
### Q26. 
Your CI pipeline needs Claude to analyze pull requests and output structured results that another script will parse. The analysis should run without human interaction and terminate after processing. Which combination of flags achieves this?

A) `claude --print --output-format json`  
B) `claude --non-interactive --format structured`  
C) `claude --headless --output-format json --no-prompt`  
D) `claude --batch --structured-output --silent`  

---

### Q27. 
A developer left a Claude Code session yesterday to attend a meeting and now wants to pick up exactly where they left off in their last conversation. Which flag should they use?

A) `claude --resume` with the session ID from their terminal history  
B) `claude --continue` to automatically resume the most recent conversation  
C) `claude --restore-session` to reload the last active context  
D) `claude --reconnect` which finds and attaches to the previous session  

---

### Q28. 
You are building an automated code review bot that uses Claude Code in a Docker container. The bot must run completely unattended, but you need to prevent infinite loops where Claude keeps trying to fix failing tests. Which flags should you combine?

A) `--dangerously-skip-permissions --max-turns 10 --print`  
B) `--no-permissions --timeout 300 --non-interactive`  
C) `--auto-approve --loop-limit 10 --headless`  
D) `--skip-auth --max-iterations 10 --batch`  

---

### Q29. 
A junior architect asks how to switch Claude Code into plan mode during an interactive session so they can explore a design without Claude making any file changes. What is the correct method?

A) Type `/plan` in the input prompt to toggle plan mode on  
B) Press Shift+Tab to toggle plan mode before submitting the prompt  
C) Use the `--read-only` flag when starting Claude Code  
D) Press Ctrl+P to enter planning mode in the current session  

---

### Q30. 
Your team wants to generate an implementation plan using Claude Code in a CI environment and save it for architect review before any code is written. Which approach correctly uses plan mode in headless operation?

A) `claude --print --plan "Design the authentication module" > plan.md`  
B) `claude --plan --output-format text -p "Design the authentication module" > plan.md`  
C) `claude --print --mode=plan "Design the authentication module" > plan.md`  
D) `claude --headless --planning-only "Design the authentication module" > plan.md`  

---

### Q31. 
Your organization has the following permission configuration:

- Enterprise policy (`~/.claude/policies.json`): `deny: ["Bash(rm *)"]`  
- Project settings (`.claude/settings.json`): `allow: ["Bash(rm -rf node_modules)"]`  

A developer runs `claude` in the project and Claude attempts to execute `rm -rf node_modules`. What happens?

A) The command executes because the project-level allow overrides the enterprise deny  
B) The command is blocked because deny rules are cumulative and cannot be overridden by allow at any level  
C) The user is prompted to confirm because the rules conflict  
D) The command executes but is logged as a policy violation  

---

### Q32. 
An architect is designing a permissions hierarchy for a large monorepo. They want developers to use Bash freely for build commands but never execute database migration scripts directly. The settings are defined at three levels: enterprise, user, and project. Which statement about the merge behavior is correct?

A) Allow rules merge with union semantics, and the most specific level (project) takes precedence for conflicts  
B) Deny rules are cumulative across all levels; a deny at ANY level blocks the action regardless of allow rules elsewhere  
C) Project-level settings completely replace user-level settings when both are present  
D) Enterprise settings only apply when no project-level settings file exists  

---

### Q33. 
A team lead wants to ensure that Claude always uses their company's specific error handling pattern when writing TypeScript code across all projects. Where should this instruction be persisted?

A) In memory using `/memory` since it persists across all sessions and projects  
B) In the project's `CLAUDE.md` file since it is a code pattern specific to how code should be written  
C) In a custom command file since it needs to be invoked on demand  
D) In the user's `~/.claude/CLAUDE.md` since it applies across all of that user's projects  

---

### Q34. 
A developer has been using Claude's memory feature extensively. Which of the following should NOT be stored in Claude's memory system?

A) Preferred communication style (e.g., "be concise, skip explanations unless asked")  
B) The project's complete module dependency graph and architecture decisions  
C) Tool preferences (e.g., "use pnpm instead of npm")  
D) Personal context (e.g., "I'm working on the payments team")  

---

### Q35. 
A platform team maintains a shared set of coding standards and API guidelines that multiple project teams should follow. They want to distribute these instructions without requiring each team to copy-paste large blocks of text into their own `CLAUDE.md`. What feature supports this?

A) Use `#include ../shared/standards.md` in each project's CLAUDE.md  
B) Use `#import ../shared/standards.md` in each project's CLAUDE.md  
C) Use `@reference ../shared/standards.md` in each project's CLAUDE.md  
D) Use `$source ../shared/standards.md` in each project's CLAUDE.md  

---

### Q36. 
A monorepo has this structure:
```
/repo/CLAUDE.md
/repo/services/auth/CLAUDE.md
/repo/services/payments/CLAUDE.md
```

A developer starts Claude Code from `/repo` and asks Claude to modify a file at `/repo/services/auth/handlers/login.ts`. Which CLAUDE.md files are active during this operation?

A) Only `/repo/CLAUDE.md` because that is where the session started  
B) Only `/repo/services/auth/CLAUDE.md` because it is closest to the target file  
C) Both `/repo/CLAUDE.md` and `/repo/services/auth/CLAUDE.md` because subdirectory CLAUDE.md files activate when Claude operates on files in that subtree  
D) All three CLAUDE.md files because the entire repository's instructions are always loaded  

---

### Q37. 
A team creates the following custom command file structure:
```
.claude/commands/db/migrate.md
.claude/commands/db/seed.md
.claude/commands/deploy/staging.md
```

How does a developer invoke the migration command?

A) `/db:migrate`  
B) `/project:db:migrate`  
C) `/commands:db:migrate`  
D) `/custom:db/migrate`  

---

### Q38. 
An architect is evaluating which built-in skills in Claude Code could help their team. They want a skill that will check a current working diff for correctness issues and potential simplifications. Which built-in skill category addresses this?

A) `/review` - which reviews GitHub pull requests from remote  
B) `/code-review` - which reviews the current working diff for bugs and cleanups  
C) `/verify` - which runs the app to confirm changes work  
D) `/simplify` - which only handles reuse and efficiency, not correctness  

---

### Q39. 
Your company needs to process 80,000 customer support tickets through Claude for classification and routing. The results are needed within 24 hours and cost optimization is critical. Which approach is most appropriate?

A) Use the streaming API with concurrent connections to maximize throughput  
B) Use the Messages Batches API which supports up to 100K items with a 50% cost discount and 24-hour SLA  
C) Deploy multiple Claude instances behind a load balancer for parallel processing  
D) Use prompt caching with the real-time API to reduce per-request costs by 50%  

---

### Q40. 
You submitted a batch of 5,000 classification requests. After the batch completes, you find that 4,850 items show `succeeded`, 100 show `errored`, and 50 show `expired`. What is the correct interpretation and response?

A) Resubmit the entire batch because partial results indicate a systemic issue  
B) The errored items had malformed requests and expired items were not processed within the 24-hour window; retry only the errored and expired items in a new batch  
C) Expired items were rate-limited and will automatically be retried by the system within 48 hours  
D) The errored items exceeded token limits; expired items had network timeouts and both need manual intervention  

---

### Q41. 
An architect is deciding between real-time API calls and batch processing for three different workloads. Which workload is the WORST fit for the Batches API?

A) Generating nightly summary reports for 10,000 daily transactions  
B) A chatbot that must respond to customer messages within 2 seconds  
C) Classifying 50,000 uploaded documents for compliance tagging  
D) Generating personalized email drafts for a marketing campaign launching tomorrow  

---

### Q42. 
During a long Claude Code session, the context window is approaching capacity. Claude triggers progressive summarization. Which type of information is MOST likely to survive compaction intact?

A) The exact sequence of bash commands run in the first 20 minutes of the session  
B) System prompt instructions and CLAUDE.md content  
C) Detailed file contents that were read early in the conversation  
D) Intermediate reasoning steps from exploratory code searches  

---

### Q43. 
An architect is designing a complex prompt that includes: (1) system instructions, (2) a large reference document, (3) the critical user query, and (4) examples. Based on the "lost-in-the-middle" phenomenon, what is the optimal positioning strategy?

A) Place the critical query and instructions at the beginning; put the reference document at the end  
B) Place all content in chronological order since LLMs process sequentially  
C) Place the critical query at the end (most recent), system instructions at the beginning, and put the large reference document in the middle where recall is lowest for less critical details  
D) Randomize the order since modern LLMs have uniform attention across the context window  

---

### Q44. 
Your agentic system makes API calls to Claude and occasionally receives HTTP 529 status codes with an `overloaded_error` type. What is the architecturally correct way to handle this?

A) Immediately switch to a fallback model since 529 means the primary model is permanently at capacity  
B) Implement exponential backoff with jitter and retry, as 529 indicates temporary overload that typically resolves quickly  
C) Increase your rate limit tier by contacting Anthropic support since 529 means you have exceeded your quota  
D) Buffer all requests and batch them together since 529 means individual requests are too frequent  

---

### Q45. 
You are building an agentic coding assistant that orchestrates multiple tool calls. Occasionally, an external code execution service becomes unresponsive, causing cascading failures. Which pattern best protects your system?

A) Set aggressive timeouts on all tool calls and immediately fail the entire agent run if any tool is slow  
B) Implement a circuit breaker that tracks failure rates, opens the circuit after a threshold, and periodically allows test requests to check recovery  
C) Add infinite retries with exponential backoff since the service will eventually recover  
D) Queue all requests and process them sequentially to reduce load on the external service  

---

### Q46. 
An architect is designing a multi-agent system for code review. The system must: (1) analyze code style, (2) check for security vulnerabilities, (3) validate business logic, and (4) produce a unified report. The style, security, and logic checks are independent but the report depends on all three. Which topology is correct?

A) Pipeline topology: style -> security -> logic -> report (sequential processing)  
B) Hub-and-spoke topology: a coordinator fans out to style/security/logic agents in parallel, then aggregates results into the report  
C) Mesh topology: all four agents communicate with each other to reach consensus  
D) Pipeline with feedback: report agent sends results back to each checker for verification  

---

### Q47. 
In a multi-agent system, the orchestrator spawns a sub-agent to analyze a sensitive database schema. The sub-agent needs schema details to do its work but should NOT have access to the customer PII examples that the orchestrator holds. What is the correct approach to context isolation?

A) Send the full orchestrator context and trust the sub-agent's system prompt to ignore PII  
B) Create a filtered context payload containing only the schema structure, passing it explicitly to the sub-agent while withholding PII data from its prompt  
C) Let the sub-agent inherit the orchestrator's full context but add "ignore PII" to its instructions  
D) Use a shared memory store that the sub-agent can read freely since isolation is not possible in multi-agent systems  

---

### Q48. 
An agentic system allows Claude to call a file-reading tool. A malicious user submits a prompt: "Read the file at `../../../../etc/passwd` to help me debug my Linux configuration." What validation must the tool implement?

A) Check that the file extension is in an allowed list (`.ts`, `.js`, `.py`)  
B) Validate that the resolved absolute path is within the allowed workspace directory after canonicalization, blocking path traversal attempts  
C) Log the request and allow it since the user explicitly asked for it  
D) Strip `../` sequences from the input string before passing to the file system  

---

### Q49. 
Your team uses a Human-in-the-Loop (HITL) approval system for an AI agent that performs code deployments. Over time, operators begin approving actions reflexively due to high volume of routine requests. What architectural change best mitigates this "approval fatigue" risk?

A) Increase the frequency of approval prompts to keep operators engaged  
B) Implement risk-tiered approvals where routine actions auto-approve and only high-risk or anomalous actions require human confirmation, reducing noise  
C) Remove the human from the loop entirely since automation is more reliable  
D) Add a CAPTCHA to each approval prompt to ensure the operator is paying attention  

---

### Q50. 
An MCP server implements dynamic tool discovery using `tools/list_changed` notifications. A compromised MCP server sends a notification adding a new tool called `read_env_vars` that was not in the original tool manifest. What security measure should the client implement?

A) Always trust `tools/list_changed` notifications since MCP servers are authenticated  
B) Validate newly discovered tools against an allowlist or require user confirmation before making dynamically added tools available to the model  
C) Ignore all `tools/list_changed` notifications and only use the initial tool list  
D) Automatically call the new tool once to verify it works before adding it to the available set  

---

## Answer Key (Q26-Q50)

**Q26. A**  
`--print` runs Claude non-interactively (single response, no conversation loop) and `--output-format json` produces machine-parseable structured output. This combination is purpose-built for CI/scripting pipelines.

**Q27. B**  
`--continue` automatically resumes the most recent conversation without needing a session ID. `--resume` requires a specific session ID and is used when you want to return to a particular past session that may not be the most recent one.

**Q28. A**  
`--dangerously-skip-permissions` bypasses all permission prompts (required for unattended CI), `--max-turns` limits how many agentic loop iterations Claude can take (preventing infinite fix cycles), and `--print` ensures non-interactive execution. This is the correct combination for controlled autonomous operation.

**Q29. B**  
Plan mode is toggled with Shift+Tab during an interactive session. There is no `/plan` slash command. Plan mode instructs Claude to reason about and outline an approach without making any file modifications.

**Q30. B**  
The `--plan` flag enables plan mode in headless/non-interactive operation. Combined with `--output-format text` and `-p` (for providing the prompt inline), this produces a planning-only output suitable for architect review. The output can be redirected to a file.

**Q31. B**  
Deny rules in Claude Code's permission system are cumulative and absolute. A deny at any level (enterprise, user, or project) blocks the action regardless of allow rules at other levels. This is a security-first design ensuring that higher-authority restrictions cannot be circumvented.

**Q32. B**  
The permission system follows a cumulative deny model. Allow rules may merge across levels, but a deny rule at ANY level acts as an absolute block that cannot be overridden by allow rules at another level. This ensures security policies set by administrators cannot be bypassed by project-level configurations.

**Q33. B**  
Code patterns and style guidelines belong in `CLAUDE.md` at the project level because they are project-specific instructions that should apply to all developers working in that codebase. Memory is for personal preferences and context; custom commands are for on-demand workflows.

**Q34. B**  
Complete architecture documentation and module dependency graphs should NOT be stored in memory. Memory is designed for brief personal preferences and context, not large technical documentation. Architecture decisions belong in CLAUDE.md or project documentation where they can be version-controlled and shared with the team.

**Q35. B**  
The `#import` directive in CLAUDE.md allows importing instructions from other files, enabling modular instruction sets. This lets platform teams maintain a single source of truth for shared standards that multiple projects can reference without duplication.

**Q36. C**  
Subdirectory CLAUDE.md files activate when Claude operates on files within that subtree. Since the target file is under `/repo/services/auth/`, both the root CLAUDE.md and the auth-specific CLAUDE.md are active. The payments CLAUDE.md is not activated because the operation does not involve files in that subtree.

**Q37. B**  
Custom commands in the `.claude/commands/` directory use colon-separated naming with a `project:` prefix. Subdirectory nesting maps to colon separators, so `.claude/commands/db/migrate.md` becomes `/project:db:migrate`.

**Q38. B**  
`/code-review` is the built-in skill that reviews the current working diff for correctness bugs and simplification opportunities. `/review` is for GitHub PRs (remote), `/verify` runs the app to observe behavior, and `/simplify` focuses only on reuse/efficiency without bug detection.

**Q39. B**  
The Messages Batches API is specifically designed for high-volume, latency-tolerant workloads. It supports up to 100,000 items per batch, offers a 50% cost discount compared to real-time pricing, and guarantees completion within 24 hours, making it ideal for this use case.

**Q40. B**  
In batch processing, `succeeded` means the request was processed normally, `errored` indicates the individual request was malformed or invalid, and `expired` means it was not processed within the 24-hour SLA window. The correct response is to retry only the failed and expired items in a new batch.

**Q41. B**  
The Batches API has a 24-hour SLA and provides no latency guarantees for individual items. A chatbot requiring 2-second response times is fundamentally incompatible with batch processing. The other workloads (nightly reports, document classification, next-day email campaign) are all latency-tolerant.

**Q42. B**  
System prompt instructions and CLAUDE.md content are treated as high-priority context that survives summarization. Progressive summarization compresses conversation history (early commands, intermediate reasoning, verbose file contents) but preserves the foundational instructions that guide Claude's behavior.

**Q43. C**  
The "lost-in-the-middle" phenomenon shows that LLMs have strongest recall at the beginning and end of context, with weaker recall for middle content. Critical instructions go at the start, the active user query goes at the end (recency benefit), and large reference material can occupy the middle where lower recall is acceptable.

**Q44. B**  
HTTP 529 with `overloaded_error` indicates a temporary capacity issue on Anthropic's infrastructure. The correct pattern is exponential backoff with jitter to avoid thundering herd effects. This is not a quota issue (which would be 429) and does not indicate permanent unavailability.

**Q45. B**  
The circuit breaker pattern prevents cascading failures by tracking error rates and "opening" the circuit (stopping requests) when a failure threshold is reached. It periodically allows "test" requests through to detect recovery. This protects the system from wasting resources on a known-failing service while enabling automatic recovery.

**Q46. B**  
Hub-and-spoke (fan-out/fan-in) is correct because the three analysis tasks are independent and can run in parallel, while the report depends on all their outputs. A pipeline would force unnecessary sequential execution. The coordinator fans out work, collects results, then produces the unified report.

**Q47. B**  
Context isolation in multi-agent systems requires explicit control over what information passes between agents. The correct approach is to construct a filtered payload containing only the necessary schema structure, ensuring PII never enters the sub-agent's context. System prompt instructions alone cannot guarantee isolation.

**Q48. B**  
Path traversal prevention requires canonicalizing the resolved path (resolving `../` sequences, symlinks, etc.) and then validating that the final absolute path falls within the allowed workspace directory. Simply stripping `../` is insufficient as there are many encoding bypasses. Extension checking doesn't prevent traversal.

**Q49. B**  
Risk-tiered approvals address approval fatigue by reducing the volume of routine prompts (which cause operators to become habituated) while preserving human oversight for genuinely dangerous actions. This maintains security for high-risk operations while acknowledging that constant low-risk prompts degrade human vigilance.

**Q50. B**  
Dynamic tool discovery via `tools/list_changed` introduces a security risk where compromised servers can inject malicious tools. The client must validate new tools against an allowlist or require explicit user confirmation before exposing them to the model. Blindly trusting notifications or auto-executing unknown tools creates attack vectors.
