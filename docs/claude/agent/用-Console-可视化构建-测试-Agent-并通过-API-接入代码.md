---
title: 用 Console 可视化构建、测试 Agent，并通过 API 接入代码
url: https://platform.claude.com/docs/en/managed-agents/onboarding
source_type: web
folder: claude/agent
author: null
tags:
- Claude Console
- Agent
- MCP
- 可视化配置
- LLM 工程
summary: 用 Claude Console 可视化配置模型、提示词、工具、MCP 与 Skills，实时测试后复制 Agent/环境 ID 接入代码，缩短
  Agent 调试周期。
fetched_at: '2026-09-20T07:02:37.044593+00:00'
---

Create, test, and iterate on agents visually in Console, then run them from your code with the API.

[Console](https://platform.claude.com/workspaces/default/agent-quickstart/) provides a visual interface for creating and configuring agents. It lets you iterate on configuration interactively before writing code.

## How to build an agent

The [visual interface](https://platform.claude.com/workspaces/default/agent-quickstart/) walks you through each field of an agent definition:

*   **Model and system prompt:** Pick a model and write the system prompt in a full-width editor.
*   **MCP servers:** Add remote MCP servers by URL and authenticate your agent to take action on your behalf.
*   **Tools:** Extend your agent's capabilities using a pre-built agent toolset and MCP tools.
*   **Skills:** Attach Anthropic or custom skills from your organization's library.

As you configure, Console shows the equivalent API request so you can copy it into your code once you're satisfied.

## Testing an agent

Console includes an inline session runner. After configuring your agent, you can start a test session directly, send messages, and watch the event stream without leaving the page. This is the fastest way to check that your system prompt and tool selection produce the behavior you expect.

## From Console to your codebase

Once your agent works as expected:

1.   Copy the agent ID and [environment ID](https://platform.claude.com/docs/en/managed-agents/environments) from Console.
2.   Reference them in your code when [creating sessions](https://platform.claude.com/docs/en/managed-agents/sessions):

Was this page helpful?