---
title: Claude API Agent Skills 入门：用容器技能生成 PPT/Excel/Word/PDF
url: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart
source_type: web
folder: claude/message
author: null
tags:
- Agent Skills
- Claude API
- 文档生成
- 渐进披露
- Code Execution
summary: 在 Claude API 中通过 container.skills 声明 Agent Skills，结合代码执行生成 PPT、Excel、Word、PDF
  等文档，再用 Files API 下载结果。
fetched_at: '2026-09-20T06:16:46.152697+00:00'
---

Learn how to use Agent Skills to create documents with the Claude API in under 10 minutes.

This tutorial shows you how to use Agent Skills to create a PowerPoint presentation. You'll learn how to enable Skills, make a request, and access the generated file.

## Prerequisites

*   A [Claude API key](https://platform.claude.com/settings/keys) or a logged-in [ant CLI](https://platform.claude.com/docs/en/cli-sdks-libraries/cli/authentication)
*   A [client SDK](https://platform.claude.com/docs/en/cli-sdks-libraries/overview) for your language, or `curl` and `jq`
*   Basic familiarity with making API requests

## Agent Skills overview

Pre-built Agent Skills extend Claude's capabilities with specialized expertise for tasks such as creating documents, analyzing data, and processing files. Anthropic provides the following pre-built Agent Skills in the API:

*   **PowerPoint (pptx):** Create and edit presentations
*   **Excel (xlsx):** Create and analyze spreadsheets
*   **Word (docx):** Create and edit documents
*   **PDF (pdf):** Generate PDF documents

## Step 1: List available Skills

First, check what Skills are available. Use the Skills API to list all Anthropic-managed Skills. Each language tab is an excerpt from one continuous script, with any imports and client setup at the top:

You see the following Skills: `pptx`, `xlsx`, `docx`, and `pdf`.

This API returns each Skill's metadata: its name and description. Claude loads this metadata at startup to determine which Skills are available. This is the first level of **progressive disclosure**, where Claude discovers Skills without loading their full instructions yet.

## Step 2: Create a presentation

Use the PowerPoint Skill to create a presentation about renewable energy. Specify Skills using the `container` parameter in the Messages API:

The request includes the following parts:

*   **`model`:** A [model that supports the code execution tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool#compatibility)
*   **`container.skills`:** Specifies which Skills Claude can use
*   **`type: "anthropic"`:** Indicates this is an Anthropic-managed Skill
*   **`skill_id: "pptx"`:** The PowerPoint Skill identifier
*   **`version: "latest"`:** The Skill version set to the most recently published
*   **`tools`:** Enables code execution (required for Skills)

When you make this request, Claude automatically matches your task to the relevant Skill. Because you asked for a presentation, Claude determines the PowerPoint Skill is relevant and loads its full instructions: the second level of progressive disclosure. Then Claude runs the Skill's code to create your presentation.

## Step 3: Download the created file

The presentation was created in the code execution container and saved as a file. The Step 2 `response` includes a file reference with a file ID. Extract the file ID and download the file with the Files API. The example saves it to your system temp directory:

## Try more examples

Try these variations:

### Create a spreadsheet

### Create a Word document

### Generate a PDF

## Next steps

Learn how to write effective Skills that Claude can discover and use successfully.

Learn how to use Agent Skills to extend Claude's capabilities through the API.

Upload your own Skills for specialized tasks.

Learn about Skills in Claude Code.

Explore example Skills and implementation patterns.