---
title: 细粒度工具流式传输（Fine-grained tool streaming）
url: https://platform.claude.com/docs/en/agents-and-tools/tool-use/fine-grained-tool-streaming
source_type: web
folder: claude/message
author: null
tags:
- fine-grained tool streaming
- Claude API
- tool use
- streaming
- eager_input_streaming
summary: 按工具设置 eager_input_streaming，跳过服务端 JSON 缓冲与校验，使大工具参数边生成边下发，降低首分片延迟。
fetched_at: '2026-09-20T04:16:41.930352+00:00'
---

Stream tool inputs without server-side JSON buffering for latency-sensitive applications.

Fine-grained tool streaming delivers a tool's input to your client as Claude generates it, without server-side buffering or JSON validation. Skipping the buffering step reduces the time to the first fragment of a large parameter, such as a document or a block of code, and the fragments arrive through the same [Streaming messages](https://platform.claude.com/docs/en/build-with-claude/streaming) events as standard tool use.

## How to use fine-grained tool streaming

All models support fine-grained tool streaming on the Claude API, [Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock), [Claude Platform on AWS](https://platform.claude.com/docs/en/build-with-claude/claude-platform-on-aws), [Google Cloud](https://platform.claude.com/docs/en/build-with-claude/claude-on-vertex-ai), and [Microsoft Foundry](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry). To use it, set `eager_input_streaming` to `true` on any user-defined tool where you want fine-grained streaming enabled, and enable streaming on your request.

The `eager_input_streaming` field is optional. Setting it to `true` turns on fine-grained streaming for that tool, and omitting it gives you standard buffered streaming, in which the API buffers and validates each parameter value before streaming it back. The exception is a request that still sends the legacy `fine-grained-tool-streaming-2025-05-14` beta header, which turns fine-grained streaming on for tools that leave the field unset. The per-tool field replaces that header, and an explicit `false` keeps buffered streaming for a tool even when a request still sends it. The legacy header cannot be combined with a [computer use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool) or [browser use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool) toolset entry: the API rejects a request that sends both, so remove the header and set `eager_input_streaming` on the user-defined tools that need it. See [Tool reference](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference) for the field definition.

The following example turns on fine-grained streaming for a `make_file` tool and asks Claude for a long poem, so the tool input is large enough to watch it stream in:

Every tab turns on fine-grained streaming for the `make_file` tool. The SDK tabs print each input fragment the moment it arrives, then print the complete accumulated input once the stream ends. The cURL tab shows the raw event stream, and the CLI tab uses `jq` to print just the fragments. Because the printed fragments join into the full tool input, the poem fills your terminal as Claude writes it:

Without `eager_input_streaming`, the API buffers and validates each parameter value before streaming it back, so nothing prints for a large parameter until Claude has finished generating it. With it, fragments start arriving as soon as Claude begins the parameter, and they are typically longer, with fewer mid-word breaks.

## Accumulating tool input deltas

The accumulation contract is the same as for standard tool-use streaming, so this section applies with and without `eager_input_streaming`. See [Input JSON delta](https://platform.claude.com/docs/en/build-with-claude/streaming#input-json-delta) in Streaming messages for the event format. Fine-grained tool streaming changes what you can assume about the result: the server streams fragments without validating them, so the accumulated string might not be valid JSON.

When a `tool_use` content block streams, the initial `content_block_start` event contains `input: {}` (an empty object). This is a placeholder. The actual input arrives as a series of `input_json_delta` events, each carrying a `partial_json` string fragment. To assemble the full input, concatenate these fragments and parse the result when the block closes.

Where your SDK provides an accumulator helper (as the Python, TypeScript, Go, Java, and Ruby tabs in the previous example do), it handles this for you. The manual pattern is for SDKs without a helper, or when you want full control over how the input is assembled.

The accumulation contract:

1.   On `content_block_start` with `type: "tool_use"`, initialize an empty string: `input_json = ""`
2.   For each `content_block_delta` with `type: "input_json_delta"`, append: `input_json += event.delta.partial_json`
3.   On `content_block_stop`, parse the accumulated string

Guard the parse, as the following SDK examples do. A response can also stop at `max_tokens` midway through a parameter. Check the [stop reason](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons) and decide whether to retry the request with a higher `max_tokens` or repair the partial input.

The type mismatch between the initial `input: {}` (object) and `partial_json` (string) is by design. The empty object marks the slot in the content array. The delta strings build the real value.

## Handling invalid JSON in tool responses

With fine-grained tool streaming, the accumulated input for a tool call might be invalid or incomplete JSON. When it is, you cannot run the tool, so report the failure back to Claude instead. The `content` of a tool result does not have to be JSON, but wrapping the raw string in a JSON object under a single key makes it unambiguous to Claude that you received invalid JSON, and preserves the original input for debugging:

Return the wrapper, serialized to a string, as the `content` of a [tool result](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls#handling-errors-with-is-error) content block with `is_error` set to `true`:

## Next steps

Understand how the context window works, how extended thinking and tool use count toward it, and how to manage context as conversations grow.

Stream Messages API responses incrementally with server-sent events, including text, tool use, and extended thinking deltas.

Parse tool_use blocks, format tool_result responses, and handle errors with is_error.

Directory of Anthropic-provided tools and reference for optional tool definition properties.