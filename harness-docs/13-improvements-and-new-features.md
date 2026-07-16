# Improvements and new features

This page is the roadmap: concrete upgrades to existing subsystems, then net-new features to build on top of the codebase. Each item explains *what*, *why it matters*, and *where in the code* it lands. The improvements target the high- and medium-severity issues from the [Critique](12-critique.md); the new features are things the existing architecture makes cheap to add and that would move Codex ahead of its competitors.

Priorities are marked P0 (do first) to P2 (opportunistic).

## Part 1 — Improvements to existing features

### P0 — Decompose `run_turn` and the `Session` god-object

**What.** Extract the inline concerns in `run_turn` (`core/src/session/turn.rs:144`) — compaction, hooks, skills/plugins injection, connectors, analytics, token budgeting — into named collaborators behind small traits, and split `Session` (`core/src/session/mod.rs`, ~4,000 lines) along the same seams.

**Why.** This is the harness's biggest maintainability liability and the maintainers already flag it (`AGENTS.md:74`). The hottest path being the hardest to change safely is exactly where bugs concentrate. Decomposition also makes the feature-flag branches testable in isolation.

**Where.** `core/src/session/`, `core/src/tasks/`. A `TurnPipeline` composed of stages (`PreSamplingCompaction`, `ContextAssembly`, `HookStage`, `SamplingStage`) each with a focused unit test would replace the monolithic loop without changing behavior.

### P0 — Document and enforce the lock hierarchy

**What.** Write a single `LOCKS.md` stating the acquisition order of the major `Mutex`/`RwLock`s (`Session.state`, `active_turn`, `TurnState`, the trackers), and add debug-only lock-order assertions.

**Why.** Correct-but-undocumented lock discipline (comments like `tasks/mod.rs:518`) is a latent deadlock/race waiting for a well-meaning refactor. This is cheap insurance on a high-consequence risk.

**Where.** A doc plus a thin `parking_lot`-style ordered-lock wrapper used in debug builds.

### P0 — Sandbox external MCP servers

**What.** Run stdio MCP servers inside the same platform sandbox the shell tools use, defaulting to a read-only-plus-workspace profile, rather than trusting them to honor the forwarded `SandboxState` hint (`core/src/mcp_tool_call.rs:734`).

**Why.** Today a malicious or buggy MCP server runs with the full privileges of the Codex user — a real hole in an otherwise best-in-class sandbox story. The sandbox machinery already exists (`sandboxing/`, `exec-server/`); wiring MCP subprocess launch through it closes the gap.

**Where.** `rmcp-client/src/stdio_server_launcher.rs` already has an `ExecutorStdioServerLauncher` that runs through an exec backend — make the sandboxed path the default for untrusted servers.

### P1 — Sign plugins and skills

**What.** Add optional signature verification to the plugin/marketplace install path (`core-plugins/src/store.rs`), with an enterprise policy to require signatures.

**Why.** Plugins are a distribution mechanism with no cryptographic integrity today — trust is source-allowlist plus hook-hash, so a compromised git repo executes arbitrary code. Signing is table stakes for a supply chain.

**Where.** `core-plugins/src/marketplace_policy.rs` (add a `require_signed` constraint), `store.rs` (verify on install).

### P1 — Make skill triggering mechanical, not prose-enforced

**What.** Add a first-class `invoke_skill` tool (or a `Skill` exposure that the router understands) so that "use this skill" is a tool call the harness executes, not an instruction the model must obey (`core-skills/src/render.rs`, `injection.rs`).

**Why.** Skill triggering currently depends on prompt adherence, which is unreliable and gets worse as the skill catalog grows. Enforce product gating at the same time (the `TODO` at `core-skills/src/model.rs:55`).

**Where.** `core/src/tools/handlers/` (new handler), `core-skills/src/injection.rs` (resolve the invocation).

### P1 — Preserve more detail through compaction

**What.** Replace the fixed 20,000-token recent-user-message window (`core/src/compact.rs:589`) with a relevance-scored selection that keeps the current task's key tool outputs (test failures, error messages, the file being edited), not just recent user turns.

**Why.** On long, tool-heavy threads the agent forgets what it did earlier in the *same task*, forcing re-derivation. A cheap heuristic — keep the last N failing command outputs and the diffs of files touched this task — would materially improve long-run coherence.

**Where.** `core/src/compact.rs` (`build_compacted_history`), reusing the `TurnDiffTracker` signal.

### P1 — Retire the dual event schema

**What.** Complete the migration off `as_legacy_events` (`core/src/session/mod.rs:1763`) and the Legacy history mode (`rollout/src/policy.rs`), leaving only the item-lifecycle model.

**Why.** Emitting every event twice doubles event volume and maintenance surface. Once no consumer needs the legacy shape, removing it simplifies the hot path and the persistence policy.

**Where.** `core/src/session/mod.rs`, `rollout/src/policy.rs`, `protocol/src/legacy_events.rs`.

### P2 — Harden the Windows read-ACL race

**What.** Replace the async-read-grant + cwd-junction workaround (`windows-sandbox-rs/src/bin/command_runner/win/cwd_junction.rs`) with synchronous grant-before-spawn, accepting the latency cost.

**Why.** The current design merely *mitigates* a race on the security boundary. On a sandbox, correctness beats latency.

**Where.** `windows-sandbox-rs/src/bin/setup_main/win.rs` (make `ReadAclsOnly` blocking on the spawn path).

### P2 — Scrub credentials from feedback diagnostics

**What.** Redact proxy-env-var values in the feedback connectivity diagnostic (`feedback/src/feedback_diagnostics.rs`), which currently reports them verbatim including embedded credentials.

**Why.** A user uploading feedback with logs can leak `user:password@proxy` — an easy, high-value fix.

**Where.** `feedback/src/feedback_diagnostics.rs` (mask everything after `//` up to `@`).

## Part 2 — New features to build on top

These leverage architecture Codex already has, so each is cheaper than it looks.

### P0 — Agent teams (peer-to-peer multi-agent coordination)

**What.** Build a "team" mode where one lead session spawns multiple peer sessions that coordinate via a shared task list and message passing — the Claude Code "agent teams" model.

**Why it makes Codex better.** Codex already has the hard parts: multi-agent spawning (`core/src/tools/handlers/multi_agents_v2/`), the parent/child spawn graph (`agent-graph-store/`), inter-agent communication (`Op::InterAgentCommunication`), and per-thread rollouts. What's missing is the *coordination layer* — a shared task board and a mailbox — and a TUI to watch it. This is the single feature most likely to close the gap with Claude Code's headline capability, and Codex's existing spawn-graph and per-thread persistence make it a smaller lift than a green-field build.

**Where.** New crate `team-coordination` over `agent-graph-store` and the existing multi-agent tools; TUI panel in `tui/src/`.

### P0 — Session checkpoints and rewind

**What.** Add filesystem checkpoints so a user can rewind the *working tree* (not just the conversation) to an earlier turn — Claude Code's `/rewind`.

**Why it makes Codex better.** Codex tracks per-turn diffs in memory (`turn_diff_tracker.rs`) and has a git baseline mechanism (`git-utils/src/baseline.rs`) it uses for the memory workspace. Extending the baseline approach to snapshot the user's working tree per turn would give true undo — a frequently-requested capability that Codex conspicuously lacks (its old ghost-commit feature was removed). The building blocks exist; they just need to be pointed at the user's repo.

**Where.** `git-utils/src/baseline.rs` (generalize beyond the memory workspace), a new `Op::Rewind`, TUI backtrack UI (`tui/src/app_backtrack.rs`, which already forks *conversation* history).

### P1 — A GitHub Action / CI-native review bot

**What.** Ship a first-party GitHub Action that runs `codex exec` on a PR, posting review comments and optional fixes — the Claude Code GitHub App equivalent.

**Why it makes Codex better.** `codex exec` already has JSONL output, `--output-schema`, resumable sessions, and a `Review` task (`exec/`, `tasks/review.rs`). A thin Action wrapper turns Codex from a local tool into a CI participant, which is where a large fraction of coding-agent value now lives. Codex has the headless engine; it lacks the packaged CI on-ramp.

**Where.** A new `.github/` action template + a `codex review --format github` mode over `tasks/review.rs`.

### P1 — LSP-backed code intelligence

**What.** Add a code-intelligence tool that queries a language server for symbol navigation and live type diagnostics, so the model can "go to definition" and see type errors instead of grepping.

**Why it makes Codex better.** Both Claude Code and Grok Build have moved toward LSP integration (Grok Build's plugins can bundle `.lsp.json` servers). Codex's tool system (`core/src/tools/`) and its exec-server (`exec-server/`, which can already run helper processes) make hosting a language server and exposing its queries as a tool straightforward. Symbol-accurate navigation reduces wasted turns spent searching.

**Where.** New `core/src/tools/handlers/lsp.rs`, an LSP client crate, launched via the exec-server.

### P1 — Scheduled / triggered cloud agents

**What.** Let a user register a `codex exec` invocation to run on a cron schedule or a repository event, using the existing app-server daemon.

**Why it makes Codex better.** The daemon (`app-server-daemon/`) already runs Codex as a managed background process with an update loop. Adding a scheduler that fires `codex exec` jobs — nightly dependency bumps, "watch this branch's CI and fix failures" — extends Codex from interactive-only to autonomous, matching Claude Code's routines.

**Where.** `app-server-daemon/src/` (add a scheduler alongside the update loop), config `[schedules]`.

### P2 — A public tool/agent SDK for embedding Codex tools elsewhere

**What.** Expose the `CoreToolRuntime` trait and the sandbox as a standalone library so third parties can reuse Codex's sandbox + tool executors in their own agents.

**Why it makes Codex better.** Codex's sandbox and tool executors are its best assets. Packaging them for reuse (the way Anthropic packaged the Claude Agent SDK) would seed an ecosystem and make Codex's safety primitives the default in other tools — strategic leverage beyond the CLI itself.

**Where.** A new `codex-agent-sdk` crate re-exporting `tools/`, `sandboxing/`, and a minimal loop.

### P2 — Model-agnostic provider adapter

**What.** Add a translation shim so a Chat-Completions-only provider (Anthropic, Google, local Llama) can be used by mapping Responses requests to that provider's format.

**Why it makes Codex better.** The Responses-only decision (`model-provider-info/src/lib.rs:55`) locks Codex to OpenAI/Azure. A shim in the provider layer that back-translates for other providers would make Codex genuinely model-agnostic — a differentiator against Claude Code (Anthropic-only) and Grok Build (xAI-first), and an insurance policy against single-vendor dependence.

**Where.** `model-provider/` (a `ResponsesToChatCompletions` adapter), gated per provider.

## How to sequence this

The P0 improvements (decompose `run_turn`, document locks, sandbox MCP servers) reduce the risk of every subsequent change and should land first. The P0 new features (agent teams, checkpoints) are the highest-visibility gaps against competitors and reuse the most existing machinery. The P1/P2 items are ecosystem and reach plays that matter once the core is de-risked. The [competitive analysis](14-competitive-analysis.md) explains which of these close specific gaps against Claude Code and Grok Build.
