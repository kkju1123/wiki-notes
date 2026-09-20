---
title: Claude Managed Agents 会话事件流与增量预览
url: https://platform.claude.com/docs/en/managed-agents/events-and-streaming
source_type: web
folder: claude/agent
author: null
tags:
- 事件流
- 增量预览
- Claude Managed Agents
- 流式渲染
- 多智能体
summary: 系统讲解 Claude Managed Agents 事件流与 event deltas 增量预览机制，重点是如何累加流式片段并以权威消息为准。
fetched_at: '2026-09-20T07:32:44.675947+00:00'
---

Send events, stream responses, and interrupt or redirect your session mid-execution.

Communication with Claude Managed Agents is event-based. You send user events to the agent, and receive agent and session events back to track status.

## Event types

Events flow in two directions.

*   **User events** and **system events** are what you send to the agent: `user.*` events start a session and steer it as it progresses; `system.message` appends system-level context that applies to the accompanying turn and all subsequent turns.
*   **Session events**, **span events**, and **agent events** are sent to you for observability into your session state and agent progress. Stream connections that opt in also receive [event deltas](https://platform.claude.com/docs/en/managed-agents/events-and-streaming#event-deltas).

Session, span, agent, user, and system event type strings follow a `{domain}.{action}` naming convention. The stream-only delta preview events (`event_start`, `event_delta`) are the exception. See [Event types](https://platform.claude.com/docs/en/managed-agents/reference#event-types) in the reference for the full catalog. [Webhook event types](https://platform.claude.com/docs/en/managed-agents/webhooks#supported-event-types) are separate, and some of their names differ from the stream's (for example, `session.status_idled` rather than `session.status_idle`).

Every persisted event includes a `processed_at` timestamp set when the event finishes processing. On events you send, `processed_at` is null while the event is still queued behind earlier events. The exceptions are `user.define_outcome`, `user.custom_tool_result`, and `user.tool_result`, which are processed on receipt and echoed back with `processed_at` already populated.

## Integrating events

## Event deltas

By default, the agent's response text reaches the stream as buffered `agent.message` events, each emitted only after the model request that produced it finishes. Event deltas let you render that text incrementally, as a live preview, while the model is still generating it. A preview is not the response: previews are a best-effort display aid, and the buffered `agent.message` is always the authoritative record. A client that ignores previews still receives a complete, correct stream.

### Opt in to previews

Previews are opt-in per stream connection. Add the `event_deltas[]` query parameter to the stream you're reading, repeating it once for each event type you want previewed. Because `[]` is a shell glob pattern, quote the URL whenever you build the request in a shell; the examples percent-encode the brackets as `%5B%5D`, which also works. Both stream endpoints accept the parameter: the session-level stream at `GET /v1/sessions/{session_id}/events/stream`, and each [session thread](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration)'s own stream at `GET /v1/sessions/{session_id}/threads/{thread_id}/stream`. The accepted values are `agent.message` and `agent.thinking`; any other value returns a 400 error, as does a request with more than 100 values. A subagent's previews appear on [that subagent's own thread stream](https://platform.claude.com/docs/en/managed-agents/events-and-streaming#preview-session-thread-events).

When a previewed event begins, the stream emits an `event_start` carrying the upcoming event's type and `id`:

For `agent.message`, the start is followed by `event_delta` events carrying incremental text. Each delta names the event it extends in `event_id` and the content block it extends in `delta.index`:

When an `agent.thinking` event is previewed, only the `event_start` is emitted. No `event_delta` events follow, and the buffered `agent.thinking` event that concludes the preview carries no thinking content; it is a progress signal, not a content carrier.

Unlike persisted events, `event_start` and `event_delta` have no `id` or `processed_at` of their own. The only identifier they carry is the `id` of the event they preview.

### Accumulate and reconcile

Every SDK that supports event deltas includes an accumulator helper that handles the `index` bookkeeping for you. The Go, Java, Ruby, and C# helpers also key the accumulating preview by the event's `id`; with the Python, TypeScript, and PHP helpers you keep that map yourself and fold each delta into the entry for its `id`. The manual pattern also works in every language when you need custom bookkeeping: apply it to the generated event types.

In the manual pattern, treat the preview as a scratch buffer and the buffered event as the record. Key the buffer by `(event_id, index)`. Reconcile per model request: a turn opens with a single `session.status_running` event, then on a turn that completes normally each model request produces, in order, `span.model_request_start`, `event_start`, the `event_delta` events, the buffered `agent.message`, and finally [`span.model_request_end`](https://platform.claude.com/docs/en/managed-agents/reference#event-types) (in the Span events tab). On the wire, this is the previewed portion of that sequence, interleaved with the connection's other buffered events:

The `event_delta` line repeats once per text fragment. Process each event as it arrives:

1.   On `event_start`, note the announced `id`. The identifiers always line up: `event_start.event.id`, every `event_delta.event_id`, and the buffered `agent.message`'s `id` are the same value.
2.   On each `event_delta`, append `delta.content.text` to the entry at `(event_id, delta.index)` and render the running text. The first delta for an `index` creates that entry.
3.   When the buffered `agent.message` arrives, match it by `id`, discard the accumulated preview, and render the message's content instead.
4.   On `span.model_request_end`, close any preview that has not been reconciled by its buffered event. No more deltas are coming for it. If the turn errors or is interrupted, the buffered event might never arrive; `span.model_request_end` still does.

Guarantees the pattern relies on:

*   Concatenating a preview's deltas in arrival order, keyed by `(event_id, index)`, gives a prefix of `content[index].text` in the buffered event (a prefix, not necessarily the whole text, because deltas might be shed under load).
*   A connection emits at most one `event_start` per `event_id`, and the buffered event is the last thing that connection delivers for that `id`.

### Preview session thread events

In a [multiagent](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration) session, every session thread has its own event stream at `GET /v1/sessions/{session_id}/threads/{thread_id}/stream`, and it takes the same `event_deltas[]` parameter with the same values. Previews are thread-scoped by design: a connection previews only the thread it's reading. A child thread's previews are delivered on that child's own stream and are never cross-posted to the session-level stream, whose previews stay scoped to the primary thread. To watch a subagent's text as the model generates it, open that subagent's thread stream.

The thread stream's path is easy to get wrong: it is `/threads/{thread_id}/stream`, not `/events/stream` (which exists only at the session level), and there is no `/threads/{thread_id}/events/stream` endpoint.

The preview events themselves don't change. `event_start` and `event_delta` have the same shape on a thread stream as on the session-level stream, and the [accumulate and reconcile](https://platform.claude.com/docs/en/managed-agents/events-and-streaming#accumulate-and-reconcile) pattern applies as written. The one adjustment is bookkeeping: run one accumulator instance per stream connection.

The read loop exits on [`session.thread_status_idle`](https://platform.claude.com/docs/en/managed-agents/reference#event-types), the event emitted when the session thread's turn finishes and the thread goes idle.

### Limitations

Previews are tuned for responsiveness. Build against these constraints:

*   **Best effort:** Under load, the server might shed deltas for an event. When it does, you receive a contiguous prefix of the text and then no further deltas for that event. The buffered `agent.message` still arrives complete. Never treat an accumulated preview as final.
*   **No replay on reconnect:** Deltas are delivered only to the connection that opted in, while it is open. This applies to the session-level stream and to each session thread stream alike, and a connection opened after a model request started receives no deltas for that in-flight event. If the stream drops, follow the [reconnect procedure](https://platform.claude.com/docs/en/managed-agents/events-and-streaming#integrating-events) in the Streaming events tab: reopen the stream and list the event history. The history includes any buffered events emitted while you were disconnected, including the `agent.message` your preview was waiting for. There is no way to re-request missed deltas.
*   **One thread, text only:** Previews cover assistant text on the thread the connection is reading. Tool use, tool results, MCP results, and activity on any other [session thread](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration) are never previewed on that connection.
*   **Start-only `agent.thinking`:** An `agent.thinking` preview emits only the `event_start` as a signal that a thinking block has started; no `event_delta` events follow it.
*   **Never persisted:**`event_start` and `event_delta` exist only on the live stream. They do not appear in the session's event history (`GET /v1/sessions/{session_id}/events`) or in any session thread's event history.

### Troubleshoot previews

If the stream doesn't behave as you expect:

| You see | What it means |
| --- | --- |
| A stream with buffered events but no `event_start` or `event_delta` | The connection you're reading didn't opt in (`event_deltas[]` applies per connection, not per session), or the turn never touched the thread you're streaming. Previews are thread-scoped, so list the session's threads (`GET /v1/sessions/{session_id}/threads`) to find which one ran. |
| A 404 on the stream URL | The path or an ID is wrong, or the request carries no managed-agents beta header at all. The thread endpoints are beta-gated, so without the header they don't exist. |
| A 400 naming `event_deltas` | Only `agent.message` and `agent.thinking` are accepted. |

## Additional scenarios

### Handling custom tool calls

When the agent invokes a [custom tool](https://platform.claude.com/docs/en/managed-agents/tools#custom-tools):

1.   The session emits an `agent.custom_tool_use` event containing the tool name and input.
2.   The session pauses with a `session.status_idle` event containing `stop_reason: requires_action`. The blocking event IDs are in the `stop_reason.event_ids` array.
3.   Execute the tool in your system and send a `user.custom_tool_result` event for each, passing the event ID in the `custom_tool_use_id` parameter along with the result content.
4.   Once all blocking events are resolved, the session transitions back to `running`.

### Tool confirmation

A tool call waits for your confirmation under an `always_ask`[permission policy](https://platform.claude.com/docs/en/managed-agents/permission-policies), or under `auto` when the server reaches no determination. When that happens:

1.   The session emits an `agent.tool_use` or `agent.mcp_tool_use` event.
2.   The session pauses with a `session.status_idle` event whose `stop_reason.type` is `requires_action`. The blocking event IDs are in the `stop_reason.event_ids` array.
3.   Send a `user.tool_confirmation` event for each, passing the event ID in the `tool_use_id` parameter. Set `result` to `"allow"` or `"deny"`. Use `deny_message` to explain a denial.
4.   Once all blocking events are resolved, the session transitions back to `running`.

Each `agent.tool_use` and `agent.mcp_tool_use` event carries `evaluated_permission` (`allow`, `ask`, or `deny`), and only events whose `evaluated_permission` is `"ask"` wait for a confirmation. Most events also carry an `evaluation` object that records which policy produced that outcome, described under [See how each call was evaluated](https://platform.claude.com/docs/en/managed-agents/permission-policies#see-how-each-call-was-evaluated). For example, a `bash` call paused under an `always_ask` policy appears on the stream as follows:

### Resuming an idle session

Sessions persist between interactions. Conversation history is preserved unless the session is explicitly deleted. When a session goes idle, its sandbox is checkpointed, preserving the full sandbox state, including the filesystem, installed packages, and any files the agent created. This allows you to resume cleanly from inactivity.

To resume a session, send a `user.message` event to it as usual:

### Reaching a session budget

A session created with a [budget](https://platform.claude.com/docs/en/managed-agents/budgets) pauses instead of overspending. When the session's tracked list cost reaches the cap, the platform pauses each thread before its next model request, and the session goes idle with a `stop_reason` of `budget_reached` rather than terminating. The request that carried the total past the cap runs to completion, so the `list_cost` reported by the `session.usage` snapshot can read [at or a fraction past the cap](https://platform.claude.com/docs/en/managed-agents/budgets#when-a-session-reaches-its-budget). On the stream, the pause arrives as three events, in order:

1.   `session.thread_status_idle` with `stop_reason: budget_reached`, for each thread as it pauses.
2.   `session.usage`, a snapshot of the session's cumulative usage and tracked list cost.
3.   `session.status_idle` with `stop_reason: budget_reached`. The `session.usage` event always immediately precedes this idle.

A thread whose final request both crosses the cap and completes its turn reports `end_turn` on its own `session.thread_status_idle` event while the session still reports `budget_reached`; key on the session-level `stop_reason` to detect the pause.

While the session is at its cap, it accepts only the events that settle work already in flight: `user.tool_confirmation`, `user.tool_result`, `user.custom_tool_result`, and `user.interrupt`. Any event that would start new work, including `user.message`, is rejected with a 400 error naming that list. When a session has both a thread waiting on a tool ask and a thread paused at the cap, the session-level `stop_reason` is `requires_action`, not `budget_reached`: settling the ask doesn't trigger a model request, so respond to it as usual.

No event resumes a session paused at its cap. Instead, update the session's budget: changing the cap to any value above the consumed list cost, or removing the budget by updating the session with `"budget": null`, resumes the paused work automatically. See [Session budgets](https://platform.claude.com/docs/en/managed-agents/budgets) for how list cost is tracked and the full budget update semantics.

### Sending system messages

Send a `system.message` event to give the agent privileged system-level context that applies to the accompanying turn and all subsequent turns. Unlike the `system` field on the agent definition (which sets the top-level system prompt), `system.message` content is appended to the session's system context as a `role: "system"` turn rather than replacing that prompt. Use it when the agent needs updated system-level guidance mid-session: a different persona, revised constraints, or context fetched at runtime that should shape the model's behavior going forward.

While the session is idle with `stop_reason: requires_action`, a `system.message` is accepted only when it trails a tool result event in the same request; sent on its own or with a `user.message`, it is rejected until the pending tool events are resolved. `content` accepts 1–1000 text items.

### Tracking usage

The session object includes a `usage` field with the session's cumulative usage: token counts, server tool use, active time, and the tracked list cost. Fetch the session after it goes idle to read the latest totals.

`input_tokens` reports uncached input tokens and `output_tokens` reports total output tokens across all model calls in the session. The `cache_read_input_tokens` field reports tokens read from the prompt cache, and the `cache_creation` object breaks down cache-creation tokens by cache lifetime (`ephemeral_5m_input_tokens` and `ephemeral_1h_input_tokens`). Cache entries use a 5-minute TTL by default, so back-to-back turns within that window benefit from cache reads, which reduce per-token cost.

`list_cost` is the session's cumulative consumption priced at public list rates, as a whole number of cents in a string, with a currency code. `active_seconds` is the cumulative time during which the session had at least one thread running; overlapping activity from concurrent threads is counted once, unlike the `active_seconds` in the session's `stats` object, which sums each thread's own active time. This deduplicated figure is the duration the session's runtime cost is priced on. `server_tool_use` counts server-executed tool requests for pricing: web search requests are priced into list cost per request, and web fetch requests carry no per-request charge and aren't metered, so `web_fetch_requests` reads `0`. Each [session thread](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration)'s own `usage` carries `list_cost` and `active_seconds` too. Per-thread figures are rounded independently and exclude the session's running-time cost, so they don't sum exactly to the session's `list_cost`; the session figure is the authoritative one.

You don't have to poll the session to observe these totals. The `session.usage` event carries the same cumulative snapshot (the `usage` object, plus the session's `budget`, which is `null` when the session has none) on the session stream and in the event history. It is emitted on idle transitions rather than on a timer: the session emits one immediately before it goes idle, whatever the stop reason, and one when a thread pauses at a [session budget](https://platform.claude.com/docs/en/managed-agents/budgets). A stream reader therefore sees the final cost of a turn, or of the work that hit a budget, without an extra fetch.

To enforce a spend limit, set a [session budget](https://platform.claude.com/docs/en/managed-agents/budgets) rather than polling usage and stopping the session yourself. The platform prices the session's consumption continuously and pauses each thread before its next model request once the session's list cost reaches the cap; see [Reaching a session budget](https://platform.claude.com/docs/en/managed-agents/events-and-streaming#reaching-a-session-budget) for what that looks like on the stream.

## Console observability

The Claude Console includes a session viewer for inspecting what an agent did without writing any code. In the Console sidebar, under **Managed Agents**, select **Sessions** to see every session in the workspace with its status, agent, token usage, cost, and creation time, then select a session to open it. The session viewer is only accessible to Developers and Admins. It shows:

*   **Timeline minimap:** A zoomable overview of the session's activity over time, with one lane per thread in [multiagent](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration) sessions. Select a lane to view that thread, or select a mark to jump to its event.
*   **Transcript:** The conversation grouped by model request, including thinking, tool calls with their inputs and results, and message text as it streams. You can filter the events and copy or download them as JSON.
*   **Inspector:** A resizable side panel with details about the session, in five tabs:
    *   **Session** shows the session's details and metadata, its cumulative cost over time, and spend against the session's [budget](https://platform.claude.com/docs/en/managed-agents/budgets) when one is set.
    *   **Events** lists every raw event on the current thread in the order the server sent it; select an event to see its JSON. A message that streamed while the page was open also has a **Deltas** view of its [event deltas](https://platform.claude.com/docs/en/managed-agents/events-and-streaming#event-deltas).
    *   **Tools** lists the tools the session's agents are configured with, along with call counts, failures, and median duration; select a tool to see its calls and jump to one in the transcript.
    *   **Resources** lists mounted [files](https://platform.claude.com/docs/en/managed-agents/files), [repositories](https://platform.claude.com/docs/en/managed-agents/github), and [memory stores](https://platform.claude.com/docs/en/managed-agents/memory) at their container paths, including the memories in each store and the changes this session made to them, plus files the agent wrote to `/mnt/session/outputs` and the [skills](https://platform.claude.com/docs/en/managed-agents/skills) attached to the session's agents.
    *   **Threads** lists every thread with its status, context size, and cost. Select a thread to view its details, such as the agent, model, context usage, and cost.

Append `?event={event_id}` to a session URL to open the session at a specific event.

With `ant beta:sessions connect`, you can open the same viewer from the `ant` CLI or follow the session in your terminal. See [Connect to a Managed Agents session from your terminal](https://platform.claude.com/docs/en/cli-sdks-libraries/cli/sessions-connect).

## Debugging tips

*   **Check session events:** Session errors are conveyed through the `session.error` event
*   **Review tool results:** Tool execution failures often explain unexpected agent behavior
*   **Track token usage:** Monitor token consumption to optimize prompts and reduce costs
*   **Use system prompts:** Add logging instructions to the system prompt to make the agent explain its reasoning
*   **Troubleshoot previews:** If a stream that opts in to event deltas doesn't behave as you expect, see [Troubleshoot previews](https://platform.claude.com/docs/en/managed-agents/events-and-streaming#troubleshoot-previews)