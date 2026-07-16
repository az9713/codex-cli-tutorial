# Frontends and protocol

Codex's most important architectural decision is that the engine is not welded to any one interface. There is a shared **protocol**, and every surface — the terminal UI, the headless CLI, the IDE server, the SDKs, the MCP server — is a *client* of that protocol. This is why one Codex build powers a terminal, VS Code, a Python SDK, and a cloud connection with the same core. This page covers the protocol and every frontend. Paths are under `codex-rs/`.

## The shared protocol

Two protocol layers exist, and they serve different consumers.

**The core protocol** (`protocol/` crate) is the internal engine contract: the `Op` enum (~26 operations *into* the engine), the `EventMsg` enum (~80 events *out*), and the `ResponseItem`/`TurnItem` conversation models. This is the SQ/EQ vocabulary the [agent loop](04-agent-loop.md) runs on.

**The app-server protocol** (`app-server-protocol/` crate) is the *public* API — a JSON-RPC surface that external programs use. It has two versions: **v1** (deprecated, retaining only the mandatory `Initialize` handshake and a few legacy methods) and **v2** (the current API). V2 exposes a clean **thread → turn → item** hierarchy with slash-namespaced methods: `thread/start`, `turn/start`, `turn/steer`, `item/*` notifications, plus config, account, MCP, filesystem, and permissions surfaces. Server-originated requests (`item/commandExecution/requestApproval`, `mcpServer/elicitation/request`) let the engine ask the client for approvals.

A crucial engineering detail: **Rust is the single source of truth for the wire types.** Every protocol type derives both `ts_rs::TS` (TypeScript) and `schemars::JsonSchema` (JSON Schema), and a code-generation step (`app-server-protocol/src/export.rs`) emits vendored TypeScript and JSON-Schema fixtures that the SDKs consume. Change a Rust struct, regenerate, and the TS SDK's types update — there is no hand-maintained second copy to drift. Experimental methods and fields are gated behind an initialize capability and stripped from the default generated schemas.

## The app-server

The app-server (`app-server/`) is the runtime that hosts the engine and speaks the v2 protocol. It runs over three transports (`app-server-transport/`):

- **stdio** — one connection over stdin/stdout, used by IDE extensions that spawn `codex app-server` as a subprocess.
- **unix socket** — a control socket at `$CODEX_HOME/app-server-control/...`, used by the daemon.
- **WebSocket** — an axum server with health checks and bearer-token auth, used by desktop/remote clients (non-loopback listeners must be authenticated).

Internally it runs two tasks joined by channels: an outbound router (slow per-connection writes off the hot path) and a processor that dispatches each request to a domain-specific handler (`thread_processor`, `turn_processor`, `config_processor`, `account_processor`, and ~20 more). Requests carry a **serialization scope** so that same-scope requests (e.g. two operations on the same thread) run sequentially while independent requests run concurrently — a clean concurrency model baked into the protocol.

The app-server talks to the engine through the exact SQ/EQ boundary from the [agent loop](04-agent-loop.md): `turn_processor` submits `Op::UserInput`/`Op::Interrupt`/`Op::Review` into a `CodexThread`'s submission queue, and a per-thread listener drains the engine's `EventMsg`s, translating each into a v2 `ServerNotification` that fans out to subscribed connections (`app-server/src/thread_state.rs`, `event_mapping.rs`).

An optional **remote control** transport inverts the direction: the app-server dials *out* to a cloud relay and receives client sessions over it, so a cloud UI (or mobile) can drive a locally-running Codex. A **daemon** (`app-server-daemon/`, Unix only) manages a standalone Codex binary as a background process, with an auto-update loop that restarts it when a new version is installed.

## The TUI

The interactive terminal UI (`tui/`) is built on ratatui and — importantly — is itself an **app-server client**, not the engine. It connects to an app-server that is embedded in-process by default, or a local daemon, or a remote endpoint (`tui/src/lib.rs`). This is why the terminal UI, VS Code, and a remote session share behavior: they're all clients of the same server.

Internally the TUI is a single-threaded `tokio::select!` loop over four event sources: an internal `AppEvent` bus, the active thread's event channel, terminal input (crossterm), and inbound app-server events. It renders a scrolling transcript of history cells plus one in-flight active cell, a bottom pane holding the composer and any modal (approval overlay, picker, elicitation), and a status line. Rich features layered on top: streaming markdown that commits stable regions to scrollback while re-rendering a mutable tail (with special handling so a growing table doesn't reflow mid-stream), `@`-mention file search, a ~55-command slash registry, an onboarding flow (auth + trust-directory), resume/fork pickers, and a **backtrack** feature (double-Escape to rewind to an earlier user message and fork before it). Approvals arrive as server-requests and are rendered as overlays; the user's choice is submitted back as the matching `Op`.

## codex exec — headless mode

`codex exec` (`exec/` crate) is the non-interactive mode for CI and scripting. It reads a prompt (argument, positional, or piped stdin), runs one turn (or resumes/reviews), and exits with a non-zero code on failure. Output is either human-readable (final message to stdout, everything else to stderr) or **JSONL** (`--json`), where every event is a `ThreadEvent` line — `thread.started`, `turn.started`, `item.started/completed`, `turn.completed`, `error` — suitable for machine parsing. Flags like `--output-schema` (structured final response), `-o` (write last message to a file), `--skip-git-repo-check`, and `--yolo` (bypass approvals and sandbox) target automation.

A subtle architectural point: `codex exec` does not call the engine directly either — it starts an **in-process app-server** and drives it over the same protocol. So even the headless CLI is a protocol client. This uniformity is a deliberate design choice with real payoff: the behavior of a turn is identical whether it runs in the TUI, in CI, or via an SDK.

## The multitool binary and arg0 dispatch

The `codex` binary is a *multitool*. Its top-level `Subcommand` enum (`cli/src/main.rs`) dispatches ~25 subcommands: `exec`, `login`, `logout`, `mcp`, `plugin`, `mcp-server`, `app-server`, `resume`, `fork`, `apply`, `cloud`, `doctor`, `sandbox`, `features`, and more. With no subcommand, it launches the TUI.

Beneath that, an **arg0 trick** (`arg0/`) lets the one binary impersonate several helper executables by branching on `argv[0]`: invoked as `codex-linux-sandbox` it runs the sandbox helper, as `apply_patch` it runs the patch tool, as `codex-execve-wrapper` it runs the shell-escalation wrapper. On startup it also plants these aliases on `PATH` (via symlinks on Unix, `.bat` shims on Windows) so the agent can call `apply_patch` without a separate install. This is how Codex ships as a single self-contained binary that still behaves like a suite of tools.

## The SDKs

Two language SDKs drive Codex as a subprocess, and they made *different* architectural choices:

- **TypeScript** (`@openai/codex-sdk`) drives `codex exec --experimental-json` — one JSONL stream per turn. Its API is `new Codex()` → `startThread()`/`resumeThread()` → `thread.run(input)` or `thread.runStreamed(input)`. Simple, one-shot-per-turn.
- **Python** (`openai-codex`) drives `codex app-server --listen stdio://` — a long-lived JSON-RPC service. This gives it richer control: multi-thread, turn steering, interrupt, login, fork. Its API mirrors the app-server: `Codex()` → `thread_start()` → `thread.run()`/`thread.turn()` → `TurnHandle` with `steer`/`interrupt`/`stream`.

Both resolve the native binary from a vendored platform package. The difference in transport (exec vs app-server) is a good illustration of the protocol's flexibility — the same engine serves a simple one-shot SDK and a full interactive-control SDK.

## npm packaging

`@openai/codex` (`codex-cli/`) is the npm distribution. The root package ships only a Node launcher shim (`bin/codex.js`); the actual native binary comes from a per-platform optional dependency (`@openai/codex-<os>-<arch>`), of which npm installs only the one matching the host. The shim resolves the platform binary, spawns it with `stdio: "inherit"`, and forwards signals for correct exit-code semantics. There is no postinstall step — binary selection is entirely at runtime.

## Cloud integration

Codex has an optional connection to OpenAI's Codex Web (`backend-client/`, `cloud-tasks/`, `chatgpt/`). The `codex cloud` subcommand lists and browses tasks that ran in the cloud, and both `codex cloud apply` and `codex apply` fetch a task's diff and apply it **locally** with `git apply` — "apply" is never a server-side mutation. Task creation (`POST .../tasks`) is the only cloud write. The base URL is `https://chatgpt.com/backend-api` by default, and all cloud features require a ChatGPT login (API-key auth is explicitly disabled on these paths). Enterprise deployments additionally fetch a signed "cloud config bundle" that layers managed settings on top of local config (`cloud-config/`).

There is also a **voice/realtime** path (`realtime-webrtc/`, macOS-only) that opens a native WebRTC peer connection for the OpenAI Realtime API, and a hardened **responses-api-proxy** that injects an API key into requests so the key never lives in the calling process's environment.

## Design assessment

The protocol-first architecture is Codex's strongest structural decision. Making every frontend a client of a versioned, code-generated protocol means the TUI, IDE, CLI, and SDKs cannot silently diverge, and it enables surfaces — remote control, a daemon, two differently-shaped SDKs — that a UI-welded design could not. The cost is indirection: even the interactive terminal talks to an in-process JSON-RPC server, which is more machinery than a direct call and shows up as the large `app-server` and `tui` crates. See [Critique](12-critique.md) for the full assessment.
