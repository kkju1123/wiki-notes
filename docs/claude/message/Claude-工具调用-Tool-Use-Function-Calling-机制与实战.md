---
title: Claude 工具调用（Tool Use / Function Calling）机制与实战
url: https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
source_type: web
folder: claude/message
author: null
tags:
- Claude
- Tool Use
- Function Calling
- Agent
- MCP
summary: 介绍 Claude 如何通过工具调用连接外部 API 与函数，覆盖客户端/服务端工具、执行往返流程、调用控制与定价模型。
fetched_at: '2026-09-20T03:31:10.382869+00:00'
---

Connect Claude to external tools and APIs. See where tools execute, when Claude calls them, and which tool fits your task.

Tool use (also called function calling) lets Claude call functions that you define or that Anthropic provides. Claude determines when to call a tool based on the user's request and the tool's description. It then returns a structured call that your application executes (client tools) or that Anthropic executes (server tools).

Here's a minimal example using a server tool, the [Web search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool), which Anthropic executes for you:

Claude runs the search on Anthropic's infrastructure and returns the cited results in the same response. To have Claude call a function that you define, pass a tool with an `input_schema`, then execute the call when Claude returns a `tool_use` block. [How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview#how-tool-use-works) shows that round trip end to end. Learn more about [defining tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools) and [handling tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls).

## How tool use works

Tools differ primarily by where the code executes. **Client tools** (including user-defined tools and tools with Anthropic-defined schemas, such as `bash` and `text_editor`) run in your application. Claude responds with `stop_reason: "tool_use"` and one or more `tool_use` blocks. Your code executes the operation and sends back a `tool_result`. **Server tools** (such as `web_search`, `web_fetch`, `code_execution`, and `tool_search`) run on Anthropic's infrastructure: you see the results directly without handling execution, unless Claude calls the tool in the same group of parallel tool calls as one of your client tools (see [Stop reasons and fallback](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons#tool-use)).

Here's that round trip in full for a client tool. The first request defines a `get_weather` tool, and Claude answers the question by calling it: the response carries a `tool_use` block, your code runs the lookup, and a second request sends the result back in a `tool_result` block so Claude can reply with the answer.

[Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls) covers each step in detail, including result formatting and error signaling; [Parallel tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use) covers responses that call several tools at once. To skip writing this round trip yourself, use [Tool Runner](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner): the SDKs execute your tools and send the results back automatically.

For the full conceptual model including the agentic loop and when to choose each approach, see [How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works).

To connect to Model Context Protocol (MCP) servers, see the [MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector). To build your own MCP client, see the Model Context Protocol guide to [building an MCP client](https://modelcontextprotocol.io/docs/develop/build-client).

## When Claude uses tools

With the default `tool_choice` of `{"type": "auto"}`, Claude determines on each turn whether to call a tool or respond directly. It calls a tool when the request maps to that tool's described capability and the answer isn't already in context. It responds directly for stable knowledge, creative tasks, and conversational turns.

This boundary is steerable through your system prompt. If Claude isn't calling tools when you expect, a light instruction such as `"Use the tools to investigate before responding."` increases tool use. A stronger form such as `"Always call a tool first before responding."` pushes further. Conversely, `"Use your judgment about whether to call a tool or respond directly."` keeps triggering behavior conservative.

To require a tool call rather than rely on prompting, set [`tool_choice`](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools#forcing-tool-use).

Each server tool's page describes its own trigger boundary in more detail.

## Choose a tool

For `type` strings, versions, and beta headers, see [Tool reference](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference).

### Your own tools

For tools you define, you write the schema and your application executes each call.

Specify tool schemas, write descriptions, and control when Claude calls your tools.

Parse `tool_use` blocks, format `tool_result` responses, and handle errors.

### Anthropic-schema client tools

Anthropic publishes the schema and trains Claude on it. Your application still executes each call and returns the `tool_result`.

Store and retrieve information across conversations in files you control.

Run shell commands in a persistent session that maintains state.

View and modify text files to debug, fix, and improve code.

Take screenshots and control the mouse and keyboard in a desktop environment.

Navigate, read, and interact with webpages in your own browser environment.

### Server tools

Server tools run on Anthropic's infrastructure, with no handler code in your application. See [Server tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools) for the mechanics they share.

Search the web for information beyond the knowledge cutoff, with cited sources.

Retrieve the full content of specified web pages and PDF documents.

Run Python and bash code in a sandboxed container to analyze data and generate files.

Let a faster executor model consult a higher-intelligence advisor model mid-generation.

Work with thousands of tools by discovering and loading them on demand.

Connect to remote MCP servers from the Messages API without a separate MCP client.

## Pricing

Tool use requests are priced based on:

1.   The total number of input tokens sent to the model (including in the `tools` parameter)
2.   The number of output tokens generated
3.   For server-side tools, additional usage-based pricing (for example, web search charges per search performed)

Client-side tools are priced the same as any other Claude API request, although server-side tools can incur additional charges based on their specific usage.

The additional tokens from tool use come from:

*   The `tools` parameter in API requests (tool names, descriptions, and schemas)
*   `tool_use` content blocks in API requests and responses
*   `tool_result` content blocks in API requests

When you use `tools`, the API also automatically includes a special system prompt for the model that enables tool use. The number of tool use tokens required for each model is listed in the following table (excluding the additional tokens listed earlier). Note that the table assumes at least 1 tool is provided. If no `tools` are provided, then a tool choice of `none` uses 0 additional system prompt tokens.

| Model | Tool choice | Tool use system prompt token count |
| --- | --- | --- |
| Claude Opus 5 | `auto`, `none` * * * `any`, `tool` | 286 tokens * * * 406 tokens |
| Claude Opus 4.8 | `auto`, `none` * * * `any`, `tool` | 290 tokens * * * 410 tokens |
| Claude Opus 4.7 | `auto`, `none` * * * `any`, `tool` | 675 tokens * * * 804 tokens |
| Claude Opus 4.6 | `auto`, `none` * * * `any`, `tool` | 497 tokens * * * 589 tokens |
| Claude Opus 4.5 | `auto`, `none` * * * `any`, `tool` | 496 tokens * * * 588 tokens |
| Claude Opus 4.1 ([retired, except on Bedrock and Google Cloud](https://platform.claude.com/docs/en/about-claude/model-deprecations)) | `auto`, `none` * * * `any`, `tool` | 313 tokens * * * 315 tokens |
| Claude Opus 4 ([retired, except on Google Cloud](https://platform.claude.com/docs/en/about-claude/model-deprecations)) | `auto`, `none` * * * `any`, `tool` | 313 tokens * * * 315 tokens |
| Claude Sonnet 5 | `auto`, `none` * * * `any`, `tool` | 354 tokens * * * 474 tokens |
| Claude Sonnet 4.6 | `auto`, `none` * * * `any`, `tool` | 497 tokens * * * 589 tokens |
| Claude Sonnet 4.5 | `auto`, `none` * * * `any`, `tool` | 496 tokens * * * 588 tokens |
| Claude Sonnet 4 ([retired, except on Bedrock and Google Cloud](https://platform.claude.com/docs/en/about-claude/model-deprecations)) | `auto`, `none` * * * `any`, `tool` | 313 tokens * * * 315 tokens |
| Claude Haiku 4.5 | `auto`, `none` * * * `any`, `tool` | 496 tokens * * * 588 tokens |
| Claude Haiku 3.5 ([retired, except on Bedrock and Google Cloud](https://platform.claude.com/docs/en/about-claude/model-deprecations)) | `auto`, `none` * * * `any`, `tool` | 264 tokens * * * 355 tokens |

These token counts are added to your normal input and output tokens to calculate the total cost of a request.

See the [Models overview](https://platform.claude.com/docs/en/models/overview#latest-models-comparison) table for current per-model prices.

When you send a tool use prompt, like any other API request, the response includes both input and output token counts in the reported `usage` metrics.

Some server tools add usage-based charges on top of tokens: see [Web search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool#usage-and-pricing) and [Code execution tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#usage-and-pricing) for their rates.

## Next steps

Understand the tool use loop, where tools execute, and when to use tools instead of prose.

A guided walkthrough from a single tool call to a production-ready agentic loop.

Directory of Anthropic-provided tools and reference for optional tool definition properties.