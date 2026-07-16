# The agent loop

The agent loop is the engine of Codex: the code that receives a user request, calls the model, dispatches the tool calls the model asks for, feeds the results back, and decides when the turn is done. Everything else in the harness — sandbox, tools, context, frontends — is a component this loop calls into.

This page traces the loop from the top-level objects down to the streaming tool-dispatch inner loop, with the source locations you need to read the code. All paths are under `codex-rs/`.

## The objects, from outside in

| Object | File | Role |
|--------|------|------|
| `ThreadManager` | `core/src/thread_manager.rs` | Top-level factory; builds Sessions, owns the model catalog and thread store |
| `CodexThread` | `core/src/codex_thread.rs` | The public handle a frontend holds: `submit(op)`, `next_event()`, `steer_input()` |
| `Session` | `core/src/session/mod.rs` + `session.rs` | The runtime: owns the queues, the active turn, the background loop |
| `SessionIo` | `core/src/session/mod.rs:392` | The queue endpoints: submission sender, event receiver |
| `TurnContext` | `core/src/session/turn_context.rs:113` | Immutable per-turn config snapshot |

A frontend never touches `Session` directly. It holds a `CodexThread`, calls `submit(op)` to send an `Op` in, and `next_event()` to read `EventMsg`s out. This is the entire public contract of the engine.

## The two queues

The loop is built on two channels, created together at `core/src/session/mod.rs:533`:

```
let (tx_sub, rx_sub) = async_channel::bounded(SUBMISSION_CHANNEL_CAPACITY);   // Submission Queue (SQ)
let (tx_event, rx_event) = async_channel::unbounded();                        // Event Queue (EQ)
```

- **Submission Queue (SQ)** — bounded, so a runaway frontend cannot flood the engine. Frontends write `Op`s here.
- **Event Queue (EQ)** — unbounded, so the engine never blocks on a slow frontend. The engine writes `EventMsg`s here.

Decoupling the two is what lets you type while the agent works and lets one engine serve many frontends. A background **submission loop**, spawned at `mod.rs:721`, drains the SQ and runs until it sees `Op::Shutdown` or the channel closes.

`Session::send_event` (`mod.rs:1730`) is the single choke point for outgoing events. It does more than push to the EQ: it records terminal errors onto the turn context, writes the event into the rollout trace (for resume), mirrors to any realtime stream, and — a migration wart — emits *both* the new event and its legacy-schema equivalent via `EventMsg::as_legacy_events` (`mod.rs:1763`), doubling event volume for backward compatibility.

## Submission dispatch

The submission loop (`core/src/session/handlers.rs:712`) is a `while let Ok(sub) = rx_sub.recv().await` matching on `sub.op`. Each arm runs inside an `op.dispatch.<kind>` tracing span. The important arms:

| Op | Effect |
|----|--------|
| `UserInput` | Steer into a running turn, or spawn a new `RegularTask` if idle (`handlers.rs:192`) |
| `Interrupt` | Abort the active turn (`abort_all_tasks(Interrupted)`) |
| `ExecApproval` / `PatchApproval` / `UserInputAnswer` / `RequestPermissionsResponse` | Resolve a pending approval oneshot (see below) |
| `Compact` | Spawn a `CompactTask` to summarize history |
| `Review` | Spawn a `ReviewTask` — a nested one-shot review turn |
| `ThreadSettings` | Change model/effort/mode mid-thread |
| `ThreadRollback` | Drop the last N user turns from context |
| `RunUserShellCommand` | Run a one-off shell command (the `!cmd` path), not a model turn |
| `Shutdown` | Break the loop and tear down |

`Op` is `#[non_exhaustive]`, so the `_` arm is ignored — new operations can be added without breaking older matches.

The `UserInput` arm (`handlers.rs:192`) is the important one. It applies any thread-settings overrides, builds a fresh `TurnContext`, and calls `Session::steer_input`. If a turn is already running, the input is *steered* into it (injected mid-flight). If not (`SteerInputError::NoActiveTurn`), it builds the turn's inputs and calls `Session::spawn_task(turn_context, task_input, RegularTask::new())`.

## Turns are tasks

A turn is a `SessionTask` — a trait (`core/src/tasks/mod.rs:214`) with `kind()`, `span_name()`, and an async `run(...)`. There are four implementations:

- **`RegularTask`** (`tasks/regular.rs`) — a normal user turn.
- **`ReviewTask`** (`tasks/review.rs`) — a nested code-review sub-turn.
- **`CompactTask`** (`tasks/compact.rs`) — a history-compaction turn.
- **`UserShellCommandTask`** (`tasks/user_shell.rs`) — an out-of-band `!cmd`.

Two lifecycle rules make the loop robust:

**One turn at a time, new preempts old.** `Session::spawn_task` (`tasks/mod.rs:314`) calls `abort_all_tasks(Replaced)` before starting — a new turn preempts any running one. This is why typing a new message mid-turn replaces the current work rather than queueing behind it.

**All completion funnels through one place.** Whether a task finishes, errors, or is aborted, it ends in `on_task_finished` (`tasks/mod.rs:568`), which drains pending steered input through hooks, emits per-turn token metrics and analytics, emits the terminal `TurnComplete` or `TurnAborted` event, clears the active turn, flushes the rollout, and drains any mailbox-triggered follow-up work. Consistent lifecycle handling in one function is a real strength of this design.

## Approvals are oneshot channels

When a tool needs the user's approval, the handler registers a `oneshot::Sender<ReviewDecision>` in the turn's `TurnState` (`core/src/state/turn.rs:87`), keyed by an id, and `await`s the receiver. The matching `Op::ExecApproval`/`PatchApproval`/`UserInputAnswer` — submitted when the user clicks Approve in the UI — resolves that oneshot. This cleanly decouples the async, potentially minutes-long human round-trip from the tool handler's control flow. The same pattern serves permission requests, elicitations, and dynamic-tool calls.

## The turn loop

`RegularTask::run` (`tasks/regular.rs:37`) emits `TurnStarted`, then loops `run_turn(...)` as long as there is pending steered input to consume. `run_turn` (`core/src/session/turn.rs:144`) is the heart of a turn. Its steps:

1. **Get a model client session** (`turn.rs:152`) — reused across retries so WebSocket connections and sticky routing survive reconnects.
2. **Pre-sampling compaction** (`turn.rs:158`) — if the model changed, downshifted, or the token budget is exhausted, compact history *before* calling the model.
3. **Build context** — capture a step context, record world-state and skills/plugins injections into history as contextual-user messages (`build_skills_and_plugins`, `turn.rs:529`).
4. **Run hooks** — session-start hooks, then input hooks that can rewrite the user input (`run_hooks_and_record_inputs`, `turn.rs:500`).
5. **The sampling loop** (`turn.rs:227`) — repeatedly:
   - drain pending steered input into history;
   - build the request input from history (`history.for_prompt(...)`, `turn.rs:273`);
   - call `run_sampling_request` (which does the actual model call);
   - check post-sampling token status; if the model wants to continue but the context is full, run mid-turn auto-compaction and loop;
   - if the model needs no follow-up, run stop hooks; if a stop hook injects a continuation, loop; else break with the final message;
   - otherwise loop (there are tool outputs to send back).

Error handling (`turn.rs:437`) is careful: an interrupt propagates, an invalid-image request sanitizes and retries, and other errors emit `EventMsg::Error` and break *without killing the thread* — the conversation survives a failed turn.

## Sampling and streaming

`run_sampling_request` (`turn.rs:1130`) builds the `ToolRouter` for this turn (which tools are enabled), fetches base instructions, creates the `ToolCallRuntime`, optionally starts a code-mode worker, and wraps everything in a retry loop driven by `handle_retryable_response_stream_error` (see [The model client](07-model-client.md)). `ContextWindowExceeded` and `UsageLimitReached` short-circuit out rather than retrying.

The real streaming happens in `try_run_sampling_request` (`turn.rs:1955`). It opens a `ResponseStream` and enters an event loop over `ResponseEvent`s. The key handlers:

- **`OutputItemAdded`** (`turn.rs:2159`) — the model started an item. Emit `ItemStarted`, and for a custom tool call, wire up a live argument-diff consumer (this is what powers the streaming apply_patch preview).
- **`OutputItemDone`** (`turn.rs:2062`) — the model finished an item. This is the tool bridge: `handle_output_item_done` (`stream_events_utils.rs:319`) converts it to a tool future or a finalized message. Tool futures are pushed to an ordered queue (`in_flight: FuturesOrdered`, `turn.rs:1999`).
- **Deltas** (`OutputTextDelta`, reasoning deltas, `ToolCallInputDelta`) — streamed to the UI as fine-grained events.
- **`Completed`** (`turn.rs:2294`) — terminal: record token usage, decide whether a follow-up is needed (`end_turn == Some(false)`), and break.

After the stream ends, `drain_in_flight` (`turn.rs:1908`) awaits each tool future *in order* and records each output into history — so tool outputs are recorded deterministically even though they were started eagerly. This is the loop's cleverest move: tool execution overlaps streaming (low latency) without reordering hazards (deterministic history).

## The tool bridge

`handle_output_item_done` (`stream_events_utils.rs:319`) takes a completed `ResponseItem` and decides what it is:

- **A tool call** — `ToolRouter::build_tool_call` converts a `FunctionCall`/`CustomToolCall` into a `ToolCall`. The call item is recorded into history immediately, then a tool future is created via `tool_runtime.handle_tool_call(call, token)`, and `needs_follow_up` is set.
- **Not a tool call** — finalize the message or reasoning as a `TurnItem`, emit `ItemStarted`/`ItemCompleted`, record it, and capture it as the potential `last_agent_message`.
- **An error the model should see** — synthesize a `FunctionCallOutput` carrying the error message and set `needs_follow_up` (the model gets a chance to recover).
- **A fatal error** — kill the turn.

## Parallel tool calls

Codex supports parallel tool calls, gated per-model by `Prompt.parallel_tool_calls = model_info.supports_parallel_tool_calls`. The concurrency mechanism is elegant (`core/src/tools/parallel.rs`): a single `Arc<RwLock<()>>` gate. A parallel-safe tool (like `shell_command` reading output) takes a **read** lock and runs concurrently with other read-lock holders; a non-parallel-safe tool takes a **write** lock and serializes against everything. So three file reads run at once, but a `git commit` blocks until it has exclusive access. Each tool's `supports_parallel_tool_calls()` declares which lock it takes.

## Interruption and cancellation

Interruption is layered:

- `Op::Interrupt` → `abort_all_tasks(Interrupted)` (`tasks/mod.rs:497`). This cancels the turn's `CancellationToken`, waits up to `GRACEFULL_INTERRUPTION_TIMEOUT_MS = 100ms` for a graceful stop, then hard-aborts.
- On interrupt, an "interrupted turn" marker is written into history so the model knows on the next turn that it was cut off (`tasks/mod.rs:97`).
- The `CancellationToken` threads all the way down: `start_task` → `run_turn` → `run_sampling_request` → the stream (via `.or_cancel(&token)`) → each per-tool token. Cancellation propagates cleanly through the whole stack.

## Sub-tasks: review, compact, multi-agent

The task abstraction lets the loop spawn nested work. `Op::Review` runs a `ReviewTask` that executes a one-shot nested Codex thread with review prompts (`codex_delegate.rs`, `codex_prompts::REVIEW_PROMPT`), emitting `EnteredReviewMode`/`ExitedReviewMode`. Compaction is a `CompactTask`. Multi-agent (v1 and v2) spawns *child threads* through the same delegate machinery, and a v2 child's terminal turn events are forwarded up to the parent (`mod.rs:1776`). This is how one Codex session can fan out into a small team of sub-agents — the topology is recorded in the `agent-graph-store` (see [Context, state, and memory](08-context-state-memory.md)).

## Design assessment

The core loop's strengths and weaknesses are covered in full in [Critique](12-critique.md), but in brief: the queue separation, the uniform `SessionTask` trait, the overlap-streaming-with-ordered-recording trick, and the oneshot-channel approval plumbing are genuinely well-designed. The cost is concentration: `session/mod.rs` is ~4,000 lines, `turn.rs` ~2,500, and `Session` is a god-object whose `run_turn` mixes compaction, hooks, skills, connectors, analytics, and token budgeting inline, guarded by a combinatorial thicket of feature flags.
