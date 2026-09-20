---
title: Claude Managed Agents 入门：创建并运行你的第一个自主智能体
url: https://platform.claude.com/docs/en/managed-agents/quickstart
source_type: web
folder: claude/agent
author: null
tags:
- Claude Managed Agents
- AI Agent
- 会话管理
- 沙箱
- 流式事件
summary: 介绍如何用 Claude Managed Agents 创建 Agent、Environment 与 Session，并通过流式事件驱动沙箱中的工具调用与任务执行。
fetched_at: '2026-09-20T06:59:55.789008+00:00'
---

Create your first autonomous agent.

This guide walks you through creating an agent, setting up an environment, starting a session, and streaming agent responses.

## Core concepts

| Concept | Description |
| --- | --- |
| **Agent** | The model, system prompt, tools, MCP servers, and skills |
| **Environment** | Configuration for where sessions run: an Anthropic-managed cloud sandbox, or a self-hosted sandbox on your own infrastructure |
| **Session** | A running agent instance within an environment, performing a specific task and generating outputs |
| **Events** | Messages exchanged between your application and the agent (user turns, tool results, status updates) |

## Prerequisites

*   A [Claude Console account](https://platform.claude.com/)
*   An [API key](https://platform.claude.com/settings/keys)

## Install the CLI

Check the installation:

## Install the SDK

Set your API key as an environment variable:

## Create your first session

1.   ### Create an agent

Create an agent that defines the model, system prompt, and available tools.

The `agent_toolset_20260401` tool type enables the full set of pre-built agent tools (bash, file operations, web search, and more). See [Tools](https://platform.claude.com/docs/en/managed-agents/tools) for the complete list and per-tool configuration options.

Save the returned `agent.id` (the CLI's [`ant apply`](https://platform.claude.com/docs/en/cli-sdks-libraries/cli/apply) prints it and records it in `claude-lock.json`). You'll reference it in every session you create. 
2.   ### Create an environment

An environment defines the sandbox where your agent runs.

Save the returned `environment.id` (also in `claude-lock.json` if you used `ant apply`). You'll reference it in every session you create. 
3.   ### Start a session

Create a session that references your agent and environment. 
4.   ### Send a message and stream the response

Open a stream, send a user event, then process events as they arrive:

The agent writes a Python script, runs it in the sandbox, and verifies the output file was created. Your output looks similar to this: 

## What's happening

When you send a user event, Claude Managed Agents:

1.   **Provisions a sandbox:** Your environment configuration determines how it's built.
2.   **Runs the agent loop:** Claude determines which tools to use based on your message.
3.   **Runs tools:** File writes, bash commands, and other tool calls run inside the sandbox.
4.   **Streams events:** You receive real-time updates as the agent works.
5.   **Goes idle:** The agent emits a `session.status_idle` event when it has nothing more to do.

## Build a complete app

Each of these quickstarts pairs Claude Managed Agents with a popular chat framework to make a complete, runnable application. In each one, the framework renders the chat surface while a managed session runs the agent loop server-side: the session holds the transcript, runs tools in a sandbox, and streams events that the front end renders.

A research analyst in a browser chat built with Vercel's Chat SDK. Each conversation is one persistent session that streams its reply while a live feed shows the tool calls. Swapping the Chat SDK adapter moves the same handler to Slack, Teams, Discord, or WhatsApp.

A spreadsheet analyst in a chat built from assistant-ui primitives. Sessions are the thread list, one reducer turns the session event log into messages and tool cards, and each bash command renders an inline Allow/Deny gate before it runs.

A personal finance assistant in a CopilotKit chat. The AG-UI adapter for Claude Managed Agents maps each chat thread to a managed session and streams replies token by token, and custom tools render interactive charts inline in the conversation.

## Next steps

Create reusable, versioned agent configurations

Customize networking and sandbox settings

Enable specific tools for your agent

Handle events and steer the agent mid-execution

Run your agent on a recurring cron schedule

Distill a document corpus once into a knowledge wiki, then answer repeated questions from it at a fraction of the cost

Was this page helpful?