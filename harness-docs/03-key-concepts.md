# Key concepts

Every term that appears across these docs, defined so it stands alone. Grouped by area. Where a term names a concrete type, the source location is given.

## Core runtime

**Thread** — A persistent conversation with the agent, addressed by a `ThreadId` and backed by a rollout file on disk. Resumable and forkable. The live handle is `CodexThread` (`core/src/codex_thread.rs`). Formerly called a "conversation."

**Session** — The in-memory engine that runs a thread's turns. Owns the submission and event queues, the active turn, and the background loop (`core/src/session/`). A near-god-object with a large method surface.

**Turn** — One user request plus all the agent work to satisfy it: model calls, tool calls, tool outputs, more model calls, ending in a final message. Modeled as a `SessionTask` (`core/src/tasks/mod.rs:214`).

**Op (Operation)** — A message *into* the engine, sent by a frontend: `UserInput`, `Interrupt`, `ExecApproval`, `Compact`, `Review`, `Shutdown`, and ~20 others (`protocol/src/protocol.rs:525`). The submission-queue payload.

**EventMsg** — A message *out of* the engine: streamed text, reasoning, tool-call begin/end, approvals requested, turn complete, errors, and ~80 others (`protocol/src/protocol.rs:1282`). The event-queue payload.

**Submission Queue (SQ) / Event Queue (EQ)** — The two channels that decouple frontends from the engine. Frontends write `Op`s into the SQ; the engine writes `EventMsg`s into the EQ. Created at `core/src/session/mod.rs:533`.

**TurnContext** — An immutable per-turn snapshot of configuration: model, provider, reasoning effort, approval policy, sandbox/permission profile, network policy, cwd, personality (`core/src/session/turn_context.rs:113`). Guarantees a turn's behavior can't shift mid-flight.

**ThreadManager** — The top-level factory that builds Sessions and owns the model catalog and thread store (`core/src/thread_manager.rs`). Formerly `ConversationManager`.

## Tools and items

**Tool** — An action the model can invoke: `shell_command`, `apply_patch`, `update_plan`, `view_image`, MCP tools, and more. Each has a name, a JSON schema, and an executor implementing `CoreToolRuntime` (`core/src/tools/registry.rs:51`).

**ToolSpec** — The schema of a tool as sent to the model, one of `Function`, `Namespace`, `Freeform`, `WebSearch`, `ToolSearch` (`tools/src/tool_spec.rs:17`).

**ToolRouter / ToolRegistry** — Per-turn structures that hold the enabled tools and route a model's tool call to the right executor (`core/src/tools/router.rs`, `registry.rs`).

**ResponseItem** — The canonical conversation/history item that maps onto the OpenAI Responses API: a `Message`, `Reasoning`, `FunctionCall`, `FunctionCallOutput`, `CustomToolCall`, etc. (`protocol/src/models.rs:804`). This is what the model sees.

**TurnItem** — The UI-facing item stream surfaced via `ItemStarted`/`ItemCompleted` events: `UserMessage`, `AgentMessage`, `Reasoning`, `CommandExecution`, `FileChange`, `McpToolCall`, `WebSearch`, and more (`protocol/src/items.rs:44`). A projection of `ResponseItem` for display.

**apply_patch** — The primary file-editing tool. The model emits a patch in a custom grammar; Codex parses and applies it (`apply-patch` crate; handler at `core/src/tools/handlers/apply_patch.rs`).

**update_plan** — The todo/checklist tool. The model posts a plan of steps with statuses; the UI renders it (`protocol/src/plan_tool.rs`). Distinct from "plan mode."

## Approvals and safety

**AskForApproval** — The approval policy: `UnlessTrusted` (ask unless known-safe), `OnRequest` (the default; model decides when to ask), `Granular` (per-flow toggles), `Never` (never ask, failures returned to the model) (`protocol/src/protocol.rs:911`).

**SandboxPolicy** — The filesystem/network restriction level: `ReadOnly`, `WorkspaceWrite` (writable roots + network toggle), `DangerFullAccess`, `ExternalSandbox` (`protocol/src/protocol.rs:998`).

**PermissionProfile** — The runtime form of a sandbox policy, resolved into concrete `(FileSystemSandboxPolicy, NetworkSandboxPolicy)` and passed to the platform sandbox backends.

**ReviewDecision** — The answer to an approval request: `Approved`, `ApprovedForSession`, `Denied`, `Abort`, plus policy-amendment variants (`protocol/src/protocol.rs:4066`).

**Guardian** — An automated reviewer that can approve/deny tool calls on the user's behalf (the "auto-review" path), as an alternative to prompting the human.

**execpolicy** — A Starlark-based policy engine that classifies whether a shell command is known-safe (`execpolicy` crate).

## Execution and sandboxing

**Seatbelt** — macOS's `sandbox-exec` mechanism; Codex generates a profile to restrict filesystem and network.

**Landlock / seccomp** — Linux kernel primitives: Landlock restricts filesystem access, seccomp filters syscalls (network). Codex applies both (`linux-sandbox/src/landlock.rs`).

**Bubblewrap (bwrap)** — A Linux mount-namespace sandbox that Codex prefers for filesystem isolation, with Landlock/seccomp as a legacy fallback (`linux-sandbox/src/bwrap.rs`).

**Restricted token** — Windows' sandbox primitive: a `CreateRestrictedToken` with per-run randomized capability SIDs stamped into file ACLs. Codex uses this instead of AppContainer (`windows-sandbox-rs/`).

**Network proxy** — A loopback-bound, egress-filtering forward proxy (HTTP/CONNECT/SOCKS5) that sandboxed processes are forced through via proxy env vars; enforces a domain allow/deny policy and can MITM-terminate TLS to inject credentials the model never sees (`network-proxy` crate).

**exec-server** — A JSON-RPC server that spawns and controls subprocesses, either in-process (local) or in a remote environment (dev container / cloud sandbox) over an encrypted Noise channel (`exec-server` crate).

**arg0 dispatch** — The trick where one binary behaves as several by branching on `argv[0]`: invoked as `codex-linux-sandbox` it runs the sandbox helper, as `apply_patch` it runs the patch tool (`arg0/src/lib.rs`).

## Context and state

**Rollout** — The durable JSONL recording of a thread: every `RolloutItem` (session meta, turn context, events, response items). The source of truth for resume/fork (`rollout` crate).

**State DB** — Five bundled SQLite files under `~/.codex` (`state_5`, `logs_2`, `goals_1`, `memories_1`, `thread_history_1`) holding a derived, queryable index over rollouts plus SQLite-primary data like the sub-agent spawn graph (`state` crate). Rebuildable from rollouts.

**Compaction** — Summarizing older conversation history to reclaim context-window space when it fills up. Triggered automatically before or mid-turn (`core/src/compact.rs`).

**Memory** — Durable facts extracted from past sessions and re-injected into new ones, via a two-stage background pipeline (`memories` crate).

**AGENTS.md** — The project-instructions file Codex discovers by walking from cwd up to the project root, merging root-to-cwd, and injecting as a user-role message (`core/src/agents_md.rs`). An `AGENTS.override.md` takes precedence per directory.

**Context fragment** — A bounded, tagged block injected into the model context (skills catalog, environment, permissions, personality). All implement `ContextualUserFragment` (`context-fragments` crate).

**Turn diff** — The net file changes made during a turn, tracked in memory by `TurnDiffTracker` (`core/src/turn_diff_tracker.rs`) and shown in the UI. Distinct from git.

## Extensibility

**Skill** — A folder with a `SKILL.md` (YAML frontmatter + markdown body) that the model reads on demand. Only the one-line description sits in context until the skill is used (`core-skills` crate; `docs/skills.md`).

**Hook** — A user script that fires at a lifecycle event (PreToolUse, PostToolUse, SessionStart, and 7 more) and can block, rewrite, or annotate agent actions (`hooks` crate). Claude-Code-compatible contract.

**Plugin** — A packaged bundle (`plugin.json` manifest) contributing skills, MCP servers, apps, hooks, and commands, installed from a marketplace (`core-plugins` crate).

**MCP (Model Context Protocol)** — An open standard for tool servers. Codex is both an MCP *client* (aggregating external servers' tools for the model, `codex-mcp`) and an MCP *server* (exposing itself as `codex`/`codex-reply` tools, `mcp-server`).

**Connector / Codex App** — A first-party ChatGPT connector (Gmail, Slack, etc.) surfaced as tools of a built-in "Codex Apps" MCP server (`connectors` crate).

**Code mode** — An experimental tool interface where the model writes JavaScript executed in an embedded V8 isolate and calls tools programmatically from the script, cutting round-trips (`code-mode`, `code-mode-host` crates).

## Frontends and protocol

**app-server** — The JSON-RPC server (v2 protocol) that is the engine's public API; IDE extensions, the desktop app, and the TUI are all clients (`app-server`, `app-server-protocol`).

**TUI** — The interactive terminal UI built on ratatui, itself an app-server client (`tui` crate).

**codex exec** — The headless, non-interactive mode for CI/scripting, with JSONL output (`exec` crate).

**SDK** — Language libraries that drive Codex as a subprocess: the TypeScript SDK (`@openai/codex-sdk`) over `codex exec`, the Python SDK (`openai-codex`) over `codex app-server` (`sdk/`).

**Cloud task** — A task that ran in OpenAI's Codex Web cloud, browsable and applicable locally via `codex cloud`; "apply" is always a local `git apply` of a backend-provided diff (`cloud-tasks` crate).

## Config and auth

**Config layering** — The precedence stack that resolves settings: managed/enterprise → admin → user → project → session flags. Enterprise layers are fetched as a signed "cloud config bundle" (`config`, `cloud-config` crates).

**Feature flag** — A `[features]` toggle with a lifecycle stage (Stable/UnderDevelopment/Deprecated/Removed/Experimental) gating most subsystems (`features` crate).

**Auth mode** — How Codex authenticates: ChatGPT OAuth login (the default), an API key, or a device-code flow. Cloud features require ChatGPT login (`login` crate).

**Originator** — A header (`originator: codex_cli_rs` by default) identifying the client to the backend; SDKs override it (`login/src/auth/default_client.rs`).
