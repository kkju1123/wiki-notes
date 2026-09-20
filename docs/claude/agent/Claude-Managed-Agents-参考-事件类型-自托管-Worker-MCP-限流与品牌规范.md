---
title: Claude Managed Agents 参考：事件类型、自托管 Worker、MCP、限流与品牌规范
url: https://platform.claude.com/docs/en/managed-agents/reference
source_type: web
folder: claude/agent
author: null
tags:
- Claude Managed Agents
- 事件流
- 自托管 Worker
- MCP
- 速率限制
summary: 汇总 Claude 托管代理的事件类型、自托管 Worker 参数、MCP 服务器支持、速率限制与品牌规范，便于集成查阅。
fetched_at: '2026-09-20T07:51:23.374896+00:00'
---

Event types, self-hosted worker CLI flags, supported MCP server types, rate limits, and branding guidelines for Claude Managed Agents.

This page collects reference material for Claude Managed Agents. For task-oriented guides, follow the links in each section. For the operations on the session resource, see [Session operations](https://platform.claude.com/docs/en/managed-agents/session-operations).

## Event types

Persisted event type strings follow a `{domain}.{action}` naming convention; the stream-only event deltas (see the Event deltas tab) are the exception. See [Session event stream](https://platform.claude.com/docs/en/managed-agents/events-and-streaming) for sending, streaming, and listing events. Webhook event types are listed separately in [Subscribe to webhooks](https://platform.claude.com/docs/en/managed-agents/webhooks#supported-event-types), and some of their names differ from the stream's (for example, `session.status_idled` rather than `session.status_idle`).

| Type | Description |
| --- | --- |
| `user.message` | A user message with text, image, or document content. |
| `user.interrupt` | Stop the agent mid-execution. |
| `user.custom_tool_result` | Response to a custom tool call from the agent. |
| `user.tool_confirmation` | Approve or deny an agent or MCP tool call when a permission policy requires confirmation. |
| `user.define_outcome` | Define an [outcome](https://platform.claude.com/docs/en/managed-agents/define-outcomes) for the agent to work toward. |
| `user.tool_result` | For sessions with `self_hosted`[environments](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes) only, your integration is responsible for providing `agent_toolset` results. The SDK helpers and CLI do this automatically. |

## Self-hosted worker

These are the `ant beta:worker` CLI flags for the pre-built worker that drives a `self_hosted` environment. See [Self-hosted sandboxes](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes) for setting up the environment, running a worker, and the SDK helper options.

| Flag | Description |
| --- | --- |
| `--environment-id` | The environment to poll for work. Also reads from `ANTHROPIC_ENVIRONMENT_ID`. |
| `--environment-key` | Authenticates the worker with this environment. Also reads from `ANTHROPIC_ENVIRONMENT_KEY`. |
| `--workdir` | Directory where skills are downloaded and tools read and write files. Defaults to `.` (the current directory); the system default working directory is `/workspace`. |
| `--on-work` | Script to call for each claimed work item instead of running tools in-process. Receives session details as environment variables. |
| `--unrestricted-paths` | Allow the file tools to read and write paths outside `--workdir`. The workdir check is a guardrail for the file tools only, not a sandbox; it does not constrain bash. |
| `--max-idle` | How long to wait after the session goes idle with an `end_turn`[stop reason](https://platform.claude.com/docs/en/api/handling-stop-reasons) before shutting down. Defaults to `60s`. |
| `--log-format` | Log output format. Use `json` for structured log ingestion. Defaults to `text`. |

The CLI worker does not mount [memory stores](https://platform.claude.com/docs/en/managed-agents/memory): a session that attaches one still runs, but the agent finds nothing at the store's `mount_path` and no changes sync back to the store. To use memory stores in sessions on a self-hosted environment, run the SDK worker instead; see [Use memory stores](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes#use-memory-stores).

## Supported MCP server types

Claude Managed Agents connects to [remote MCP servers](https://platform.claude.com/docs/en/agents-and-tools/remote-mcp-servers) that expose an HTTP endpoint, or to private MCP servers through [MCP tunnels](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview). The server should support the MCP protocol's streamable HTTP transport; servers that only support the deprecated SSE transport still work through an automatic fallback. See [MCP connector](https://platform.claude.com/docs/en/managed-agents/mcp-connector) for declaring servers on an agent.

For more information on MCP and building MCP servers, see the [MCP documentation](https://modelcontextprotocol.io/).

## Rate limits

Managed Agents endpoints are rate-limited per organization:

| Operation | Limit |
| --- | --- |
| Create endpoints (such as agents, sessions, and environments) | 300 requests per minute |
| Read endpoints (such as retrieve, list, and stream) | 1,200 requests per minute |

Organization-level [spend limits and usage-tier rate limits](https://platform.claude.com/docs/en/api/rate-limits) also apply.

## Branding guidelines

For partners integrating Claude Managed Agents, use of Claude branding is optional. When referencing Claude in your product:

**Allowed:**

*   "Claude Agent" (preferred for dropdown menus)
*   "Claude" (when within a menu already labeled "Agents")
*   "{YourAgentName} Powered by Claude" (if you have an existing agent name)

**Not permitted:**

*   "Claude Code" or "Claude Code Agent"
*   "Claude Cowork" or "Claude Cowork Agent"
*   Claude Code-branded ASCII art or visual elements that mimic Claude Code

Your product should maintain its own branding and not appear to be Claude Code, Claude Cowork, or any other Anthropic product. For questions about branding compliance, contact the Anthropic [sales team](https://www.anthropic.com/contact-sales).