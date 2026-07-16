# Tools and approvals

Tools are the agent's hands: the actions the model can take beyond emitting text. This page covers how Codex defines tools, generates their schemas, routes a model's tool call to an executor, runs tools in parallel, and — critically — gates each call through an approval and sandbox pipeline before it touches your machine.

Two crates cooperate. `tools/` (`codex-tools`) holds transport-neutral primitives: tool specs, JSON-schema types, the executor trait. `core/src/tools/` owns per-turn assembly, dispatch, approvals, and every concrete handler.

## The built-in tools

Codex ships a large, feature-gated tool set. This is the catalog (model-facing name → where it's defined). Not all are active in every session — availability depends on model capabilities, feature flags, and config.

| Tool | What it does | Handler |
|------|--------------|---------|
| `shell_command` | Run a shell command (classic one-shot) | `handlers/shell/shell_command.rs` |
| `exec_command` / `write_stdin` | Persistent PTY sessions (unified exec) | `handlers/unified_exec/` |
| `apply_patch` | Apply a code patch in a custom grammar | `handlers/apply_patch.rs` |
| `update_plan` | Post a todo/checklist of steps | `handlers/plan.rs` |
| `view_image` | Attach a local image to context | `handlers/view_image.rs` |
| `request_user_input` | Ask the user a question mid-turn | `handlers/request_user_input.rs` |
| `request_permissions` | Ask to widen sandbox permissions | `handlers/request_permissions.rs` |
| `web_search` | Hosted web search (provider-side) | hosted spec, no local handler |
| `list_mcp_resources` / `read_mcp_resource` | Access MCP server resources | `handlers/mcp_resource/` |
| `spawn_agent` / `send_message` / `wait_agent` (v1 & v2) | Multi-agent collaboration | `handlers/multi_agents*/` |
| `spawn_agents_on_csv` / `report_agent_job_result` | Batch agent jobs over CSV | `handlers/agent_jobs/` |
| `tool_search` | Load deferred tools on demand (BM25 search) | `handlers/tool_search.rs` |
| `get_context_remaining` / `new_context` | Token-budget introspection | `handlers/` |
| MCP tools (`server__tool`) | External MCP server tools | `handlers/mcp.rs` |
| code-mode `exec` / `wait` | JavaScript tool interface | code-mode handlers |

Each handler implements two traits: `ToolExecutor<ToolInvocation>` (the base contract: `tool_name()`, `spec()`, `exposure()`, `handle()`) and `CoreToolRuntime` (`core/src/tools/registry.rs:51`, the core-owned extension adding hook payloads and diff consumers).

## Schema generation

Tool schemas are the contract the model reads. Codex models them as `ToolSpec` (`tools/src/tool_spec.rs:17`), a serde-tagged enum with five shapes:

- `Function` — a normal JSON-schema tool.
- `Namespace` — a group of related function tools coalesced under one namespace (used for MCP servers).
- `Freeform` — a grammar-based tool whose input is raw text, not JSON (this is how `apply_patch` works).
- `WebSearch` / `ToolSearch` — hosted and deferred-tool variants.

The JSON-schema type (`tools/src/json_schema.rs:41`) mirrors OpenAI's Structured-Outputs subset. Each handler's schema is produced by a free `create_*_tool(...)` function in a sibling `*_spec.rs` file, so the schema lives next to but separate from the executor. `create_tools_json_for_responses_api` serializes the enabled specs into the Responses API `tools` array.

## Per-turn registry assembly

The tool set is rebuilt every turn, because it depends on the current model, feature flags, provider capabilities, and configured MCP servers. `spec_plan.rs::build_tool_specs_and_registry` (`core/src/tools/spec_plan.rs:167`) is the assembler. It runs a sequence of `add_*` functions — shell tools, MCP-resource tools, core-utility tools, collaboration tools, injected MCP tools, extension tools, dynamic tools, then hosted specs — dedupes by name, coalesces same-namespace tools, and filters out anything the provider can't handle.

Availability is decided by several layers stacked together:

- **Feature flags** (`codex_features::Feature`) — `ShellTool`, `UnifiedExec`, `TokenBudget`, `RequestPermissionsTool`, `CodeMode`, `SpawnCsv`, and more.
- **Model info** — `shell_type` (ShellCommand vs UnifiedExec vs disabled), `apply_patch_tool_type`, `supports_search_tool`, `tool_mode` (Direct vs CodeMode).
- **Provider capabilities** — `namespace_tools`, `web_search`, `image_generation`.
- **Tool exposure** (`tool_executor.rs:15`) — `Direct` (in the list), `Deferred` (hidden until `tool_search` loads it), `DirectModelOnly` (list-only, excluded from code mode), `Hidden` (dispatch-only).

The `Deferred` exposure is worth noting: instead of putting every MCP tool's schema in the model's context (expensive), rarely-used tools are hidden behind a single `tool_search` tool that the model queries to load them on demand. This is the same token-saving pattern Claude Code calls "tool search."

## Dispatch

When the model emits a tool call, `ToolRouter::build_tool_call` (`router.rs:112`) converts the wire item into a `ToolCall` and `registry.dispatch_any_with_terminal_outcome` (`registry.rs:408`) runs the pipeline:

1. Look up the handler by name (unknown → an error returned to the model, not a crash).
2. Run **PreToolUse hooks**, which can block the call or rewrite its input (see [Extensibility](09-extensibility.md)).
3. Call the handler's `handle(...)`.
4. Run **PostToolUse hooks**, which can block or annotate the result.
5. Return the output, which `AnyToolResult::into_response` converts into a `FunctionCallOutput`/`CustomToolCallOutput` to feed back to the model.

## The approval and sandbox gate

This is the most important part of the tool system: the gate that stands between "the model wants to run `rm -rf build/`" and the command actually running. Exec-like tools (shell, apply_patch) route through `ToolOrchestrator::run` (`core/src/tools/orchestrator.rs:137`), which computes an approval requirement, selects a sandbox, runs the attempt, and escalates on failure.

**Computing the requirement.** `default_exec_approval_requirement` (`sandboxing.rs:198`) maps the approval policy against the sandbox policy:

| Approval policy | Effect |
|-----------------|--------|
| `Never` | Skip approval; failures returned to the model |
| `OnRequest` / `Granular` | Ask only when the filesystem policy is `Restricted` |
| `UnlessTrusted` | Always ask |

`execpolicy` (a Starlark policy engine) can additionally classify a command as known-safe and skip the prompt entirely, or as trusted-enough to bypass the sandbox.

**Two reviewers.** An approval request goes to either the **Guardian** (an automated model-based reviewer — the "auto-review" path) or the **User** (an interactive prompt) (`approvals.rs:137`). Permission-request hooks run first and can short-circuit with Allow/Deny. Session caching means an `ApprovedForSession` decision is remembered, keyed by a serialized `ApprovalKey`, so you don't re-approve the same command (apply_patch caches per-file, so a subset request auto-approves).

**Escalation on failure.** If a sandboxed command is denied by the OS sandbox and the policy allows it, the orchestrator can prompt to *retry without the sandbox* — the classic "this command needs network access, allow it?" flow (`orchestrator.rs:137`). It re-runs escalated only after approval.

**Network approvals** are a separate path (`network_approval.rs`): a command that wants network access beyond the allowlist triggers its own approval, mapping an approved `curl *` into a network-policy amendment.

The result is a defense-in-depth model: the approval policy decides *whether to ask*, the sandbox decides *what the command can touch*, and escalation provides a *controlled way to widen* both — with every decision logged as an `otel.tool_decision` telemetry event.

## Shell, unified exec, and apply_patch specifics

**`shell_command`** builds an `ExecParams` (command, cwd, env from the shell-environment policy), intercepts any embedded `apply_patch`, computes its approval requirement via execpolicy, and runs through the orchestrator with a `ShellRuntime`. Two backends exist: `Classic` and `ZshFork` (which intercepts `execve` via a wrapper to track subprocesses). It supports parallel calls and waits for runtime teardown on cancellation.

**`exec_command` + `write_stdin`** provide persistent PTY sessions via a `UnifiedExecProcessManager` — the model can start a long-running process (a REPL, a dev server), read its output incrementally, and write to its stdin. This is selected only when the model supports it and ConPTY is available on the platform.

**`apply_patch`** is a Freeform tool: the model emits a patch in a custom grammar (`apply_patch.lark`), Codex parses it with `codex_apply_patch::parse_patch`, verifies the affected paths against the sandbox context, computes per-file permissions, and applies it. It can also be invoked inline inside a shell command (`intercept_apply_patch`), and per-file `ApprovedForSession` caching means approving one file's change doesn't re-prompt for the rest of the same patch.

## Design assessment

The tool system's strengths are the clean two-trait executor contract, the per-turn registry that adapts to model and provider capabilities, the deferred-tool mechanism that keeps context small, and the layered approval gate with escalation. Its weaknesses (detailed in [Critique](12-critique.md)) are the combinatorial feature-flag surface that decides which tools exist, and the fact that `Deferred`/`DirectModelOnly`/`Hidden` exposures plus code-mode augmentation make "what tools does the model actually see this turn" genuinely hard to reason about from the code.
