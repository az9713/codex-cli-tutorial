# Codex overview

Codex CLI is OpenAI's local coding agent, distributed as a single Rust binary and driven by the OpenAI Responses API. You run `codex` in a terminal, describe a task, and the agent reads your code, runs commands in a sandbox, edits files, and iterates until the task is done — all on your machine, under your approval policy.

This page builds the mental model: what Codex is, how its ~100 Rust crates are organized, and how a single request flows from your keystroke to a model call and back.

## The mental model

Codex is a **thread-centric agent runtime** wrapped in **many frontends**, all speaking one **protocol**.

- A **thread** is a persistent conversation with the agent, backed by a rollout file on disk so it can be resumed or forked. (Older code and docs call this a "conversation"; the type was renamed — `ThreadManager` was `ConversationManager`, `CodexThread` was `CodexConversation`, with deprecated aliases still exported at `core/src/lib.rs:124`.)
- A **turn** is one user request and all the agent's work to satisfy it — model calls, tool calls, tool outputs, more model calls — ending in a final message.
- A **Session** is the live engine that runs turns: it owns a submission queue (things coming in) and an event queue (things going out), and a background loop that drives the two.

Everything a user sees — the terminal UI, the `codex exec` headless mode, the VS Code extension, the Python SDK — is a *client* that submits operations and reads events. The engine does not know or care which frontend it is talking to.

## The crate map

The workspace lives under `codex-rs/`. It is large (~100 crates), but the crates cluster into clear layers. This table is your index; each layer links to the deep dive.

| Layer | Key crates | What it does | Deep dive |
|-------|-----------|--------------|-----------|
| **Core loop** | `core` (esp. `session/`, `tasks/`, `codex_thread.rs`, `thread_manager.rs`) | The agent runtime: turns, tool dispatch, streaming | [Agent loop](04-agent-loop.md) |
| **Protocol** | `protocol`, `app-server-protocol`, `code-mode-protocol`, `exec-server-protocol` | Shared wire types: `Op`, `EventMsg`, `ResponseItem`, JSON-RPC schemas | [Frontends and protocol](10-frontends-and-protocol.md) |
| **Tools** | `tools`, `core/src/tools/`, `apply-patch` | Tool specs, registry, router, executors, approvals | [Tools and approvals](05-tools-and-approvals.md) |
| **Model client** | `codex-api`, `model-provider-info`, `model-provider`, `codex-client`, `models-manager` | Responses API wire client, streaming, retry, model catalog | [Model client](07-model-client.md) |
| **Execution & sandbox** | `sandboxing`, `linux-sandbox`, `windows-sandbox-rs`, `bwrap`, `execpolicy`, `exec-server`, `network-proxy`, `shell-command`, `process-hardening` | Run commands under OS-enforced restrictions; filter network egress | [Execution and sandboxing](06-execution-and-sandboxing.md) |
| **State & context** | `rollout`, `state`, `thread-store`, `memories`, `context-fragments`, `file-search`, `file-watcher`, `git-utils`, `agent-graph-store` | Session recording, SQLite index, compaction, memory, prompt fragments | [Context, state, and memory](08-context-state-memory.md) |
| **Extensibility** | `skills`, `core-skills`, `hooks`, `plugin`, `core-plugins`, `codex-mcp`, `rmcp-client`, `code-mode`, `code-mode-host`, `connectors`, `external-agent-migration` | Skills, hooks, plugins, MCP client, code mode, importers | [Extensibility](09-extensibility.md) |
| **Frontends** | `tui`, `cli`, `exec`, `app-server`, `app-server-*`, `mcp-server`, `sdk/` (TS/Python), `codex-cli` (npm) | Terminal UI, headless CLI, IDE server, MCP server, SDKs, packaging | [Frontends and protocol](10-frontends-and-protocol.md) |
| **Config, auth, infra** | `config`, `cloud-config`, `features`, `login`, `keyring-store`, `secrets`, `otel`, `analytics`, `feedback`, `http-client`, `websocket-client`, `install-context`, `codex-home` | Config layering, auth, feature flags, telemetry, HTTP/WS, updates | [Config, auth, and telemetry](11-config-auth-telemetry.md) |
| **Cloud** | `backend-client`, `cloud-tasks`, `cloud-tasks-client`, `chatgpt`, `realtime-webrtc`, `responses-api-proxy` | Codex Web task list/apply, `codex apply`, voice, key-injection proxy | [Frontends and protocol](10-frontends-and-protocol.md), [Config](11-config-auth-telemetry.md) |

A design rule the maintainers enforce shapes this map: `core` is deliberately kept from growing. `AGENTS.md:74` says *"resist adding code to codex-core"* — new concepts get their own crate. That is why there are ~100 crates instead of a monolith.

## How a request flows, end to end

Follow one user message through the system. Suppose you type "add a test for the parser" in the terminal.

1. **Frontend → submission.** The TUI packages your text as an `Op::UserInput` and submits it. But the TUI does not call the engine directly — it sends the op over JSON-RPC to an **app-server** (embedded in-process by default), which forwards it into the engine's **submission queue** (`core/src/session/mod.rs:533`).

2. **The loop picks it up.** A background `submission_loop` (`core/src/session/handlers.rs:712`) receives the op. With no turn currently running, it builds a fresh `TurnContext` (an immutable snapshot of model, sandbox mode, approval policy, cwd) and spawns a `RegularTask` — one turn of work (`tasks/regular.rs:37`).

3. **The prompt is assembled.** The turn builds the model request: base/system instructions for the current model, your `AGENTS.md` project instructions, an `<environment_context>` block (cwd, OS, sandbox, network), the skills catalog, and the conversation history — then your new message. (See [Context, state, and memory](08-context-state-memory.md) for the exact ordering.)

4. **The model is called, streaming.** The model client opens a Responses API stream (WebSocket if the provider supports it, else HTTP server-sent events) and yields typed events: text deltas, reasoning deltas, and tool calls (`codex-api/src/sse/responses.rs`).

5. **Tool calls are dispatched.** When the model emits a tool call — say `shell_command` running `cargo test` — the router converts it to a `ToolCall`, the approval gate decides whether it needs your OK, and the executor runs it **inside the sandbox** (Seatbelt/Landlock/Windows token, with the network proxy in front). The output is captured and recorded into history (`core/src/tools/`, `core/src/stream_events_utils.rs:319`).

6. **The loop repeats.** The tool output is fed back to the model, which calls again — reading a file, writing the test, running it — until it stops asking for tools and emits a final answer. Each streamed piece becomes an `EventMsg` on the **event queue**.

7. **Events flow back to you.** The app-server relays events to the TUI, which renders streaming text, live command output, diffs, and approval prompts. The whole turn is written to a **rollout file** (JSONL) and indexed in a **SQLite** database, so you can resume or fork it later.

8. **Persistence and memory.** After the turn, token usage is metered, the turn diff is tracked, and — if enabled — a background pipeline may extract durable **memories** to inject into future sessions.

That is the entire harness in one trip: frontend → protocol → loop → prompt → model → tools → sandbox → events → persistence.

## What Codex is *not*

- **Not a cloud service by default.** The agent runs locally. There is an optional connection to Codex Web (the `codex cloud` subcommand and `codex apply`) for browsing and applying tasks that ran in OpenAI's cloud, but the local agent is self-contained.
- **Not tied to one model provider in principle.** It ships built-in providers for OpenAI, Amazon Bedrock, and local OSS runtimes (Ollama, LM Studio), and the provider abstraction is pluggable — though the codebase has fully removed Chat Completions support and now speaks only the **Responses API** wire format (`model-provider-info/src/lib.rs:55`).
- **Not a single program.** The `codex` binary is a *multitool*: one executable that dispatches to a dozen subcommands and, via an `arg0` trick, also impersonates helper binaries like the Linux sandbox and the apply-patch tool (see [Frontends and protocol](10-frontends-and-protocol.md)).

## Where to go next

- To understand the engine, read [The agent loop](04-agent-loop.md).
- To understand safety, read [Execution and sandboxing](06-execution-and-sandboxing.md).
- For a term you don't recognize, check [Key concepts](03-key-concepts.md).
- To see how Codex stacks up against Claude Code and Grok Build, jump to the [Competitive analysis](14-competitive-analysis.md).
