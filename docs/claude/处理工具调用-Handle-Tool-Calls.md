---
title: 处理工具调用（Handle Tool Calls）
url: https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls
source_type: web
folder: claude
author: null
tags:
- Tool Use
- Claude API
- Function Calling
- Agent
- 工具调用
summary: 讲解如何识别Claude的tool_use停止原因、执行工具并回传tool_result，实现多轮工具调用循环。
fetched_at: '2026-09-20T03:38:27.327365+00:00'
---

[Claude Platform Docs](https://platform.claude.com/docs/en/home)
*   [Messages](https://platform.claude.com/docs/en/intro)
*   [Managed Agents](https://platform.claude.com/docs/en/managed-agents/overview)
*   [Admin](https://platform.claude.com/docs/en/manage-claude/admin-api)
*   
Resources
    *   [Best practices](https://platform.claude.com/docs/en/about-claude/use-case-guides/overview)
    *   [Models & pricing](https://platform.claude.com/docs/en/models/overview)
    *   [CLI, SDKs, and libraries](https://platform.claude.com/docs/en/cli-sdks-libraries/overview)
    *   [Claude API skill](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/claude-api-skill)
    *   [Release notes](https://platform.claude.com/docs/en/release-notes/overview)

[API reference](https://platform.claude.com/docs/en/api/overview)

English

[Console](https://platform.claude.com/)[Log in](https://platform.claude.com/login?returnTo=%2Fdocs%2Fen%2Fagents-and-tools%2Ftool-use%2Fhandle-tool-calls)



Search Ctrl K

First steps

[Intro to Claude](https://platform.claude.com/docs/en/intro)[Get your API key](https://platform.claude.com/docs/en/get-api-key)[Quickstart](https://platform.claude.com/docs/en/get-started)[Authentication](https://platform.claude.com/docs/en/manage-claude/authentication)

Building with Claude

[Features overview](https://platform.claude.com/docs/en/build-with-claude/overview)[Using the Messages API](https://platform.claude.com/docs/en/build-with-claude/working-with-messages)[Stop reasons and fallback](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons)[Refusals and fallback](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)[Fallback credit](https://platform.claude.com/docs/en/build-with-claude/fallback-credit)

Model capabilities

[Effort](https://platform.claude.com/docs/en/build-with-claude/effort)[Task budgets (beta)](https://platform.claude.com/docs/en/build-with-claude/task-budgets)[Fast mode (research preview)](https://platform.claude.com/docs/en/build-with-claude/fast-mode)[Structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)[Citations](https://platform.claude.com/docs/en/build-with-claude/citations)[Streaming Messages](https://platform.claude.com/docs/en/build-with-claude/streaming)[Batch processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)[Search results](https://platform.claude.com/docs/en/build-with-claude/search-results)[Streaming refusals](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/handle-streaming-refusals)[Multilingual support](https://platform.claude.com/docs/en/build-with-claude/multilingual-support)[Embeddings](https://platform.claude.com/docs/en/build-with-claude/embeddings)

[Thinking](https://platform.claude.com/docs/en/build-with-claude/thinking)

Tools

[Overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)[How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works)[Tutorial: Build a tool-using agent](https://platform.claude.com/docs/en/agents-and-tools/tool-use/build-a-tool-using-agent)[Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)[Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)[Parallel tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use)[Tool Runner (SDK)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner)[Strict tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use)[Server tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools)[Web search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool)[Web fetch tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-fetch-tool)[Code execution tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool)[Advisor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool)[Tool search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)[Memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)[Bash tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool)[Text editor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool)[Computer use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool)[Browser use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool)[Troubleshooting](https://platform.claude.com/docs/en/agents-and-tools/tool-use/troubleshooting-tool-use)

Tool infrastructure

[Tool reference](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference)[Manage tool context](https://platform.claude.com/docs/en/agents-and-tools/tool-use/manage-tool-context)[Tool combinations](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-combinations)[Tool use with prompt caching](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-use-with-prompt-caching)[Programmatic tool calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling)[Fine-grained tool streaming](https://platform.claude.com/docs/en/agents-and-tools/tool-use/fine-grained-tool-streaming)

Context management

[Context windows](https://platform.claude.com/docs/en/build-with-claude/context-windows)[Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)[Context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing)[Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)[Mid-conversation system messages and tool changes](https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages)[Build an orchestration mode](https://platform.claude.com/docs/en/build-with-claude/mid-conversation-effort-example)[Cache diagnostics (beta)](https://platform.claude.com/docs/en/build-with-claude/cache-diagnostics)[Token counting](https://platform.claude.com/docs/en/build-with-claude/token-counting)

Working with files

[Files API](https://platform.claude.com/docs/en/build-with-claude/files)[PDF support](https://platform.claude.com/docs/en/build-with-claude/pdf-support)

[Images and vision](https://platform.claude.com/docs/en/build-with-claude/vision)

Skills

[Overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)[Quickstart](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart)[Best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)[Skills for enterprise](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/enterprise)[Skills in the API](https://platform.claude.com/docs/en/build-with-claude/skills-guide)

MCP

[Remote MCP servers](https://platform.claude.com/docs/en/agents-and-tools/remote-mcp-servers)[MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector)

[MCP tunnels](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview)

Claude on cloud platforms

[Amazon Bedrock (Opus 4.7 and later)](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock)[Amazon Bedrock (Opus 4.6 and earlier)](https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock-legacy)[Claude Platform on AWS](https://platform.claude.com/docs/en/build-with-claude/claude-platform-on-aws)[Google Cloud](https://platform.claude.com/docs/en/build-with-claude/claude-on-vertex-ai)[Microsoft Foundry](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry)

[Console](https://platform.claude.com/)

[Messages](https://platform.claude.com/docs/en/intro)Tools

# Handle tool calls

Copy page



Parse tool_use blocks, format tool_result responses, and handle errors with is_error.

Copy page



This page covers the tool-call lifecycle: reading `tool_use` blocks from Claude's response, formatting `tool_result` blocks in your reply, and signaling errors. For the SDK abstraction that handles this automatically, see [Tool Runner](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner).



**Simpler with Tool Runner:** The manual tool handling described on this page is automatically managed by [Tool Runner](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner). Use this page when you need custom control over tool execution.

Claude's response differs based on whether it uses a [client or server tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview#how-tool-use-works).

## Handling results from client tools

The response will have a `stop_reason` of `tool_use` and one or more `tool_use` content blocks that include:

*   `id`: A unique identifier for this particular tool use block. This will be used to match up the tool results later.
*   `name`: The name of the tool being used.
*   `input`: An object containing the input being passed to the tool, conforming to the tool's `input_schema`.

A `tool_use` block for a member of the [computer use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool) or [browser use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool) toolset also carries a `toolset_name` field (`"computer"` or `"browser"`). Its `name` is the member tool Claude is calling, such as `screenshot` or `navigate`, so dispatch those blocks on both fields.

### Example API response with a `tool_use` content block

JSON



```
{
  "id": "msg_01Aq9w938a90dw8q",
  "model": "claude-opus-5",
  "stop_reason": "tool_use",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "I'll check the current weather in San Francisco for you."
    },
    {
      "type": "tool_use",
      "id": "toolu_01A09q90qw90lq917835lq9",
      "name": "get_weather",
      "input": { "location": "San Francisco, CA", "unit": "celsius" }
    }
  ]
}
```

When you receive a tool use response for a client tool, you should:

1.   Extract the `name`, `id`, and `input` from the `tool_use` block.
2.   Run the actual tool in your codebase corresponding to that tool name, passing in the tool `input`.
3.   Continue the conversation by sending a new message with the `role` of `user`, and a `content` block containing the `tool_result` type and the following information:
    *   `tool_use_id`: The `id` of the tool use request this is a result for.
    *   `content` (optional): The result of the tool, as a string (for example, `"content": "15 degrees"`), a list of nested content blocks (for example, `"content": [{"type": "text", "text": "15 degrees"}]`), or a list of document blocks (for example, `"content": [{"type": "document", "source": {"type": "text", "media_type": "text/plain", "data": "15 degrees"}}]`). These content blocks can use the `text`, `image`, `document`, or [`search_result`](https://platform.claude.com/docs/en/build-with-claude/search-results) types.
    *   `is_error` (optional): Set to `true` if the tool execution resulted in an error.

A `tool_result` that answers a computer use or browser use member block must also echo the same `toolset_name` value as the `tool_use` block; a member result that omits it is rejected. Its `content` is also narrower: a member result may contain only `text` and `image` blocks, and a browser use result may add one [`browser_state`](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool#track-tabs-and-page-state) block (the [tab-management members](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool#tab-management-results) return only that block).



**Important formatting requirements:**

*   Tool result blocks must immediately follow their corresponding tool use blocks in the message history. You cannot include any messages between the assistant's tool use message and the user's tool result message.
*   In the user message containing tool results, the tool_result blocks must come FIRST in the content array. Any text must come AFTER all tool results.
*   If the assistant turn also called a [server tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools) that has no result block yet, the user message must contain only `tool_result` blocks. Text after the results ends the turn early; for a server tool Claude called directly, the request then fails with a 400 error that names the unresolved server tool. See [Stop reasons and fallback](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons#tool-use).

For example, this will cause a 400 error:

```
{
  "role": "user",
  "content": [
    { "type": "text", "text": "Here are the results:" }, // ❌ Text before tool_result
    { "type": "tool_result", "tool_use_id": "toolu_01" /* ... */ }
  ]
}
```



This is correct when the assistant turn calls only client tools:

```
{
  "role": "user",
  "content": [
    { "type": "tool_result", "tool_use_id": "toolu_01" /* ... */ },
    { "type": "text", "text": "What should I do next?" } // ✅ Text after tool_result
  ]
}
```



If you receive an error like "tool_use ids were found without tool_result blocks immediately after", check that your tool results are formatted correctly.



Tool results often carry content from sources outside your control: web pages, inbound email, user uploads, third-party APIs. Treat that content as untrusted: an attacker who can influence it may embed instructions that try to redirect Claude (indirect prompt injection). Keep untrusted content inside `tool_result` blocks rather than `system` prompts or plain user `text` blocks, and see [Mitigate jailbreaks and prompt injections](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/mitigate-jailbreaks#indirect-prompt-injection) for further hardening.

### Example of successful tool result

JSON



```
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
      "content": "15 degrees"
    }
  ]
}
```

### Example of tool result with images

JSON



```
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
      "content": [
        { "type": "text", "text": "15 degrees" },
        {
          "type": "image",
          "source": {
            "type": "base64",
            "media_type": "image/jpeg",
            "data": "/9j/4AAQSkZJRg..."
          }
        }
      ]
    }
  ]
}
```

### Example of empty tool result

JSON



```
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9"
    }
  ]
}
```

### Example of tool result with documents

JSON



```
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
      "content": [
        { "type": "text", "text": "The weather is" },
        {
          "type": "document",
          "source": {
            "type": "text",
            "media_type": "text/plain",
            "data": "15 degrees"
          }
        }
      ]
    }
  ]
}
```

After receiving the tool result, Claude will use that information to continue generating a response to the original user prompt.

## Handling results from server tools

Claude executes the tool internally and incorporates the results directly into its response without requiring additional user interaction.



A response can contain both a client `tool_use` block and a `server_tool_use` block that has no result block. That server tool call is not finished yet, and its result block arrives in a later response. Reply with a user message that contains only the `tool_result` blocks for the client tools and keep the same `tools` array; for a server tool Claude called directly, the API runs it on that request and the next response starts with its result block. See [Stop reasons and fallback](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons#tool-use).



**Differences from other APIs**

Unlike APIs that separate tool use or use special roles like `tool` or `function`, the Claude API integrates tools directly into the `user` and `assistant` message structure.

Messages contain arrays of `text`, `image`, `tool_use`, and `tool_result` blocks. `user` messages include client content and `tool_result`, while `assistant` messages contain AI-generated content and `tool_use`.

## Handling errors with is_error

There are a few different types of errors that can occur when using tools with Claude:

### Tool execution error

If the tool itself throws an error during execution (for example, a network error when fetching weather data), you can return the error message in the `content` along with `"is_error": true`:

JSON



```
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
      "content": "ConnectionError: the weather service API is not available (HTTP 500)",
      "is_error": true
    }
  ]
}
```

Claude will then incorporate this error into its response to the user. For example: "I'm sorry, I was unable to retrieve the current weather because the weather service API is not available. Please try again later."



Write instructive error messages. Instead of generic errors like `"failed"`, include what went wrong and what Claude should try next (for example, `"Rate limit exceeded. Retry after 60 seconds."`). This gives Claude the context it needs to recover or adapt without guessing.

### Invalid tool name

If Claude's attempted use of a tool is invalid (for example, missing required parameters), it usually means that there wasn't enough information for Claude to use the tool correctly. Your best bet during development is to try the request again with more-detailed `description` values in your tool definitions.

However, you can also continue the conversation forward with a `tool_result` that indicates the error, and Claude will try to use the tool again with the missing information filled in:

JSON



```
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
      "content": "Error: Missing required 'location' parameter",
      "is_error": true
    }
  ]
}
```

If a tool request is invalid or missing parameters, Claude will retry 2-3 times with corrections before apologizing to the user.



To eliminate invalid tool calls entirely, use [strict tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use) with `strict: true` on your tool definitions. This guarantees that tool inputs will always match your schema exactly, preventing missing parameters and type mismatches.

### Server tool errors

When server tools encounter errors (for example, network issues with Web Search), Claude will transparently handle these errors and attempt to provide an alternative response or explanation to the user. Unlike client tools, you do not need to handle `is_error` results for server tools.

For web search specifically, possible error codes include:

*   `too_many_requests`: Rate limit exceeded
*   `invalid_input`: Invalid search query parameter
*   `max_uses_exceeded`: Maximum web search tool uses exceeded
*   `query_too_long`: Query exceeds maximum length
*   `unavailable`: An internal error occurred

## Next steps



[Parallel tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use)

Handle responses where Claude calls several tools in a single turn.



[Tool Runner (SDK)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner)

Let the SDK manage the `tool_use` loop, result formatting, and retries for you.



[Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)

Write schemas and descriptions that steer Claude toward the right tool.

Was this page helpful?



*   [Handling results from client tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls#handling-results-from-client-tools)
*   [Handling results from server tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls#handling-results-from-server-tools)
*   [Handling errors with is_error](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls#handling-errors-with-is-error)
*   [Next steps](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls#next-steps)

[Claude Platform Docs](https://platform.claude.com/docs/en/home)

[](https://x.com/claudeai)[](https://www.threads.com/@claudeai)[](https://www.linkedin.com/showcase/claude)[](https://www.youtube.com/@anthropic-ai)[](https://instagram.com/claudeai)



### Solutions

*   [AI agents](https://claude.com/solutions/agents)
*   [Code modernization](https://claude.com/solutions/code-modernization)
*   [Coding](https://claude.com/solutions/coding)
*   [Customer support](https://claude.com/solutions/customer-support)
*   [Financial services](https://claude.com/solutions/financial-services)
*   [Government](https://claude.com/solutions/government)
*   [Higher education](https://claude.com/solutions/education)
*   [K-12 teachers](https://claude.com/solutions/teachers)
*   [Life sciences](https://claude.com/solutions/life-sciences)

### Partners

*   [Claude on AWS](https://claude.com/partners/amazon-bedrock)
*   [Claude on Google Cloud](https://claude.com/partners/google-cloud-vertex-ai)

### Learn

*   [Blog](https://claude.com/blog)
*   [Courses](https://claude.com/resources/courses)
*   [Use cases](https://claude.com/resources/use-cases)
*   [Connectors](https://claude.com/partners/mcp)
*   [Customer stories](https://claude.com/customers)
*   [Engineering at Anthropic](https://www.anthropic.com/engineering)
*   [Events](https://www.anthropic.com/events)
*   [Powered by Claude](https://claude.com/partners/powered-by-claude)
*   [Service partners](https://claude.com/partners/services)
*   [Startups program](https://claude.com/programs/startups)

### Company

*   [Anthropic](https://www.anthropic.com/company)
*   [Careers](https://www.anthropic.com/careers)
*   [Economic Futures](https://www.anthropic.com/economic-futures)
*   [Research](https://www.anthropic.com/research)
*   [News](https://www.anthropic.com/news)
*   [Responsible Scaling Policy](https://www.anthropic.com/news/announcing-our-updated-responsible-scaling-policy)
*   [Security and compliance](https://trust.anthropic.com/)
*   [Transparency](https://www.anthropic.com/transparency)

### Learn

*   [Blog](https://claude.com/blog)
*   [Courses](https://claude.com/resources/courses)
*   [Use cases](https://claude.com/resources/use-cases)
*   [Connectors](https://claude.com/partners/mcp)
*   [Customer stories](https://claude.com/customers)
*   [Engineering at Anthropic](https://www.anthropic.com/engineering)
*   [Events](https://www.anthropic.com/events)
*   [Powered by Claude](https://claude.com/partners/powered-by-claude)
*   [Service partners](https://claude.com/partners/services)
*   [Startups program](https://claude.com/programs/startups)

### Help and security

*   [Availability](https://www.anthropic.com/supported-countries)
*   [Status](https://status.claude.com/)
*   [Support](https://support.claude.com/)
*   [Discord](https://www.anthropic.com/discord)

### Terms and policies

*   [Privacy policy](https://www.anthropic.com/legal/privacy)
*   [Responsible disclosure policy](https://www.anthropic.com/responsible-disclosure-policy)
*   [Terms of service: Commercial](https://www.anthropic.com/legal/commercial-terms)
*   [Terms of service: Consumer](https://www.anthropic.com/legal/consumer-terms)
*   [Usage policy](https://www.anthropic.com/legal/aup)

Ask Docs