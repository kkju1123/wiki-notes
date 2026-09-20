---
title: Claude 在 Microsoft Foundry 中的接入与调用
url: https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry
source_type: web
folder: claude/message
author: null
tags:
- Claude
- Microsoft Foundry
- Azure AI
- LLM 推理
- 企业身份与计费
summary: 介绍通过 Azure Foundry 资源与部署调用 Claude 模型的方法，涵盖托管选项、认证方式和 SDK 调用。
fetched_at: '2026-09-20T03:29:08.345647+00:00'
---

[Messages](https://platform.claude.com/docs/en/intro)Claude on cloud platforms

Access Claude models through Microsoft Foundry with Azure-native endpoints and authentication.

This guide shows you how to set up and make API calls to Claude in Microsoft Foundry using one of Anthropic's client SDKs or direct HTTP requests. When you access Claude in Microsoft Foundry, you are billed for Claude usage in the Azure Marketplace. You can use Claude models including Claude Fable 5.1, Claude Opus 5, Claude Opus 4.8, and Claude Sonnet 5, and features such as the [1M-token context window](https://platform.claude.com/docs/en/build-with-claude/context-windows), while managing costs through your Azure subscription.

Claude is available in Global Standard and US Data Zone Standard deployment types in Foundry resources, billed in Claude Consumption Units through the Azure Marketplace. Visit [Claude in Microsoft Foundry pricing](https://platform.claude.com/docs/en/about-claude/pricing#claude-in-microsoft-foundry-pricing) for details.

## Hosting options

Claude models in Microsoft Foundry are available in two hosting options. You choose the hosting option when you configure the deployment.

|  | Hosted on Azure | Hosted on Anthropic |
| --- | --- | --- |
| Where inference runs | Anthropic-operated service running on Azure infrastructure | Anthropic-operated service running on Anthropic infrastructure |
| Model availability | The latest models in the Opus, Sonnet, and Haiku families | All Claude models available on Microsoft Foundry |
| Deployment types | Global Standard, US Data Zone Standard | Global Standard |
| Recommended for | Most workloads | [Access to features or models not yet hosted on Azure](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry#additional-features-not-supported-when-hosted-on-azure) |

## Prerequisites

Before you begin, ensure you have:

*   An active Azure subscription
*   Access to the [Foundry portal](https://ai.azure.com/)
*   The [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) installed (required for the Entra ID cURL example, optional otherwise)
*   An Azure RBAC role that allows you to use the resource, such as **Foundry User** (formerly Azure AI User) or **Cognitive Services User**

## Install an SDK

Anthropic's [client SDKs](https://platform.claude.com/docs/en/cli-sdks-libraries/overview) support Foundry through a platform-specific package or client class. The examples on this page also show requests with cURL and the ant CLI. To set up the CLI, see [CLI quickstart](https://platform.claude.com/docs/en/cli-sdks-libraries/cli/quickstart).

## Provisioning

Foundry uses a two-level hierarchy: **resources** contain your security and billing configuration, while **deployments** are the model instances you call through the API. You'll first create a Foundry resource, then create one or more Claude deployments within it.

### Provisioning Foundry resources

Create a Foundry resource, which is required to use and manage services in Azure. You can follow these instructions to create a [Foundry resource](https://learn.microsoft.com/en-us/azure/ai-services/multi-service-resource?pivots=azportal#create-a-new-azure-ai-foundry-resource). Alternatively, you can start by creating a [Foundry project](https://learn.microsoft.com/en-us/azure/foundry/how-to/create-projects), which involves creating a Foundry resource.

To provision your resource:

1.   Navigate to the [Foundry portal](https://ai.azure.com/).
2.   Create a new Foundry resource or select an existing one.
3.   Configure access management using Azure-issued API keys or Entra ID (formerly Azure Active Directory) for role-based access control.
4.   Optionally configure the resource to be part of a private network (Azure Virtual Network) to restrict network access to your resource.
5.   Note your resource name. You'll use this as `{resource}` in API endpoints (for example, `https://{resource}.services.ai.azure.com/anthropic/v1/*`).

### Creating Foundry deployments

After creating your resource, deploy a Claude model to make it available for API calls. These steps describe the new Foundry portal (the **New Foundry** toggle is on):

1.   Sign in to the Foundry portal. From the portal homepage, select **Discover** in the upper-right navigation, then **Models** in the left pane to open the model catalog.
2.   Search for and select a Claude model (for example, ). Each model appears once in the catalog regardless of how many hosting options it supports.
3.   On the model card, select **Deploy**, then **Custom settings** to open the deployment settings pane. If you choose **Default settings** instead, the deployment is automatically configured as Hosted on Azure for models available in both hosting options.
4.   On your first Claude deployment, review the Azure Marketplace terms, select an industry, and select **Agree and Proceed** to accept the terms and subscribe to the Azure Marketplace offer.
5.   Configure the deployment:
    *   **Deployment name:** Defaults to the model ID, but you can customize it (for example, `my-claude-deployment`). The deployment name cannot be changed after creation.
    *   **Region scope:** Select Global, or for models hosted on Azure, Data Zone. Selecting Data Zone creates a US Data Zone Standard deployment, which keeps inference within the United States and is equivalent to setting [`inference_geo: "us"`](https://platform.claude.com/docs/en/manage-claude/data-residency#inference-geo) on the Claude API.
    *   **Model version:** Expand **Model version settings** and select a version from the **Model version** dropdown menu. Each [hosting option](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry#hosting-options) is listed as a separate model version, labeled with its hosting option (for example, version 1 for Hosted on Anthropic, version 2 for Hosted on Azure).

6.   Select **Deploy** and wait for provisioning to complete.
7.   Once deployed, select **Build** in the upper-right navigation, then **Models** in the left pane, and open your deployment. The **Details** tab shows the **Target URI** (your endpoint URL) and **Key** (your API key).

If the **New Foundry** toggle is off, you are in the classic portal layout. There, open **Model catalog** in the left pane to find and deploy a model, and open **Models + endpoints** (under **My assets**) to view your deployments and their endpoint details.

## Authentication

Claude in Microsoft Foundry supports two authentication methods: API keys and Entra ID tokens. Both methods use Azure-hosted endpoints in the format `https://{resource}.services.ai.azure.com/anthropic/v1/*`.

### API key authentication

After provisioning your Foundry Claude resource, you can obtain an API key from the Foundry portal:

1.   In the Foundry portal, select **Build** in the upper-right navigation, then **Models** in the left pane.
2.   Open your Claude deployment and select the **Details** tab.
3.   Copy the **Key** value (and note the **Target URI** for your endpoint).
4.   Use either the `api-key` or `x-api-key` header in your requests, or provide it to the SDK.

The Foundry SDKs require an API key and either a resource name or base URL. The C#, Java, PHP, Python, and TypeScript SDKs automatically read these from the following environment variables if they are defined:

*   `ANTHROPIC_FOUNDRY_API_KEY` - Your API key
*   `ANTHROPIC_FOUNDRY_RESOURCE` - Your resource name (for example, `example-resource`)
*   `ANTHROPIC_FOUNDRY_BASE_URL` - Alternative to resource name: the full base URL (for example, `https://example-resource.services.ai.azure.com/anthropic/`). The C# SDK does not read this variable: it always constructs the base URL from the resource name.

**Example using API key:**

### Microsoft Entra authentication

Entra ID authentication lets you manage access with Azure RBAC, integrate with your organization's identity management, and avoid handling API keys manually. To use Entra ID tokens:

1.   Enable [Microsoft Entra ID authentication](https://learn.microsoft.com/en-us/azure/ai-foundry/model-inference/how-to/configure-entra-id) for your Foundry resource.
2.   Obtain an access token from Entra ID.
3.   Use the token in the `Authorization: Bearer {TOKEN}` header.

**Example using Entra ID:**

## Correlation request IDs

Foundry includes request identifiers in HTTP response headers for debugging and tracing. When contacting support, provide both the `request-id` and `apim-request-id` (Azure API Management) values to help teams quickly locate and investigate your request across both Anthropic and Azure systems.

## Feature support

Claude in Microsoft Foundry supports most Claude features. You can find all the features currently supported in [Features overview](https://platform.claude.com/docs/en/build-with-claude/overview).

### Context window

Claude Fable 5.1, Claude Fable 5, Claude Opus 5, Claude Opus 4.8, Claude Opus 4.7, Claude Opus 4.6, Claude Sonnet 5, and Claude Sonnet 4.6 have a [1M-token context window](https://platform.claude.com/docs/en/build-with-claude/context-windows) on Microsoft Foundry. Other Claude models, including Claude Sonnet 4.5, have a 200k-token context window.

### Claude features not supported for Claude in Microsoft Foundry

*   Admin API
*   Advisor tool
*   Claude Managed Agents
*   Compliance API
*   Models API
*   Message Batches API
*   Server-side fallback (the [`fallbacks` parameter](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#server-side-fallback); use the [client-side fallback pattern](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#client-side-fallback) instead)
*   [Computer use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool) and [browser use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/browser-use-tool) toolsets (`computer_toolset_20260801` and `browser_toolset_20260801` are not currently available on Microsoft Foundry; the beta computer use tool versions remain available)

### Additional features not supported when hosted on Azure

The following features are available for deployments hosted on Anthropic but are not supported for deployments hosted on Azure:

*   [Code execution](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool)
*   [Web search](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool) and [web fetch](https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-fetch-tool) tool versions later than `web_search_20250305` and `web_fetch_20250910`. Deployments hosted on Azure support only these basic versions, so dynamic filtering, response inclusion, and cache bypass are not available.
*   [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
*   [Programmatic tool calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling)
*   [Files API](https://platform.claude.com/docs/en/build-with-claude/files)

Requests that use these features against a deployment hosted on Azure return a `400 Bad Request` error by design. Claude Code detects deployments hosted on Azure and automatically adapts its feature set.

## API responses

API responses from Claude in Microsoft Foundry follow the standard [Claude API response format](https://platform.claude.com/docs/en/api/messages/create). This includes the `usage` object in response bodies, which provides detailed token consumption information for your requests. The `usage` object is consistent across all platforms (Claude API, Amazon Bedrock, Claude Platform on AWS, Foundry, and Google Cloud).

For details on response headers specific to Foundry, see [Correlation request IDs](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry#correlation-request-ids).

## API model IDs and deployments

Lifecycle terms (Deprecated, Retired) are defined in [Model deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations). Microsoft Foundry follows the Claude API lifecycle schedule.

The following Claude models are available through Foundry:

| Model | Default deployment name | Hosted on Azure | Hosted on Anthropic |
| --- | --- | --- | --- |
| Claude Fable 5.1 |  |  | ✓ |
| Claude Fable 5 |  |  | ✓ |
| Claude Opus 5 |  | ✓ | ✓ |
| Claude Opus 4.8 |  | ✓ | ✓ |
| Claude Opus 4.7 |  |  | ✓ |
| Claude Opus 4.6 |  |  | ✓ |
| Claude Opus 4.5 |  |  | ✓ |
| Claude Sonnet 5 |  | ✓ | ✓ |
| Claude Sonnet 4.6 |  |  | ✓ |
| Claude Sonnet 4.5 |  |  | ✓ |
| Claude Haiku 4.5 |  | ✓ | ✓ |

By default, deployment names match the model IDs shown in the preceding table. However, you can create custom deployments with different names in the Foundry portal to manage different configurations, versions, or rate limits. Use the deployment name (not necessarily the model ID) in your API requests.

## Billing

Claude in Microsoft Foundry bills through the [Azure Marketplace](https://azuremarketplace.microsoft.com/). Usage is denominated in Claude Consumption Units (CCUs), metered hourly, and invoiced monthly in arrears on your Azure bill. CCUs are not prepaid credits. There is no CCU balance or commitment.

For the CCU price, conversion mechanics, and per-model token rates, see [Claude in Microsoft Foundry pricing](https://platform.claude.com/docs/en/about-claude/pricing#claude-in-microsoft-foundry-pricing).

## Migrating between hosting options

To move an existing deployment from one hosting option to the other:

1.   Create a new deployment of the model's other hosting version (Hosted on Azure or Hosted on Anthropic). This can be in the same Foundry resource, or a new one.
2.   Update your application to pass the new deployment name in the `model` parameter.
3.   Delete the old deployment once traffic has moved.

If the new deployment is in the same Foundry resource, your endpoint URL and authentication are unchanged. If you created a new resource, update your application's endpoint and credentials to point to it.

## Monitoring and logging

Azure provides monitoring and logging for your Claude usage through standard Azure patterns:

*   **Azure Monitor:** Track API usage, latency, and error rates
*   **Azure Log Analytics:** Query and analyze request/response logs
*   **Cost Management:** Monitor and forecast costs associated with Claude usage

Anthropic recommends logging your activity on at least a 30-day rolling basis to understand usage patterns and investigate any potential issues.

## Troubleshooting

### Authentication errors

**Error:**`401 Unauthorized` or `Invalid API key`

*   **Solution:** Verify your API key is correct. You can find it in the Foundry portal on your deployment's **Details** tab (under **Build**>**Models**).
*   **Solution:** If using Microsoft Entra ID, ensure your access token is valid and hasn't expired. Tokens typically expire after 1 hour.

**Error:**`403 Forbidden`

*   **Solution:** Your Azure account may lack the necessary permissions. Ensure you have the appropriate Azure RBAC role assigned (for example, **Foundry User** (formerly Azure AI User) or **Cognitive Services User**).

### Rate limiting

**Error:**`429 Too Many Requests`

*   **Solution:** You've exceeded your rate limit. Implement exponential backoff and retry logic in your application.
*   **Solution:** Consider requesting rate limit increases through the Azure portal or Azure support.

Foundry does not include Anthropic's standard rate limit headers (`anthropic-ratelimit-tokens-limit`, `anthropic-ratelimit-tokens-remaining`, `anthropic-ratelimit-tokens-reset`, `anthropic-ratelimit-input-tokens-limit`, `anthropic-ratelimit-input-tokens-remaining`, `anthropic-ratelimit-input-tokens-reset`, `anthropic-ratelimit-output-tokens-limit`, `anthropic-ratelimit-output-tokens-remaining`, and `anthropic-ratelimit-output-tokens-reset`) in responses. Manage rate limiting through Azure's monitoring tools instead.

### Model and deployment errors

**Error:**`Model not found` or `Deployment not found`

*   **Solution:** Verify you're using the correct deployment name. If you haven't created a custom deployment, use the default model ID (for example, ).
*   **Solution:** Ensure the model/deployment is available in your Azure region.

**Error:**`Invalid model parameter`

*   **Solution:** The model parameter should contain your deployment name, which can be customized in the Foundry portal. Verify the deployment exists and is properly configured.

## Next steps

Explore Claude's advanced features and capabilities.

Learn about Anthropic's pricing structure for models and features.

As safer and more capable models launch, Anthropic regularly retires older ones. See all API deprecations, along with recommended replacements.

## Additional resources

Browse Anthropic models in the Foundry catalog.

View Microsoft's pricing details for Azure AI Foundry.

View Anthropic's per-model pricing details.

Manage your Azure resources.