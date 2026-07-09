# Claude Certified Architect — Full Mock Exam 3

**Time:** 90 minutes | **Questions:** 50 | **Passing:** 80%

---

## Questions

### Q1 [D1]
Your agentic workflow processes customer support tickets. After Claude generates a response, you receive `stop_reason: "end_turn"` in the API response. A junior developer asks what this means and whether the agent should continue looping. What is the correct interpretation?

A) The model hit the `max_tokens` limit and was cut off mid-response — you should increase the token budget and retry  
B) The model voluntarily finished its response and has nothing more to say — the agent loop can terminate unless there are pending tool results  
C) The model encountered a tool_use block and is waiting for you to execute the tool and return results  
D) The context window overflowed and the response was truncated — you should compact the conversation and retry  

---

### Q2 [D1]
You are building a complex reasoning agent that must show its chain-of-thought for compliance auditing. You enable extended thinking with `budget_tokens: 10000` on an older model. During testing, you notice the API rejects your request. What is the most likely cause?

A) You set `temperature` to 0.7 — extended thinking with `budget_tokens` requires `temperature` to be exactly 1  
B) You set the thinking budget below the minimum of 1024 tokens  
C) You included a `system` prompt, which is incompatible with extended thinking  
D) You are using a model that does not support extended thinking, such as Haiku 3.5  

---

### Q3 [D1]
Your enterprise compliance system uses Claude with extended thinking enabled. In production, you occasionally receive thinking blocks with `type: "redacted"` instead of the actual thinking content. Your logging pipeline crashes when it encounters these. What is the correct architectural response?

A) Switch to a model that never produces redacted thinking blocks  
B) Design your system to gracefully handle redacted thinking blocks as a normal occurrence — they appear when internal safety classifiers trigger, and the model's final response remains usable  
C) Retry the request immediately, as redacted thinking blocks indicate a transient server error  
D) Disable extended thinking entirely, since redacted blocks mean the model cannot reason about your use case  

---

### Q4 [D1]
You are designing a multi-turn agent loop using the Anthropic SDK. The agent needs to call multiple tools, process results, and continue reasoning. Which pattern correctly describes the SDK's expected flow?

A) Send the initial request, receive a response with `stop_reason: "tool_use"`, extract tool_use blocks, execute tools, append the assistant message and tool results to the conversation, then send another request — repeating until `stop_reason: "end_turn"`  
B) Send all possible tool calls upfront in the system prompt, and the SDK automatically executes them in sequence without further API calls  
C) Use a webhook endpoint that Claude calls directly when it needs tool results, eliminating the need for a loop  
D) Send the initial request and poll a status endpoint until the model finishes all tool calls server-side  

---

### Q5 [D1]
Your team is selecting a model for a coding agent that generates and iterates on complex software architectures. The agent typically runs 15-30 tool-use turns per session. Cost is a concern but quality of architectural decisions is paramount. Which model selection rationale is most appropriate?

A) Use Haiku 3.5 for all turns — it is the fastest and cheapest, and quality differences are negligible for code generation  
B) Use Opus 4 for all turns — its superior sustained performance over long agentic tasks justifies the higher cost for complex architectural work  
C) Use Sonnet 4 for the initial planning turn, then switch to Haiku 3.5 for all subsequent tool-use turns to save cost  
D) Use Opus 4 only for the final summary turn and Haiku 3.5 for everything else, since the final output is what matters most  

---

### Q6 [D1]
You need to estimate costs for a batch processing pipeline before submitting requests. The pipeline processes documents of varying length. Which approach correctly uses the Count Tokens API?

A) Send the full messages array to the `/v1/messages/count_tokens` endpoint to get the exact input token count, including any system prompt and tool definitions, before deciding whether to submit the request  
B) Divide the character count by 4 to estimate tokens — the Count Tokens endpoint is only for output token estimation  
C) Use the Count Tokens endpoint after the request completes to verify billing — it cannot be called before sending a message  
D) The Count Tokens endpoint only works with the Batch API and cannot be used for individual message requests  

---

### Q7 [D1]
Your agent receives `stop_reason: "max_tokens"` on a response that was generating a detailed analysis. What is the correct architectural interpretation and recovery strategy?

A) The model chose to stop at this point — treat the response as complete  
B) The response was truncated because it hit the `max_tokens` output limit — you should either increase `max_tokens` or append the partial response to the conversation and ask the model to continue  
C) The context window is full — you must start a new conversation  
D) A rate limit was hit — implement exponential backoff and retry the identical request  

---

### Q8 [D1]
You are building a routing layer that classifies incoming requests and dispatches them to specialized sub-agents. For the classification step (which produces a single label from a fixed set), what is the optimal model and configuration choice?

A) Opus 4 with extended thinking enabled — classification requires deep reasoning  
B) Haiku 3.5 with low `max_tokens` — classification is a simple task where latency and cost matter more than deep reasoning  
C) Sonnet 4 with `temperature` set to 0 — you need deterministic outputs for routing  
D) Any model with `stop_sequences` set to the classification labels — this forces exact outputs  

---

### Q9 [D1]
Your extended thinking configuration uses `budget_tokens: 5000` within a request that has `max_tokens: 8000`. A colleague asks how billing works for the thinking tokens. What is correct?

A) Thinking tokens are free and only the visible output tokens count toward billing  
B) Thinking tokens are billed at the input token rate since they are internal processing  
C) Thinking tokens are billed at the output token rate — both thinking and visible response tokens count against `max_tokens` and are billed as output  
D) Thinking tokens are billed at a special discounted rate that is 50% of the output token rate  

---

### Q10 [D1]
Your multi-agent system has an orchestrator that delegates to specialist agents. During execution, a specialist agent returns a `stop_reason: "stop_sequence"` response. What does this indicate and how should the orchestrator handle it?

A) The model encountered an error and stopped — retry the request  
B) The model's output contained one of the custom `stop_sequences` you specified in the request — the orchestrator should parse the output up to that point and take the appropriate next action based on which stop sequence was hit  
C) The model ran out of context window space and was forced to stop  
D) The model detected harmful content in its own output and self-censored — escalate to a human reviewer  

---

### Q11 [D2]
Your MCP server exposes tools that change dynamically based on user permissions. After the initial connection, a user's role is upgraded and new tools become available. How should your server communicate this to the client?

A) Close the connection and force the client to reconnect and re-discover tools  
B) Send a `notifications/tools/list_changed` notification, which signals the client to call `tools/list` again to get the updated tool set  
C) Push the new tool definitions directly to the client via a `tools/update` message containing the full tool schemas  
D) Wait for the client to poll `tools/list` on its next request — MCP has no push mechanism for tool changes  

---

### Q12 [D2]
You are designing an MCP server that provides access to a knowledge base. Some content is semi-structured reference material that the LLM should have available as context (not something it calls like a function). Which MCP primitive is most appropriate?

A) Expose the content as Tools that return the reference material when called  
B) Expose the content as Resources — these are data/content primitives that can be loaded into context, distinct from callable tools  
C) Embed all reference material in the MCP server's initialization response  
D) Use Prompts to encode the reference material as pre-written templates  

---

### Q13 [D2]
Your MCP server provides a `delete_customer` tool. You want to ensure that MCP clients properly present this as a destructive, irreversible operation that requires confirmation. Which tool annotation combination is correct?

A) `{"readOnlyHint": false, "destructiveHint": true, "idempotentHint": false}` — marks it as a write operation that is destructive and not safely repeatable  
B) `{"destructiveHint": true, "idempotentHint": true}` — destructive but idempotent since deleting twice has the same effect  
C) `{"readOnlyHint": true, "destructiveHint": true}` — read-only but destructive to signal it returns data about what would be deleted  
D) `{"destructiveHint": false, "idempotentHint": false}` — hide the destructive nature so the LLM does not avoid calling it  

---

### Q14 [D2]
A developer on your team defines a tool's input schema but omits `additionalProperties: false` from the JSON Schema object. During testing, Claude sometimes sends extra fields not defined in the schema. What is the architectural impact?

A) The extra fields are harmless — JSON Schema ignores undefined properties by default and the tool should just ignore them  
B) Omitting `additionalProperties: false` means the model may hallucinate extra parameters, and your tool handler may receive unexpected input — always include this property to constrain the model to only the defined parameters  
C) The API rejects any tool definition that does not include `additionalProperties: false`  
D) The extra fields cause a JSON parsing error on the client side since strict mode is always enabled  

---

### Q15 [D2]
Your tool returns a 50KB JSON response containing a full database query result. You notice that subsequent model responses degrade in quality as this large result consumes context. What is the best truncation strategy?

A) Let the model handle it — Claude automatically summarizes large tool results internally  
B) Before returning the result, truncate it server-side to the most relevant subset (e.g., top 10 rows with a note indicating total count), keeping the tool result concise and preserving context space for reasoning  
C) Split the 50KB result across multiple tool_result messages in sequence  
D) Compress the JSON using gzip encoding within the tool_result content block  

---

### Q16 [D2]
You are implementing an MCP client that connects to a third-party MCP server. During the connection lifecycle, what is the correct sequence of operations?

A) Send `initialize` request, wait for server's `initialize` response with capabilities, then send `initialized` notification — only then can you begin normal operations like `tools/list`  
B) Send `tools/list` immediately upon TCP connection — the server auto-initializes on first request  
C) Send `initialized` notification first, then `initialize` request, then begin normal operations  
D) Send a `handshake` request containing your client capabilities, then wait for the server to send `ready`  

---

### Q17 [D2]
Your team is designing an MCP server that provides prompt templates for common workflows. A template called "code-review" accepts a `language` parameter and returns a structured system prompt plus user message. Which MCP primitive are you using?

A) Tools — define a `generate_prompt` tool that returns the formatted prompt  
B) Resources — expose each template as a URI-addressable resource  
C) Prompts — these are the MCP primitive specifically designed for parameterized prompt templates that return structured message arrays  
D) Sampling — use the server's built-in LLM sampling to generate prompts dynamically  

---

### Q18 [D2]
An MCP tool annotation specifies `{"openWorldHint": true}`. What does this communicate to the client about the tool's behavior?

A) The tool accesses external/open-world resources (like the internet) beyond the local environment — clients may apply additional confirmation or sandboxing policies  
B) The tool accepts any input format without schema validation  
C) The tool is available to all users without authentication  
D) The tool can discover and call other tools dynamically  

---

### Q19 [D2]
Your MCP server crashes and restarts during an active session. The client detects the disconnection. What is the correct client-side recovery behavior per the MCP specification?

A) Resume sending requests immediately — the server maintains session state across restarts  
B) Re-establish the connection and repeat the full initialization handshake (`initialize` → `initialized`) before resuming operations  
C) Wait 30 seconds for the server's internal recovery mechanism to restore the session  
D) Notify the user and terminate — MCP does not support reconnection  

---

### Q20 [D2]
You define a tool with the annotation `{"idempotentHint": true, "readOnlyHint": false}`. A client is implementing retry logic for failed tool calls. How should this annotation combination influence the client's behavior?

A) The client should never retry — `readOnlyHint: false` means retries could cause side effects  
B) The client can safely retry the operation on failure because `idempotentHint: true` indicates that calling it multiple times has the same effect as calling it once, even though it performs writes  
C) The client should retry only once — idempotent operations are safe to retry but only a single time  
D) These annotations contradict each other and the client should reject the tool definition  

---

### Q21 [D3]
Your monorepo has a root `CLAUDE.md` and a subdirectory `packages/api/CLAUDE.md`. The subdirectory file contains `@../../shared/conventions.md`. A developer asks what happens when Claude Code is invoked from within `packages/api/`. What is correct?

A) Only `packages/api/CLAUDE.md` is loaded — subdirectory files override root files completely  
B) Both files are loaded (root provides base context, subdirectory adds specifics), and the `@path` import pulls in the contents of `shared/conventions.md` as if it were inlined at that point  
C) The `@path` import syntax is not a real feature — Claude Code only reads literal CLAUDE.md file content  
D) The `@path` import syntax triggers a git submodule fetch of the referenced file  

---

### Q22 [D3]
Your CI pipeline needs to run Claude Code non-interactively to generate PR descriptions. The pipeline must not prompt for any permissions. Which command configuration achieves this safely?

A) `claude --dangerously-skip-permissions -p "Generate a PR description for the current diff"` — this flag bypasses all permission prompts, enabling fully headless CI operation  
B) `claude --yes -p "Generate a PR description"` — the `--yes` flag auto-approves all prompts  
C) `claude --no-permissions -p "Generate a PR description"` — this disables the permission system  
D) `claude -p "Generate a PR description"` — print mode is automatically headless and never prompts  

---

### Q23 [D3]
You want to build a CI integration that parses Claude Code's output programmatically in real-time as it streams. Which output format should you use?

A) `--output-format json` — returns a single JSON object after completion  
B) `--output-format stream-json` — emits newline-delimited JSON objects as they are generated, allowing real-time parsing of assistant messages, tool calls, and results  
C) `--output-format xml` — returns structured XML that can be parsed with SAX parsers  
D) `--output-format text` — pipe the plain text output through a custom regex parser  

---

### Q24 [D3]
You are configuring Claude Code for a long-running refactoring task that should stop after processing 20 files (one file per tool call). Which mechanism correctly limits the agent's execution?

A) `--max-budget-usd 10.00` — this caps the dollar spend for the invocation, preventing unbounded execution  
B) `--max-tokens 20` — this limits output to 20 tokens, forcing the agent to stop  
C) Set `max_iterations: 20` in CLAUDE.md — the agent reads this as a runtime constraint  
D) `--timeout 20` — this stops execution after 20 seconds  

---

### Q25 [D3]
Your company has an enterprise-managed deployment of Claude Code. An individual developer sets `allow` permissions in their local `.claude/settings.json` that conflict with the enterprise policy's `deny` list. What happens?

A) The developer's local settings take precedence — enterprise policies are only defaults  
B) The enterprise policy override wins — deny rules from enterprise policy cannot be overridden by local user or project settings  
C) The system throws a configuration conflict error and refuses to start  
D) The settings are merged with union semantics — both allow and deny rules coexist, creating an inconsistent state  

---

### Q26 [D3]
A developer asks whether they should put their personal coding style preferences in CLAUDE.md (checked into the repo) or in Claude Code's memory system. What is the correct guidance?

A) Put everything in CLAUDE.md — the memory system is deprecated  
B) CLAUDE.md is for project-level, team-shared instructions (coding standards, architecture decisions, build commands) while the memory system stores personal, user-specific preferences that should not be imposed on the whole team  
C) The memory system is for facts Claude has learned; CLAUDE.md is for personality configuration only  
D) They are identical in function — use whichever is more convenient  

---

### Q27 [D3]
You want to configure a "stop hook" that runs a linting check every time Claude Code finishes a coding task, before presenting results to the user. How are stop hooks configured?

A) Add a `PostComplete` section in CLAUDE.md with shell commands  
B) Configure the hook in `.claude/settings.json` under the hooks system with the `Stop` event matcher — it runs a specified command when the agent finishes its response  
C) Stop hooks are triggered by adding `--on-stop "lint.sh"` to the CLI invocation  
D) Place an executable file named `.claude-stop-hook` in the project root — Claude Code auto-detects it  

---

### Q28 [D3]
Your CI pipeline uses `claude --dangerously-skip-permissions` for automated code review. A security engineer raises concerns. What is the primary risk this flag introduces?

A) It disables all model safety filters, allowing harmful content generation  
B) It allows Claude Code to execute any tool (file writes, shell commands, network access) without human confirmation — if the model is manipulated via prompt injection in the codebase, it could execute arbitrary commands  
C) It sends all code to third-party servers without encryption  
D) It disables rate limiting, potentially causing excessive API charges  

---

### Q29 [D3]
You are debugging why Claude Code ignores instructions in your CLAUDE.md file. The file is located at `./CLAUDE.md` in the project root and contains build instructions. What is the most likely issue?

A) CLAUDE.md must be named in all-caps with a `.txt` extension  
B) The file exceeds the maximum allowed size and is silently truncated — check if the file has grown too large, or if it uses `@path` imports that pull in excessive content  
C) CLAUDE.md is only read in interactive mode — your headless invocation skips it  
D) The file must be listed in `.claude/settings.json` to be activated  

---

### Q30 [D3]
You configure `--max-budget-usd 2.00` for a Claude Code session. The agent uses $1.50 on tool calls and produces a partial response. What happens if the task is not yet complete by the agent's own assessment?

A) The agent ignores the limit and continues until the task is done  
B) The agent stops at or before the budget limit — if it reaches $2.00 in spend, execution halts regardless of task completion, and the user receives whatever output has been produced  
C) The agent enters a "summary mode" and produces a compressed final answer  
D) The system automatically doubles the budget limit when it detects incomplete work  

---

### Q31 [D4]
You are implementing prompt caching for a conversational assistant using Claude Sonnet. Your system prompt is 800 tokens. You add `cache_control: {"type": "ephemeral"}` to it but notice caching never activates. What is the issue?

A) The `"ephemeral"` type is deprecated — use `"persistent"` instead  
B) The cached content must meet a minimum token threshold to be cached — the minimum varies by model (e.g., 1024 tokens for Sonnet 4.5, 2048 for Sonnet 4.6, 4096 for Opus), and your 800-token system prompt falls below all thresholds  
C) Prompt caching only works with the Batch API, not real-time requests  
D) You must enable caching in your account settings before using `cache_control`  

---

### Q32 [D4]
You are configuring extended thinking for a complex multi-step reasoning task. Your request has `max_tokens: 16000` and you want to allocate thinking budget. What is the minimum valid `budget_tokens` value, and what happens if thinking tokens exceed the budget?

A) Minimum is 512 tokens; if exceeded, thinking is truncated and the model proceeds with partial reasoning  
B) Minimum is 1024 tokens; the model will use up to the budget for thinking, and both thinking and response tokens together must fit within `max_tokens`  
C) Minimum is 2048 tokens; thinking tokens are unlimited and do not count against `max_tokens`  
D) Minimum is 4096 tokens; if exceeded, the request fails with an error  

---

### Q33 [D4]
You have a multi-turn conversation with a large system prompt, followed by tool definitions, and then user/assistant turns. You want to maximize cache hits. Where should you place `cache_control` breakpoints?

A) Only on the very last user message — caching works backward from the end  
B) On the system prompt and at the end of the tool definitions — place breakpoints at boundaries of stable content so that varying user messages do not invalidate the cached prefix  
C) On every single message in the conversation — more breakpoints means more caching  
D) Only on assistant messages — system and user messages cannot be cached  

---

### Q34 [D4]
Your tool-calling agent sometimes calls multiple tools in parallel when you need strictly sequential execution (each tool depends on the previous result). How do you enforce sequential tool use?

A) Set `tool_choice: {"type": "auto", "disable_parallel_tool_use": true}` — this forces the model to make only one tool call per response, ensuring you can process results sequentially  
B) Include "DO NOT USE PARALLEL TOOLS" in the system prompt — this is the only mechanism available  
C) Set `max_tokens: 1` to force the model to output only one tool call  
D) Remove all tools except one from the request, rotating which tool is available each turn  

---

### Q35 [D4]
You submitted 1,000 requests to the Batch API. After the 24-hour processing window, you retrieve results. You find that 950 have `result.type: "succeeded"`, 30 have `result.type: "errored"`, and 20 have `result.type: "expired"`. What does the `expired` status mean?

A) The API key used for those requests expired during processing  
B) Those requests did not begin processing within the 24-hour batch window — they were never attempted and you can resubmit them  
C) The cached prompts for those requests expired before processing  
D) Those requests exceeded a per-item timeout of 60 seconds  

---

### Q36 [D4]
You are structuring a complex prompt that includes instructions, examples, and user input. A colleague argues for Markdown headers while you prefer XML tags. For a prompt that will be processed by Claude with tool use, which structuring approach is most appropriate?

A) Always use Markdown — Claude was exclusively trained on Markdown and ignores XML  
B) Use XML tags for clear semantic boundaries (`<instructions>`, `<examples>`, `<user_input>`) — Claude handles XML tags well for delineating prompt sections, and they nest cleanly without ambiguity, especially when content itself contains Markdown  
C) Use JSON format for all prompt structure — it maps directly to the API format  
D) Use plain text with no structural markers — any formatting confuses the model  

---

### Q37 [D4]
You need to send a PDF document and an image to Claude for analysis in a single request. How should you structure the content blocks in the user message?

A) Convert both to base64 and include them as text content blocks with a note about the encoding  
B) Use separate content blocks with appropriate types — an `image` content block (base64 or URL source) for the image and a `document` content block (base64 source, `media_type: "application/pdf"`) for the PDF, alongside a `text` content block with your question  
C) Upload both files to a URL and pass the URL as the text of the user message  
D) Multi-modal input only supports images — convert the PDF to images first, then send all as image content blocks  

---

### Q38 [D4]
You are implementing the Batch API for overnight processing of 10,000 customer feedback items. Each batch item can have a different model configuration. You want 9,000 processed with Sonnet and 1,000 complex cases with Opus. How do you handle this?

A) Submit two separate batches — the Batch API requires all items in a batch to use the same model  
B) Include all 10,000 items in a single batch JSONL file, specifying the appropriate model in each item's request body — the Batch API processes each item independently with its own parameters  
C) Submit one batch and use a `model_override` header to specify per-item models  
D) Batch API only supports Haiku — use the standard API for Sonnet and Opus  

---

### Q39 [D4]
Your prompt caching implementation targets Claude Opus (4.8). You set `cache_control` on a content block containing exactly 1500 tokens. Cache reads never appear in your usage metrics. Why?

A) Opus does not support prompt caching  
B) Claude Opus 4.8/4.7/4.6/4.5 requires a minimum of 4096 tokens in the cached prefix — your 1500 tokens fall below this threshold, so the cache is never created  
C) You must wait 60 seconds between requests for the cache to populate  
D) Cache reads are not reported in usage metrics — they happen silently  

---

### Q40 [D4]
You use `tool_choice: {"type": "tool", "name": "classify", "disable_parallel_tool_use": true}` in your request. What behavior does this produce?

A) The model must call the `classify` tool exactly once and only that tool, and cannot call multiple tools in parallel — after calling it, the response ends with `stop_reason: "tool_use"`  
B) The model can choose whether to call `classify` or respond with text  
C) The model calls `classify` repeatedly until it gets a satisfactory result  
D) This configuration is invalid — `disable_parallel_tool_use` cannot be combined with a specific tool name  

---

### Q41 [D5]
Your production agent processes long documents and you notice degraded performance after ~150K tokens of context usage. You want to implement proactive compaction. At what threshold should compaction typically trigger, and why?

A) At 50% of context window — early compaction preserves the most information  
B) At 95% of context window — wait as long as possible to avoid unnecessary summarization  
C) At approximately 75-80% of context window — this provides enough remaining space for the model to generate useful responses while preserving conversation state before hitting hard limits  
D) At exactly 100% — the API automatically truncates old messages when the window is full  

---

### Q42 [D5]
Your agent cites multiple sources of information. It directly read a file (source A), received a tool result from a database query (source B), made an inference based on patterns in the conversation (source C), and recalled something from its training data (source D). A user asks which source to trust most. What is the correct provenance hierarchy?

A) Training knowledge > inference > tool result > direct observation (training data is most comprehensive)  
B) Direct observation > tool result > inference > training knowledge (firsthand evidence is strongest, training data is least reliable for specific current facts)  
C) Tool result > direct observation > training knowledge > inference (tools are always authoritative)  
D) All sources are equally reliable — provenance does not affect trustworthiness  

---

### Q43 [D5]
A developer estimates that their prompt is "about 2,000 characters, so roughly 2,000 tokens." Why is this estimate problematic for capacity planning?

A) The estimate is actually correct — 1 character equals 1 token for English text  
B) The approximation of 1 token per 4 characters means 2,000 characters is roughly 500 tokens, not 2,000 — overestimating by 4x leads to unnecessarily expensive architectural decisions like premature compaction  
C) Characters and tokens have no predictable relationship whatsoever  
D) The estimate is wrong because tokens are counted after compression, making them unpredictable  

---

### Q44 [D5]
You are using extended thinking with a budget of 8,000 tokens. Your finance team asks how thinking tokens affect costs. What is the correct billing explanation?

A) Thinking tokens are free — they are internal computation only  
B) Thinking tokens are billed at the input token rate  
C) Thinking tokens are billed at the output token rate — this means an 8,000-token thinking budget could cost the same as 8,000 output tokens, and this should be factored into cost projections  
D) Thinking tokens are billed at 10% of the output token rate as a "compute fee"  

---

### Q45 [D5]
Your agent operates on a fast-moving codebase where files change frequently. It read a file 45 minutes and 15 tool-call turns ago. The agent is now making a decision based on that file's contents. What reliability concern should your architecture address?

A) No concern — once read, file contents are permanently reliable within a session  
B) Information freshness decay — in active codebases, file contents read many turns ago may be stale, and the agent should re-read critical files before making decisions that depend on their current state  
C) The model automatically tracks file modification timestamps and invalidates stale reads  
D) Files are immutable once opened — the OS locks them for the session  

---

### Q46 [D5]
Current Claude models have different context window sizes. Which statement is correct, and how should this inform your architecture?

A) 100K tokens for all models — design for aggressive compaction starting at 50K  
B) Fable 5, Opus 4.8/4.7/4.6, Sonnet 5/4.6 have 1M tokens; Haiku 4.5 has 200K — implement compaction well before the limit since performance can degrade in the later portion of the window  
C) 500K tokens for all models — context is effectively unlimited for most use cases  
D) Each model has a different context window — Opus has 200K, Sonnet 150K, Haiku 100K  

---

### Q47 [D5]
Your Batch API pipeline processes 50,000 items. After completion, 200 items have `errored` status due to transient server issues. What is the recommended architectural pattern for handling these failures?

A) Resubmit the entire batch of 50,000 items — there is no way to identify which specific items failed  
B) Implement a dead letter queue pattern — collect errored items by their `custom_id`, log the error details, and resubmit only the failed items in a new batch, with alerting if items repeatedly fail  
C) Ignore errored items — a 0.4% error rate is within acceptable tolerances  
D) Contact support for each errored item — transient errors indicate account issues  

---

### Q48 [D5]
Your conversation history has grown to 180K tokens in a 200K context window. You need to add a new user message and receive a response. What happens if you send the request as-is?

A) The API automatically truncates older messages to make room  
B) The request will likely fail or produce a severely constrained response — at 90% context usage, there is minimal room for output, and you risk HTTP errors if input alone exceeds the limit. You should compact the conversation before sending  
C) The model processes it normally but responds with exactly one token  
D) The API queues the request until context space frees up  

---

### Q49 [D5]
You are designing a retrieval-augmented generation system. Retrieved chunks are injected as tool results. When the agent encounters conflicting information between a retrieved chunk (from 2023) and its training knowledge, how should your system prompt instruct prioritization?

A) Always trust training knowledge — it was curated by Anthropic  
B) Always trust retrieved content over training knowledge for factual claims about specific current state — retrieved content is more recent and specific, but instruct the model to note conflicts and flag uncertainty to the user  
C) Average the two sources — the truth is usually in the middle  
D) Discard both sources and ask the user — conflicting information is always irreconcilable  

---

### Q50 [D5]
Your enterprise system uses prompt caching with a 5-minute TTL. During peak hours, requests come every 30 seconds to the same cached prefix. During off-peak hours, requests come every 8 minutes. What is the cost implication?

A) Caching costs are identical regardless of request frequency  
B) During peak hours, most requests benefit from cache reads (lower cost per request), but during off-peak hours the cache expires between requests — each request pays the cache write cost without benefiting from subsequent reads, potentially making caching more expensive than not caching at all during low-traffic periods  
C) The cache TTL automatically extends based on usage patterns  
D) Off-peak requests are cheaper because the system has more capacity  

---

## Answer Key

**Q1: B** [D1]
`stop_reason: "end_turn"` means the model naturally concluded its response. This is distinct from `"tool_use"` (model wants to call a tool), `"max_tokens"` (output was truncated), or `"stop_sequence"` (a custom stop sequence was encountered). The agent loop should check for pending tool_use blocks before fully terminating.

**Q2: A** [D1]
On older models using `budget_tokens`, extended thinking requires `temperature` to be set to exactly 1. Any other temperature value causes the API to reject the request. The budget of 10,000 exceeds the minimum of 1024. Note: On newer models (Opus 4.7/4.8, Fable 5), `budget_tokens` is removed entirely — use `thinking: {type: "adaptive"}` instead, and `temperature`/`top_p`/`top_k` are no longer accepted.

**Q3: B** [D1]
Redacted thinking blocks are a normal part of the extended thinking feature. They occur when Anthropic's internal safety classifiers determine the thinking content should not be exposed. The model's final response remains valid and usable. Systems must be designed to handle these gracefully rather than treating them as errors.

**Q4: A** [D1]
The standard multi-turn agent loop pattern is: send request → receive response with `stop_reason: "tool_use"` → extract tool_use blocks → execute tools locally → append both the assistant message and tool_result messages to the conversation → send a new request. This continues until `stop_reason: "end_turn"`. Claude does not execute tools server-side.

**Q5: B** [D1]
Opus 4 excels at sustained performance over long agentic tasks, making it ideal for complex multi-turn architectural work where each decision builds on previous context. While more expensive per token, the quality of architectural decisions across 15-30 turns justifies the cost. Haiku 3.5 would sacrifice quality on the complex reasoning needed for architecture.

**Q6: A** [D1]
The Count Tokens API (`/v1/messages/count_tokens`) accepts the same messages array (including system prompt and tools) as a Messages request and returns the exact input token count. This enables pre-flight cost estimation and context window budget management before committing to a full API call.

**Q7: B** [D1]
`stop_reason: "max_tokens"` indicates the response was cut off because it reached the `max_tokens` output limit. The response is incomplete. Recovery options include increasing `max_tokens` or continuing the conversation by appending the truncated response and asking the model to continue from where it stopped.

**Q8: B** [D1]
Classification into a fixed set of labels is a low-complexity task where Haiku 3.5's speed and cost advantages outweigh the marginal quality gains from larger models. Low `max_tokens` is appropriate since the output is a single label. Extended thinking would add unnecessary latency and cost for a simple routing decision.

**Q9: C** [D1]
Thinking tokens count as output tokens for billing purposes. Both the thinking tokens and the visible response text tokens are billed at the output token rate and together must fit within the `max_tokens` budget. This is important for cost planning — a large thinking budget directly increases output token costs.

**Q10: B** [D1]
`stop_reason: "stop_sequence"` means the model's output matched one of the custom stop sequences specified in the API request. The response content is everything generated up to (but not including) the stop sequence. This is commonly used in routing/parsing architectures where stop sequences signal state transitions.

**Q11: B** [D2]
MCP provides the `notifications/tools/list_changed` notification mechanism for exactly this purpose. When tools change (added, removed, or modified), the server sends this notification to connected clients. Clients should respond by calling `tools/list` again to refresh their tool catalog. This avoids connection cycling and supports dynamic tool discovery.

**Q12: B** [D2]
MCP Resources are the correct primitive for semi-structured content that should be available as context. Resources are data/content identified by URIs that can be read into the LLM's context. They differ from Tools (which are callable functions) and Prompts (which are templates for user interactions). Resources are ideal for reference documentation, configuration, and knowledge bases.

**Q13: A** [D2]
The correct combination is `readOnlyHint: false` (it modifies state), `destructiveHint: true` (deletion is irreversible), and `idempotentHint: false` (attempting to delete an already-deleted customer may produce an error, not the same result). Option B is debatable but less correct — most deletion implementations return errors on double-delete rather than being truly idempotent.

**Q14: B** [D2]
Without `additionalProperties: false`, the JSON Schema allows any extra properties, and Claude may generate parameters beyond what your tool expects. This can cause subtle bugs if your handler processes unexpected fields, or waste tokens on hallucinated parameters. Best practice is to always include `additionalProperties: false` to tightly constrain model output to defined parameters.

**Q15: B** [D2]
Server-side truncation to a relevant subset is the best strategy. The model does not automatically summarize tool results — large results consume context window space and can degrade subsequent reasoning. Truncating to key information (with metadata about what was omitted) preserves context budget while providing actionable information. The other options either do not work or are not supported.

**Q16: A** [D2]
The MCP lifecycle is: client sends `initialize` request → server responds with its capabilities and protocol version → client sends `initialized` notification → normal operations begin. This handshake ensures both sides agree on protocol version and capabilities before any tool/resource operations occur. Skipping this sequence violates the protocol.

**Q17: C** [D2]
MCP Prompts are parameterized templates that accept arguments and return structured message arrays (with roles and content). They are specifically designed for reusable prompt templates that clients can discover and invoke with parameters. This is distinct from Tools (callable functions) and Resources (data to read into context).

**Q18: A** [D2]
The `openWorldHint: true` annotation indicates the tool interacts with external systems beyond the local environment, such as making internet requests or accessing third-party APIs. Clients can use this signal to apply additional policies like requiring user confirmation or sandboxing the tool call, since open-world access introduces unpredictability.

**Q19: B** [D2]
MCP does not maintain session state across disconnections. When a server crashes and restarts, the client must re-establish the transport connection and perform the full initialization handshake again (`initialize` → `initialized`). Any in-progress operations are lost, and the client should re-discover capabilities since the server may have different tools available after restart.

**Q20: B** [D2]
`idempotentHint: true` with `readOnlyHint: false` means the tool performs writes but is safe to call multiple times with the same input (same effect as calling once). This is the classic definition of idempotency. Clients can safely implement retry logic for transient failures without risking duplicate side effects, similar to HTTP PUT semantics.

**Q21: B** [D3]
Claude Code loads CLAUDE.md files hierarchically — the root file provides base context, and subdirectory files add specifics when working in that directory. The `@path` import syntax (e.g., `@../../shared/conventions.md`) is real syntax that inlines the referenced file's contents at that point in the CLAUDE.md, allowing shared conventions to be maintained in one place and imported where needed.

**Q22: A** [D3]
`--dangerously-skip-permissions` is the flag designed for CI/headless environments where no human is available to approve permission prompts. It bypasses all confirmation dialogs, making Claude Code fully non-interactive. The flag name intentionally signals the security tradeoff. Combined with `-p` for print mode, it enables fully automated pipeline integration.

**Q23: B** [D3]
`--output-format stream-json` emits newline-delimited JSON objects in real-time as Claude Code processes. Each object represents an event (assistant text, tool call, tool result, etc.), enabling programmatic real-time parsing in CI pipelines. This is distinct from `--output-format json` which returns a single complete JSON object only after the entire operation finishes.

**Q24: A** [D3]
`--max-budget-usd` limits the dollar spend for the invocation. Setting it to 10.00 caps execution at $10 of API spend. This is the correct mechanism for bounding agent execution by cost rather than by time or tokens.

**Q25: B** [D3]
Enterprise policy overrides take precedence over all other configuration levels. If enterprise policy denies a permission, neither project-level `.claude/settings.json` nor user-level settings can override that denial. This ensures organizational security policies are enforced consistently regardless of individual developer configurations.

**Q26: B** [D3]
CLAUDE.md is version-controlled and shared with the team — it should contain project-level instructions like coding standards, architecture patterns, and build/test commands. The memory system is for personal, user-specific preferences (like editor habits or individual workflow preferences) that should not be imposed on teammates via version control.

**Q27: B** [D3]
Stop hooks are configured in the hooks system within `.claude/settings.json` (or `.claude/settings.local.json`). The `Stop` event fires when the agent completes its response. The hook runs a specified command, which could be a linter, formatter, or test runner. This is part of Claude Code's extensible hook system that supports multiple lifecycle events.

**Q28: B** [D3]
`--dangerously-skip-permissions` removes the human-in-the-loop safety mechanism. If malicious content in the codebase (e.g., a crafted comment) tricks the model via prompt injection, there is no confirmation step before file writes or shell command execution. This is why the flag is named "dangerously" — it trades safety for automation.

**Q29: B** [D3]
The most common cause of ignored CLAUDE.md instructions is excessive file size. If the file (including `@path` import expansions) grows too large, it may be truncated or partially processed. Other issues could include syntax errors in `@path` import paths. CLAUDE.md is read in all modes (interactive and headless) and does not require explicit activation.

**Q30: B** [D3]
`--max-budget-usd` is a hard spending cap. When the agent reaches the specified dollar amount, execution stops regardless of whether the agent considers the task complete. The user receives all output produced up to that point. The agent does not get special behavior at the limit — it simply cannot spend more.

**Q31: B** [D4]
Prompt caching requires minimum token thresholds in the cached prefix. The minimum varies by model: Sonnet 4.5/4.1/4/3.7 require 1024 tokens; Fable 5, Sonnet 4.6, Haiku 3.5/3 require 2048 tokens; Opus 4.8/4.7/4.6/4.5 and Haiku 4.5 require 4096 tokens. An 800-token system prompt falls below all thresholds, so the cache is never created. You need to increase the cached content to meet the minimum for your model.

**Q32: B** [D4]
The minimum `budget_tokens` for extended thinking is 1024 tokens. Thinking tokens and response tokens together count against the `max_tokens` limit — they share the output token budget. So with `max_tokens: 16000` and `budget_tokens: 10000`, the model has up to 10K tokens for thinking and the remainder for the visible response.

**Q33: B** [D4]
Cache breakpoints should be placed at boundaries between stable and variable content. The system prompt and tool definitions rarely change between requests, so placing breakpoints after these ensures they remain cached even as user messages vary. Putting breakpoints on every message is wasteful (max 4 breakpoints allowed) and does not improve hit rates.

**Q34: A** [D4]
`disable_parallel_tool_use: true` within the `tool_choice` object forces the model to emit only one tool_use block per response. This guarantees sequential execution since you process each tool call, return results, and the model decides the next call. This is the proper API-level mechanism rather than relying on prompt engineering.

**Q35: B** [D4]
`expired` status means those batch items were not processed within the batch's 24-hour processing window. They were never attempted — no tokens were consumed and no results exist. You can collect these items and resubmit them in a new batch. This is different from `errored` (attempted but failed) or `canceled` (batch was manually canceled).

**Q36: B** [D4]
XML tags provide unambiguous semantic boundaries that nest cleanly and do not conflict with content that may itself contain Markdown formatting. When a prompt contains code examples, documentation, or user input with Markdown, using XML tags for structural sections avoids ambiguity. Claude processes both well, but XML has structural advantages for complex prompts.

**Q37: B** [D4]
The Messages API supports multi-modal content blocks. Images use `type: "image"` with base64 or URL source, and PDFs use `type: "document"` with base64 source and `media_type: "application/pdf"`. These content blocks sit alongside `type: "text"` blocks in the user message's `content` array, allowing mixed-media input in a single request.

**Q38: B** [D4]
The Batch API processes each item independently. Each line in the JSONL file specifies its own request body, including the model. You can mix models, parameters, and even system prompts within a single batch. Items are processed independently, so there is no requirement for uniformity within a batch.

**Q39: B** [D4]
Claude Opus 4.8/4.7/4.6/4.5 and Haiku 4.5 require a minimum of 4096 tokens in the cached prefix for caching to activate. Fable 5, Sonnet 4.6, and Haiku 3.5/3 require 2048 tokens. Sonnet 4.5/4.1/4/3.7 require 1024 tokens. At 1500 tokens, you are below Opus's 4096-token minimum threshold. The cache is never created, so you never see cache read hits. Increase the cached content above 4096 tokens to enable caching on Opus.

**Q40: A** [D4]
`tool_choice: {"type": "tool", "name": "classify"}` forces the model to call the specified tool. Adding `disable_parallel_tool_use: true` ensures only one call is made (though with a specific named tool, parallel calls are already unlikely). The response will have `stop_reason: "tool_use"` and contain exactly one tool_use block for `classify`.

**Q41: C** [D5]
Compaction at 75-80% of the context window is the recommended threshold. This provides a buffer — enough remaining space (40-50K tokens in a 200K window) for the model to generate substantial responses and handle multi-turn tool use. Waiting until 95%+ risks hard failures, while 50% discards useful context prematurely.

**Q42: B** [D5]
The provenance hierarchy from most to least reliable is: direct observation (Claude read the file itself) > tool result (data returned from an executed query) > inference (patterns Claude noticed) > training knowledge (may be outdated or generalized). For specific, current facts, firsthand evidence always trumps recalled training data.

**Q43: B** [D5]
The standard approximation for English text is ~4 characters per token. So 2,000 characters is approximately 500 tokens, not 2,000. Overestimating by 4x could lead to premature compaction, unnecessary model downgrades, or over-provisioning of token budgets. Accurate token estimation (or using the Count Tokens API) is essential for capacity planning.

**Q44: C** [D5]
Thinking tokens are billed at the output token rate. An 8,000-token thinking budget is billed equivalently to 8,000 tokens of visible output. For cost projections, teams must account for thinking budget as a direct multiplier on output costs. This makes extended thinking a significant cost factor that should be reserved for tasks requiring genuine complex reasoning.

**Q45: B** [D5]
Information freshness decay is a real reliability concern in agentic systems. Files read many turns ago in an active codebase may have been modified by other processes, by the agent itself in earlier turns, or by concurrent users. Critical decisions should be preceded by re-reading the relevant files to ensure current state, especially in long-running sessions.

**Q46: B** [D5]
Current Claude models have different context windows: Fable 5, Opus 4.8/4.7/4.6, Sonnet 5, and Sonnet 4.6 all have 1M token context windows, while Haiku 4.5 has 200K. Architectures should implement compaction well before hitting the limit (at 75-80%) since model performance can degrade with very long contexts and you need space for output generation.

**Q47: B** [D5]
A dead letter queue pattern is the standard approach: collect failed items by their `custom_id` (which you assigned when creating the batch), log error details for observability, and resubmit only the failed items in a new batch. Alert if items repeatedly fail across retries. This avoids reprocessing the 49,800 successful items.

**Q48: B** [D5]
At 180K of a 200K context window (90% — this example uses Haiku 4.5's 200K window), there are only 20K tokens remaining for the new message AND the model's response. If the new user message is large enough, the request may fail entirely. Even if it succeeds, the response will be severely constrained. Compaction should have been triggered at 75-80% — this is now an emergency requiring immediate conversation compaction. Note: Opus/Sonnet models now have 1M windows, but the same principle applies — implement compaction well before hitting the limit.

**Q49: B** [D5]
Retrieved content should be prioritized over training knowledge for specific factual claims because it is more recent and contextually relevant. However, the system should instruct the model to flag conflicts transparently to the user rather than silently choosing one source. This maintains trust and allows human judgment on ambiguous cases.

**Q50: B** [D5]
Prompt caching has a 5-minute TTL. During peak hours (30-second intervals), each cache write is amortized over ~10 subsequent cache reads, providing significant savings. During off-peak (8-minute intervals), the cache expires before the next request — you pay the cache write premium without any cache read benefits, making it potentially more expensive than uncached requests. Architecture should consider disabling caching during low-traffic periods or designing request patterns to maintain cache freshness.
