---
title: Event streaming - Docs by LangChain
url: https://docs.langchain.com/oss/python/langgraph/event-streaming
source_type: web
folder: langGraph
author: null
tags: []
summary: ''
fetched_at: '2026-09-20T08:55:50.592050+00:00'
---

Event streaming is the recommended in-process streaming model for most LangGraph application code. It returns a run stream object that can be consumed in multiple ways at the same time.

## Quickstart

To stream against a graph deployed behind an Agent Server, see the [LangSmith Streaming API](https://docs.langchain.com/langsmith/streaming).

## How the pieces fit together

The streaming stack has two main layers:

1.   **Streaming** emits raw graph execution events from the Pregel engine.
2.   **Event streaming** normalizes those events, runs them through stream transformers, and exposes typed projections.

Pregel engine

Runs graph steps

emits

Raw Pregel events

`updates`, `values`, `messages`, `custom`, `checkpoints`, `tasks`, `debug`

sent to

Event router

Routes each event through the transformer pipeline

cascades through

Stream transformers

ValuesTransformer

MessagesTransformer

…

Custom transformers

produces

Event Stream

Projected events for application code

The event router is the bridge between the two layers. It receives normalized Pregel events and passes each event through the registered stream transformers. Built-in transformers create standard projections such as `stream.messages`, `stream.values`, `stream.subgraphs`, and `stream.output`. Custom transformers can add application-specific projections under `stream.extensions`.

## What event streaming provides

The run stream exposes typed projections over one underlying event flow:

| Projection | Use |
| --- | --- |
| `stream` | Iterate every protocol event. |
| `stream.messages` | Stream chat model messages and token deltas. |
| `stream.values` | Iterate state snapshots and await the final value. |
| `stream.output` | Await the final output. |
| `stream.subgraphs` | Discover and observe nested graph executions. |
| `stream.interrupts` | Inspect human-in-the-loop interrupt payloads. |
| `stream.interrupted` | Check whether the run paused for human input. |
| `stream.extensions` | Consume custom stream transformer projections. |

Multiple consumers can read these projections concurrently. Reading `stream.messages` does not consume events needed by `stream.values`, `stream.subgraphs`, or `stream.output`.Event streaming sits one level above [streaming](https://docs.langchain.com/oss/python/langgraph/streaming), which exposes raw graph execution events through `stream_mode` modes such as `updates`, `values`, `messages`, `custom`, `checkpoints`, `tasks`, and `debug`. Use streaming when you need low-level access to those modes; use event streaming when application code benefits from typed projections.

## Stream messages

Use `stream.messages` for chat model output:

`message.text` is iterable in synchronous code. Iterate it for token-by-token output, or call `str(message.text)` for the complete text.`message.reasoning` exposes reasoning deltas, and `message.tool_calls` exposes tool-call argument chunks. If you need text, reasoning, and tool-call chunks in exact arrival order, iterate the message stream’s raw events instead of each projection separately.

## Stream subgraphs

Use `stream.subgraphs` to observe nested graph work without parsing namespace strings:

`subgraph.graph_name` is the `name` of the compiled graph or agent. A named agent dispatched from a tool (for example, a `create_agent(name=...)` invoked through the Deep Agents `task` tool) surfaces here under that name, and the `lifecycle` event that opens the scope carries a `cause` linking back to the dispatching tool call. See [Lifecycle](https://docs.langchain.com/oss/python/langgraph/event-streaming#lifecycle) for more information.For product-specific streams, see [Deep Agents streaming](https://docs.langchain.com/oss/python/deepagents/event-streaming) for subagent streams and [LangChain agent streaming](https://docs.langchain.com/oss/python/langchain/streaming) for tool calls and middleware events.

## Stream state

Use `stream.values` to stream full state snapshots after each step:

## Stream multiple projections

For concurrent consumption in async code, use `astream_events` with `asyncio.gather`:

For synchronous code, use `stream.interleave(...)` to consume multiple projections in strict arrival order:

## Resume after an interrupt

When a graph pauses for human input, inspect `stream.interrupted` and `stream.interrupts`, then resume by calling `stream_events(..., version="v3")` again with `Command`.Resume requires a graph compiled with a checkpointer and a config carrying a thread ID — see [persistence](https://docs.langchain.com/oss/python/langgraph/persistence).

## Stream all protocol events

Use the run object itself when you want the raw protocol event stream:

Each event is a `ProtocolEvent` envelope wrapping a channel-specific payload. The same shape is what a transformer’s `process(event)` receives.

The `namespace` is a path from the root graph to the scope that emitted the event. The root is the empty array `[]`. Each child execution adds one `"name:runtime_id"` segment, so a nested tool call inside a subgraph looks like `["researcher:6f4d", "tools:91ac"]`. The name before `:` is the stable graph or node name; the suffix is a per-invocation runtime ID. Filter raw events by namespace yourself when you only care about a specific subtree — `stream.subgraphs` already does this for nested graph executions.

## Channels and event lifecycle

Raw events flow on channels. The channel name appears as the event’s `method`; each channel emits a specific event shape.

| Channel | Purpose |
| --- | --- |
| `values` | Full graph state snapshots. |
| `updates` | Per-node state deltas. |
| `messages` | Content-block-centric chat model output. |
| `tools` | Tool call start, streamed output, finish, and error events. |
| `lifecycle` | Run, subgraph, and subagent status changes. |
| `checkpoints` | Lightweight checkpoint envelopes for branching and time travel. |
| `input` | Human-in-the-loop input requests and responses. |
| `tasks` | Pregel task creation and result events. |
| `custom` | User-defined payloads from graph code. |
| `custom:<name>` | Application-defined stream transformer output. |

The typed projections (`stream.messages`, `stream.values`, etc.) are built from these channels. The channel name appears as the `method` field on raw events when you iterate the run object directly.

### Messages

The `messages` channel models output as content blocks. The data’s `event` field is one of:

*   `message-start`
*   `content-block-start`
*   `content-block-delta`
*   `content-block-finish`
*   `message-finish`

Content blocks have explicit boundaries: a block starts, emits zero or more deltas, and finishes before the next block in the same message starts. This makes token streaming, reasoning blocks, tool-call blocks, and multimodal content explicit without requiring provider-specific formats. `message-finish` may include token usage; unrecoverable model-call failures arrive as message error events.To consume raw content-block events directly instead of using the `stream.messages` projection:

### Tools

The `tools` channel exposes tool execution. The data’s `event` field is one of:

*   `tool-started`
*   `tool-output-delta`
*   `tool-finished`
*   `tool-error`

Tool events are correlated by tool call ID, so a tool execution can be joined back to its originating tool-call content block on the `messages` channel.

### Lifecycle

The `lifecycle` channel tracks root run, subgraph, and subagent status. The data’s `event` field is one of:

*   `started`
*   `running`
*   `completed`
*   `failed`
*   `interrupted`

Beyond `event`, lifecycle data may include an optional `graph_name`, `error`, and `cause` describing why a child scope started (parent tool call, fan-out send, edge transition).

## Build your own projection

Stream transformers are the projection layer in event streaming. They observe protocol events, keep their own state, and expose derived views of a run — things like tool activity, token totals, progress events, artifacts, or messages for another protocol. `StreamChannel` is the projection primitive transformers use to publish those views.Built-in projections (`stream.messages`, `stream.values`, `stream.subgraphs`, `stream.output`) and product-specific projections (LangChain’s `stream.tool_calls`, Deep Agents’ `stream.subagents`) are themselves transformers using this same contract. User transformers stack on top via compile-time or call-time registration, and their projections appear under `stream.extensions`.Write one when the existing projections don’t match the shape an application needs.

### How transformers work

Event streaming starts with streaming output from the LangGraph Pregel engine. The runtime normalizes those chunks into protocol events, then a stream handler routes each event through a stack of stream transformers.

The stream handler is the central dispatcher for one stream. For every protocol event, it:

1.   Calls each registered transformer’s `process(event)` hook in order.
2.   Wires named `StreamChannel` pushes back onto the protocol event stream.
3.   Stores the event in the run stream unless a transformer suppresses it.
4.   Calls `finalize()` or `fail()` on every transformer when the run ends.

Transformers are observational. They do not call back into the graph runtime. Instead, they consume events and push derived values into `StreamChannel`, promises, or other projection objects.

### Transformer shape

A transformer implements the `StreamTransformer` interface:

*   `init()` creates the projection object. User transformer projections appear under `stream.extensions`.
*   `process()` observes each protocol event. See [Stream all protocol events](https://docs.langchain.com/oss/python/langgraph/event-streaming#stream-all-protocol-events) for the `ProtocolEvent` shape. Return `false` only when you intentionally want to suppress the original event.
*   `finalize()` closes or resolves non-channel projections after a successful stream.
*   `fail()` propagates errors to non-channel projections.

### Declaring required stream modes

`required_stream_modes` controls which Pregel stream modes the underlying graph emits during the stream. The runtime takes the union of every registered transformer’s `required_stream_modes` and passes that union as the `stream_mode` argument to the graph’s `.stream()` call. **Modes that no transformer requests are never emitted** — declaring `("custom",)` is what causes `custom` events to flow through the run at all.

`process()` receives every event the graph emits and is responsible for filtering by `event["method"]`. The declaration turns on upstream emission; it does not narrow what `process()` sees. Valid values are the Pregel stream modes: `"messages"`, `"tools"`, `"custom"`, `"values"`, `"updates"`, `"checkpoints"`, `"tasks"`, `"debug"`. Each transformer must declare every mode it acts on — an omitted mode is not emitted by the graph and never reaches `process()`.

### StreamChannel

`StreamChannel` is the projection primitive a transformer uses for streaming values. It always exposes an iterable stream on `stream.extensions.<name>`. The constructor argument decides whether each `push()` also flows into the run’s main event stream as a `custom:<name>` event—that is, whether the projection’s values show up when iterating raw protocol events.

| Need | Use |
| --- | --- |
| Side-channel projection only | `StreamChannel()` |
| Also flow each push into the main event stream | `StreamChannel(name)` |

Named channel payloads must be serializable, because each pushed value also becomes a `custom:<name>` protocol event in the main stream. Keep promises, async iterables, class instances, and other in-process handles in unnamed channels.The stream handler owns channel lifecycle. Once `init()` returns a channel, the handler closes or fails it for you when the run ends. Transformers only push values.

### Example: named channel

Pass a string name to `StreamChannel` to expose a streaming projection through `stream.extensions`_and_ forward each pushed value into the run’s main event stream as a `custom:<name>` protocol event:

### Example: unnamed channel

Without a name, the channel is a side-channel projection only — accessible on `stream.extensions` but not visible to consumers iterating raw events. This is the right choice for projections that hold in-process handles (promises, async iterables, class instances) that can’t be serialized onto the main event stream.The example below pairs an unnamed channel with `get_stream_writer`, which lets graph nodes emit `custom`-channel events that the transformer then drains into the projection:

### Example: final-value projection

Use unnamed streams, promises, or other in-process objects when the projection should not flow into the main event stream:

### Register at call time or compile time

Pass transformers at call time for local experimentation:

Compile transformers into the graph when every run of that graph should produce the projection:

### Built-in: `ToolCallTransformer`

LangGraph ships `ToolCallTransformer` as a built-in. Register it to expose `stream.tool_calls` on a plain `StateGraph`:

LangGraph defines the streaming primitives. For using streaming with LangChain or Deep Agents, review the relevant product docs:

*   [LangChain agent streaming](https://docs.langchain.com/oss/python/langchain/event-streaming) covers ReAct-style agent messages, tool calls, and middleware updates.
*   [Deep Agents streaming](https://docs.langchain.com/oss/python/deepagents/event-streaming) covers subagents, nested messages, and subagent tool calls.
*   [LangChain frontend patterns](https://docs.langchain.com/oss/python/langchain/frontend/overview) and [LangGraph frontend patterns](https://docs.langchain.com/oss/python/langgraph/frontend/overview) show UI use cases built on top of streamed state.
*   [LangSmith Streaming API](https://docs.langchain.com/langsmith/streaming) covers streaming against a graph deployed behind an Agent Server.

The wire-level event and command formats are defined in the [Agent Protocol](https://github.com/langchain-ai/agent-protocol) repository and consumable as [`langchain-protocol`](https://pypi.org/project/langchain-protocol/) on PyPI and [`@langchain/protocol`](https://www.npmjs.com/package/@langchain/protocol) on npm.

* * *