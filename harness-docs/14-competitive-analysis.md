# Competitive analysis: Codex vs Claude Code vs Grok Build

This is a feature-by-feature comparison of the three leading coding-agent harnesses as of mid-2026: OpenAI **Codex CLI**, Anthropic **Claude Code**, and xAI **Grok Build**. It inventories each product's features, compares them dimension by dimension, identifies what exists in one but not the others, calls out where each harness is superior and why, and — grounded strictly in known facts — infers each one's likely direction.

**Sourcing discipline.** Codex claims are grounded in its source tree (this documentation set). Claude Code claims come from Anthropic's docs, changelog, and engineering blog. Grok Build claims come from its GitHub repo, xAI docs, and reputable writeups, and are marked where they rest on a single or lower-reliability source. Where a competitor's behavior could not be verified, it is labeled *unverified*. Nothing here is invented.

## The three products in one paragraph each

**Codex CLI** is OpenAI's open-source (Apache-2.0) local coding agent, ~100 Rust crates, driven by the Responses API. It runs locally as a self-contained binary with a protocol-first architecture (every frontend is a client of an internal JSON-RPC app-server), best-in-class multi-OS sandboxing plus a credential-brokering network proxy, an advanced memory pipeline, and deep Claude-Code compatibility (reads `AGENTS.md`/`CLAUDE.md`, runs Claude-compatible hooks, imports Claude/Cursor config). It has an optional cloud arm (Codex Web) but is fundamentally a local tool.

**Claude Code** is Anthropic's closed-source Node/TypeScript coding agent, the most broadly-distributed of the three: CLI, VS Code and JetBrains extensions, desktop app, web (claude.ai/code), mobile monitoring, a GitHub Action/App, Slack and Chrome integrations, and a first-class **Agent SDK** (TS + Python) that exposes the same loop as a library. It leads on **surface breadth**, **agent teams** (peer multi-agent coordination), **cloud sandboxes** with session teleport, and **enterprise provider support** (Bedrock/Vertex/Foundry). It runs on Anthropic models only.

**Grok Build** is xAI's open-source-but-closed-development (Apache-2.0, no external contributions) Rust coding agent + full-screen mouse-interactive TUI, launched May 2026. It is terminal-native with headless and ACP-based IDE-embedding modes, notable for **git-worktree-per-subagent** parallelism, interactive **plan editing** (approve/comment/rewrite individual steps), an official plugin marketplace with LSP-server support, and drop-in reuse of `AGENTS.md`/`CLAUDE.md`/MCP. It runs on Grok models (launched on `grok-build-0.1`, later Grok 4.5). It suffered a serious data-exfiltration incident in v0.2.93 (see the safety note below).

## Feature inventory and comparison

Legend: ✅ present · ⚠️ partial/limited · ❌ absent · ? unverified.

### Interaction surfaces

| Surface | Codex | Claude Code | Grok Build |
|---------|:-----:|:-----------:|:----------:|
| Terminal CLI/TUI | ✅ ratatui | ✅ | ✅ full-screen, mouse-interactive |
| Headless / `-p` mode | ✅ `codex exec`, JSONL | ✅ `claude -p`, text/json/stream-json | ✅ `grok -p`, streaming-json |
| VS Code extension | ✅ (via app-server) | ✅ native diff, bundles CLI | ⚠️ via ACP (editors unverified) |
| JetBrains plugin | ⚠️ (IDE docs) | ✅ | ? |
| Desktop app | ✅ `codex app` (macOS/Win) | ✅ (macOS/Win) | ❌ |
| Web UI | ✅ Codex Web (cloud) | ✅ claude.ai/code | ❌ |
| Mobile | ❌ | ✅ monitor/steer cloud sessions | ❌ |
| GitHub Action / App | ⚠️ IDE docs mention; no first-party Action found in repo | ✅ Action + App + auto-fix PRs | ❌ |
| Slack integration | ❌ | ✅ "Claude in Slack" | ❌ |
| Browser automation | ❌ | ✅ Claude in Chrome (GA) | ❌ |
| Language SDK | ✅ TS + Python | ✅ Agent SDK (TS + Python) | ❌ (ACP for embedding) |

**Superiority: Claude Code, decisively.** It is available on more surfaces than the other two combined, and the mobile-monitors-cloud-session and Slack/Chrome integrations have no equivalent.

### Agent loop and tools

| Capability | Codex | Claude Code | Grok Build |
|-----------|:-----:|:-----------:|:----------:|
| Shell execution | ✅ shell + unified-exec PTY | ✅ Bash + background | ✅ |
| File edit | ✅ apply_patch (grammar) | ✅ Read/Write/Edit | ✅ |
| Codebase search | ✅ nucleo/ripgrep | ✅ Glob/Grep (ripgrep) | ✅ |
| Web search | ✅ hosted | ✅ WebSearch/WebFetch | ✅ + live X search |
| Plan/todo tool | ✅ update_plan | ✅ TodoWrite | ✅ + interactive plan editing |
| Plan mode (read-only) | ✅ collaboration modes | ✅ `--permission-mode plan` | ✅ (headline feature) |
| Sub-agents | ✅ v1/v2, spawn graph | ✅ background, 5 levels deep | ✅ worktree-per-agent |
| Parallel tool calls | ✅ RwLock gate | ✅ | ✅ |
| Clarifying questions | ✅ request_user_input | ✅ AskUserQuestion | ✅ multiple-choice |
| Code-mode (JS tool interface) | ✅ V8 (experimental) | ❌ | ❌ |
| Background tasks survive restart | ⚠️ | ✅ (incl. Windows) | ⚠️ |

**Superiority: mixed.** Codex is unique in **code mode** (programmatic V8 tool calls). Grok Build leads on **interactive plan editing** (approve/comment/rewrite individual steps). Claude Code leads on **background-task durability** and depth of sub-agent lifecycle.

### Sandboxing and security

| Capability | Codex | Claude Code | Grok Build |
|-----------|:-----:|:-----------:|:----------:|
| macOS Seatbelt | ✅ | ✅ | ? (mechanism undetailed) |
| Linux Landlock/seccomp/bwrap | ✅ (+ bundled bwrap) | ✅ Bubblewrap | ? |
| Windows sandbox | ✅ restricted tokens + ACLs + WFP | ⚠️ | ⚠️ Windows best-effort |
| Network egress filtering | ✅ proxy w/ domain allowlist | ✅ UDS→proxy allowlist | ? |
| Credential brokering (model never sees real secrets) | ✅ dummy-swap MITM | ⚠️ block secret reads | ❌ |
| Secure remote execution | ✅ post-quantum Noise channel | ✅ cloud sandbox | ? |
| Devcontainer support | ✅ | ✅ | ? |
| Permission modes | ✅ 4 policies + granular | ✅ 5 modes + Auto | ⚠️ ask/always-approve (+plan/code, naming varies) |

**Superiority: Codex, on depth.** The credential-brokering proxy (the model literally never sees your real GitHub token) and the post-quantum secure remote-execution channel are unique in published detail. Claude Code's sandboxing is well-documented and reduced its internal permission prompts by 84%, and it covers more *surfaces* (cloud). Grok Build's sandbox mechanism is undetailed in accessible sources — and its v0.2.93 incident (below) is a cautionary counterweight.

### Context management

| Capability | Codex | Claude Code | Grok Build |
|-----------|:-----:|:-----------:|:----------:|
| Auto-compaction | ✅ 3 tiers (summarize/token-budget/remote) | ✅ tunable threshold | ✅ `/compact` |
| Manual compact with focus | ✅ | ✅ (`/compact` w/ instructions) | ✅ |
| Project instructions file | ✅ AGENTS.md (+ override) | ✅ CLAUDE.md hierarchy + rules/ | ✅ AGENTS.md **and** CLAUDE.md |
| @-mention file imports | ✅ | ✅ | ? |
| Cross-session memory | ✅ extract/consolidate/inject pipeline | ⚠️ /memory files | ❌ |
| Context window (default model) | 400K (Codex model) | 1M (Sonnet 5) | 256K (grok-build-0.1) |

**Superiority: split.** Claude Code has the **largest context window** (1M) and the most refined CLAUDE.md hierarchy (nested, path-scoped `rules/`, `@`-imports). Codex has the most **sophisticated memory** (a real extract→consolidate→inject loop, not static files). Grok Build's headline here is *compatibility* — it reads both `AGENTS.md` and `CLAUDE.md` out of the box.

### Extensibility

| Capability | Codex | Claude Code | Grok Build |
|-----------|:-----:|:-----------:|:----------:|
| Skills (SKILL.md) | ✅ | ✅ (originated the format) | ✅ |
| Hooks (lifecycle) | ✅ 10 events, Claude-compatible | ✅ 11+ events incl. team events | ✅ (event names unverified) |
| Slash commands | ✅ built-in enum | ✅ + custom `.md` commands | ✅ |
| Plugins + marketplace | ✅ (syncs openai/plugins) | ✅ + marketplaces, namespaced | ✅ official marketplace repo |
| Plugin can bundle LSP server | ❌ | ⚠️ code-intelligence plugins | ✅ `.lsp.json` |
| MCP client | ✅ stdio + HTTP + OAuth | ✅ + tool search | ✅ zero-reconfig |
| MCP server (be driven by MCP) | ✅ `mcp-server` | ✅ | ? |
| Output styles | ❌ | ✅ | ? |
| Import from other agents | ✅ Claude Code + Cursor | ❌ | ⚠️ reuses CLAUDE.md natively |

**Superiority: near-parity, with Claude Code slightly ahead** on breadth (output styles, custom commands, team-aware hook events) since it originated the skills/hooks/plugins model that both competitors adopted. Codex uniquely offers **migration from other agents**. Grok Build uniquely lets plugins **bundle LSP servers**.

### Multi-agent

| Capability | Codex | Claude Code | Grok Build |
|-----------|:-----:|:-----------:|:----------:|
| Sub-agents (report to caller) | ✅ | ✅ | ✅ |
| Peer agent teams (coordinate w/ each other) | ❌ | ✅ experimental (shared task list + mailbox) | ⚠️ parallel subagents, worktree-isolated |
| Worktree isolation per agent | ⚠️ (worktree option exists) | ✅ isolation:worktree | ✅ (headline: each on own branch) |
| Spawn-graph persistence | ✅ agent-graph-store | ✅ | ? |
| Batch/fan-out jobs (CSV) | ✅ spawn_agents_on_csv | ⚠️ dynamic workflows | ? |

**Superiority: Claude Code (teams) and Grok Build (worktree parallelism) each lead a dimension Codex lacks.** Claude Code's agent teams — a lead spawning peers that self-coordinate via a shared task list and mailbox — is the most advanced multi-agent model. Grok Build's git-worktree-per-subagent (reportedly up to 8, single-source) is the cleanest *parallel-edit* isolation. Codex has the persistence substrate (spawn graph, per-thread rollouts) but not the coordination layer — which is exactly why "agent teams" is a P0 in the [roadmap](13-improvements-and-new-features.md).

### Session persistence and review

| Capability | Codex | Claude Code | Grok Build |
|-----------|:-----:|:-----------:|:----------:|
| Resume / continue | ✅ | ✅ | ✅ |
| Fork | ✅ | ✅ | ✅ |
| Working-tree checkpoints / rewind | ❌ (removed) | ✅ /rewind + checkpoints | ⚠️ checkpoints (workspace) |
| Conversation backtrack | ✅ TUI double-Esc | ✅ | ? |
| Code review command | ✅ `/review`, review task | ✅ /review, /code-review (→cloud ultra) | ? |
| Security review | ⚠️ (security-review skill) | ✅ /security-review | ? |

**Superiority: Claude Code and Grok Build both have working-tree rewind; Codex does not** (its ghost-commit feature was removed). Claude Code's multi-level `/code-review` escalating to a cloud "ultra" pass is the most developed review flow.

### Enterprise and providers

| Capability | Codex | Claude Code | Grok Build |
|-----------|:-----:|:-----------:|:----------:|
| Model providers | OpenAI, Bedrock, Ollama, LM Studio | Anthropic, Bedrock, Vertex, Foundry, AWS | Grok (OpenRouter routing unverified) |
| Managed/enterprise config | ✅ signed cloud-config bundles | ✅ managed settings + admin console | ✅ config.toml (enterprise ZDR mentioned) |
| Custom CA / proxy | ✅ | ✅ | ? |
| OTEL monitoring | ✅ | ✅ (lines-of-code metric, workflow attrs) | ? |
| Fallback models | ⚠️ | ✅ up to 3 | ? |

**Superiority: mixed.** Claude Code has the widest **provider** support (four major clouds) and the most mature managed-settings/monitoring story. Codex matches on managed config (signed bundles), custom CA, and OTEL, and adds **local OSS models** (Ollama/LM Studio) that Claude Code lacks.

## What exists in one but not the others

**Only in Codex:**
- Credential-brokering network proxy (model never sees real secrets).
- Post-quantum secure remote-execution channel.
- Code mode (JavaScript/V8 programmatic tool interface).
- An extract→consolidate→inject **memory pipeline** (vs static memory files).
- Import/migration from Claude Code *and* Cursor.
- Local OSS model providers (Ollama, LM Studio) with first-party support.

**Only in Claude Code:**
- Agent teams (peer coordination via shared task list + mailbox).
- Mobile app that monitors and steers cloud sessions.
- Slack and Chrome integrations.
- Four-cloud provider support (Bedrock/Vertex/Foundry/AWS).
- A first-class Agent SDK exposing the same loop as an embeddable library.
- Working-tree /rewind with checkpoints; multi-level /code-review with a cloud "ultra" pass.
- Output styles.

**Only in Grok Build:**
- Git-worktree-per-subagent as the default parallelism model.
- Interactive plan editing (approve/comment/rewrite *individual* plan steps).
- Plugins that bundle LSP servers (`.lsp.json`).
- `grok inspect` — a pre-flight dump of all discovered config/skills/plugins/hooks/MCP.
- Full-screen mouse-interactive TUI as the default terminal experience.

## Where each harness is superior, and why

**Codex is superior at safety and architecture.** Its three-OS sandbox, credential brokering, post-quantum remote execution, and protocol-first design (every frontend a client of a code-generated JSON-RPC API) are the deepest engineering of the three. The reason is structural: Codex is open-source Rust with a maintainer culture that resists shortcuts (the `AGENTS.md` rules, the fail-closed sandboxes, the JSONL-canonical persistence). If you care about *what the agent can and cannot do to your machine*, Codex is the most defensible.

**Claude Code is superior at reach and coordination.** It is available everywhere (IDE, desktop, web, mobile, Slack, Chrome, GitHub, SDK), it originated the skills/hooks/plugins model the others copied, and its agent teams and cloud-teleport features are the most advanced multi-agent and session-mobility stories. The reason is strategy and head start: Anthropic built Claude Code as a *platform* first, monetized through plan integration, and iterates fast on Node. If you care about *using the agent from wherever you are and coordinating many of them*, Claude Code wins.

**Grok Build is superior at parallel-edit ergonomics.** Worktree-per-subagent and step-level plan editing are genuinely nice for the specific workflow of "fan out several risky changes and review each." The reason is focus: Grok Build is a newer, narrower product (terminal-only, xAI-model-only) that optimized a few workflows hard. If you care about *parallel branches with tight human review*, Grok Build's ergonomics lead — though its safety story is unproven and its data-handling history is a red flag.

## ⚠️ Safety note on Grok Build

Grok Build v0.2.93 was found to upload entire git repositories — all tracked files, full commit history, *unread* files, and *unredacted `.env` secrets* — to xAI cloud storage, independent of what the model actually read (a researcher measured ~5.1 GiB uploaded vs ~192 KB seen by the model). The "Improve the model" toggle did not stop it; only a server-side flag did. xAI shipped a kill switch on July 13, 2026 and points users to a `/privacy` command. This corroborated incident (TheHackerNews + independent wire-level analysis) is a material differentiator: it is the opposite of the credential-brokering, redaction-first posture Codex takes, and it should weigh heavily for any security-sensitive user considering Grok Build.

## Inferred future directions (grounded)

These are inferences, each tied to a known fact — not speculation for its own sake.

**Codex** will likely (1) close the multi-agent-coordination gap, because it already has the spawn graph, per-thread rollouts, and inter-agent messaging — the substrate for teams exists and only the coordination layer is missing; (2) restore working-tree rewind, because the git-baseline machinery it built for the memory workspace generalizes directly and the capability is a conspicuous hole; (3) push code mode from experimental to default, because it invested in an out-of-process V8 host that already ships Stable; and (4) deepen its Claude-Code-compatibility on-ramp (hooks, `CLAUDE.md`, migration), because that is a clear, repeated strategic pattern in the codebase. It is *unlikely* to become broadly model-agnostic soon, given the deliberate Responses-only wire commitment.

**Claude Code** will likely (1) graduate agent teams from experimental to GA and add nested teams, because the changelog shows steady investment (auto-cleanup, split-pane modes, quality-gate hooks) and the current one-team-per-session limit is explicitly temporary; (2) expand cloud/mobile as the primary surface, because that is where its recent feature velocity concentrates (teleport, remote control, routines, web); and (3) keep widening enterprise provider support, having just added the AWS upstream provider. It will remain **Anthropic-model-only** — that is the business model.

**Grok Build** will likely (1) formalize its IDE story beyond ACP, because ACP is explicitly positioned as an integration protocol and named editors are the obvious next step; (2) lean into worktree parallelism as its signature (more agents, better merge UX), because that is its clearest differentiator; and (3) be forced to invest heavily in data-handling transparency and sandboxing after the v0.2.93 incident, because a corroborated exfiltration bug on a security-sensitive tool demands a visible response — the `/privacy` command and kill switch are the start, not the end. Its trajectory is the most uncertain: closed development, no external contributions, binaries shipped out-of-band, and a young codebase mean its public signals are thin.

## What Codex should adopt

Distilling the comparison into concrete borrowings, cross-referenced to the [roadmap](13-improvements-and-new-features.md):

| From | Feature | Why for Codex | Roadmap item |
|------|---------|---------------|--------------|
| Claude Code | Agent teams (shared task list + mailbox) | Codex has the spawn graph and messaging; only the coordination layer is missing — highest-visibility gap | P0 new feature |
| Claude Code | Working-tree /rewind + checkpoints | Codex removed ghost commits; the git-baseline mechanism generalizes; frequently requested | P0 new feature |
| Claude Code | GitHub Action / CI review bot | `codex exec` already has JSONL, output schemas, and a review task; CI is where agent value concentrates | P1 new feature |
| Claude Code | Scheduled / triggered agents (routines) | The daemon already runs Codex as a managed background process with an update loop | P1 new feature |
| Grok Build | LSP-backed code intelligence | Symbol-accurate navigation cuts wasted search turns; the exec-server can host a language server | P1 new feature |
| Grok Build | Interactive plan editing (step approve/comment/rewrite) | Codex has `update_plan` and an approval framework; step-level editing is a UX upgrade | (fold into `update_plan` UX) |
| Grok Build | `inspect`-style config preflight | A `codex inspect` dumping discovered skills/plugins/hooks/MCP would aid debugging | (small TUI/CLI add) |

The reciprocal is worth stating plainly: **Codex's credential brokering, three-OS sandbox depth, and memory pipeline are things the others should adopt from it.** In a field where Claude Code leads on reach and Grok Build on niche ergonomics, Codex's path to being *the best* harness is to keep its safety-and-architecture lead while closing the coordination and session-mobility gaps — which is precisely what the roadmap sequences.
