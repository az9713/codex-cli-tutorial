# Critique: strengths and weaknesses

This is an honest engineering assessment of the Codex harness, grounded in the code. It is organized by subsystem, then by cross-cutting theme. Every claim cites a source location so you can check it. The goal is not to praise or condemn but to tell a maintainer where the harness is strong, where it is fragile, and why.

## What Codex does exceptionally well

**Sandboxing is best-in-class.** Three real OS backends — Seatbelt, Linux Landlock/seccomp/bwrap, Windows restricted tokens — plus a filtering network proxy with **credential brokering** (`network-proxy/`, the dummy-secret swap that means the model never sees your real GitHub token) and **post-quantum secure remote execution** (`exec-server/`, the `Noise_hybridIK_X25519+MLKEM768` channel). No competitor documents a network-hardening story this deep. The bundled, SHA256-verified `bwrap` fallback (`linux-sandbox/src/bundled_bwrap.rs`) means the sandbox works even on machines without bubblewrap installed. This is the harness's crown jewel.

**The protocol-first architecture is the right structural bet.** Every frontend — TUI, `codex exec`, IDE, SDKs — is a client of a versioned, code-generated JSON-RPC protocol (`app-server-protocol/`). Rust is the single source of truth; TypeScript and JSON-Schema types are generated from it (`export.rs`), so the SDK types cannot drift. This enables surfaces a UI-welded design could not: remote control, a daemon, and two differently-shaped SDKs from one engine.

**The core loop's concurrency model is elegant.** Tool execution overlaps streaming (futures started eagerly as items complete) yet outputs are recorded deterministically in stream order (`FuturesOrdered` + `drain_in_flight`, `core/src/session/turn.rs:1908`). Parallel-tool concurrency is expressed as a single `RwLock` read/write gate (`tools/parallel.rs:133`) rather than ad-hoc locking. Approvals are oneshot channels keyed by id in `TurnState` (`state/turn.rs:87`), cleanly decoupling minute-long human round-trips from tool control flow.

**Persistence fails safe.** JSONL rollouts are the source of truth; the five SQLite DBs are a rebuildable index (`state/`, backfill in `rollout/src/metadata.rs`), so DB corruption never loses work. The writer buffers unwritten suffixes and retries (`recorder.rs:1622`), and reverse-scan ordinal recovery avoids full re-reads on resume.

**The memory pipeline is genuinely advanced.** A leased, watermarked, idempotent extract → consolidate → inject loop (`memories/`), with secret redaction and a *sandboxed, network-less consolidation sub-agent*. This is a real memory system, not a static file — more sophisticated than any competitor's.

## Where Codex is fragile

### The `core` god-object

The maintainers' own `AGENTS.md:74` says *"resist adding code to codex-core"* — an admission that the crate is bloated. The numbers bear it out: `core/src/session/mod.rs` is ~4,000 lines, `turn.rs` ~2,500, `client.rs` ~2,400, and `Session` is a god-object with a huge method surface (`core/src/session/`). `run_turn` (`turn.rs:144`) mixes compaction, hooks, skills/plugins injection, connectors, analytics, and token budgeting inline, carrying multiple `#[allow(clippy::too_many_arguments)]`. This is the single biggest maintainability liability: the hottest code path is also the hardest to modify safely.

### Combinatorial feature flags

Most subsystems branch on `codex_features::Feature` flags — `DeferredExecutor`, `TokenBudget`, `RemoteCompactionV2`, `ConcurrentReasoningSummaries`, `MemoryTool`, `CodeMode`, `MultiAgentVersion`, and dozens more. The result is combinatorial behavior: the set of code paths that can be simultaneously active is enormous, much of it lightly exercised. A concrete symptom is that a feature's plumbing can ship enabled while its user-facing surface is gated off (`code_mode_host` Stable/on, `code_mode` under-development/off — `features/src/lib.rs`), so "what does the model actually see" diverges from "what the code contains."

### Windows sandboxing is inherently precarious

Because Windows has no Landlock/Seatbelt equivalent, `windows-sandbox-rs/` improvises with restricted tokens and per-file ACLs, and the seams show: the unelevated backend *cannot* enforce deny-read (`lib.rs:539`), full-disk-write profiles are unenforceable (`resolved_permissions.rs:49`), read grants are applied asynchronously with a race merely *mitigated* by a cwd junction (`command_runner/win/cwd_junction.rs`), the elevated backend needs a UAC prompt, and firewall setup fails closed if group policy overrides it. It is impressive engineering around a missing primitive, but it is the most brittle part of the harness.

### Two-schema and dual-path migration warts

Several places carry a migration in progress:

- **Legacy vs Paginated history modes** create two persistence code paths in `rollout/src/policy.rs`.
- **Dual event emission** — every event is emitted in both the new and legacy schemas via `as_legacy_events` (`core/src/session/mod.rs:1763`), doubling event volume.
- **Deprecated `cwd`** on `TurnContext` (`turn_context.rs`) coexists with the new multi-environment model.
- **v1 vs v2 app-server protocol** — v1 lingers for the `Initialize` handshake and a few legacy methods (`app-server-protocol/src/protocol/v1.rs`).

None is broken, but each is a maintenance surface that must be kept consistent.

### Extensibility relies on prose, not enforcement

Skill triggering has no dedicated tool or slash mechanism in the core crate — it depends on the model obeying the injected how-to text (`core-skills/src/render.rs`), a prompt-adherence risk. Product gating is parsed but not enforced (`TODO` at `core-skills/src/model.rs:55`). External MCP servers are **not sandboxed** — a stdio server runs with the Codex user's privileges, and isolation depends on the server voluntarily honoring a forwarded `SandboxState` hint (`core/src/mcp_tool_call.rs:734`). Plugins have **no cryptographic signing** — trust is source-allowlist plus hook-hash, so a trusted git clone executes arbitrary repo content.

### Compaction is lossy by construction

Summarization compaction keeps only ~20,000 tokens of recent user messages plus one assistant summary (`core/src/compact.rs:589`), so mid-thread tool detail is discarded unless the model re-derives it. This is a reasonable tradeoff, but on long, tool-heavy threads it means the agent can "forget" specifics of what it did earlier in the same task.

## Cross-cutting themes

**Surface area is the recurring cost.** Nearly every strength — three sandbox backends, five SQLite DBs, a rollout+trace+history+index persistence quartet, a protocol with v1/v2 and legacy events, a feature-flag matrix — buys capability by adding surface to keep consistent. Codex is a *large* system, and its complexity is not accidental fat but the price of doing hard things thoroughly. The risk is that the complexity is now high enough that no single engineer holds the whole harness in their head, which is exactly the condition under which subtle inconsistencies (a lock-ordering bug, a feature-flag interaction, a Windows race) slip in.

**Lock discipline is load-bearing and under-documented.** Heavy `Arc<Mutex>`/`RwLock` sharing across `Session`, `TurnContext`, and the various trackers is correct today, but several comments flag ordering subtleties (e.g. clearing pending approvals after cancellation, `tasks/mod.rs:518`). There is no single document that states the lock hierarchy, so a future change could deadlock or race.

**The Responses-only bet narrows the ecosystem.** Removing Chat Completions (`model-provider-info/src/lib.rs:55`) simplified the client and unlocked Responses-only features, but it also means only OpenAI and Azure can serve Codex out of the box; the local OSS providers must speak Responses-shaped requests. A harness that wanted to be genuinely model-agnostic could not have made this choice.

## Severity summary

| Issue | Severity | Why |
|-------|----------|-----|
| `core` god-object / `run_turn` complexity | High | Hottest path, hardest to change; maintainers acknowledge it |
| Windows sandbox fragility | High | Race-prone, several unenforceable cases, on the security boundary |
| Combinatorial feature flags | Medium | Testability and reasoning cost; shipped-but-gated divergence |
| External MCP servers unsandboxed | Medium | Real trust hole, mitigated only by cooperation |
| Plugins unsigned | Medium | Supply-chain risk on a distribution mechanism |
| Compaction detail loss | Medium | Degrades long-task performance |
| Dual-path / two-schema warts | Low | Consistency burden, not correctness bugs |
| Undocumented lock hierarchy | Low–Medium | Latent risk, no current failure |

The improvements in [Improvements and new features](13-improvements-and-new-features.md) target the high- and medium-severity items directly.
