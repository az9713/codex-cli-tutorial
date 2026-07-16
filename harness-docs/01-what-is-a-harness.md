# What is an agent harness?

An agent harness is the software around a language model that turns "a model that predicts text" into "an agent that does work." The model contributes intelligence; the harness contributes everything else — the loop that keeps calling the model, the tools the model can invoke, the sandbox that contains what those tools do, the memory of what happened, and the interface a human uses to watch and steer.

If you have used Codex, Claude Code, or Grok Build, you have used a harness. This page explains what one is made of and how the pieces are implemented, so the rest of these docs — which go deep on Codex's specific implementation — have a shared vocabulary to build on.

## The problem a harness solves

A raw language model is a function: text in, text out. It has no memory between calls, cannot run a command, cannot read a file, and cannot stop itself when it is done. To make it *build software*, you must wrap it in a program that:

1. Sends the model the user's request plus enough context to act.
2. Lets the model ask to run tools (read a file, run a shell command, edit code), executes those tools, and feeds the results back.
3. Repeats until the model signals it is finished.
4. Keeps the whole thing safe, observable, and resumable.

That wrapper is the harness. The model is swappable; the harness is the product.

## The mental model: a conversation that runs commands

The cleanest way to picture a harness is a **turn loop**. A *turn* is one user request and everything the agent does to satisfy it. Inside a turn:

```
user request
  → build a prompt (instructions + context + history + the request)
  → call the model (streaming)
  → model emits text and/or tool calls
      → for each tool call: check it is allowed, run it (maybe in a sandbox), capture output
      → feed tool outputs back into the conversation
  → call the model again with the new outputs
  → ... repeat until the model stops asking for tools ...
  → final answer
```

The loop is the beating heart. Everything else is a component that the loop calls into.

## The eight components of a harness

Every serious coding agent — Codex, Claude Code, Grok Build — is built from the same eight components. They differ in *how well* each is implemented, not in *which* exist.

| Component | Job | Codex's implementation (see) |
|-----------|-----|------------------------------|
| **Agent loop** | Drive turns; call the model; dispatch tool calls; decide when a turn ends | [The agent loop](04-agent-loop.md) |
| **Tool system** | Define the actions the model can take; generate their schemas; execute them; return results | [Tools and approvals](05-tools-and-approvals.md) |
| **Sandbox / execution** | Run shell commands and file edits under OS-enforced restrictions so a mistake cannot wreck the machine | [Execution and sandboxing](06-execution-and-sandboxing.md) |
| **Model client** | Speak the provider's wire protocol; stream responses; retry failures; count tokens | [The model client](07-model-client.md) |
| **Context manager** | Assemble the prompt; inject project docs and environment; compact history when it overflows; remember across sessions | [Context, state, and memory](08-context-state-memory.md) |
| **Extensibility** | Let users add capabilities without forking: skills, hooks, plugins, external tool servers | [Extensibility](09-extensibility.md) |
| **Frontends / protocol** | The surfaces a human or another program uses to talk to the loop — TUI, headless CLI, IDE server, SDK | [Frontends and protocol](10-frontends-and-protocol.md) |
| **Config / auth / telemetry** | Load settings; authenticate to the provider; measure what happens | [Config, auth, and telemetry](11-config-auth-telemetry.md) |

The rest of this page walks each component at the concept level.

### 1. The agent loop

The loop is usually built around two queues: a **submission queue** the frontend writes into (user messages, approvals, interrupts) and an **event queue** the loop writes out of (streamed text, tool-call notifications, completion). Decoupling the two with queues means the interface and the engine run on separate schedules — you can type while the agent works, and one engine can serve many frontends.

Inside the loop, one turn is a unit of work. A well-built loop makes a turn cancellable (the user hits Escape), preemptible (a new message replaces the running turn), and resumable (the machine crashes; you pick up where you left off). Codex models this explicitly with a `SessionTask` trait and a submission/event channel pair — see [The agent loop](04-agent-loop.md).

### 2. The tool system

Tools are the agent's hands. Each tool has a **name**, a **JSON schema** describing its arguments, and an **executor**. The harness serializes the schemas into the model request (in OpenAI's world, the `tools` array), the model responds with a tool call, and the harness routes that call to the executor.

Design questions every harness must answer:

- **What are the built-in tools?** At minimum: run a shell command, read/edit a file, search the codebase. Optionally: web search, image viewing, a plan/todo tool, sub-agent spawning.
- **How are tool calls approved?** Auto-run everything (fast, dangerous), ask the human each time (safe, slow), or something adaptive.
- **Can tools run in parallel?** Reading three files at once is safe; running two `git commit`s is not. The system needs to express which tools are concurrency-safe.

### 3. The sandbox

A coding agent runs arbitrary commands the model invents. The sandbox is what stops `rm -rf /` — or a subtler mistake — from being catastrophic. Sandboxing is OS-specific and hard:

- **macOS** uses Seatbelt (`sandbox-exec`) profiles.
- **Linux** uses Landlock (filesystem) + seccomp (syscalls), often wrapped by Bubblewrap for mount namespaces.
- **Windows** has no equivalent primitive, so harnesses improvise with restricted tokens and per-file ACLs.

A good sandbox restricts the filesystem (writable only inside the workspace) *and* the network (no exfiltration), while still letting legitimate work through. The tension is real: too tight and the agent can't install a dependency; too loose and it can leak your secrets. Codex's answer — three OS backends plus a filtering network proxy — is one of the most thorough in the field (see [Execution and sandboxing](06-execution-and-sandboxing.md)).

### 4. The model client

The client speaks the provider's HTTP/WebSocket protocol. The subtleties are all in the details: **streaming** (parsing server-sent events into typed deltas so the UI updates live), **retry/backoff** (models rate-limit and disconnect constantly), **token accounting** (knowing when you are about to overflow the context window), and **reasoning** (newer models emit hidden chain-of-thought that must be round-tripped without being shown to the user).

### 5. The context manager

The model's context window is finite. The context manager decides what goes into every request: the system prompt (who the agent is), project instructions (an `AGENTS.md` or `CLAUDE.md` file), environment facts (cwd, OS, sandbox mode), the conversation history, and the current request. When history grows past the window, it **compacts** — summarizes older turns to reclaim space. Advanced harnesses add **memory**: durable facts extracted from past sessions and re-injected into new ones.

### 6. Extensibility

No harness can ship every capability, so all of them expose extension points:

- **Skills** — markdown files (`SKILL.md`) with a description and instructions the model reads on demand. Cheap, because only the one-line description sits in context until the skill is used.
- **Hooks** — user scripts that fire at lifecycle events (before a tool runs, after a turn ends) and can block, modify, or annotate.
- **Plugins / marketplaces** — packaged bundles of skills, hooks, and tool servers, installed from a catalog.
- **MCP (Model Context Protocol)** — an open standard for tool servers. A harness that is an MCP *client* can use any MCP server's tools; one that is an MCP *server* can be driven by any MCP client.

### 7. Frontends and protocol

The loop needs a way to talk to the outside world. The best-architected harnesses define a **protocol** — a typed request/response contract — and then build every frontend as a *client* of that protocol: the interactive terminal UI, the headless CLI for CI, the JSON-RPC server that IDE extensions connect to, the language SDKs. This is why Codex's terminal UI is a thin app-server client rather than the engine itself (see [Frontends and protocol](10-frontends-and-protocol.md)).

### 8. Config, auth, and telemetry

Mundane but load-bearing. Configuration layers (system → user → project → session) determine behavior. Authentication proves you're allowed to call the model (an OAuth login or an API key). Telemetry (opt-in, ideally) tells the maintainers what's slow or broken. Self-update keeps the binary current.

## How harnesses are implemented in practice

Three implementation choices dominate the design of a harness, and the three products in these docs made different bets:

**Language.** Codex and Grok Build are written in **Rust** — chosen for a single self-contained binary, memory safety while spawning untrusted subprocesses, and performance in a long-running TUI. Claude Code is written in **Node/TypeScript** — chosen for fast iteration and the npm ecosystem. The tradeoff is stark: Rust gives you a fast, safe, dependency-free binary at the cost of slower iteration; Node gives you the reverse.

**Protocol boundary.** A harness can bake the loop directly into the UI (simplest) or split them with an internal protocol (more work, more flexible). Codex splits aggressively: the engine is an *app-server* speaking JSON-RPC, and every frontend — TUI, IDE, SDK — is a client. This is why one Codex build powers a terminal, VS Code, and a Python SDK.

**Tool interface.** The classic design gives the model one tool call per turn. A newer design — **code mode** — gives the model a JavaScript sandbox where it writes a script that calls many tools in a loop, cutting round-trips. Codex ships an experimental code-mode built on an embedded V8 engine (see [Extensibility](09-extensibility.md)).

## What good looks like

The quality of a harness is not the length of its feature list; it is how the components compose under pressure. A great harness:

- **Fails safe.** A sandbox that fails closed, an approval flow that defaults to deny, a hook contract where a crash blocks rather than silently passes.
- **Stays observable.** Every turn recorded, every tool decision logged, every token counted, so a failure can be diagnosed.
- **Degrades gracefully.** WebSocket drops to HTTP; a missing sandbox binary falls back to a bundled one; an out-of-process worker that dies is respawned.
- **Is resumable.** You can crash, close the laptop, or `/clear` and pick the session back up.
- **Extends without forking.** New capability arrives as a skill, a plugin, or an MCP server — not a code change.

With that vocabulary in hand, [Codex overview](02-codex-overview.md) shows how Codex assembles these eight components into a working system.
