---
title: Amazon Bedrock 上的 Claude 集成（Opus 4.6 及更早版本）
url: https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock-legacy
source_type: web
folder: claude
author: null
tags:
- AWS Bedrock
- Claude
- 模型集成
- IAM认证
- inference profiles
summary: 介绍如何通过 AWS Bedrock 调用 Claude 模型，涵盖模型订阅、推理配置文件、SDK 调用及功能差异。
fetched_at: '2026-09-20T03:22:50.121386+00:00'
---

[Messages](https://platform.claude.com/docs/en/intro)Claude on cloud platforms

The legacy Amazon Bedrock integration for Claude models, using InvokeModel and Converse APIs with ARN-versioned model identifiers.

Calling Claude through Bedrock slightly differs from how you would call Claude on the Claude API directly. This guide walks you through completing an API call to Claude on Bedrock using one of Anthropic's [client SDKs](https://platform.claude.com/docs/en/cli-sdks-libraries/overview).

Note that this guide assumes you have already signed up for an [AWS account](https://portal.aws.amazon.com/billing/signup) and configured programmatic access.

## Install and configure the AWS CLI

1.   [Install a version of the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html) at or newer than version `2.13.23`.
2.   Configure your AWS credentials using the AWS configure command (see [Configure the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)) or find your credentials by navigating to "Command line or programmatic access" within your AWS dashboard and following the directions in the modal window.
3.   Verify that your credentials are working:

## Install an SDK for accessing Bedrock

Anthropic's [client SDKs](https://platform.claude.com/docs/en/cli-sdks-libraries/overview) support Bedrock. You can also use an AWS SDK like `boto3` directly.

## Accessing Bedrock

### Subscribe to Anthropic models

Go to the [AWS Console > Bedrock > Model Access](https://console.aws.amazon.com/bedrock/home?region=us-west-2#/modelaccess) and request access to Anthropic models. Note that Anthropic model availability varies by region. See [AWS documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html) for latest information.

#### API model IDs

Lifecycle terms (Deprecated, Retired) are defined in [Model deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations). Lifecycle dates on partner-operated platforms are set by the partner and can differ from the Claude API schedule. For the current retirement date of any model on Amazon Bedrock, see [Amazon Bedrock's model lifecycle page](https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle.html).

AWS offers newer Claude models through [cross-region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html) rather than on-demand throughput. For these models, a request that passes the base model ID fails with an HTTP 400 error like the following:

To invoke these models, pass an inference profile instead of the base model ID. The inference profile ID is the base model ID with a prefix from a column marked "Yes" in the following table, for example . You can also pass the full inference profile ARN, in the form `arn:aws:bedrock:{region}:{account-id}:inference-profile/{inference-profile-id}`. For AWS's authoritative list of available inference profiles, see [Supported Regions and models for inference profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-support.html). To learn how the prefixes affect routing and pricing, see the [Global versus regional endpoints](https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock-legacy#global-vs-regional-endpoints) section.

| Model | Base Bedrock model ID | `global` | `us` | `eu` | `jp` | `apac` |
| --- | --- | --- | --- | --- | --- | --- |
| Claude Opus 4.6 |  | Yes | Yes | Yes | Yes | Yes |
| Claude Sonnet 4.6 |  | Yes | Yes | Yes | Yes | No |
| Claude Sonnet 4.5 |  | Yes | Yes | Yes | Yes | No |
| Claude Sonnet 4 Deprecated. |  | Yes | Yes | Yes | No | Yes |
| Claude Sonnet 3.7 Retired. |  | No | No | No | No | No |
| Claude Opus 4.5 |  | Yes | Yes | Yes | No | No |
| Claude Opus 4.1 Deprecated. |  | No | Yes | No | No | No |
| Claude Opus 4 Retired. |  | No | No | No | No | No |
| Claude Haiku 4.5 |  | Yes | Yes | Yes | No | No |
| Claude Haiku 3.5 Deprecated. |  | No | Yes | No | No | No |

### List available models

The following examples show how to print a list of all the Claude models available through Bedrock:

### Making requests

The following examples show how to generate text from Claude on Bedrock:

See the [client SDKs](https://platform.claude.com/docs/en/cli-sdks-libraries/overview) for more details, and the [official Bedrock documentation](https://docs.aws.amazon.com/bedrock/).

### Bearer token authentication

You can authenticate with Bedrock using bearer tokens instead of AWS credentials. This is useful in corporate environments where teams need access to Bedrock without managing AWS credentials, IAM roles, or account-level permissions.

The simplest approach is to set the `AWS_BEARER_TOKEN_BEDROCK` environment variable, which each SDK detects automatically when resolving credentials from the environment.

To provide a token programmatically:

## Activity logging

Bedrock provides an [invocation logging service](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html) that allows you to log the prompts and completions associated with your usage.

Anthropic recommends that you log your activity on at least a 30-day rolling basis to understand your activity and investigate any potential misuse.

## Feature support

For the full feature list with Amazon Bedrock availability, see [Features overview](https://platform.claude.com/docs/en/build-with-claude/overview).

### Supported feature highlights

*   [Messages API](https://platform.claude.com/docs/en/api/messages/create)
*   [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
*   [Thinking](https://platform.claude.com/docs/en/build-with-claude/thinking)
*   [Tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview), including the [Bash tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool), [Computer use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool), [Memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool), and [Text editor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool)
*   [Citations](https://platform.claude.com/docs/en/build-with-claude/citations)
*   [Structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)

### Features not supported

*   Input sources (URL sources for images and documents, Files API)
*   Server-side tools (code execution, web search, web fetch, advisor)
*   Agent infrastructure (Agent Skills, MCP connector, programmatic tool calling)
*   API endpoints (Message Batches, Models, Admin, Compliance, Usage and Cost)
*   Claude Managed Agents
*   Server-side fallback (the [`fallbacks` parameter](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#server-side-fallback); use the [client-side fallback pattern](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#client-side-fallback) instead)
*   Automatic prompt caching (the [top-level `cache_control` field](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#automatic-caching); use [explicit cache breakpoints](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#explicit-cache-breakpoints) instead)
*   [Computer use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool) and [browser use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool) toolsets (`computer_toolset_20260801` and `browser_toolset_20260801` are not currently available on Amazon Bedrock; the beta computer use tool versions remain available)

### PDF support on Bedrock

PDF support is available on Bedrock through both the Converse API and InvokeModel API. For detailed information about PDF processing capabilities and limitations, see [Amazon Bedrock PDF support](https://platform.claude.com/docs/en/build-with-claude/pdf-support#amazon-bedrock-pdf-support).

**Important considerations for Converse API users:**

*   Visual PDF analysis (charts, images, layouts) requires citations to be enabled
*   Without citations, only basic text extraction is available
*   For full control without forced citations, use the InvokeModel API

### Mid-conversation system messages on Bedrock

[Mid-conversation system messages](https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages) are available through the InvokeModel API for Claude Fable 5.1, Claude Fable 5, Claude Opus 5, and Claude Opus 4.8. As described in the note under [API model IDs](https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock-legacy#api-model-ids), these requests are served by the same infrastructure as the [Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock) endpoint. No beta header is required. This feature is not available on Claude Sonnet 5. Use the top-level `system` field instead. It is not available for the ARN-versioned models in the model table on this page.

**For Converse API users:** the Converse API accepts system instructions through its top-level [`system` parameter](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_Converse.html). To add system instructions mid-conversation, use the InvokeModel API.

### Context window

Claude Fable 5.1, Claude Fable 5, Claude Opus 5, Claude Opus 4.8, Claude Opus 4.7, Claude Opus 4.6, Claude Sonnet 5, and Claude Sonnet 4.6 have a [1M-token context window](https://platform.claude.com/docs/en/build-with-claude/context-windows) on Amazon Bedrock. Other Claude models, including Sonnet 4.5 and Sonnet 4 (deprecated), have a 200k-token context window.

Bedrock limits request payloads to 20 MB. When sending large documents or many images, you may reach this limit before the token limit.

## Global versus regional endpoints

Starting with **Claude Sonnet 4.5 and all future models**, Bedrock offers two endpoint types:

*   **Global endpoints:** Dynamic routing for maximum availability
*   **Regional endpoints:** Guaranteed data routing through specific geographic regions

Regional endpoints include a 10% pricing premium over global endpoints.

### When to use each option

**Global endpoints (recommended):**

*   Provide maximum availability and uptime
*   Dynamically route requests to regions with available capacity
*   No pricing premium
*   Best for applications where data residency is flexible

**Regional endpoints (CRIS):**

*   Route traffic through specific geographic regions
*   Required for data residency and compliance requirements
*   Available for US, EU, Japan, and Asia-Pacific
*   10% pricing premium reflects infrastructure costs for dedicated regional capacity

### Implementation

**Using global endpoints (default for Opus 4.6, Sonnet 4.6, and Sonnet 4.5):**

The model IDs for Claude Opus 4.6, Sonnet 4.6, and Sonnet 4.5 already include the `global.` prefix:

**Using regional endpoints (CRIS):**

To use regional endpoints, replace the `global.` prefix with a regional prefix such as `us.`:

## Additional resources

*   **Bedrock pricing:**[Amazon Bedrock pricing page](https://aws.amazon.com/bedrock/pricing/)
*   **AWS pricing documentation:**[Bedrock pricing guide](https://docs.aws.amazon.com/bedrock/latest/userguide/bedrock-pricing.html)
*   **AWS blog post:**[Introducing Claude Sonnet 4.5 in Amazon Bedrock](https://aws.amazon.com/blogs/aws/introducing-claude-sonnet-4-5-in-amazon-bedrock-anthropics-most-intelligent-model-best-for-coding-and-complex-agents/)
*   **Anthropic pricing details:**[Cloud platform pricing](https://platform.claude.com/docs/en/about-claude/pricing#cloud-platform-pricing)