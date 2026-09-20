---
title: Anthropic 工具组合：研究、编码与长时运行智能体的选型模式
url: https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-combinations
source_type: web
folder: claude/message
author: null
tags:
- Anthropic
- tool use
- 智能体
- 工具组合
- Claude
summary: 解析 Claude 智能体工具组合：搜索+执行、编辑+Bash、记忆正交及浏览器/桌面自动化，帮助按任务选型。
fetched_at: '2026-09-20T04:11:14.435964+00:00'
---

Common Anthropic tool pairings for research agents, coding agents, and long-running agents.

Anthropic-provided tools are designed to work together. Common agent patterns pair tools that cover complementary stages of a workflow: one tool gathers or discovers, another processes or acts. The combinations below are starting points, not prescriptions. Mix them to fit your task.

Each snippet shows only the `tools` array. See [Handle tool calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls) for the full request shape.

## Research agent: web_search + code_execution

Search finds sources; code execution analyzes and synthesizes. Claude searches for data, then writes Python to process, tabulate, or visualize it. This pairing is a good fit for questions that require both up-to-date information and nontrivial computation over that information, such as "compare this quarter's earnings across the top five cloud providers."

The flow is typically search, then execute, then optionally search again if the first pass surfaced a gap. Code execution runs server-side, so there's no client-side sandbox to manage.

## Coding agent: text_editor + bash

The text editor reads and modifies files; bash runs tests and build commands. This is the canonical software-development loop: inspect the code, make an edit, run the tests, repeat. Both tools are client-executed, so your application controls which files and commands are accessible.

Pair this with a constrained working directory and a command allowlist if the agent operates on untrusted code. See [Text editor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool) and [Bash tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool) for the execution contracts.

## Cite-then-fetch: web_search + web_fetch

Search surfaces candidate URLs; fetch retrieves full page content for the relevant ones. This avoids fetching everything upfront. Claude runs a search, inspects the snippets, picks the two or three results that actually look relevant, and fetches only those.

This pairing is useful when the answer lives in long-form content (documentation pages, articles, specifications) that a search snippet can't fully capture. Fetch pulls the complete page so Claude can cite specific passages.

## Long-running agent: memory + any other tools

Memory persists state across conversations; the other tools do the work. Add memory to any agent that needs to remember prior sessions, such as a support agent that recalls a customer's earlier issues or a project assistant that tracks decisions made last week.

Add your other tools alongside `memory` in the same array.

Memory is orthogonal to your other tools. It doesn't change how they behave; it gives Claude a place to write down and later retrieve facts that would otherwise be lost when the context window resets. See [Memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) for the storage model.

## All-in-one: computer_use

The computer use tool subsumes most others by operating a full desktop. Claude sees screenshots and issues mouse and keyboard actions, which means it can drive any application a human can. Use this when the task requires arbitrary GUI interaction that more specific tools can't reach: legacy software without an API, visual verification steps, or workflows that span multiple desktop apps.

The toolset entry takes no `name` or display dimensions: coordinates are expressed in the pixel space of the screenshots you return, and you can turn individual actions off through the entry's `configs` field.

Computer use is the most general option and also the slowest, because Claude typically needs a fresh screenshot after each batch of actions. Prefer narrower tools when they cover your use case, and reach for computer use when nothing else fits. If the task stays inside a web browser, use the browser agent pattern in the next section. See [Computer use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool) for the sandbox setup.

## Browser agent: browser_use

When the whole task happens inside webpages (filling forms, reading page content, working across tabs), the browser use tool is a closer fit than computer use. Your application drives a browser it controls and returns screenshots or page state; Claude calls page-aware member tools such as `read_page`, `find`, `form_input`, and `get_page_text` alongside clicks and typing, so it can act on element references in addition to pixel coordinates.

Like the computer use toolset, the entry takes no `name`, and you turn individual member tools off through its `configs` field. See [Browser use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool) for the execution contract.

## Next steps

Full catalog of Anthropic-provided tools with type strings and parameters.

How tool use works and when to use Anthropic tools versus defining your own.