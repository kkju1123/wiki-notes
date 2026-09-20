---
title: Fault tolerance - Docs by LangChain
url: https://docs.langchain.com/oss/python/langgraph/fault-tolerance
source_type: web
folder: langGraph
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:55:40.354996+00:00'
---

When a node fails—from a slow external API, a transient network error, or an unhandled exception—LangGraph gives you three composable mechanisms to respond:

*   [**Retries**](https://docs.langchain.com/oss/python/langgraph/fault-tolerance#retries): automatically re-run failed attempts based on exception type and backoff settings
*   [**Timeouts**](https://docs.langchain.com/oss/python/langgraph/fault-tolerance#timeouts): cap how long a single attempt may run
*   [**Error handling**](https://docs.langchain.com/oss/python/langgraph/fault-tolerance#error-handling): run a recovery function after all retries are exhausted

Use [**`set_node_defaults`**](https://docs.langchain.com/oss/python/langgraph/fault-tolerance#graph-defaults) to configure these mechanisms once for all nodes instead of repeating them on every `add_node` call.These compose in a fixed order: when a node attempt raises any exception (including [`NodeTimeoutError`](https://reference.langchain.com/python/langgraph/errors/NodeTimeoutError) from a timeout), the retry policy decides whether to retry. Only after retries are exhausted does the error handler run.For stopping a run cleanly at a superstep boundary and resuming later, see [Graceful shutdown](https://docs.langchain.com/oss/python/langgraph/fault-tolerance#graceful-shutdown).

## Retries

A retry policy automatically re-runs a failed node attempt based on exception type and backoff settings.Pass `retry_policy=` to [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node):

### Default behavior

By default, `retry_on` uses `default_retry_on`, which retries on **any** exception except the following (and their subclasses):

*   `ValueError`
*   `TypeError`
*   `ArithmeticError`
*   `ImportError`
*   `LookupError`
*   `NameError`
*   `SyntaxError`
*   `RuntimeError`
*   `ReferenceError`
*   `StopIteration`
*   `StopAsyncIteration`
*   `OSError`

For exceptions from popular HTTP libraries such as `requests` and `httpx`, it only retries on 5xx status codes. [`NodeTimeoutError`](https://reference.langchain.com/python/langgraph/errors/NodeTimeoutError) is retryable by default.

### Parameters

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `max_attempts` | `int` | `3` | Maximum number of attempts, including the first. |
| `initial_interval` | `float` | `0.5` | Seconds before the first retry. |
| `backoff_factor` | `float` | `2.0` | Multiplier applied to the interval after each retry. |
| `max_interval` | `float` | `128.0` | Maximum seconds between retries. |
| `jitter` | `bool` | `True` | Add random jitter to the interval. |
| `retry_on` | `type[Exception] | Sequence[type[Exception]] | Callable[[Exception], bool]` | `default_retry_on` | Exceptions to retry on, or a callable returning `True` for retryable exceptions. |

### Custom retry logic

Pass a callable or exception type to `retry_on`. Import `default_retry_on` to extend the default behavior:

### Inspect retry state

Use execution info inside a node to inspect the current attempt number. This is useful for switching to a fallback when the primary call keeps failing:

`execution_info` exposes the following fields:

| Attribute | Type | Description |
| --- | --- | --- |
| `node_attempt` | `int` | Current attempt number (1-indexed). `1` on the first try, `2` on the first retry, etc. |
| `node_first_attempt_time` | `float | None` | Unix timestamp of when the first attempt started. Constant across retries. |
| `thread_id` | `str | None` | Thread ID for the current execution. `None` without a checkpointer. |
| `run_id` | `str | None` | Run ID for the current execution. `None` when not provided in config. |
| `checkpoint_id` | `str` | Checkpoint ID for the current execution. |
| `task_id` | `str` | Task ID for the current execution. |

`execution_info` is available even without a retry policy—`node_attempt` defaults to `1`.

## Timeouts

The `timeout=` parameter on [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node) caps how long a single node attempt may run. Pass a number (seconds), a `timedelta`, or a [`TimeoutPolicy`](https://reference.langchain.com/python/langgraph/types/TimeoutPolicy) for separate run and idle limits:

### Run timeout

`run_timeout` is a hard wall-clock cap on a single attempt. It is never refreshed, regardless of node activity:

When the limit is exceeded, LangGraph raises [`NodeTimeoutError`](https://reference.langchain.com/python/langgraph/errors/NodeTimeoutError), clears any writes from the failed attempt, and lets the retry policy decide whether to retry.

### Idle timeout

`idle_timeout` is a progress-resetting cap. It fires only when the node stops making observable progress for the specified duration—unlike `run_timeout`, the clock resets whenever the node produces a progress signal:

You can set `run_timeout` and `idle_timeout` together. Whichever fires first cancels the attempt.

#### Progress signals

Under the default `refresh_on="auto"`, the idle clock resets on any of the following:

*   State writes via `CONFIG_KEY_SEND`
*   Stream output (yielded async stream chunks)
*   Child-task scheduling
*   Runtime stream-writer calls
*   Any LangChain callback event from the node or its descendants (LLM tokens, tool calls, chain start/end, etc.)

#### Heartbeat mode

Set `refresh_on="heartbeat"` to narrow the refresh source to explicit `runtime.heartbeat()` calls only. This is useful when you want a strict idle definition that isn’t reset by chatty subordinates:

#### Manual heartbeats

For long-running work that doesn’t naturally emit progress signals, call `runtime.heartbeat()` to manually reset the idle clock:

`runtime.heartbeat()` is a no-op outside an idle-timed attempt, so you can call it unconditionally.

### NodeTimeoutError

When a timeout fires, LangGraph raises [`NodeTimeoutError`](https://reference.langchain.com/python/langgraph/errors/NodeTimeoutError) with structured context about which limit was hit:

| Attribute | Type | Description |
| --- | --- | --- |
| `node` | `str` | Name of the node whose execution timed out. |
| `elapsed` | `float` | Seconds elapsed before the timeout fired. |
| `kind` | `Literal["idle", "run"]` | Which timeout fired. |
| `idle_timeout` | `float | None` | The configured idle timeout (seconds), if any. |
| `run_timeout` | `float | None` | The configured run timeout (seconds), if any. |

`NodeTimeoutError` is retryable by default. Combining `timeout` with a retry policy works out of the box—the timeout clock resets on each new attempt, and writes from a timed-out attempt are cleared before the next retry:

### Dynamic timeouts with Send

When using [`Send`](https://reference.langchain.com/python/langgraph/types/Send) to dispatch nodes dynamically (for example, in map-reduce patterns), you can pass a timeout directly on the `Send` to override the target node’s static timeout for that specific push:

If the timeout is omitted on the `Send`, the target node’s timeout (set at [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node) time) applies. This lets you set a default timeout on the node and tighten it for individual calls.

## Error handling

An error handler runs after a node fails and all retries are exhausted. It receives the current state and can update it or route to a different node using [`Command`](https://reference.langchain.com/python/langgraph/types/Command). This is useful for compensation flows (Saga patterns) where you want to recover gracefully rather than abort the entire graph.Pass `error_handler=` to [`add_node`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/add_node):

The handler fires only after the retry policy is exhausted, or immediately if no retry policy is configured. The retry policy and the error handler stay decoupled: configure when to retry and when to compensate independently.

### NodeError

Error handlers receive failure context through a typed `error: NodeError` parameter, injected by type annotation (the same pattern as `runtime: Runtime`):

[`NodeError`](https://reference.langchain.com/python/langgraph/errors/NodeError) is a frozen dataclass with two fields:

| Attribute | Type | Description |
| --- | --- | --- |
| `node` | `str` | Name of the node whose execution failed. |
| `error` | `BaseException` | The exception raised by the failed node. |

The `error: NodeError` parameter is opt-in. Handlers that don’t need failure context can use simpler signatures like `(state)` or `(state, runtime)`.

### Route with Command

Error handlers can return a [`Command`](https://reference.langchain.com/python/langgraph/types/Command) to update state and route to a specific node, enabling Saga / compensation patterns:

`charge_payment` retries on `ConnectionError` up to 3 times. If retries are exhausted (or the error isn’t a `ConnectionError`), the handler compensates by updating state and routing to `finalize` instead of aborting the graph.

### Resume-safe failures

### Behavior with `interrupt()`

### Subgraph failures

If a node wraps a subgraph and the subgraph raises an unhandled exception, that exception surfaces to the parent node. If the parent node has an error handler, the handler fires with the subgraph’s exception in `error.error`.

## Graph defaults

Instead of repeating the same `retry_policy=`, `error_handler=`, `timeout=`, or `cache_policy=` on every `add_node` call, use [`set_node_defaults`](https://reference.langchain.com/python/langgraph/graph/state/StateGraph/set_node_defaults) to configure graph-wide defaults in one place:

Both `step_a` and `step_b` now share the same retry policy, error handler, and timeout without any duplication.

### Precedence

Per-node values passed directly to `add_node()` always override the defaults set by `set_node_defaults()`. Defaults are resolved at `compile()` time, so you can call `set_node_defaults()` before or after `add_node()` in any order:

### Default error handler

The `error_handler` default is particularly valuable when every graph run maps to an external process (for example a background job row) and any unhandled node failure should mark that process as failed, without repeating `error_handler=` on every `add_node`. Per-node handlers still take precedence when a step needs its own logic:

If `fetch_data` fails after retries, `mark_process_failed` runs. If `charge_payment` fails after retries, `refund_payment` runs instead because the per-node handler overrides the default.The handler accepts the same `(state, error: NodeError)` signature described in [Error handling](https://docs.langchain.com/oss/python/langgraph/fault-tolerance#error-handling). It also accepts `RunnableConfig` as an optional third argument if you need access to config values such as `thread_id`:

### Applicability matrix

Not all defaults apply to all node types. Error-handler nodes (those registered via `add_node(error_handler=...)`) are excluded from certain defaults to prevent unsafe behavior:

| `set_node_defaults` parameter | Applies to regular nodes | Applies to error-handler nodes | Reason |
| --- | --- | --- | --- |
| `retry_policy` | ✅ | ✅ | Handlers should be retried on transient failures |
| `timeout` | ✅ | ✅ | Stuck handlers should be cancelled like stuck regular nodes |
| `error_handler` | ✅ | ❌ | Handlers must never catch themselves |
| `cache_policy` | ✅ | ❌ | Caching handler results is unsafe |

### Scope

Defaults set on a parent graph are **not** inherited by subgraphs. Each graph maintains its own defaults.

## Functional API

The same `timeout=` and `retry_policy=` parameters are available on `@task` and `@entrypoint` in the functional API:

The behavior is identical to `add_node`: `NodeTimeoutError` is raised on timeout, buffered writes are cleared, and the retry policy decides whether to retry.

## Graceful shutdown

Cooperative shutdown lets you stop an in-flight graph run after the current superstep completes and save a resumable checkpoint. This is useful for handling SIGTERM signals or any external supervisor that needs to reclaim resources without losing work.

Create a [`RunControl`](https://reference.langchain.com/python/langgraph/runtime/RunControl) and pass it as `control=` to `invoke` or `stream`. Call `request_drain()` from any thread to signal that the run should stop:

### Semantics

Drain is cooperative and operates between supersteps, never preempting work that is already running:

| Scenario | Behavior |
| --- | --- |
| Node mid-execution | Runs to completion. Drain takes effect on the next superstep. |
| Node with a retry policy currently retrying | Retry loop runs to exhaustion or success. Drain takes effect after. |
| Graph finishes naturally on the same tick as drain | Returns normally. Inspect `control.drain_requested` to distinguish from a normal run. |
| More supersteps remain | Raises `GraphDrained(reason)`. Checkpoint is saved and resumable. |
| Subgraph requests drain | `GraphDrained` bubbles up through the parent and stops it at its own next superstep boundary. |

### Resume after drain

Resume a drained run with `invoke(None, config)` using the same `thread_id`:

### Read drain state inside a node

Access drain state through the `runtime` parameter to adjust node behavior before the superstep boundary is reached:

### SIGTERM hook pattern

The recommended pattern for handling process shutdown:

## Limitations

*   **Timeouts are async-only**: sync nodes with a `timeout` are rejected at compile time.
*   **One handler per node**: each node can have at most one `error_handler`.
*   **Handler failures bubble up**: if the error handler itself raises, that exception propagates as if the node had no handler.
*   **`set_node_defaults` is not inherited by subgraphs**: each graph manages its own defaults independently.

* * *