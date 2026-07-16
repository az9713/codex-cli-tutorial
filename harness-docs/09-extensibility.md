# Extensibility

Extensibility is how users add capability to Codex without forking it. There are six mechanisms, each with a distinct purpose: **skills** (on-demand knowledge), **hooks** (lifecycle interception), **plugins** (packaged bundles), **MCP** (external tool servers, both directions), **code mode** (a programmatic tool interface), and **external-agent migration** (importing from Claude Code and Cursor). This page covers each. Paths are under `codex-rs/`.

A cross-cutting fact: nearly every extension system is gated by a `[features]` flag with a lifecycle stage. `hooks`, `plugins`, `skill_search`, and `code_mode_host` are Stable and on by default; `code_mode`, `code_mode_only`, and `external_agent_memory_import` are under development and off. This matters when reading the code — a system's plumbing may ship enabled while its model-facing surface is still gated.

## Skills

A skill is a folder with a `SKILL.md` file: YAML frontmatter (`name`, required `description`, optional metadata) plus a free-form markdown body of instructions the model reads on demand. The design principle is **progressive disclosure**: only the one-line description sits in the model's context until the skill is actually used, so a large skill library costs almost nothing.

Skills are discovered from a layered set of roots (`core-skills/src/loader.rs`): project `.codex/skills`, repo `.agents/skills` (for each directory between project root and cwd), user `~/.agents/skills`, an embedded system-skills cache, admin `/etc/codex/skills`, and plugin-contributed roots. Precedence is repo → user → system → admin, with a repo skill shadowing a same-named user or system skill. Codex ships bundled system skills (`imagegen`, `openai-docs`, `skill-creator`, `review-agent`, and more).

At thread start, the skills *catalog* — one `- name: description (file: path)` line per skill — is injected into the developer context, budgeted at 2% of the model's context window. The model self-selects a skill by matching its description, or the user explicitly names one with `$skill-name`, at which point the full `SKILL.md` body is injected for that turn (`core-skills/src/injection.rs`). There is no dedicated "invoke skill" tool — triggering relies on the model obeying the prose contract in the injected how-to text, which is a prompt-adherence risk rather than an enforced mechanism.

A known limitation: skill metadata declares product-gating and implicit-invocation policy, but a `TODO` in `model.rs:55` notes that product gating is parsed and stored but not yet enforced in selection.

## Hooks

Hooks are user scripts that fire at lifecycle events and can block, rewrite, or annotate agent actions. Codex implements a **Claude-Code-compatible** hook engine (`hooks/` crate), which is significant: it means Claude Code hooks largely work in Codex unchanged, and it is a deliberate compatibility bet.

There are ten events:

| Event | Can it block/modify? |
|-------|---------------------|
| PreToolUse | Deny the tool, rewrite its input, add context |
| PermissionRequest | Allow or deny an approval |
| PostToolUse | Block with feedback, add context |
| PreCompact / PostCompact | Force-stop |
| SessionStart / SubagentStart | Inject context |
| SubagentStop / Stop | Block / provide a continuation prompt |
| UserPromptSubmit | Block the prompt, add context |

Hooks are external processes: Codex writes event-specific JSON to stdin and reads a JSON decision from stdout (or uses exit code 2 with stderr as a block reason). The I/O contract mirrors Claude Code's. Hooks are discovered from config layers (`hooks.json` files or inline TOML) and from plugin bundles, and are subject to a **trust model**: a non-managed hook must be trusted (its normalized hash recorded) before it runs, because a trusted hook runs *outside* the sandbox. Only `command`-type handlers actually execute — Claude Code's prompt and agent handler types are parsed but skipped with a warning.

The wiring runs through `core/src/hook_runtime.rs`, which builds each event's request, previews it (emitting `HookStarted` UI rows), runs the matched handlers concurrently, and maps the outcome to a runtime action (block a tool, inject context, force-stop).

## Plugins

A plugin is a packaged directory with a `plugin.json` manifest contributing skills, MCP servers, apps (connectors), hooks, and commands. Plugins are installed from **marketplaces** — JSON catalogs (`marketplace.json`) that can point at local, git, or npm sources — and Codex syncs a curated `openai/plugins` marketplace on startup (`core-plugins/`).

Plugin identity is `<plugin>@<marketplace>`, which is also the `[plugins]` config key. A plugin's hooks flow into the hook engine as `PluginHookSource` values (subject to the same trust model), and its slash-commands are *migrated into skills* at install time (a lossy conversion that drops templates using dynamic arguments). The system is compatibility-minded: it reads `.claude-plugin/` and `.cursor-plugin/` manifest locations and sets `CLAUDE_PLUGIN_*` environment variables alongside its own.

Security is source-allowlist plus hook-hash trust — there is **no cryptographic signing**, so a trusted git clone executes whatever is in the repo. Enterprise deployments can restrict marketplaces to an allowed-source list.

## MCP: client and server

Codex speaks the Model Context Protocol in both directions.

**As a client** (`codex-mcp`, `rmcp-client`), Codex connects out to external MCP servers configured under `[mcp_servers]`, aggregates their tools, and exposes them to the model. It supports two transports (stdio and streamable-HTTP), OAuth for remote servers, and per-tool controls (allow/deny lists, four approval modes). Tool names are namespaced by server and made collision-safe with SHA-1 suffixes. A notable limitation: external MCP servers are **not truly sandboxed** — a stdio server runs as a child process with the Codex user's privileges, and isolation depends on the server voluntarily honoring a `SandboxState` hint that Codex forwards in request metadata.

**As a server** (`mcp-server`), Codex exposes itself as two MCP tools — `codex` (run a session) and `codex-reply` (continue one) — so any MCP client (Claude Desktop, an inspector) can drive a whole Codex session. Approvals are surfaced back to the calling client as MCP elicitations. This is a prototype surface; the richer, newer integration path is the [app-server](10-frontends-and-protocol.md).

**Connectors / Codex Apps** (`connectors`) are a special case: first-party ChatGPT connectors (Gmail, Slack, etc.) surfaced as tools of a single built-in "Codex Apps" MCP server, with their catalog and approval policy managed by the connectors layer.

## Code mode

Code mode is an experimental alternative to one-tool-call-per-turn. Instead of emitting a tool call, the model writes **JavaScript** executed in an **embedded V8 isolate** and calls tools programmatically — `await tools.exec_command(...)`, `await tools.mcp__server__tool(...)` — from inside the script. The model calls a single top-level `exec` tool whose input is raw JavaScript; a companion `wait` tool resumes a script that runs past a yield deadline (`code-mode/`, `code-mode-protocol/`).

The value is fewer round-trips: the model can loop, branch, and filter tool calls in one script instead of N separate model turns. The JavaScript sandbox is capability-restricted — no `fetch`, no filesystem, no `require`/`import`, no `console`, no threads — so the script's only outside effect is calling tools that already passed Codex's normal permission pipeline. Each `exec` runs in a fresh isolate, CPU-bounded by V8's `terminate_execution`, and by default runs **out of process** (`code-mode-host`) so a V8 crash can't take down Codex, with automatic fallback to in-process if the host binary is missing.

Code mode's plumbing (`code_mode_host`) is Stable and on by default, but the model-facing surface (`code_mode`) is under development and off — the feature ships enabled-but-gated.

## External-agent migration

The `/import` command imports another agent's setup into Codex (`external-agent-migration/`). It supports exactly two sources: **Claude Code** (`.claude/`, `CLAUDE.md`, `settings.json`) and **Cursor** (`.cursor/`, `.cursorrules`). It migrates ten item types — config, skills, AGENTS.md, plugins, MCP servers, subagents, hooks, commands, memory (Claude only), and past chat sessions — converting each into its Codex equivalent (e.g. `CLAUDE.md` → `AGENTS.md`, Claude subagent `.md` → Codex `.codex/agents/*.toml`, Claude session `.jsonl` → Codex rollout items), with term-rewriting so imported docs read natively.

This is a deliberate on-ramp: Codex is explicitly designed to make switching from Claude Code low-friction, from reading its `AGENTS.md`/`CLAUDE.md` to running its hooks to importing its whole configuration.

## Design assessment

Codex's extensibility story is broad and, in places, deeper than its competitors': the memory-writing consolidation sub-agent, the code-mode V8 interface, and the credential-brokering that protects MCP-adjacent secrets have no direct equivalent elsewhere. The strategic thread running through all of it is **Claude Code compatibility** — the hook contract, the plugin manifest locations, the `CLAUDE.md` import — which lowers the switching cost for Claude Code users. The weaknesses (see [Critique](12-critique.md)) are that skill triggering is prose-enforced rather than mechanical, external MCP servers aren't sandboxed, plugins aren't signed, and several capabilities (code mode, product gating, external memory import) are shipped-but-gated, so the documented surface and the enabled surface diverge.
