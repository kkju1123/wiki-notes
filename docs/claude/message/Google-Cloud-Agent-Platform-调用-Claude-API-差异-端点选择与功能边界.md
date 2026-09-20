---
title: Google Cloud Agent Platform 调用 Claude：API 差异、端点选择与功能边界
url: https://platform.claude.com/docs/en/build-with-claude/claude-on-vertex-ai
source_type: web
folder: claude/message
author: null
tags:
- Claude
- Google Cloud
- Agent Platform
- Messages API
- LLM 部署
summary: 介绍在 Google Cloud Agent Platform 上调用 Claude 的请求格式差异、端点类型、SDK 用法及功能支持边界。
fetched_at: '2026-09-20T03:27:15.220473+00:00'
---

[Messages](https://platform.claude.com/docs/en/intro)Claude on cloud platforms

The API for accessing Claude on Google Cloud's Agent Platform is nearly identical to the [Messages API](https://platform.claude.com/docs/en/api/messages/create), with two key differences in request format:

*   On Agent Platform, `model` is not passed in the request body. Instead, it is specified in the Google Cloud endpoint URL.
*   On Agent Platform, `anthropic_version` is passed in the request body (rather than as a header), and must be set to the value `vertex-2023-10-16`.

Agent Platform is also supported by Anthropic's official [client SDKs](https://platform.claude.com/docs/en/cli-sdks-libraries/overview). This guide walks you through making a request to Claude on Agent Platform using one of Anthropic's client SDKs.

Note that this guide assumes you already have a Google Cloud project that is able to use Agent Platform. See [Anthropic Claude models on Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/claude) for more information on the setup required and a full walkthrough.

## Install an SDK for accessing Agent Platform

First, install Anthropic's [client SDK](https://platform.claude.com/docs/en/cli-sdks-libraries/overview) for your language of choice.

## Accessing Agent Platform

### Model availability

Note that Anthropic model availability varies by region. Search for "Claude" in the [Model Garden](https://cloud.google.com/model-garden) or go to [Anthropic Claude models](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/claude) for the latest information.

#### API model IDs

Lifecycle terms (Deprecated, Retired) are defined in [Model deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations). Lifecycle dates on partner-operated platforms are set by the partner and can differ from the Claude API schedule. For the current retirement date of any model on Agent Platform, see [Google Cloud's documentation for Claude models on Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/claude).

| Model | Agent Platform API model ID |
| --- | --- |
| Claude Fable 5.1 |  |
| Claude Fable 5 |  |
| Claude Opus 5 |  |
| Claude Opus 4.8 |  |
| Claude Opus 4.7 |  |
| Claude Opus 4.6 |  |
| Claude Opus 4.5 |  |
| Claude Opus 4.1 ([deprecated](https://platform.claude.com/docs/en/about-claude/model-deprecations)) |  |
| Claude Opus 4 ([deprecated](https://platform.claude.com/docs/en/about-claude/model-deprecations)) |  |
| Claude Sonnet 5 |  |
| Claude Sonnet 4.6 |  |
| Claude Sonnet 4.5 |  |
| Claude Sonnet 4 ([deprecated](https://platform.claude.com/docs/en/about-claude/model-deprecations)) |  |
| Claude Haiku 4.5 |  |
| Claude Haiku 3.5 ([deprecated](https://platform.claude.com/docs/en/about-claude/model-deprecations)) |  |

### Making requests

Before running requests you might need to run `gcloud auth application-default login` to authenticate with Google Cloud.

The following examples show how to generate text from Claude on Agent Platform:

See the [client SDKs](https://platform.claude.com/docs/en/cli-sdks-libraries/overview) and the official [Agent Platform docs](https://cloud.google.com/vertex-ai/docs) for more details.

Claude is also available through [Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock), [Claude Platform on AWS](https://platform.claude.com/docs/en/build-with-claude/claude-platform-on-aws), and [Microsoft Foundry](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry).

## Data retention

Data handling for this offering is governed by Google Cloud. For details, see [Agent Platform and zero data retention](https://cloud.google.com/vertex-ai/generative-ai/docs/data-governance).

## Activity logging

Agent Platform provides a [request-response logging service](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/request-response-logging) that allows you to log the prompts and completions associated with your usage.

Anthropic recommends that you log your activity on at least a 30-day rolling basis to understand your activity and investigate any potential misuse.

## Feature support

For the full feature list with Google Cloud availability, see [Features overview](https://platform.claude.com/docs/en/build-with-claude/overview).

### Supported feature highlights

*   [Messages API](https://platform.claude.com/docs/en/api/messages/create)
*   [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
*   [Thinking](https://platform.claude.com/docs/en/build-with-claude/thinking)
*   [Tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview), including the [Bash tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool), [Browser use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool), [Computer use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool), [Memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool), and [Text editor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool)
*   [Web search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool)
*   [Citations](https://platform.claude.com/docs/en/build-with-claude/citations)
*   [Structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)

### Features not supported

*   Input sources (URL sources for images and documents, Files API)
*   Server-side tools (code execution, web fetch, advisor)
*   Agent infrastructure (Agent Skills, MCP connector, programmatic tool calling)
*   API endpoints (Message Batches, Models, Admin, Compliance, Usage and Cost)
*   Claude Managed Agents
*   Server-side fallback (the [`fallbacks` parameter](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#server-side-fallback); use the [client-side fallback pattern](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#client-side-fallback) instead)

### Context window

Claude Fable 5.1, Claude Fable 5, Claude Opus 5, Claude Opus 4.8, Claude Opus 4.7, Claude Opus 4.6, Claude Sonnet 5, and Claude Sonnet 4.6 have a [1M-token context window](https://platform.claude.com/docs/en/build-with-claude/context-windows) on Agent Platform. Other Claude models, including Sonnet 4.5 and Sonnet 4 (deprecated), have a 200k-token context window.

Agent Platform limits request payloads to 30 MB. When sending large documents or many images, you might reach this limit before the token limit.

## Global, multi-region, and regional endpoints

Agent Platform offers three endpoint types:

*   **Global endpoints:** Dynamic routing for maximum availability
*   **Multi-region endpoints:** Dynamic routing within a geographic area (for example, the United States or the European Union) for data residency with high availability
*   **Regional endpoints:** Guaranteed data routing through specific geographic regions

Regional and multi-region endpoints include a 10% pricing premium over global endpoints.

### When to use each option

**Global endpoints (recommended):**

*   Provide maximum availability and uptime
*   Dynamically route requests to regions with available capacity
*   No pricing premium
*   Best for applications where data residency is flexible
*   Only supports pay-as-you-go traffic (provisioned throughput requires regional endpoints)

**Multi-region endpoints:**

*   Dynamically route requests across regions within a geographic area (currently `us` and `eu`)
*   Useful when you need data residency within a broad geography but want higher availability than a single region
*   10% pricing premium over global endpoints
*   Only supports pay-as-you-go traffic (provisioned throughput requires regional endpoints)

**Regional endpoints:**

*   Route traffic through specific geographic regions
*   Required for single-region data residency, strict compliance mandates, or provisioned throughput
*   Support both pay-as-you-go and provisioned throughput
*   10% pricing premium reflects infrastructure costs for dedicated regional capacity

### Implementation

**Using global endpoints (recommended):**

Set the `region` parameter to `"global"` when initializing the client:

**Using multi-region endpoints:**

Set the `region` parameter to a multi-region identifier: `"us"` for the United States or `"eu"` for the European Union. The SDK routes requests to the corresponding multi-region endpoint (`https://aiplatform.us.rep.googleapis.com` or `https://aiplatform.eu.rep.googleapis.com`), which dynamically balances traffic across regions within that geography.

**Using regional endpoints:**

Specify a specific region such as `"us-east5"` or `"europe-west1"`:

## Additional resources

*   **Agent Platform pricing:**[Generative AI pricing on cloud.google.com](https://cloud.google.com/vertex-ai/generative-ai/pricing)
*   **Claude models documentation:**[Claude on Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/claude)
*   **Google blog post:**[Global endpoint for Claude models](https://cloud.google.com/blog/products/ai-machine-learning/global-endpoint-for-claude-models-generally-available-on-vertex-ai)
*   **Anthropic pricing details:**[Cloud platform pricing](https://platform.claude.com/docs/en/about-claude/pricing#cloud-platform-pricing)