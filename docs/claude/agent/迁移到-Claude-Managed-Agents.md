---
title: 迁移到 Claude Managed Agents
url: https://platform.claude.com/docs/en/managed-agents/migration
source_type: web
folder: claude/agent
author: null
tags:
- Claude Managed Agents
- Agent 迁移
- 事件驱动架构
- Claude Agent SDK
- Messages API
summary: 解析将基于 Messages API 或 Agent SDK 的 Agent 迁移到 Claude Managed Agents 的变化、取舍与迁移步骤。
fetched_at: '2026-09-20T07:05:22.765058+00:00'
---

Move an existing agent built on the Messages API or the Claude Agent SDK to Claude Managed Agents.

Claude Managed Agents replaces your hand-written agent loop with managed infrastructure. This page covers what changes when you migrate from a custom loop built on the [Messages API](https://platform.claude.com/docs/en/build-with-claude/working-with-messages) or from the [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview).

## From a Messages API agent loop

If you built an agent by calling `messages.create` in a `while` loop, running tool calls yourself, and appending results to the conversation history, most of that code goes away.

### What you stop managing

| Before | After |
| --- | --- |
| You maintain the conversation history array and pass it back on every turn. | The session stores history server-side. Send events, receive events. |
| You iterate `tool_use` content blocks, run each tool, and loop back with `tool_result` messages. | Pre-built tools run inside the sandbox automatically. You only handle custom tools through `agent.custom_tool_use` events. |
| You provision your own sandbox for running agent-generated code. | The session sandbox handles code execution, file operations, and bash. |
| You decide when the loop is done. | The session emits `session.status_idle` when the agent has nothing more to do. |

### Code comparison

**Before** (Messages API loop, simplified):

**After** (Claude Managed Agents):

### What you still control

*   **System prompt and model:** Same fields, now on the agent definition.
*   **Custom tools:** Still declared with JSON Schema. Execution moves from inline handling to responding to `agent.custom_tool_use` events. See [Session event stream](https://platform.claude.com/docs/en/managed-agents/events-and-streaming).
*   **Web search and web fetch settings:** Same `allowed_domains`, `blocked_domains`, `max_content_tokens`, and `user_location` fields, now set once on the `web_search` and `web_fetch` entries of the agent toolset's `configs` array instead of on every request. The `max_uses`, `citations`, and `cache_control` fields are not available. See [Restrict web search and web fetch domains](https://platform.claude.com/docs/en/managed-agents/tools#restrict-web-search-and-web-fetch-domains).
*   **Context:** You can still inject context through the system prompt, [file resources](https://platform.claude.com/docs/en/managed-agents/files), or [skills](https://platform.claude.com/docs/en/managed-agents/skills).

## From the Claude Agent SDK

If you built with the [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview), you're already working with agents, tools, and sessions as concepts. The difference is where they run: the SDK runs in a process you operate, while Managed Agents runs in Anthropic's infrastructure. Most of the migration is mapping SDK configuration objects to their API-side equivalents.

### What changes

| Agent SDK | Managed Agents |
| --- | --- |
| `ClaudeAgentOptions(...)` constructed per run | `client.beta.agents.create(...)` once; the Agent is persisted and versioned server-side. See [Agent setup](https://platform.claude.com/docs/en/managed-agents/agent-setup). |
| `async with ClaudeSDKClient(...)` or `query(...)` | `client.beta.sessions.create(...)` then send and receive [events](https://platform.claude.com/docs/en/managed-agents/events-and-streaming). |
| `@tool`-decorated functions dispatched automatically by the SDK | Declare as `{"type": "custom", ...}` on the Agent; your client handles `agent.custom_tool_use` events and replies with `user.custom_tool_result`. See [Tools](https://platform.claude.com/docs/en/managed-agents/tools). |
| Built-in tools run in your process against your filesystem | `{"type": "agent_toolset_20260401"}` runs the same tools inside the session sandbox against `/workspace`. |
| `cwd`, `add_dirs` point at local paths | Upload or mount [files](https://platform.claude.com/docs/en/managed-agents/files) as session resources. |
| `system_prompt` and the `CLAUDE.md` hierarchy | A single `system` string on the Agent. Each update that changes the agent produces a new server-side version; pin sessions to a specific version to promote or roll back without a deploy. See [Agent setup](https://platform.claude.com/docs/en/managed-agents/agent-setup). |
| `mcp_servers` configured and authenticated in one place | Declare servers on the Agent; provide credentials through a [Vault](https://platform.claude.com/docs/en/managed-agents/vaults) on the Session. |
| `permission_mode`, `can_use_tool` | Per-tool [`permission_policy`](https://platform.claude.com/docs/en/managed-agents/permission-policies) (`always_allow`, `always_ask`, or `auto`); send `user.tool_confirmation` events for calls that pause for your approval. |

### Code comparison

**Before** (Agent SDK):

**After** (Managed Agents):

The Agent and Environment are created once and reused across sessions. The tool function still runs in your process; the difference is that you read the `agent.custom_tool_use` event and send the result explicitly instead of the SDK dispatching it for you.

### Features that move to your client

The tradeoff for Anthropic running the agent loop is that a few things the SDK handled automatically become your client's responsibility.

| SDK feature | Managed Agents approach |
| --- | --- |
| Plan mode | Run a planning-only session first, then a second session to run the plan. |
| Output styles, slash commands | Apply in your client before sending `user.message` or after receiving `agent.message`. |
| `PreToolUse` / `PostToolUse` hooks | Your client already sees every `agent.custom_tool_use` event before responding; put the logic there. For built-in tools, use `permission_policy: always_ask` to review every call. [`auto`](https://platform.claude.com/docs/en/managed-agents/permission-policies#let-the-server-evaluate-each-call-with-auto) lets the server evaluate each call instead, but if the server evaluates a call as safe, it runs without reaching your client. |
| `max_turns` | Count turns client-side. |

## Migration checklist

1.   [Create an environment](https://platform.claude.com/docs/en/managed-agents/environments) with the networking and runtimes your agent needs.
2.   Port your system prompt and tool selection to an [agent definition](https://platform.claude.com/docs/en/managed-agents/agent-setup).
3.   Replace your loop with [`sessions.create`](https://platform.claude.com/docs/en/managed-agents/sessions) and [`sessions.events.stream`](https://platform.claude.com/docs/en/managed-agents/events-and-streaming).
4.   For any local files the agent reads, upload them through the [Files API](https://platform.claude.com/docs/en/managed-agents/files) and mount them as `resources`.
5.   For any custom tool handlers, move execution into your event loop as responses to `agent.custom_tool_use` events.
6.   Verify with a test session before pointing production traffic at the new flow.

## Migrating between model versions

When a new Claude model is released, migrating a Claude Managed Agents integration is typically a one-field change: update `model` on your [agent definition](https://platform.claude.com/docs/en/managed-agents/agent-setup) and the change takes effect on the next session you create.

Most model-level behavior changes documented in the [Messages API migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide) do not require action on your side:

*   **Request parameter changes** (`max_tokens` defaults, `thinking` configuration) are handled by the Claude Managed Agents runtime. These fields are not exposed on the agent definition.
*   **Assistant message prefilling** does not exist in the event-based session model, so its removal on newer models is a no-op.
*   **Tool argument JSON escaping** is parsed by the runtime before you receive `agent.custom_tool_use` events. You see structured data, not raw strings.

The behavior descriptions in the Messages API guide (what the model does differently) still apply. The migration steps (how to change your request code) do not.