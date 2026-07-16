# Context, state, and memory

Two problems define this subsystem. First, the model's context window is finite, so Codex must decide what goes into every request and what to do when history overflows. Second, a session must survive a crash, a `/clear`, or a restart, so Codex must record everything durably and be able to resume or fork. This page covers prompt assembly, the persistence layer, compaction, and the memory pipeline. Paths are under `codex-rs/`; `CODEX_HOME` is `~/.codex`.

## Prompt assembly

Every model request is built from two distinct channels, and confusing them is the most common mistake when reading this code:

1. **Base/system instructions** → the Responses API top-level `instructions` field. This is "who the agent is."
2. **Context fragments** → developer/user *messages* injected into the input array each turn. This is "what the agent needs to know right now."

### Base instructions

The system prompt is per-model. The fallback lives at `protocol/src/prompts/base_instructions/default.md` — the full "You are a coding agent running in the Codex CLI…" prompt covering planning, task execution, validation, and final-answer formatting. Per-model prompts are embedded in the model registry (`models-manager/models.json`), and a model may carry an `instructions_template` with a `{{ personality }}` placeholder that is substituted with the chosen personality (`Friendly`, `Pragmatic`, or none) before use (`protocol/src/openai_models.rs:476`). Resolution priority at session start (`core/src/session/mod.rs:595`): an explicit config override → instructions recovered from a resumed conversation → the model's default instructions.

### Context fragments, in order

Everything else is assembled per turn into three buckets — developer sections, contextual-user sections, and separate developer sections — and packed into messages. The effective order the model sees is:

1. **API `instructions`** — base prompt plus baked-in personality.
2. **Developer bundle** (one message): model-switch notice → permissions instructions (sandbox + approval policy) → the user's custom developer instructions → collaboration-mode instructions → personality spec (if not already baked in) → skills catalog → token-budget guidance → apps/plugins instructions.
3. **Separate developer messages** — e.g. the guardian policy.
4. **Contextual-user bundle** (one message): AGENTS.md instructions → `<environment_context>` → recommended plugins.
5. **Conversation history / the user's new turn.**

Every block implements `ContextualUserFragment` (`context-fragments/`): `render()` wraps a body in start/end markers, and those markers double as recognizers so the diff logic can detect and replace a previously-injected fragment across turns without rewriting history.

### The environment context

The `<environment_context>` block (`core/src/context/world_state/environment.rs`) tells the model where it is: cwd, shell, current date, timezone, and structured `<network>` (allowed/denied domains) and `<filesystem>` (workspace roots, permission profile, per-entry read/write/deny access) blocks. Notably, approval policy and sandbox mode are *not* here — they are rendered separately as permissions instructions.

### AGENTS.md discovery

Project instructions come from `AGENTS.md` (`core/src/agents_md.rs`). Discovery walks *up* from cwd to the project root (defined by `project_root_markers`, default `.git`), collects candidate files root-to-cwd, and prefers `AGENTS.override.md` over `AGENTS.md` per directory. Files are merged root-first, byte-budgeted (default 32 KiB, `0` disables), and injected as a **user-role** message wrapped in `# AGENTS.md instructions` / `<INSTRUCTIONS>` markers. When the set of applicable AGENTS.md files changes mid-session, a replacement notice ("These AGENTS.md instructions replace all previously provided…") is emitted so the model updates cleanly. Codex reads `AGENTS.md` natively; it does not read `CLAUDE.md` (though its migration tool imports one — see [Extensibility](09-extensibility.md)).

## Persistence: rollouts and SQLite

Codex persists at two layers with a clear hierarchy: **JSONL rollout files are the source of truth; SQLite is a derived, rebuildable index.**

### Rollout files

A rollout is the durable, append-only, human-readable recording of a thread, at `~/.codex/sessions/YYYY/MM/DD/rollout-<timestamp>-<uuid>.jsonl`. Each line is a `RolloutLine` — a timestamp, an optional ordinal, and a flattened `RolloutItem`: `SessionMeta` (always the first line: ids, cwd, model, git info), `ResponseItem` (the model-visible turn items), `TurnContext`, `WorldState` (snapshots for resumable world-state diffing), `Compacted` (compaction checkpoints), and lifecycle `EventMsg`s. A background worker compresses cold sessions to `.jsonl.zst`, and readers handle both transparently (`rollout/src/`).

The writer (`RolloutRecorder`) runs on a Tokio task with a bounded channel. File creation is deferred until the first write. On an I/O error it keeps the unwritten suffix buffered and retries on the next barrier — crash-safe by design. Resume re-opens the same file for append; **fork** starts a *new* rollout file whose `SessionMeta.forked_from_id` points at the source, with fork modes to truncate before the nth user message or preserve an interrupted turn (`thread_manager.rs:141`).

### The SQLite state DBs

Five bundled SQLite files under `CODEX_HOME` hold the derived index and coordination state (`state/`):

| File | Holds |
|------|-------|
| `state_5.sqlite` | The `threads` table (fast list/search index projected from rollouts), the `thread_spawn_edges` multi-agent graph, and the `agent_jobs` batch queue |
| `thread_history_1.sqlite` | The paginated history projection — per-turn and per-item rows keyed by rollout byte offset, so a UI can page through huge threads without parsing the whole file |
| `logs_2.sqlite` | Structured tracing sink |
| `goals_1.sqlite` | Long-running goal tracking with token/time budgets |
| `memories_1.sqlite` | The memory extraction/consolidation job queue |

Because the state DB is a projection, corruption is non-fatal: a **backfill** worker (`rollout/src/metadata.rs`) rescans the sessions directory and rebuilds `threads` rows, self-healing and catching files written by other processes. Listing tries the DB first and falls back to a filesystem scan-and-repair.

## Compaction

When history nears the context window, Codex compacts. The trigger (`core/src/session/context_window.rs`) fires after model sampling when active tokens cross the model's `auto_compact_token_limit` (or hit the hard context cap). There are three implementations:

1. **Summarization compaction** (`core/src/compact.rs`) — the classic. Runs a model turn with a "context checkpoint" prompt that produces a handoff summary, then builds new history from the most recent user messages (up to 20,000 tokens) plus the summary. On a context-exceeded error mid-compaction it drops the oldest item and retries, preserving the prefix cache.
2. **Token-budget compaction** (`core/src/compact_token_budget.rs`) — skips the model entirely and just starts a fresh window with rebuilt initial context. Cheaper, but loses detail.
3. **Remote compaction** (`core/src/compact_remote_v2.rs`) — when the provider supports it (OpenAI/Azure), the server performs summarization and returns a checkpoint.

Compaction is itself recorded as a rollout item, so windows chain by UUID and survive resume. The tradeoff is inherent: summarization keeps only ~20k tokens of recent user messages plus one assistant summary, so mid-thread tool detail is lost unless the model re-derives it.

## Memory

Memory is durable knowledge carried across sessions — distinct from within-session history. The pipeline (`memories/` crate, gated by `Feature::MemoryTool`) has three stages, all leased and watermarked so they run safely across concurrent processes:

**Phase 1 — per-rollout extraction.** On root session start (async, non-blocking, rate-limit-guarded), it claims eligible past rollouts, filters each to memory-relevant items (with secret redaction), and calls a model with a strict JSON schema to extract `{raw_memory, rollout_summary, slug}`, storing the result in `memories_1.sqlite`.

**Phase 2 — global consolidation.** Under a single global lock, it selects the top memories by usage and recency, rebuilds filesystem artifacts (`~/.codex/memories/MEMORY.md`, `raw_memories.md`, `rollout_summaries/`), and diffs the workspace against a git baseline. If anything changed, it spawns an internal **consolidation sub-agent** — running with no approvals, no network, and local-write-only — to update `MEMORY.md` and `memory_summary.md`.

**Injection.** In a new session, Codex renders a developer-instructions block containing the `memory_summary.md` contents plus guidance telling the model to do a "quick memory pass" — grep `MEMORY.md`, open a summary or two, search the cited rollout for evidence. The memories directory is added as a sandbox-readable root. So injection is *guidance plus a summary*, not bulk content; the model pulls detail on demand. Usage and citations are telemetered so the pipeline knows which memories earn their keep.

This is the most sophisticated memory system among the three harnesses in these docs — a genuine extract/consolidate/inject loop rather than a static file — but it costs extra model calls at startup and its quality depends on the extraction model (the read prompt itself warns that memory may be stale).

## Supporting subsystems

- **File search / @-mentions** (`file-search/`) — a `nucleo` fuzzy matcher over a gitignore-aware `ignore`-crate walk, streamed into the TUI's `@`-mention popup.
- **File watcher** (`file-watcher/`) — a `notify`-crate wrapper with a multi-subscriber model, used to hot-reload config and skills.
- **Turn diff** (`core/src/turn_diff_tracker.rs`) — the net unified diff of a turn, computed purely in memory from apply_patch deltas and shown in the UI. There is **no live per-turn git ghost-commit/undo** — that legacy feature was removed and its rollout lines are stripped on read.
- **Message history** (`message-history/`) — a global `~/.codex/history.jsonl` of prompts, shell-history-style, independent of rollouts.
- **Rollout trace** (`rollout-trace/`) — a separate, opt-in observability bundle for debugging inference attempts and tool dispatch.

## Design assessment

The JSONL-canonical / SQLite-derived split is the right call: the DB is a rebuildable cache, so corruption never loses your work, and the crash-safe deferred writer plus reverse-scan ordinal recovery make resume robust. The memory pipeline is genuinely advanced. The cost is surface area — five SQLite files plus JSONL plus a trace bundle plus a history file plus a session index is a lot of moving parts to keep consistent — and two persistence code paths (Legacy vs Paginated history modes) that both have to be maintained. See [Critique](12-critique.md) for the full assessment.
