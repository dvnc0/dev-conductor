# LLM and Agent Security

Reference for the `security-and-hardening` skill when the code calls a model, builds prompts, exposes tools, uses retrieval or memory, or orchestrates agents. The themes line up with the OWASP Top 10 for LLM Applications and the OWASP Top 10 for Agentic Applications. Check the current official lists at genai.owasp.org for exact IDs.

**The core rule:** model output is untrusted input. Anything that reaches the model's context (user text, retrieved documents, web pages, emails, tool results, other agents' messages) can contain instructions, and the model may follow them. **A system prompt is not an access control.** Whatever the model is able to do, assume an attacker who controls any of its inputs can ask it to do.

Same method as the rest of the skill: find where untrusted text enters the context (the source), find what the model's output can trigger (the sink), and show the missing control between them.

---

## 1. Prompt injection (direct and indirect)
- **Sources:** user messages; retrieved documents; fetched web pages; emails, tickets, calendar items; file contents; tool results; other agents' output.
- **Sinks:** tool calls; actions taken with the app's or user's privileges; data shown to a different user; persisted memory or plans.
- **Grep:** `messages\s*=|system_?prompt|role.*system|chat\.completions|messages\.create|generate_content|invoke\(|ChatPromptTemplate|PromptTemplate|f".*\{.*\}.*"` near model calls · `tools\s*=|tool_choice|function_call|tool_use|@tool\b|mcp`.
- **Trail:** attacker-controlled text lands in the model's context (cite where it's added), and the model's response then drives a privileged action or exposes data the attacker couldn't reach directly.
- **Controls:** treat context content as data (delimit and label it, but don't rely on that alone); limit what tools and data the model can reach (section 3); require confirmation for consequential actions; authorize every tool call against the end user, not the app.
- **Traps:** output shown only to the same user who wrote the input, with no tools or secrets in context; a human approval step showing the real action before it runs.

## 2. Sensitive information disclosure
- **Sources:** conversation history, RAG corpora, fine-tuning data, other users' data in shared context or caches, secrets placed in prompts.
- **Sinks:** model responses, logs and traces, shared semantic caches, prompts sent to third-party providers.
- **Grep:** `chat_history|memory|ConversationBuffer|messages\.append|semantic_cache|GPTCache|cache` on prompts or responses · secrets or PII fields interpolated into prompts · `api_key|password|token` inside prompt templates.
- **Trail:** data belonging to user or tenant A is placed into context or a cache that a request from B can reach; or a secret placed in a prompt can be extracted; or regulated data goes to a provider the data policy doesn't allow.
- **Controls:** scope history, caches, and retrieval per user and tenant; keep secrets out of prompts (tools hold credentials, the model never sees them); redact before logging.
- **Traps:** context scoped per user; the data is public; the provider is approved for that data class.

## 3. Excessive agency and tool misuse
- **Sources:** model-chosen tool calls and arguments; chains of individually harmless tools.
- **Sinks:** tools that write, delete, send, pay, execute, or change permissions; broad credentials held by the agent.
- **Grep:** `tools\s*=|@tool\b|function_declarations|allowed_tools|tool_choice|auto_approve|bypassPermissions|dangerously|requires_approval|run_shell|subprocess|send_email|delete|transfer`.
- **Trail:** the agent holds a capability broader than its task, or acts with the app's full privileges instead of the end user's, and no confirmation, scope check, or argument validation sits between a model decision (steerable via section 1) and an irreversible effect. Include chains: "read a secret" plus "make an HTTP request" equals exfiltration.
- **Controls:** the narrowest tool set per task; read-only by default; validate tool arguments against a schema and allow-list at the tool boundary; tools enforce the end user's authorization; human confirmation for irreversible or external actions; per-session spend and action budgets.
- **Traps:** per-call human approval that shows the real arguments; tools that re-authorize against the end user; read-only tools.

## 4. Improper output handling
- **Sources:** model output.
- **Sinks:** HTML or markdown rendering, SQL, shell, `eval`, file writes, URL fetches, downstream API calls, redirects.
- **Grep:** `innerHTML|dangerouslySetInnerHTML|v-html|markdown|marked\(|eval\(|exec\(|subprocess|execute\(|json\.loads\(.*(response|completion|content)|JSON\.parse\(.*(response|completion|content)|open\(.*(response|output)`.
- **Trail:** model output reaches an injection sink without validation, encoding, or sandboxing. Write it up as a normal injection trail (catalog A05) with the model as one hop. Watch markdown renderers: an image tag pointing at an attacker URL with data in the query string is an exfiltration channel.
- **Controls:** structured output validated against a schema; allow-lists for anything that becomes a command, path, or URL; the same escaping as any user input; strip or proxy remote images and links in rendered model output.
- **Traps:** output schema-validated into an enum or typed value; rendered through a sanitizer.

## 5. Retrieval, embeddings, and memory poisoning
- **Sources:** documents ingested into vector stores; user feedback; memory writes; shared knowledge bases.
- **Sinks:** retrieval results placed in context; long-term memory re-injected in later sessions; fine-tuning datasets.
- **Grep:** `similarity_search|as_retriever|vector_?store|embed|upsert|add_documents|add_texts|namespace|collection|filter=|memory\.(save|add|update)|long_term`.
- **Trail:** retrieval has no tenant or ACL filter, so one user's query returns another's chunks; or attacker-written content is stored in shared memory or an index and later trusted as instruction or fact in other users' sessions.
- **Controls:** tenant and permission filters enforced server-side on every retrieval; provenance tracked on stored content; review gates on shared corpora; memory scoped per user and labeled as untrusted when re-injected.
- **Traps:** per-tenant namespaces enforced by the server; ingestion limited to curated sources.

## 6. Unbounded consumption (denial of wallet)
- **Sources:** user-controlled prompt size, file sizes, loop or step counts, concurrency, retries.
- **Sinks:** paid model calls, agent loops, embedding jobs, tool fan-out.
- **Grep:** `max_tokens|max_output_tokens|max_(iterations|steps|turns)|recursion_limit|while True|retry|backoff|batch|concurrency|rate_?limit|quota|budget`.
- **Trail:** an anonymous or low-privilege user can trigger unbounded or amplified model calls (no token cap, step limit, per-user quota, or spend budget), causing outage or attacker-driven cost.
- **Controls:** caps on input size, output tokens, steps, recursion, and concurrency; per-user and global rate limits and budgets; alerts on spend.
- **Traps:** gateway or provider quota already enforced; caps set in a shared wrapper.

## 7. Model and tool supply chain
- **Sources:** model weights, adapters, datasets, prompt templates, plugins, MCP servers, and agent tools loaded at runtime.
- **Sinks:** model loaders, plugin or tool registries, remote-code flags.
- **Grep:** `trust_remote_code=True|from_pretrained\(|torch\.load|pickle\.load|hf_hub_download|snapshot_download` · `npx -y|uvx |mcp.*(command|url|servers)|load_plugin|importlib\.import_module|@latest|:latest`.
- **Trail:** an unpinned or unverified artifact is loaded from a source an attacker can influence and executes code on load (pickle-based weights, remote code), or a tool or MCP server whose description or code can change after approval is trusted automatically.
- **Controls:** pin to versions or hashes; prefer safetensors; avoid `trust_remote_code`; allow-list tool servers; require re-approval when a tool's definition changes.
- **Traps:** pinned by hash; internal registry only; dev notebook not shipped.

## 8. Code execution by agents
- **Sources:** model-written code or commands.
- **Sinks:** interpreters, shells, notebooks, package installs, write-then-execute paths.
- **Grep:** `code_interpreter|exec_code|run_python|PythonREPL|PythonAstREPLTool|subprocess|exec\(|eval\(|pip install|npm install`.
- **Trail:** model output (steerable via section 1) reaches an executor running with host access, credentials, or network beyond a sandbox. Cite the executor and what the sandbox doesn't cover.
- **Controls:** isolated sandbox (container or VM) with no ambient credentials, restricted network and filesystem, resource limits; command allow-lists; approval gates.
- **Traps:** a real isolation boundary with no secrets and no network; allow-listed commands only.

## 9. Identity and inter-agent trust
- **Sources:** credentials the agent holds or inherits (service accounts, passed-through user tokens, cached tokens); messages between agents over queues, HTTP, MCP, or shared state.
- **Sinks:** downstream services acting on the agent's credentials; agents acting on messages from other agents.
- **Grep:** `service_account|impersonat|on_behalf_of|delegat|token.*cache|a2a|agent_to_agent|send_message|handoff|from_agent|publish\(|subscribe\(`.
- **Trail:** the agent acts with broader credentials than the requesting user has (confused deputy), or reuses a token across users or sessions; or a message is trusted as coming from a specific agent with no authentication or integrity check, and it drives tool calls or state changes.
- **Controls:** per-user, scoped, short-lived credentials; downstream services re-authorize the end user; authenticated and integrity-protected inter-agent messages; receivers treat message content as untrusted data.
- **Traps:** the downstream service enforces end-user authorization; in-process calls with no external writer.

## 10. Human oversight and runaway behavior
- **Approval UIs:** if a human approves actions, the approval must show the real action and arguments, not an agent-written summary that attacker-influenced content can shape.
- **Cascades:** chained or delegating agents need step, depth, and fan-out limits, and validation between hops, so one bad output doesn't become a loop or a mass action.
- **Self-modification:** an agent should not be able to edit its own permissions, prompts, tools, or schedules, or spawn agents with broader scope, without out-of-band approval. Look for `create_agent|spawn|add_tool|edit.*(prompt|permission|settings)|schedule|cron`.
- **Kill switch:** there is a way to stop a running agent and revoke its credentials.
