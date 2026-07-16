# Codex CLI — Harness Documentation

Codex CLI is OpenAI's open-source local coding agent: a Rust workspace (~100 crates under `codex-rs/`) that wraps a language model in an **agent harness** — the loop, tools, sandbox, context manager, and frontends that turn a model into an autonomous coding assistant. This documentation explains that harness in depth, critiques it against its own code, and compares it feature-by-feature with Anthropic's Claude Code and xAI's Grok Build.

It is written for two readers: a new engineer who wants to understand Codex from scratch in the shortest time, and an experienced engineer deciding how to extend it or catch it up to its competitors.

---

## Documentation

| Section | What's inside |
|---------|---------------|
| [What is an agent harness?](01-what-is-a-harness.md) | The concept, its components, and how harnesses are implemented — the vocabulary you need before reading the rest |
| [Codex overview](02-codex-overview.md) | What Codex is, the mental model, the crate map, and how a request flows end to end |
| [Key concepts](03-key-concepts.md) | Glossary of every load-bearing term used across these docs |
| [The agent loop](04-agent-loop.md) | The core turn loop: submission/event queues, tasks, sampling, streaming, tool dispatch |
| [Tools and approvals](05-tools-and-approvals.md) | The tool system, schema generation, parallel execution, and the approval/safety gate |
| [Execution and sandboxing](06-execution-and-sandboxing.md) | Per-platform sandboxes (Seatbelt/Landlock/bwrap/Windows tokens), the network proxy, and the exec-server |
| [The model client](07-model-client.md) | Responses API wire client, SSE/WebSocket streaming, retry/backoff, prompt assembly |
| [Context, state, and memory](08-context-state-memory.md) | Rollout files, the SQLite state DB, compaction, the memory pipeline, AGENTS.md, file search |
| [Extensibility](09-extensibility.md) | Skills, hooks, plugins, MCP (client and server), code mode, connectors, external-agent migration |
| [Frontends and protocol](10-frontends-and-protocol.md) | The shared protocol, the TUI, `codex exec`, the app-server, the TS/Python SDKs, npm packaging |
| [Config, auth, and telemetry](11-config-auth-telemetry.md) | The config system, ChatGPT/API-key auth, model providers, OTEL/analytics/feedback, self-update |
| [Critique: strengths and weaknesses](12-critique.md) | An honest engineering critique of the harness, grounded in code references |
| [Improvements and new features](13-improvements-and-new-features.md) | Concrete upgrades to existing subsystems and net-new features to build on top |
| [Competitive analysis](14-competitive-analysis.md) | Codex vs Claude Code vs Grok Build: feature inventory, gaps, where each wins, future directions, and what Codex should adopt |

**New here?** Read [What is an agent harness?](01-what-is-a-harness.md) then [Codex overview](02-codex-overview.md). Together they take about 20 minutes and give you the mental model the rest of the docs build on.

---

## How this documentation was produced

Every claim about Codex is grounded in the source tree at `codex-rs/` and cites `crate/src/file.rs:line`. Claims about Claude Code and Grok Build are grounded in published documentation, changelogs, and repositories, and are marked where a fact rests on a single or lower-reliability source. Nothing here is invented; where a competitor's behavior could not be verified, the docs say so.
