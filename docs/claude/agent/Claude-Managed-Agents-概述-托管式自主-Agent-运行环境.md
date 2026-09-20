---
title: Claude Managed Agents 概述：托管式自主 Agent 运行环境
url: https://platform.claude.com/docs/en/managed-agents/overview
source_type: web
folder: claude/agent
author: null
tags:
- Claude API
- Managed Agents
- Agent 运行时
- 长时任务
- Anthropic
summary: 介绍 Anthropic 的预构建 Agent 托管运行时，如何用 Agent/Environment/Session/Events 四概念支撑长时异步任务。
fetched_at: '2026-09-20T06:56:58.200543+00:00'
---

Managed Agents First steps

Pre-built, configurable agent harness that runs in managed infrastructure. Best for long-running tasks and asynchronous work.

Anthropic offers two ways to build with Claude, each suited to different use cases:

|  | Messages API | Claude Managed Agents |
| --- | --- | --- |
| **What it is** | Direct model prompting access | Pre-built, configurable agent harness that runs in managed infrastructure |
| **Best for** | Custom agent loops and fine-grained control | Long-running tasks and asynchronous work |

Claude Managed Agents provides the harness and infrastructure for running Claude as an autonomous agent. Instead of building your own agent loop, tool execution, and runtime, you get a fully managed environment where Claude can read files, run commands, browse the web, and run code securely. The harness supports built-in prompt caching, compaction, and other performance optimizations for high-quality, efficient agent outputs. To build your own agent loop with direct model access instead, see [Using the Messages API](https://platform.claude.com/docs/en/build-with-claude/working-with-messages).

Create your first agent session

Create a session and send your first event

Event types, rate limits, CLI flags, and other lookup tables

## Core concepts

Claude Managed Agents is built around four concepts:

| Concept | Description |
| --- | --- |
| **Agent** | The model, system prompt, tools, MCP servers, and skills |
| **Environment** | Configuration for where sessions run: an Anthropic-managed cloud sandbox, or a self-hosted sandbox on your own infrastructure |
| **Session** | A running agent instance within an environment, performing a specific task and generating outputs |
| **Events** | Messages exchanged between your application and the agent (user turns, tool results, status updates) |

## How it works

1.   ### Create an agent

Define the model, system prompt, tools, MCP servers, and skills. Create the agent once and reference it by ID across sessions. 
2.   ### Create an environment

Configure where the agent runs: a cloud sandbox, or a [self-hosted sandbox](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes) on your own infrastructure. 
3.   ### Start a session

Launch a session that references your agent and environment configuration. 
4.   ### Send events and stream responses

Send user messages as events. Claude autonomously runs tools and streams back results through server-sent events (SSE). Event history is persisted server-side and can be fetched in full. 
5.   ### Steer or interrupt

Send additional user events to guide the agent mid-execution, or interrupt it to change direction. 

## When to use Claude Managed Agents

Claude Managed Agents is best for workloads that need:

*   **Long-running execution:** Tasks that run for minutes or hours with multiple tool calls
*   **Cloud infrastructure:** Secure sandboxes with pre-installed packages and network access
*   **Self-hosted execution:** Sandboxes on infrastructure you control for compliance or data-residency requirements
*   **Minimal infrastructure:** No need to build your own agent loop, sandbox, or tool execution layer
*   **Stateful sessions:** Persistent filesystems and conversation history across multiple interactions
*   **Scheduled execution:** Recurring agent runs on a cron schedule through [scheduled deployments](https://platform.claude.com/docs/en/managed-agents/scheduled-deployments)

## Supported tools

Claude Managed Agents gives Claude access to a set of built-in tools:

*   **Bash:** Run shell commands in the sandbox
*   **File operations:** Read, write, edit, glob, and grep files in the sandbox
*   **Web search and fetch:** Search the web and retrieve content from URLs, optionally restricted to an allowlist or blocklist of domains
*   **MCP servers:** Connect to external tool providers

See [Tools](https://platform.claude.com/docs/en/managed-agents/tools) for the full list and configuration options.

## Beta access

To get started, you need:

1.   A [Claude API key](https://platform.claude.com/settings/keys)
2.   The `managed-agents-2026-04-01` beta header on all requests
3.   Access to Claude Managed Agents (enabled by default for all API accounts)

Within the beta, [MCP tunnels](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview) and [dreaming](https://platform.claude.com/docs/en/managed-agents/dreams) are in a more limited research preview. [Request access](https://claude.com/form/claude-managed-agents) to enable them.

Claude Managed Agents is stateful by design: sessions are long-running, resume cleanly after pauses, and store conversation history, sandbox state, and outputs server-side. Because of this, Managed Agents is not currently eligible for [Zero Data Retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention#zero-data-retention-zdr-scope) or HIPAA Business Associate Agreement (BAA) coverage. You retain control over this data: you can [delete sessions](https://platform.claude.com/docs/en/managed-agents/session-operations#deleting-a-session), and separately delete any [files](https://platform.claude.com/docs/en/build-with-claude/files#delete-a-file) you uploaded, at any time through the API. For eligibility across all features, see [API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention#feature-eligibility).

See [Rate limits](https://platform.claude.com/docs/en/managed-agents/reference#rate-limits) and [Branding guidelines](https://platform.claude.com/docs/en/managed-agents/reference#branding-guidelines) in the reference.