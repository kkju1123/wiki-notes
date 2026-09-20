---
title: 流式消息：用 Server-Sent Events 增量返回 Claude 响应
url: https://platform.claude.com/docs/en/build-with-claude/streaming
source_type: web
folder: claude
author: null
tags:
- Anthropic Claude
- Server-Sent Events
- 流式 API
- Tool Use
- Thinking
summary: 通过 SSE 增量返回 Claude 文本、工具调用与思考增量，并说明事件流结构和 SDK 用法。
fetched_at: '2026-09-20T02:32:55.776200+00:00'
---

Stream Messages API responses incrementally with server-sent events, including text, tool use, and extended thinking deltas.

When creating a Message, you can set `"stream": true` to incrementally stream the response using [server-sent events](https://developer.mozilla.org/en-US/Web/API/Server-sent%5Fevents/Using%5Fserver-sent%5Fevents) (SSE).

## Streaming with SDKs

The [Python SDK](https://github.com/anthropics/anthropic-sdk-python) and [TypeScript SDK](https://github.com/anthropics/anthropic-sdk-typescript) offer multiple ways of streaming. The [PHP SDK](https://github.com/anthropics/anthropic-sdk-php) provides streaming through `createStream()`. The Python SDK allows both sync and async streams. See the documentation in each SDK for details.

## Get the final message without handling events

If you don't need to process text as it arrives, the SDKs provide a way to use streaming internally while returning the complete `Message` object, identical to what `.create()` returns. This is especially useful for requests with large `max_tokens` values, where the SDKs require streaming to avoid HTTP timeouts.

The `.stream()` call keeps the HTTP connection alive with server-sent events, then `.get_final_message()` (Python) or `.finalMessage()` (TypeScript) accumulates all events and returns the complete `Message` object. In Go, you call `message.Accumulate(event)` inside the stream loop to build the same complete `Message`. In Java, use `MessageAccumulator.create()` and call `accumulator.accumulate(event)` on each event. In C#, await the stream's `.Aggregate()` extension method to get the complete `Message`, or pass a `MessageContentAggregator` to `.CollectAsync()` to aggregate while handling events. In Ruby, call `.accumulated_message` on the stream. In the PHP SDK, you iterate over stream events manually to accumulate the response.

## Event types

Each server-sent event includes a named event type and associated JSON data. Each event uses an SSE event name (for example, `event: message_stop`), and includes the matching event `type` in its data.

Each stream uses the following event flow:

1.   `message_start`: contains a `Message` object with empty `content`. Under the [`thinking-binding-controls-2026-08-01`](https://platform.claude.com/docs/en/build-with-claude/thinking#preserved-thinking-controls) beta header, this `Message` object also carries the `input_transformations` array. After a mid-stream [server-side fallback](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#server-side-fallback), the final `message_delta` event carries the array again with the serving model's entries.
2.   A series of content blocks, each of which has a `content_block_start`, one or more `content_block_delta` events, and a `content_block_stop` event. Each content block has an `index` that corresponds to its index in the final Message `content` array. One exception: during [server-side fallback](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#server-side-fallback) responses, a `fallback` content block arrives at each model boundary as a `content_block_start` and `content_block_stop` pair with no deltas in between.
3.   One or more `message_delta` events, indicating top-level changes to the final `Message` object.
4.   A final `message_stop` event.

### Ping events

Event streams may also include any number of `ping` events.

### Error events

The API may occasionally send [errors](https://platform.claude.com/docs/en/api/errors) in the event stream. For example, during periods of high usage, you may receive an `overloaded_error`, which would normally correspond to an HTTP 529 in a non-streaming context:

### Other events

In accordance with the [versioning policy](https://platform.claude.com/docs/en/api/versioning), new event types may be added, and your code should handle unknown event types gracefully.

## Content block delta types

Each `content_block_delta` event contains a `delta` of a type that updates the `content` block at a given `index`.

### Text delta

A `text` content block delta looks like:

### Input JSON delta

The deltas for `tool_use` content blocks correspond to updates for the `input` field of the block. To support maximum granularity, the deltas are _partial JSON strings_, whereas the final `tool_use.input` is always an _object_.

You can accumulate the string deltas and parse the JSON once you receive a `content_block_stop` event, by using a library like [Pydantic](https://docs.pydantic.dev/latest/concepts/json/#partial-json-parsing) to do partial JSON parsing, or by using the [SDKs](https://platform.claude.com/docs/en/cli-sdks-libraries/overview), which provide helpers to access parsed incremental values.

A `tool_use` content block delta looks like:

Note: Current models only support emitting one complete key and value property from `input` at a time. As such, when using tools, there may be delays between streaming events while the model is working. Once an `input` key and value are accumulated, they are emitted as multiple `content_block_delta` events with chunked partial JSON so that the format can automatically support finer granularity in future models.

### Thinking delta

When using [thinking](https://platform.claude.com/docs/en/build-with-claude/thinking#streaming-thinking) with streaming enabled, you'll receive thinking content through `thinking_delta` events. These deltas correspond to the `thinking` field of the `thinking` content blocks.

For thinking content, a special `signature_delta` event is sent just before the `content_block_stop` event. This signature is used to verify the integrity of the thinking block.

When `display: "omitted"` is set on the thinking configuration, no thinking text is streamed. The thinking block opens, receives a `thinking_delta` with an empty `thinking` string and then a single `signature_delta`, and closes. With `display: "updates"` (beta), reasoning blocks stream the same way, and only the [progress updates](https://platform.claude.com/docs/en/build-with-claude/thinking#progress-updates) that some models write between tool calls stream `thinking_delta` events that carry text. See [Controlling thinking display](https://platform.claude.com/docs/en/build-with-claude/thinking#controlling-thinking-display).

A typical thinking delta looks like:

The signature delta looks like:

## Full HTTP stream response

Use the [client SDKs](https://platform.claude.com/docs/en/cli-sdks-libraries/overview) when using streaming mode. However, if you are building a direct API integration, you need to handle these events yourself.

A stream response consists of:

1.   A `message_start` event
2.   Potentially multiple content blocks, each of which contains:
    *   A `content_block_start` event
    *   Potentially multiple `content_block_delta` events
    *   A `content_block_stop` event

3.   One or more `message_delta` events
4.   A `message_stop` event

There may be `ping` events dispersed throughout the response as well. See [Event types](https://platform.claude.com/docs/en/build-with-claude/streaming#event-types) for more details on the format.

### Basic streaming request

### Streaming request with tool use

This request asks Claude to use a tool to report the weather.

### Streaming request with thinking

This request enables thinking with streaming. The `display: "summarized"` setting streams a condensed summary of Claude's reasoning rather than the full chain of thought.

### Streaming request with web search tool use

This request asks Claude to search the web for current weather information.

## Error recovery

### Claude 4.5 and earlier

For Claude 4.5 models and earlier, you can recover a streaming request that was interrupted because of network issues, timeouts, or other errors by resuming from where the stream was interrupted. This approach saves you from re-processing the entire response.

The basic recovery strategy involves:

1.   **Capture the partial response:** Save all content that was successfully received before the error occurred.
2.   **Construct a continuation request:** Create a new API request that includes the partial assistant response as the beginning of a new assistant message.
3.   **Resume streaming:** Continue receiving the rest of the response from where it was interrupted.

### Claude 4.6 and later

For Claude 4.6 and later models, the same capture-and-resume strategy applies, but step 2 changes: instead of placing the partial response in an assistant message, add a user message that instructs the model to continue from where it left off.

1.   **Capture the partial response:** Save all content that was successfully received before the error occurred.
2.   **Construct a continuation request:** Create a new API request with a user message containing the partial response and an instruction to continue, for example:
3.   **Resume streaming:** Continue receiving the rest of the response from where it was interrupted.

### Error recovery best practices

1.   **Use SDK features:** Leverage the SDK's built-in message accumulation and error handling capabilities.
2.   **Handle content types:** Be aware that messages can contain multiple content blocks (`text`, `tool_use`, `thinking`). Tool use and extended thinking blocks cannot be partially recovered. You can resume streaming from the most recent text block.

## Next steps

Handle each `stop_reason` value once a stream completes.

Stream tool input JSON without server-side buffering for lower latency.

Stream thinking output with `thinking_delta` and `signature_delta` events.

Use the official SDKs, which handle streaming, accumulation, and reconnection for you.

Process large volumes of requests asynchronously when you don't need real-time responses.