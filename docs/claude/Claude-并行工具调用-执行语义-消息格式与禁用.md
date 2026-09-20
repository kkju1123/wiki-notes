---
title: Claude 并行工具调用：执行语义、消息格式与禁用
url: https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use
source_type: web
folder: claude
author: null
tags:
- Claude
- 工具调用
- 并行执行
- Agent
- Anthropic API
summary: 讲解 Claude 并行工具调用的执行语义、消息历史格式要求、禁用方式与故障排查，确保多工具调用协作可靠。
fetched_at: '2026-09-20T03:40:11.844377+00:00'
---

Enable, format, and disable parallel tool calls, with message-history guidance and troubleshooting.

By default, Claude may call multiple tools in a single response. This page covers how to run those calls, how to format the message history so parallelism keeps working, and how to disable parallel tool use when you need to. For the single-call flow, see [Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls).

## Execution semantics

When Claude calls tools, the response has a `stop_reason` of `tool_use` and can contain several `tool_use` blocks in a single assistant turn. How you run those calls is your decision. The API doesn't prescribe an execution order: you can run the calls concurrently (`Promise.all`, `asyncio.gather`), sequentially in the order they appear, or in any combination that suits your tools.

Choose the strategy based on what your tools do. Independent, read-only operations are usually safe to run in parallel for lower latency. Tools with side effects, shared state, or ordering requirements might be better run sequentially.

Whichever strategy you use, return one `tool_result` for each `tool_use` block, all together in the next user message. Match each result to its call with `tool_use_id`, and put every `tool_result` block before any text content in that message. See [Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls) for the full formatting rules. If you choose not to run a particular call (for example, because you ran the batch sequentially and an earlier call failed), still return a `tool_result` for it with `is_error: true` and a brief explanation.

The [computer use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool#batch-actions) and the [browser use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool#batch-actions) are stricter. When Claude returns several of their member tool calls in one turn (a batch action), run them sequentially in the order they appear and stop at the first failure; each tool defines the exact text to return for the calls you skip.

## Test parallel tool calls

The following script sends a request that should trigger parallel tool calls, verifies the response contains them, and formats the tool results so parallelism keeps working. Run it with `ANTHROPIC_API_KEY` set in your environment:

The summary lines at the end restate the two formatting rules that keep parallelism working: every tool result returns in a single user message, and no text content appears before the tool results in that message.

## Maximizing parallel tool use

Claude 4 and later models make parallel tool calls by default when a request benefits from multiple tools. For all models, you can increase the likelihood of parallel tool calls with targeted prompting:

## Disable parallel tool use

Parallel tool use is on by default. To turn it off, set `disable_parallel_tool_use: true` inside the [`tool_choice`](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools#forcing-tool-use) object. It is not a top-level request parameter. The effect depends on the `tool_choice` type.

### At most one tool call

When `tool_choice` type is `auto` (the default), setting `disable_parallel_tool_use: true` means Claude calls at most one tool per response. Claude can still answer in plain text without calling any tool. The highlighted lines are the only change from a standard tool use request:

### Exactly one tool call

When `tool_choice` type is `any` or `tool`, setting `disable_parallel_tool_use: true` means Claude calls exactly one tool. Claude Fable 5.1 and Claude Mythos 5.1 don't support these `tool_choice` types (see [Forcing tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools#forcing-tool-use)). The following example uses `any`. The same field works with `tool`:

## Troubleshooting

If Claude isn't making parallel tool calls when expected, check these common issues:

**1. Incorrect tool result formatting**

The most common issue is formatting tool results incorrectly in the conversation history. This "teaches" Claude to avoid parallel calls.

Specifically for parallel tool use:

*   **Wrong:** a separate user message for each tool result
*   **Correct:** all tool results together in a single user message

See [Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls) for other formatting rules.

**2. Weak prompting**

Default prompting might not be sufficient. Use the stronger system prompt from [Maximizing parallel tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use#maximizing-parallel-tool-use).

**3. Measuring parallel tool usage**

To verify parallel tool calls are working:

**4. Calls in a batch appear to depend on each other**

Execution order is your choice. If your tools have ordering dependencies, running the batch sequentially and stopping on the first failure is a valid strategy (and the required one for the [computer use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool#batch-actions) and [browser use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool#batch-actions) tools): return `is_error: true` for any call you didn't run. If you run in parallel and a call fails because its prerequisite hadn't completed, return `is_error: true` with the natural error message. Claude will reissue the call on the next turn. To reduce dependent calls appearing together, add this to your system prompt: "Only batch tool calls that are independent of each other."

## Next steps

Use the SDK's Tool Runner abstraction to handle the agentic loop, error wrapping, and type safety automatically.

Parse tool_use blocks, format tool_result responses, and handle errors with is_error.

Specify tool schemas, write effective descriptions, and control when Claude calls your tools.