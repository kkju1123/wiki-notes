---
title: Server tools：服务端工具调用、暂停与混合轮次机制
url: https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools
source_type: web
folder: claude
author: null
tags:
- Claude API
- server_tool_use
- pause_turn
- tool_use
- ZDR
summary: 解析 Claude API 中服务端工具调用的 srvtoolu_ 块、pause_turn 续接、混合服务端与客户端工具轮次及域过滤机制。
fetched_at: '2026-09-20T03:46:06.062093+00:00'
---

Work with Anthropic-executed tools: server_tool_use blocks, pause_turn continuation, mixed server and client tool turns, and domain filtering.

Server-executed tools share these mechanics: the `server_tool_use` block, `pause_turn` continuation, turns that mix server and client tools, Zero Data Retention (ZDR) eligibility, and domain filtering. For individual tools, see the [tool reference](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference).

## The server_tool_use block

The `server_tool_use` block appears in Claude's response when a server-executed tool runs. Its `id` field uses the `srvtoolu_` prefix to distinguish it from client tool calls:

The API executes the tool internally. You see the call and its result in the response, but you don't handle execution. Unlike client `tool_use` blocks, you don't need to respond with a `tool_result`. The tool's result block (for example, `web_search_tool_result` for web search) follows the `server_tool_use` block in the same assistant turn, paired by `tool_use_id`. If Claude calls one of your client tools at the same time, the `server_tool_use` block appears without its result, and the response ends with `stop_reason: "tool_use"`. The API runs the tool when you return the client `tool_result` blocks in your next request.

## The server-side loop and pause_turn

When using server tools such as web search, the API executes tool calls in a server-side agentic loop. On a long-running turn, the API might pause that loop and return a `pause_turn` stop reason.

Here's how to handle the `pause_turn` stop reason:

When handling `pause_turn`:

*   **Continue the conversation:** Pass the paused response back as-is in a subsequent request to let Claude continue its turn.
*   **Preserve tool state:** Include the same tools in the continuation request. A paused turn can end with a `server_tool_use` block whose tool has not run yet, and the API returns a validation error if that tool is missing from the continuation.
*   **Repeat as needed:** A continued turn can pause again. Check `stop_reason` on each response and continue until you get a different stop reason, capping the number of continuations as you would any retry loop.

For the other `stop_reason` values and general handling patterns, see [Stop reasons and fallback](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons).

## Mixing server tools and client tools in one turn

Claude can call a server tool and a client tool in the same group of parallel tool calls, for example, `web_fetch` together with a user-defined tool. A client tool is any tool that your code executes and that produces a `tool_use` block, whether it is user-defined or an Anthropic-schema client tool such as the [Bash tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool). When that happens, the API does not run the server tool. It returns immediately so that you can run the client tool first:

*   `stop_reason` is `"tool_use"`, not `"pause_turn"`.
*   `content` contains the `server_tool_use` block and the client `tool_use` block, but no result block for the server tool: that call is not finished.
*   There is no other marker. Detect the state by looking for a `server_tool_use` block whose `id` has no matching result block in the response. An `mcp_tool_use` block from the [MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector) behaves the same way. Server tool calls that already have their result block in the same response are complete and need nothing from you.

To continue the turn, run the client tools and send a user message whose content is only the `tool_result` blocks, one for each `tool_use` block in that response. Keep the same `tools` array: a resume request that no longer defines the waiting server tool fails with a 400 whose message ends `but no `web_fetch` tool was provided`.

The API attaches your results to the still-open assistant turn, runs the deferred server tool (for paused code execution, resumes it), and then lets Claude continue. For a server tool Claude called directly, the next response begins with the result block that answers the previous response's `server_tool_use``id`, followed by the newly generated content and a fresh `stop_reason`:

A `server_tool_use` block and its result block pair up by `tool_use_id`, not by position: in this flow they arrive in two different responses, and the `server_tool_use` block is not repeated in the second one. On later requests, keep the whole exchange in your `messages` array in order: the first response as an `assistant` message, the `tool_result` user message, and then the next response as another `assistant` message, the same way you accumulate any other tool-use exchange.

**How this differs from `pause_turn`:** A [`pause_turn` response](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools#the-server-side-loop-and-pause-turn) can also end with a `server_tool_use` block that has not run, but it never leaves a client `tool_use` block waiting on you, so you continue it by re-sending the assistant content as-is. A response that leaves a client `tool_use` block waiting on you never has a `stop_reason` of `pause_turn`: when Claude stops to call your tools, `stop_reason` is `tool_use`, and you continue it by sending the client `tool_result` blocks rather than by re-sending the response. In both cases the API runs the pending server tool at the start of the next request.

The following example enables web fetch together with a user-defined `run_command` tool and handles the mixed response:

This code is also correct when Claude does not mix the two kinds of call. A turn with only client `tool_use` blocks takes the same continuation path, and a turn with only server tool calls needs no client `tool_result` blocks from you: its result blocks are normally already present, and one that comes back suspended, such as a [`pause_turn` response](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools#the-server-side-loop-and-pause-turn), is re-sent as-is instead.

## ZDR and allowed_callers

The basic versions of web search (`web_search_20250305`) and web fetch (`web_fetch_20250910`) are eligible for [Zero Data Retention (ZDR)](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention).

The `_20260209` and later versions with dynamic filtering are **not** ZDR-eligible by default because dynamic filtering relies on code execution internally.

To use a `_20260209` or later server tool with ZDR, disable dynamic filtering by setting `"allowed_callers": ["direct"]` on the tool:

This restricts the tool to direct invocation only, bypassing the internal code execution step.

`allowed_callers` controls how a tool can be invoked: directly by Claude (`"direct"`), from inside a code execution container (for example, `"code_execution_20260120"`), or both. The `_20260209` versions of the web tools default to the code execution caller only; earlier versions default to `["direct"]`. On models that don't support programmatic tool calling, these versions require `allowed_callers: ["direct"]`; without it the API returns a validation error that says to set it.

## Domain filtering

Server tools that access the web accept `allowed_domains` and `blocked_domains` parameters to control which domains Claude can reach. Both are fields on the tool object:

When using domain filters:

*   Domains should not include the HTTP/HTTPS scheme (use `example.com` instead of `https://example.com`).
*   Subdomains are automatically included (`example.com` covers `docs.example.com`).
*   Specific subdomains restrict results to only that subdomain (`docs.example.com` returns only results from that subdomain, not from `example.com` or `api.example.com`).
*   Subpaths are supported for web search and match anything after the path (`example.com/blog` matches `example.com/blog/post-1`).
*   Web fetch matches on the domain only: an entry that includes a path never matches a web fetch URL.
*   You can use either `allowed_domains` or `blocked_domains`, but not both in the same request.

**Wildcard support:**

*   Wildcards (`*`) are not allowed in the domain itself, only in the path after it.
*   Valid: `example.com/*`, `example.com/*/articles`
*   Invalid: `*.example.com`, `ex*.com`

Invalid domain formats are rejected at request time with a 400 `invalid_request_error`.

[Claude Managed Agents](https://platform.claude.com/docs/en/managed-agents/overview) uses the same `allowed_domains` and `blocked_domains` fields on the `web_search` and `web_fetch` entries of the agent toolset. On Managed Agents, each list holds at most 64 entries, domains listed for `web_fetch` cannot include a path, and fields specific to the Messages API tools, such as `max_uses`, `citations`, and `cache_control`, are not available. See [Restrict web search and web fetch domains](https://platform.claude.com/docs/en/managed-agents/tools#restrict-web-search-and-web-fetch-domains) for the full rules.

Organization-level web search and web fetch settings in the Claude Console apply to Messages API requests only; they do not apply to Managed Agents sessions, which use only the per-tool lists on the agent toolset.

## Dynamic filtering with code execution

The `_20260209` and later versions of web search and web fetch use code execution internally to apply dynamic filters against search results.

## Streaming server-tool events

Server-tool events stream as part of the normal server-sent events (SSE) flow. A `server_tool_use` block that Claude calls directly streams like a client `tool_use` block: a `content_block_start` event followed by `input_json_delta` events. The result block arrives complete in a single `content_block_start` event, with no deltas.

See [Streaming](https://platform.claude.com/docs/en/build-with-claude/streaming) for the full event reference. Individual tool pages document tool-specific event names where they differ.

## Batch requests

All server tools support batch processing. In a batch, the agentic loop runs just as it does for synchronous requests, with a higher per-turn iteration limit. If the loop reaches that limit, the response ends with `stop_reason: "pause_turn"`; you can continue it by submitting a follow-up request with the returned content. See [Server tools and the agentic loop](https://platform.claude.com/docs/en/build-with-claude/batch-processing#server-tools-and-the-agentic-loop) for details.

Common batch workloads include enriching a dataset with information from the web, checking a large set of documents against current sources, and running analysis code over many files.

## Next steps

Fix the most common tool-use errors with symptom-to-fix diagnostic tables.

Search the web and cite results.

Fetch and read content from specific URLs to augment Claude's context with live web content.

Run Python and bash code in a sandboxed container to analyze data, generate files, and iterate on solutions.

Discover and load tools on demand.