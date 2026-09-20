---
title: 在 Amazon Bedrock 上使用 Claude（Opus 4.7 及之后版本）
url: https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock
source_type: web
folder: claude
author: null
tags:
- Claude
- Amazon Bedrock
- AWS IAM
- Messages API
- 企业安全合规
summary: 介绍如何通过 Amazon Bedrock 以 AWS 原生认证、计费和安全边界访问 Claude 模型，涵盖认证、SDK 调用、支持模型与功能差异。
fetched_at: '2026-09-20T03:20:06.848472+00:00'
---

[Messages](https://platform.claude.com/docs/en/intro)Claude on cloud platforms

Access Claude models through Amazon Bedrock with AWS-native authentication, billing, and security boundaries.

This guide walks you through setting up and making API calls to Claude in Amazon Bedrock. Claude in Amazon Bedrock runs on AWS-managed infrastructure with zero operator access (Anthropic personnel have no access to the inference infrastructure), letting you build sensitive applications entirely inside the AWS security boundary while using the same Messages API shape you use with Anthropic's first-party API.

## Access

Amazon Bedrock sets access criteria for each Claude model individually. Claude Fable 5.1, Claude Fable 5, Claude Opus 4.8, Claude Sonnet 5, Claude Opus 4.7, and Claude Haiku 4.5 are open to all Amazon Bedrock customers. For any other model's current criteria, check [Amazon Bedrock model access](https://console.aws.amazon.com/bedrock/home#/modelaccess) in the AWS console. Claude Mythos Preview requires an invitation through [Project Glasswing](https://anthropic.com/glasswing). For region availability, see [Regions](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock#regions).

## Prerequisites

Before you begin, ensure you have:

*   An AWS account with [Amazon Bedrock model access](https://console.aws.amazon.com/bedrock/home#/modelaccess) enabled for the Claude models you intend to use.
*   The [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured (optional, for credential management).

Claude Mythos Preview additionally requires a dedicated AWS account that has been allowlisted by the Bedrock Marketplace team. Your Anthropic account executive can submit your account ID for allowlisting (typically processed within 24 hours), and AWS sends a welcome email once it's complete.

## Authentication

Claude in Amazon Bedrock supports three authentication paths. Choose the one that best fits your security requirements.

### Bedrock service role (recommended)

Use a Bedrock service role with AWS-managed keys for the most secure, long-lived access:

1.   ### Admin: provision the service role

An AWS administrator provisions a Bedrock service role and grants developers `iam:PassRole` permission on the service role ARN. 
2.   ### Developer: pass the role

When calling the API, Bedrock assumes the service role on your behalf. See the [Amazon Bedrock documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/bedrock-mantle.html) for how to associate the role with your requests. 

### IAM assumed roles

For identity-federated access with a 12-hour maximum session:

1.   ### Admin: configure the IAM role

Create an IAM role scoped to your Claude models. The trust policy names your identity provider (SAML, OIDC, or AWS Identity Center). The permissions policy grants `bedrock-mantle:CreateInference` only on the allowed model ARNs. 
2.   ### Developer: authenticate and assume

Authenticate through your corporate identity provider, then assume the IAM role. AWS STS issues temporary credentials that the SDK or CLI uses to sign requests. 

### Bearer tokens

For short-term access without IAM roles (12-hour maximum, least preferred):

1.   ### Admin: restrict token types

Block long-term keys by attaching a policy that denies `bedrock:CallWithBearerToken` unless the `bedrock:BearerTokenType` condition matches a short-term token. 
2.   ### Developer: mint a token

Use the `aws-bedrock-token-generator` CLI to mint a bearer token. Pass it in the `x-api-key` header on each request. 

## Install an SDK

Anthropic's [client SDKs](https://platform.claude.com/docs/en/cli-sdks-libraries/overview) support Claude in Amazon Bedrock through a Bedrock-specific package or module.

## Making your first request

The endpoint follows the pattern `https://bedrock-mantle.{region}.api.aws/anthropic/v1/messages`. Unlike the `InvokeModel`-based integration, this endpoint uses standard SSE streaming and the same request body shape as Anthropic's first-party API.

The SDK resolves credentials and region using the standard AWS precedence: constructor arguments, then environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AWS_REGION`), then the AWS config file and credential chain (SSO, assumed roles, ECS task role, IMDS).

## Supported models

Model IDs in Claude in Amazon Bedrock carry an `anthropic.` provider prefix. Model capabilities and behaviors are documented on the [Models overview](https://platform.claude.com/docs/en/models/overview) page.

| Model | Model ID | Access |
| --- | --- | --- |
| Claude Fable 5.1 |  | Open |
| Claude Fable 5 |  | Open |
| Claude Opus 5 |  | See [Access](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock#access) |
| Claude Opus 4.8 |  | Open |
| Claude Opus 4.7 |  | Open |
| Claude Sonnet 5 | `anthropic.claude-sonnet-5` | Open |
| Claude Haiku 4.5 |  | Open |
| Claude Mythos Preview |  | Invitation only ([Project Glasswing](https://anthropic.com/glasswing)) |

Use Claude Code 2.1.255 or later with Claude Fable 5.1 on Amazon Bedrock; run `claude update` to upgrade.

## Feature support

For the full feature list with Amazon Bedrock availability, see [Features overview](https://platform.claude.com/docs/en/build-with-claude/overview).

### Supported feature highlights

*   [Messages API](https://platform.claude.com/docs/en/api/messages/create) (`/anthropic/v1/messages`)
*   [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
*   [Thinking](https://platform.claude.com/docs/en/build-with-claude/thinking)
*   [Tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview), including the [Bash tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool), [Computer use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool), [Memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool), and [Text editor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool)
*   [Citations](https://platform.claude.com/docs/en/build-with-claude/citations)

### Features not supported

*   [Structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)
*   Input sources (URL sources for images and documents, Files API)
*   Server-side tools (code execution, web search, web fetch, advisor)
*   Agent infrastructure (Agent Skills, MCP connector, programmatic tool calling)
*   API endpoints (Message Batches, Models, Admin, Compliance, Usage and Cost)
*   Claude Managed Agents
*   Server-side fallback (the [`fallbacks` parameter](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#server-side-fallback); use the [client-side fallback pattern](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#client-side-fallback) instead)
*   [Computer use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool) and [browser use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool) toolsets (`computer_toolset_20260801` and `browser_toolset_20260801` are not currently available on Amazon Bedrock; the beta computer use tool versions remain available)

## Regions

Claude in Amazon Bedrock is available in the following AWS regions. Amazon Bedrock offers two endpoint types:

*   **Global:** dynamic routing across all available regions for maximum availability. No pricing premium.
*   **Regional:** the endpoint resolves to the single AWS region you specify, for data-residency requirements. Regional endpoints carry a 10% pricing premium over global endpoints. To route across multiple regions within a geography, use an [inference profile](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html) (US, EU, JP, or AU). Regions marked **In-region only** in the table support direct single-region routing without an inference profile.

The global endpoint is available for Claude Fable 5.1, Claude Fable 5, Claude Opus 5, Claude Opus 4.8, Claude Opus 4.7, Claude Sonnet 5, and Claude Haiku 4.5. For Claude Fable 5.1, regional endpoints are currently available in `us-east-1` only. Claude Mythos Preview is regional only and is available in `us-east-1`.

| AWS region | Location | Endpoint types |
| --- | --- | --- |
| `af-south-1` | Africa (Cape Town) | Global |
| `ap-northeast-1` | Asia Pacific (Tokyo) | Global, JP, In-region only |
| `ap-northeast-2` | Asia Pacific (Seoul) | Global |
| `ap-northeast-3` | Asia Pacific (Osaka) | Global, JP |
| `ap-south-1` | Asia Pacific (Mumbai) | Global |
| `ap-south-2` | Asia Pacific (Hyderabad) | Global |
| `ap-southeast-1` | Asia Pacific (Singapore) | Global |
| `ap-southeast-2` | Asia Pacific (Sydney) | Global, AU |
| `ap-southeast-3` | Asia Pacific (Jakarta) | Global |
| `ap-southeast-4` | Asia Pacific (Melbourne) | Global, AU, In-region only |
| `ca-central-1` | Canada (Central) | Global, US |
| `ca-west-1` | Canada West (Calgary) | Global |
| `eu-central-1` | Europe (Frankfurt) | Global, EU |
| `eu-central-2` | Europe (Zurich) | Global, EU |
| `eu-north-1` | Europe (Stockholm) | Global, EU, In-region only |
| `eu-south-1` | Europe (Milan) | Global, EU |
| `eu-south-2` | Europe (Spain) | Global, EU |
| `eu-west-1` | Europe (Ireland) | Global, EU, In-region only |
| `eu-west-2` | Europe (London) | Global, EU |
| `eu-west-3` | Europe (Paris) | Global, EU |
| `il-central-1` | Israel (Tel Aviv) | Global |
| `me-central-1` | Middle East (UAE) | Global |
| `sa-east-1` | South America (São Paulo) | Global |
| `us-east-1` | US East (N. Virginia) | Global, US, In-region only |
| `us-east-2` | US East (Ohio) | Global, US, In-region only |
| `us-west-1` | US West (N. California) | Global, US |
| `us-west-2` | US West (Oregon) | Global, US, In-region only |

## Quotas

Default quota is 2 million input tokens per minute (TPM). You can request up to 5 million input TPM and 500,000 output TPM without additional Anthropic approval. AWS enforces requests-per-minute (RPM) limits on the Bedrock side; contact AWS support for RPM adjustments.

## Data retention

Data handling for this offering is governed by Amazon Bedrock. For details, see [Data protection in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html).

## Monitoring and logging

Claude in Amazon Bedrock emits logs to both CloudWatch and CloudTrail. Anthropic recommends retaining activity logs on at least a 30-day rolling basis to understand usage patterns and investigate potential issues.

## Support

For support, contact **[bedrock-ant-eap@amazon.com](mailto:bedrock-ant-eap@amazon.com)**. Include your AWS account ID and the `request-id` from any failed API responses.