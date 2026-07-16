# Execution and sandboxing

The sandbox is what makes a coding agent safe to run. Codex invents and executes arbitrary shell commands; the sandbox is the OS-enforced boundary that stops a mistake — or a prompt injection — from deleting your files or exfiltrating your secrets. This is one of the most thoroughly engineered parts of the harness, with a distinct backend for each of the three major operating systems plus a filtering network proxy and a remote-execution server.

This page covers the full command-execution pipeline, each platform's sandbox, the network proxy, and the exec-server. Paths are under `codex-rs/`.

## The execution pipeline

A shell tool call becomes a spawned process through this sequence:

1. **Safety assessment.** `execpolicy` (a Starlark policy engine) and the approval logic classify the command: known-safe (auto-run), needs approval, or forbidden.
2. **Approval.** Per the [approval gate](05-tools-and-approvals.md), the command is auto-approved, sent to the Guardian reviewer, or shown to the user.
3. **Sandbox selection.** Based on the `SandboxPolicy` and platform, a backend is chosen: macOS Seatbelt, Linux Landlock/seccomp/bwrap, or Windows restricted tokens. `should_require_platform_sandbox` (`sandboxing/src/policy_transforms.rs:538`) decides whether a sandbox is needed at all.
4. **Spawn.** The command runs wrapped by the sandbox, with the network proxy env vars injected.
5. **Escalation.** If the sandbox denies the command and policy allows, the orchestrator can prompt to retry without the sandbox.

The `SandboxPolicy` (`protocol/src/protocol.rs:998`) is the user-facing knob, with four settings:

| Policy | Filesystem | Network |
|--------|-----------|---------|
| `ReadOnly` | Read anywhere, write nowhere | Restricted |
| `WorkspaceWrite` | Write inside workspace roots (+ TMPDIR); `.git`/`.codex`/`.agents` protected read-only | Per config |
| `DangerFullAccess` | Unrestricted | Enabled |
| `ExternalSandbox` | Delegated to an outer sandbox | Per config |

This maps through a `PermissionProfile` into concrete `(FileSystemSandboxPolicy, NetworkSandboxPolicy)` that each platform backend enforces.

## macOS: Seatbelt

macOS uses Apple's Seatbelt (`sandbox-exec`). Codex generates a Seatbelt profile from the permission profile and wraps the command. Because the profile is generated per-invocation, the writable roots, readable roots, and network policy are all expressed declaratively in the profile. (When Codex spawns a Seatbelt child, it sets `CODEX_SANDBOX=seatbelt` so nested tests can detect they're already sandboxed.)

## Linux: Landlock, seccomp, and Bubblewrap

Linux is the most elaborate backend, living in `linux-sandbox/` and driven host-side by `sandboxing/src/landlock.rs`. It has two layers and prefers the stronger one.

**Bubblewrap (primary).** `bwrap.rs` builds a Bubblewrap argument list that creates a mount namespace: `--ro-bind / /` for read access, `--bind <root> <root>` for each writable root, `--tmpfs`/`--ro-bind-data /dev/null` masks over unreadable paths, and metadata masks that keep `.git`/`.agents`/`.codex` read-only inside writable roots. It handles symlink safety by failing closed when a read-only path crosses a writable symlink (a TOCTTOU defense), and expands unreadable globs with ripgrep. Codex prefers the *system* bwrap when present and falls back to a **bundled, SHA256-verified** `bwrap` binary (`bundled_bwrap.rs`) executed via `/proc/self/fd` — so the sandbox works even on machines without bubblewrap installed.

**Landlock + seccomp (legacy fallback).** `landlock.rs` applies, in-process, Landlock V5 filesystem rules (read-all, write only to writable roots) plus a seccomp network filter. The seccomp filter (`install_network_seccomp_filter_on_current_thread`) unconditionally denies `ptrace` and `io_uring`, and in `Restricted` mode denies `connect`/`bind`/`listen`/`sendto` while allowing only `AF_UNIX` sockets — so a sandboxed process cannot open a network connection. A `ProxyRouted` mode instead allows IP sockets but only to the loopback bridge (for the managed network proxy).

**Two-stage execution.** The Linux helper runs as a separate `codex-linux-sandbox` process (reached via the arg0 trick). The outer stage sets up the bwrap mount view; it then re-invokes itself with `--apply-seccomp-then-exec` as the inner stage, which applies the seccomp filter inside the namespace and `execve`s the real command. Signal forwarding, a protected-metadata inotify monitor (which removes any `.git`/`.codex` the sandboxed process tries to create), and `PR_SET_PDEATHSIG` round out the hardening.

WSL1 is explicitly unsupported (no user namespaces) and fails closed with a clear error pointing at WSL2.

## Windows: restricted tokens and ACLs

Windows has no Landlock or Seatbelt equivalent, so `windows-sandbox-rs/` improvises. There is **no AppContainer**; enforcement is **restricted tokens** (`CreateRestrictedToken` with `WRITE_RESTRICTED`) plus **per-run randomized capability SIDs** stamped into file DACLs. There are two backends:

- **Unelevated "restricted-token" backend** — runs the child as the current user but with a restricted token whose restricting SIDs are the capability SIDs. It can enforce write restrictions but *cannot* enforce deny-read (a limitation of `WRITE_RESTRICTED`), so it requires full-disk read.
- **Elevated backend** — used when network isolation or deny-read is needed. It creates dedicated local user accounts (`CodexSandboxOffline`/`CodexSandboxOnline`), runs the child as that user via `CreateProcessWithLogonW`, applies deny-read ACLs, and blocks network egress with Windows Firewall rules plus kernel-level Windows Filtering Platform (WFP) filters scoped to the sandbox account. Setup requires a one-time UAC elevation.

The Windows implementation is remarkably deep — DPAPI-protected sandbox-account passwords, a `mklink /J` cwd junction to work around async read-ACL propagation, hidden sandbox users, Job Objects for kill-on-close — but it is also the most fragile: read grants are applied asynchronously (a race mitigated but not eliminated), firewall setup fails closed if group policy overrides it, and full-disk-write profiles are simply unenforceable.

## The network proxy

Filesystem sandboxing stops a bad *write*; the network proxy stops a bad *send*. `network-proxy/` is a loopback-bound, egress-filtering forward proxy (HTTP CONNECT + plain HTTP + SOCKS5) that sandboxed processes are forced through via ~30 proxy environment variables (`HTTP_PROXY`, `npm_config_proxy`, `PIP_PROXY`, `DOCKER_*`, and more).

Its capabilities:

- **Domain allow/deny policy.** Every connection is checked against a globset allowlist/denylist. Deny always wins; local/private IPs are blocked unless explicitly allowlisted; hostnames are resolved and re-classified to defend against DNS rebinding, with a second SSRF check at actual dial time.
- **TLS interception (MITM).** For hosts that need it, the proxy terminates TLS using a **process-local, self-signed CA** (whose private key is never persisted) and re-inspects the inner HTTP request against method/host/hook policy. Sandboxed children are pointed at a generated CA bundle via `SSL_CERT_FILE`/`NODE_EXTRA_CA_CERTS`/etc.
- **Credential brokering.** This is the standout feature. Before launching a child, the proxy replaces real secrets in the environment (a GitHub token, an OpenAI key) with **shaped dummy values** of the same length and prefix. The child — and therefore the model, which sees the environment and tool output — only ever sees the dummy. When the child makes an HTTPS request to the matching host, the proxy (having terminated TLS) swaps the dummy back for the real secret in the `Authorization` header. **The model never sees your real credentials, even though the agent can use them.**
- **Attribution.** Each execution is scoped by an attribution token so audit events and policy decisions are attributed to the right turn.

This is a level of network hardening that neither of Codex's competitors matches in published detail.

## The exec-server

`exec-server/` abstracts *where* execution happens. It is a JSON-RPC server that spawns and controls subprocesses, so the core can run a command either **locally in-process** (the default, no server) or in a **remote environment** — a dev container or cloud sandbox — over a network. The two capability traits are `ExecBackend` (spawn a process) and `ExecutorFileSystem` (read/write files); an `Environment` bundles both plus an HTTP client, and is either local (`LocalProcess`) or remote (`RemoteProcess`).

The remote transport is genuinely sophisticated. It runs across an untrusted rendezvous relay that routes frames by `stream_id` but must not see plaintext, so an end-to-end **Noise** channel is layered inside the relay WebSocket. The handshake uses a **hybrid post-quantum** suite (X25519 *and* ML-KEM-768, `Noise_hybridIK_X25519+MLKEM768_AESGCM_SHA256`), with a registry acting as the independent authority both endpoints consult to validate keys. A recovery FSM reconnects and replays missed process output after a disconnect. This is production-grade secure remote execution — the plumbing that lets a local Codex drive a cloud dev container.

Selection is by environment variable: `CODEX_EXEC_SERVER_URL` unset → local in-process; `ws://...` → remote WebSocket; Noise-registry vars set → remote Noise; `none` → no model-facing shell/FS tools at all.

## Design assessment

The sandboxing layer is Codex's crown jewel. Three real OS backends, a filtering network proxy with credential brokering, and post-quantum secure remote execution together form a defense-in-depth story that is more thorough than any competitor's published design. The weaknesses (detailed in [Critique](12-critique.md)) are concentrated on Windows — where the absence of a real OS sandbox primitive forces a complex, race-prone restricted-token-and-ACL scheme — and in the sheer surface area, which makes the sandbox one of the hardest parts of the codebase to audit end to end.
