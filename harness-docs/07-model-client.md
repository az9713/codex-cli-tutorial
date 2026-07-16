# The model client

The model client is the code that actually talks to the language model: it speaks the provider's wire protocol, streams responses into typed events, retries failures, counts tokens, and assembles the request body. This page covers all of that. Paths are under `codex-rs/`.

The layer is split across crates: `codex-api` is the low-level wire client (HTTP + WebSocket, SSE parsing, error mapping); `core/src/client.rs` orchestrates requests per turn; `model-provider-info` holds provider config; `model-provider` adds auth resolution; and the transport retry primitive lives in `codex-client/src/retry.rs`.

## Responses API only

A foundational fact: **Codex speaks only the OpenAI Responses API.** Chat Completions support has been fully removed. The `WireApi` enum has a single variant, `Responses`, and its deserializer explicitly rejects `wire_api = "chat"` with a migration error (`model-provider-info/src/lib.rs:55`). This simplifies the client enormously — there is one request shape, one streaming format, one set of item types — at the cost of locking Codex to OpenAI's newer protocol.

Within the Responses API there is a second, orthogonal choice: **WebSocket vs HTTP**. If the provider supports WebSockets, the client tries Responses-over-WebSocket first and falls back to Responses-over-HTTP (server-sent events) on failure (`core/src/client.rs:1787`). The WebSocket path supports incremental request reuse — sending only new input items with a `previous_response_id` when prior request fields match — which cuts bandwidth on long conversations.

## Streaming

The model streams its response as a sequence of events, parsed in `codex-api/src/sse/responses.rs`. The pipeline reads response headers first and emits header-derived events (the server's chosen model, rate limits, whether reasoning was included) *before* the body, then loops over server-sent events with a per-event idle timeout. Each event is deserialized and mapped to a typed `ResponseEvent` (`codex-api/src/common.rs:73`):

| Wire event | `ResponseEvent` |
|-----------|-----------------|
| `response.output_item.added` | `OutputItemAdded` |
| `response.output_item.done` | `OutputItemDone(ResponseItem)` |
| `response.output_text.delta` | `OutputTextDelta` |
| `response.custom_tool_call_input.delta` | `ToolCallInputDelta` |
| `response.reasoning_summary_text.delta/.done` | `ReasoningSummaryDelta` / `ReasoningSummaryDone` |
| `response.reasoning_text.delta` | `ReasoningContentDelta` |
| `response.completed` | `Completed { response_id, token_usage, end_turn }` |
| `response.failed` / `.incomplete` | mapped to fatal or retryable errors |

A key design point: the SSE layer does *not* accumulate deltas into full strings — it forwards each delta verbatim, and the turn loop (see [The agent loop](04-agent-loop.md)) assembles them. Full items, including a tool call's final arguments and any encrypted reasoning, arrive as `OutputItemDone`. The WebSocket path reuses the exact same event model and parser.

## Retry and backoff

Retry happens at three independent layers, which is worth understanding because they handle different failures:

**Transport-level** (`codex-client/src/retry.rs`) — retries a single HTTP call on 5xx and network/timeout errors with exponential backoff plus jitter. Notably, **429 rate-limit is *not* retried here**; it is surfaced up as a usage error instead. The default is 4 attempts (`request_max_retries`).

**Stream-disconnect + transport fallback** (`core/src/responses_retry.rs`) — the per-turn loop for stream errors (disconnects, retryable stream failures). When retries are exhausted *and* the WebSocket is active, it switches the fallback transport from WebSocket to HTTPS, resets the retry counter, and tries again on HTTP. Otherwise it backs off (honoring a server-requested delay if present) and reconnects, showing the user a "Reconnecting… n/max" message. The default is 5 stream retries (`stream_max_retries`).

**401 recovery** (`core/src/client.rs`) — a separate loop that, on an unauthorized response, drives token refresh via the auth manager once and retries.

The stream body also classifies failure codes: `context_length_exceeded` and `insufficient_quota` are fatal, `server_is_overloaded`/`slow_down` are retryable, and a `rate_limit_exceeded` message is parsed for a "try again in N" delay.

## Token and context accounting

Usage arrives on every completed response (`ResponseCompletedUsage` → `TokenUsage`), mapping input tokens, cached input tokens, cache-write tokens, output tokens, and reasoning tokens. This feeds telemetry and, crucially, the context-budget logic that triggers compaction.

The context window itself is a per-model fact (`ModelInfo` in `protocol/src/openai_models.rs:368`), fetched from the backend `/models` endpoint. Key fields: `context_window`, an `effective_context_window_percent` (default 95%), and an `auto_compact_token_limit` computed as the minimum of a configured limit and 90% of the resolved window. When usage crosses that limit, the turn loop compacts (see [Context, state, and memory](08-context-state-memory.md)).

## Reasoning

Newer models emit hidden chain-of-thought that must be handled carefully: shown to the user only if they opt in, but round-tripped to the server so the model keeps its reasoning across turns. Codex always adds `include: ["reasoning.encrypted_content"]` to every request so the server returns encrypted reasoning it can send back. The `Reasoning` request field carries the effort level (mapping the internal `Ultra` to the wire's `Max`) and, when supported, a summary setting. Reasoning arrives streamed as summary-text deltas and raw-content deltas, and the full reasoning item — including its encrypted blob — arrives as `OutputItemDone`. A server header, `x-reasoning-included`, tells the client the server already accounted for past reasoning tokens so it must not re-estimate.

## Providers

The provider abstraction is `ModelProviderInfo` (`model-provider-info/src/lib.rs:87`), carrying base URL, auth (env-key, command-backed bearer, or AWS SigV4), wire API, headers, retry limits, timeouts, and capability flags (`requires_openai_auth`, `supports_websockets`). Four providers are built in:

- **openai** — the default; base URL `https://chatgpt.com/backend-api/codex` for ChatGPT auth or `https://api.openai.com/v1` for API-key auth; supports WebSockets.
- **amazon-bedrock** — AWS SigV4 auth, no WebSockets.
- **ollama** and **lmstudio** — local OSS runtimes at `localhost:11434`/`localhost:1234`, configurable via `CODEX_OSS_PORT`/`CODEX_OSS_BASE_URL`.

Users can extend the built-ins via config. The runtime auth wrapping (bearer injection, error mapping) is applied by `create_model_provider` in `model-provider/`.

## Request assembly

The request body is built by `ModelClient::build_responses_request` (`core/src/client.rs:823`) from a `Prompt` (input items, tools, base instructions, output schema). It:

- Formats the input from history, stripping image detail for "responses-lite" models and stripping internal metadata for non-OpenAI providers.
- Serializes the enabled tools into the `tools` array (or, in responses-lite mode, splices them into the input prefix as a developer message and leaves `tools` empty).
- Sets `instructions` to the base/system prompt (see [Context, state, and memory](08-context-state-memory.md) for how that prompt is built).
- Adds reasoning config, verbosity, `include: ["reasoning.encrypted_content"]`, a `prompt_cache_key` (the session id, for server-side prompt caching), the service tier, and client metadata.
- Sets `tool_choice: "auto"`, `stream: true`, and `parallel_tool_calls` per model capability.

Compaction uses the same builder but POSTs to `/responses/compact` with a longer idle timeout.

## Design assessment

The model client is clean and well-layered: one wire format, a shared event model across HTTP and WebSocket, and a sensible three-tier retry story with graceful WebSocket→HTTP degradation. The main design bet — dropping Chat Completions entirely — is a double-edged sword: it makes the client simpler and lets Codex use Responses-only features (encrypted reasoning, server-side prompt caching, incremental WebSocket reuse), but it forecloses any provider that doesn't implement the Responses API, which today is effectively only OpenAI and Azure. The local OSS providers (Ollama, LM Studio) exist but must speak Responses-shaped requests to work.
