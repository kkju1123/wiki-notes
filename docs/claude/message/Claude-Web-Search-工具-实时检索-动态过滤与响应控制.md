---
title: Claude Web Search 工具：实时检索、动态过滤与响应控制
url: https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool
source_type: web
folder: claude/message
author: null
tags:
- Claude
- Web Search Tool
- Dynamic Filtering
- Tool Use
- Agentic Workflows
summary: Claude API 的 web search 工具提供带引用的实时检索，通过动态过滤、域名控制与本地化减少无关上下文并降低成本。
fetched_at: '2026-09-20T03:47:37.227829+00:00'
---

Give Claude access to current web content with cited sources, optional dynamic filtering, and domain controls.

The web search tool gives Claude direct access to real-time web content, allowing it to answer questions with up-to-date information beyond its knowledge cutoff. The response includes citations for sources drawn from search results.

With `web_search_20260209` and later versions, Claude can write and run code that filters the search results before they reach the context window (**dynamic filtering**), keeping only relevant information. Dynamic filtering is available with Claude 4.6 and later models and [Claude Mythos Preview](https://anthropic.com/glasswing).

Three versions of the web search tool are available:

*   `web_search_20250305`: basic web search
*   `web_search_20260209`: adds [dynamic filtering](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool#dynamic-filtering)
*   `web_search_20260318`: adds [response inclusion](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool#response-inclusion) control for agentic workflows

The examples on this page use `web_search_20250305` for basic search and `web_search_20260318` for dynamic filtering.

For web search's Zero Data Retention eligibility and the related `allowed_callers` configuration, see [Server tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools#zdr-and-allowed-callers).

For model support, see the [Tool reference](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference).

## How web search works

When you add the web search tool to your API request:

1.   Claude determines when to search based on the prompt.
2.   The API runs the searches and provides Claude with the results. This process can repeat multiple times throughout a single request.
3.   At the end of its turn, Claude provides a final response with cited sources.

### When Claude searches

Claude searches when the request depends on information that is current, changing, or outside its training data:

*   Recent events, news, or announcements
*   Current prices, rates, scores, or statistics
*   Information about specific organizations, people, or products that might have changed
*   Explicit requests to search or look something up

Claude answers directly without searching when the request draws on stable knowledge:

*   Established facts, math, science fundamentals, or coding concepts
*   Creative writing or brainstorming
*   Analysis of content already provided in the conversation
*   Conversational turns and greetings

Triggering is steerable through your system prompt: you can encourage Claude to search more readily or to prefer answering directly. For a hard constraint, use `max_uses` to cap the number of searches for each request.

### Dynamic filtering

With basic web search, every search result is loaded into Claude's context window, and much of that content can be irrelevant to the request. With `web_search_20260209` or later, Claude instead writes and runs code that filters the results first, so only relevant content reaches the context window. This reduces token use on search-heavy requests.

Dynamic filtering runs web search from inside [code execution](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool): on `web_search_20260209` and later, the tool's `allowed_callers` field defaults to `["code_execution_20260120"]`, and when dynamic filtering runs, the API provisions the code execution it needs for the request automatically. You don't need to add the code execution tool to `tools` yourself. There are no additional charges for code execution calls made this way beyond the standard token costs.

To call web search directly, without dynamic filtering, set `allowed_callers: ["direct"]`. Models that don't support programmatic tool calling require this setting. Without it, the API returns a 400 error that tells you to set it.

The following examples use `web_search_20260318`:

## How to use web search

These organization-level settings in the Claude Console apply to Messages API requests only. [Claude Managed Agents](https://platform.claude.com/docs/en/managed-agents/overview) sessions use only the per-tool `allowed_domains` and `blocked_domains` lists on the agent toolset; see [Restrict web search and web fetch domains](https://platform.claude.com/docs/en/managed-agents/tools#restrict-web-search-and-web-fetch-domains).

Provide the web search tool in your API request:

## Tool definition

The web search tool supports the following parameters:

All web search tool versions accept `allowed_callers`, which controls whether Claude calls web search directly or from code execution through [dynamic filtering](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool#dynamic-filtering). On `web_search_20260209` and later it defaults to `["code_execution_20260120"]` instead of `["direct"]`. See [Server tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools#zdr-and-allowed-callers) for how to configure it. `web_search_20260318` and later also accept [`response_inclusion`](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool#response-inclusion).

### Max uses

The `max_uses` parameter limits the number of searches performed. If Claude attempts more searches than allowed, the `web_search_tool_result` is an error with the `max_uses_exceeded` error code.

Simple factual queries typically use 1–3 searches; comparative or multientity research can use 10 or more. For guidance on choosing a value, see [Server tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools).

### Domain filtering

Provide `allowed_domains` or `blocked_domains`, not both. If a request includes both, the API returns a 400 error. Entries are bare domains with an optional path, for example `example.com` or `example.com/blog`, without a scheme.

For the full domain filtering rules, see [Domain filtering](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools#domain-filtering) in the Server tools guide.

On [Claude Managed Agents](https://platform.claude.com/docs/en/managed-agents/overview), set these fields on the `web_search` entry of the agent toolset; see [Restrict web search and web fetch domains](https://platform.claude.com/docs/en/managed-agents/tools#restrict-web-search-and-web-fetch-domains).

### Localization

The `user_location` parameter allows you to localize search results based on a user's location. Provide at least one of `city`, `region`, `country`, or `timezone`.

*   `type`: The type of location (must be `approximate`)
*   `city`: The city name
*   `region`: The region or state
*   `country`: The two-letter [ISO 3166-1 alpha-2](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2) country code. The API rejects unsupported country codes with a 400 error.
*   `timezone`: The [IANA timezone ID](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

On Claude Managed Agents, the `web_search` entry of the agent toolset accepts a `user_location` object with the same fields. The API rejects an unsupported `country` code with a 400 error when you create or update the agent, or when you create or update a session that supplies the setting. See [Restrict web search and web fetch domains](https://platform.claude.com/docs/en/managed-agents/tools#restrict-web-search-and-web-fetch-domains).

### Response inclusion

The `response_inclusion` parameter controls how search result blocks appear in the API response when the result was consumed by a completed [code execution](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool) call in the same turn. Set `"response_inclusion": "excluded"` to drop those nested `server_tool_use` and result block pairs entirely from the response, reducing output token costs for agentic workflows that don't need to echo raw search content back to the client. The default is `"full"`. Results from direct calls, or from code execution calls that paused before completing, are always returned in full so they can be sent back on the next turn.

## Response

Here's an example response structure:

This example shows a direct search. When a search runs through [dynamic filtering](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool#dynamic-filtering), the response also contains the [code execution tool's](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool) result blocks, and each nested `server_tool_use` and `web_search_tool_result` pair carries a `caller` field identifying the code execution call that made it.

### Search results

Search results include:

*   `url`: The URL of the source page
*   `title`: The title of the source page
*   `page_age`: When the site was last updated
*   `encrypted_content`: Encrypted content that you must pass back in multi-turn conversations

To continue a conversation that contains search results, send the assistant's content blocks back exactly as you received them, including each result's `encrypted_content`. The API decrypts that content on later turns to restore the search results in Claude's context. If `encrypted_content` is missing or modified, the request fails with a 400 validation error.

### Citations

Citations are always enabled for web search, and each `web_search_result_location` includes:

*   `url`: The URL of the cited source
*   `title`: The title of the cited source
*   `encrypted_index`: A reference that must be passed back for multi-turn conversations
*   `cited_text`: Up to 150 characters of the cited content

The web search citation fields `cited_text`, `title`, and `url` do not count toward input or output token usage.

### Errors

When the web search tool encounters an error (such as hitting rate limits), the Claude API still returns a 200 (success) response. The error is represented within the response body using the following structure:

On an error, `content` is a single error object rather than a list of result blocks. A search that succeeds but matches no results returns an empty `content` list, not an error.

These are the possible error codes:

*   `too_many_requests`: Rate limit exceeded
*   `invalid_tool_input`: Invalid search query parameter
*   `max_uses_exceeded`: Maximum web search tool uses exceeded
*   `query_too_long`: Query exceeds maximum length
*   `request_too_large`: The search request is too large, typically because of a long domain filter list
*   `unavailable`: An internal error occurred

### `pause_turn` stop reason

The API can pause a long-running search turn and return `stop_reason: "pause_turn"`. To continue, send the paused assistant message back unchanged in a new request.

If Claude calls web search and one of your client tools in the same group of parallel tool calls, the API returns `stop_reason: "tool_use"` instead and does not run the search yet. To continue, return the client tool results, and the API runs the search in the next request. See [Mixing server tools and client tools in one turn](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools#mixing-server-tools-and-client-tools-in-one-turn).

For the server-side loop and `pause_turn` handling, see [The server-side loop and pause_turn](https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools#the-server-side-loop-and-pause-turn) in the Server tools guide.

## Prompt caching

To cache tool definitions across turns, see [Tool use with prompt caching](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-use-with-prompt-caching).

## Streaming

With streaming enabled, you'll receive search events as part of the stream. There will be a pause while the search runs:

## Batch requests

You can include the web search tool in the [Messages Batches API](https://platform.claude.com/docs/en/build-with-claude/batch-processing). Web search tool calls through the Messages Batches API are priced the same as those in regular Messages API requests.

To protect shared capacity, the Batches API throttles web search requests per organization, so large batches with many searches might take longer to complete. You can see your organization's web search rate limit on the [Rate limits](https://platform.claude.com/settings/limits) page in the Claude Console. To request a higher limit, contact sales from that page.

## Usage and pricing

Web search usage is charged in addition to token usage:

Web search is available on the Claude API for **$10 per 1,000 searches**, plus standard token costs for search-generated content. Web search results retrieved throughout a conversation are counted as input tokens, in search iterations executed during a single turn and in subsequent conversation turns.

Each web search counts as one use, regardless of the number of results returned. If an error occurs during web search, the web search will not be billed.

## Next steps

Fetch and read content from specific URLs to augment Claude's context with live web content.

Work with Anthropic-executed tools: server_tool_use blocks, pause_turn continuation, and domain filtering.

Directory of Anthropic-provided tools and reference for optional tool definition properties.