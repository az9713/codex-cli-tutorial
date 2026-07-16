# Config, auth, and telemetry

The unglamorous but load-bearing plumbing: how Codex loads its settings, proves it's allowed to call the model, decides which model to use, measures what happens, and keeps itself up to date. Paths are under `codex-rs/`.

## Configuration

Codex's configuration is `config.toml` under `CODEX_HOME` (`~/.codex`), resolved through a **layered precedence stack**. Later layers override earlier ones:

1. **Managed / enterprise** — fetched from the backend as a signed "cloud config bundle" (see below).
2. **Admin** — system-level config.
3. **User** — `~/.codex/config.toml`.
4. **Project** — `.codex/config.toml` in the repo.
5. **Session flags** — CLI `-c key=value` overrides and `--profile` selections.

The config surface is large — model providers, approval and sandbox settings, feature flags, MCP servers, skills, hooks, memory, telemetry, and profiles (named bundles of settings). The schema is validated (`config/`), and a `just write-config-schema` step keeps `core/config.schema.json` in sync.

**Feature flags** (`features/` crate) are a first-class subsystem. Each `[features]` toggle has a lifecycle **stage** — Stable, UnderDevelopment, Deprecated, Removed, Experimental — and a default-enabled flag, and most subsystems in the harness are gated by one. `codex features list/enable/disable` manages them, and the CLI's `--enable`/`--disable` flags fold into config overrides. This is why the same binary can ship a feature's plumbing enabled while its user-facing surface stays off.

**Enterprise cloud config** (`cloud-config/` crate) delivers managed policy from OpenAI's backend to business/enterprise/edu ChatGPT accounts. It fetches a bundle of managed `config.toml` and `requirements.toml` fragments, validates and caches them (an HMAC-signed, identity-scoped, one-hour-TTL on-disk cache), and refreshes in the background every 15 minutes. The bundle layers into the config stack as the highest-precedence managed layer — the mechanism by which an org can pin allowed models, sandbox policy, or marketplace sources.

## Authentication

Codex supports three auth modes (`login/` crate):

- **ChatGPT OAuth** — the default and recommended path (`codex login` → "Sign in with ChatGPT"). Uses the user's Plus/Pro/Business/Enterprise plan. Required for all cloud features.
- **API key** — `codex login --with-api-key` (read from stdin) or `--api-key`. Points at `api.openai.com`.
- **Device code** — `codex login --device-auth` for headless machines.

Auth material lives in `auth.json` under `CODEX_HOME`, with a `keyring-store` for OS keychain integration and a `secrets` crate for handling. Token refresh is automatic: a 401 triggers an unauthorized-recovery flow that refreshes the token and retries the request. Cloud paths explicitly disable API-key auth (`enable_codex_api_key_env=false`) — they require a ChatGPT login and an account id.

Outbound requests carry an `originator` header (`codex_cli_rs` by default) and a composed `User-Agent` identifying the client, OS, and terminal; the SDKs override the originator to identify themselves (`login/src/auth/default_client.rs`).

## Models and providers

The model catalog is data-driven. `models-manager/` fetches model metadata from the backend `/models` endpoint and merges it with an embedded `models.json`, producing a `ModelInfo` per model slug: context window, reasoning levels, verbosity support, apply-patch tool type, shell type, personality support, and base instructions. The `/model` command and config select among them.

The provider abstraction (`model-provider-info/`) ships four built-in providers — **openai** (default), **amazon-bedrock** (SigV4), **ollama** and **lmstudio** (local OSS at `localhost`) — and users can extend them via config. All speak the Responses API wire format (see [The model client](07-model-client.md)).

## HTTP and networking infrastructure

Shared transport plumbing (`http-client/`, `websocket-client/`) centralizes concerns every crate needs: a reqwest wrapper with W3C trace-header injection, request compression (zstd), an enterprise **custom-CA** policy (`CODEX_CA_CERTIFICATE`/`SSL_CERT_FILE` force a rustls backend layering the custom cert onto native roots), and **route-aware outbound proxy** resolution that respects the OS system proxy (WinHTTP PAC/WPAD on Windows, SystemConfiguration/CFNetwork on macOS) with env-var fallback. The WebSocket dialer implements Happy Eyeballs (interleaved IPv4/IPv6) and proxy-tunnel support. A dedicated cookie jar stores *only* Cloudflare infrastructure cookies for ChatGPT hosts and refuses all auth cookies.

## Telemetry

Telemetry is three separate subsystems with a clear PII boundary (`otel/`, `analytics/`, `feedback/`):

**OpenTelemetry** (`otel/`) emits logs, traces, and metrics on three independent channels. The critical design is target-based **PII segregation**: a `log_only` target carries potentially-sensitive fields (raw prompts if `otel.log_user_prompt` is enabled, tool arguments/output, account id/email), while a `trace_safe` target carries only aggregated lengths and counts. Metrics default to a hardcoded Statsig OTLP endpoint in release builds, but this is **disabled in debug builds** and whenever analytics is turned off. Logs and traces are off by default (opt-in via `[otel]`).

**Analytics** (`analytics/`) sends product events (thread initialized, tool calls, plugin usage) to the ChatGPT backend, reduced from JSON-RPC facts by a stateful correlator. It sends no raw prompt or output text — only ids, counts, and shapes. It is on unless `analytics.enabled=false`, and a debug-only capture-file mode disables network delivery. Notably, "accepted-line fingerprints" are computed but *blanked* before upload — only aggregate counts and a repo hash leave the machine.

**Feedback** (`feedback/`) is never automatic. Only on an explicit `/feedback` upload does it send a Sentry event with a ring-buffered log tail and diagnostic attachments, after a user consent gate.

The privacy posture is defensible: nothing ambient sends prompt text (unless the user opts in via `otel.log_user_prompt`), the trace channel is aggregate-only, and debug builds are silent. One sharp edge worth noting: the feedback connectivity diagnostic reports proxy env-var *values verbatim*, including any embedded credentials, but only when the user includes logs.

## Self-update

Codex checks for updates on startup (config-gated, release-only). It detects its install channel — npm/bun/pnpm, standalone, or Homebrew (`install-context/`) — maps that to the right upgrade command, and checks GitHub releases (plus npm-registry readiness for npm installs) at most every 20 hours, caching the result in `version.json`. If a newer version exists and wasn't dismissed, it shows a modal offering Update now / Skip / Skip-until-next. "Update now" exits the TUI and runs the channel-appropriate command (`npm install -g @openai/codex`, `brew upgrade --cask codex`, or the standalone install script). The daemon has its own auto-update loop that restarts the managed binary hourly when a new version lands.

## Design assessment

The config/auth/infra layer is solid and, in the enterprise dimensions, ahead of its competitors: signed cloud-config bundles, custom-CA support, route-aware system-proxy resolution, and OS-keychain auth storage are all present and thorough. The feature-flag system is a genuine strength for shipping incrementally, though it is also the source of the combinatorial complexity that shows up elsewhere in the harness (see [Critique](12-critique.md)). The telemetry PII boundary is well-designed. The main friction is the sheer breadth of the config surface, which — like the rest of Codex — trades simplicity for capability.
