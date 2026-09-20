---
title: MCP Connector：通过 Messages API 直接集成远程 MCP 工具服务器
url: https://platform.claude.com/docs/en/agents-and-tools/mcp-connector
source_type: web
folder: claude/message
author: null
tags:
- MCP
- Claude API
- 工具调用
- 模型上下文协议
- 集成
summary: Messages API 可直接连接远程 MCP 服务器，无需自建客户端，并支持工具白名单、黑名单及细粒度配置。
fetched_at: '2026-09-20T06:23:49.418747+00:00'
---

Connect to remote MCP servers directly from the Messages API without an MCP client, and allowlist, denylist, or configure individual tools.

Claude's Model Context Protocol (MCP) connector feature enables you to connect to remote MCP servers directly from the Messages API without a separate MCP client.

## Key features

*   **Direct API integration:** Connect to MCP servers without implementing an MCP client
*   **Tool calling support:** Access MCP tools through the Messages API
*   **Flexible tool configuration:** Enable all tools, allowlist specific tools, or denylist unwanted tools
*   **Per-tool configuration:** Configure individual tools with custom settings
*   **OAuth authentication:** Support for OAuth Bearer tokens for authenticated servers
*   **Multiple servers:** Connect to multiple MCP servers in a single request

## When Claude uses MCP tools

Once an MCP server is connected, Claude calls its tools when the user's request maps to a tool's described capability, either explicitly ("search Jira for open bugs") or implicitly ("what's blocking the release?" with a Jira server attached).

Claude does **not** call an MCP tool for general knowledge questions about a connected service. Asking "how do Notion databases work?" with a Notion server attached is answered directly; asking "what's in my Projects database?" triggers the tool.

You can steer how readily Claude calls MCP tools through your system prompt. See [When Claude uses tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview#when-claude-uses-tools) for general guidance and example phrasings.

## Limitations

*   Of the feature set of the [MCP specification](https://modelcontextprotocol.io/introduction#explore-mcp), only [tool calls](https://modelcontextprotocol.io/docs/concepts/tools) are currently supported.
*   The server must be publicly exposed through HTTP (supports both Streamable HTTP and SSE transports). Local STDIO servers cannot be connected directly.

## Using the MCP connector in the Messages API

The MCP connector uses two components:

1.   **MCP server definition** (`mcp_servers` array): Defines server connection details (URL, authentication)
2.   **MCP toolset** (`tools` array): Configures which tools to enable and how to configure them

### Basic example

This example enables all tools from an MCP server with default configuration:

## MCP server configuration

Each MCP server in the `mcp_servers` array defines the connection details:

### Field descriptions

| Property | Type | Required | Description |
| --- | --- | --- | --- |
| `type` | string | Yes | Currently only "url" is supported. |
| `url` | string | Yes | The URL of the MCP server. Must start with https://. |
| `name` | string | Yes | A unique identifier for this MCP server. Must be referenced by exactly one MCPToolset in the `tools` array. |
| `authorization_token` | string | No | OAuth authorization token if required by the MCP server. See [Authentication](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector#authentication) for how to obtain one, or the [MCP specification](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization) for protocol details. |

## MCP toolset configuration

The MCPToolset lives in the `tools` array and configures which tools from the MCP server are enabled and how they should be configured.

### Basic structure

### Field descriptions

| Property | Type | Required | Description |
| --- | --- | --- | --- |
| `type` | string | Yes | Must be "mcp_toolset". |
| `mcp_server_name` | string | Yes | Must match a server name defined in the `mcp_servers` array. |
| `default_config` | object | No | Default configuration applied to all tools in this set. Individual tool configs in `configs` override these defaults. |
| `configs` | object | No | Per-tool configuration overrides. Keys are tool names, values are configuration objects. |
| `cache_control` | object | No | [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) cache breakpoint configuration for this toolset. |

### Tool configuration options

Each tool (whether configured in `default_config` or in `configs`) supports the following fields:

| Property | Type | Default | Description |
| --- | --- | --- | --- |
| `enabled` | boolean | `true` | Whether this tool is enabled. |
| `defer_loading` | boolean | `false` | If true, tool description is not sent to the model initially. Used with [Tool search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool). |

For the full directory of Anthropic-provided tools and optional properties such as `defer_loading`, see the [Tool reference](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-reference). To search across large tool sets, see [Tool search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool).

### Configuration merging

Configuration values merge with this precedence (highest to lowest):

1.   Tool-specific settings in `configs`
2.   Set-level `default_config`
3.   System defaults

Example:

Results in:

*   `search_events`: `enabled: false` (from configs), `defer_loading: true` (from default_config)
*   All other tools: `enabled: true` (system default), `defer_loading: true` (from default_config)

## Common configuration patterns

### Enable all tools with default configuration

The simplest pattern: enable all tools from a server:

### Allowlist: enable only specific tools

Set `enabled: false` as the default, then explicitly enable specific tools:

### Denylist: disable specific tools

Enable all tools by default, then explicitly disable unwanted tools. Denylisting write or destructive tools is recommended when building read-only assistants, or when you want a human confirmation step before state changes:

### Mixed: allowlist with per-tool configuration

Combine allowlisting with custom configuration for each tool:

In this example:

*   `search_events` is enabled with `defer_loading: false`
*   `list_events` is enabled with `defer_loading: true` (inherited from default_config)
*   All other tools are disabled

## Validation rules

The API enforces these validation rules:

*   **Server must exist:** The `mcp_server_name` in an MCPToolset must match a server defined in the `mcp_servers` array
*   **Server must be used:** Every MCP server defined in `mcp_servers` must be referenced by exactly one MCPToolset
*   **Unique toolset per server:** Each MCP server can only be referenced by one MCPToolset
*   **Unknown tool names:** If a tool name in `configs` doesn't exist on the MCP server, a backend warning is logged but no error is returned (MCP servers may have dynamic tool availability)

## Response content types

When Claude uses MCP tools, the response includes two new content block types:

### MCP tool use block

### MCP tool result block

## Multiple MCP servers

You can connect to multiple MCP servers by including multiple server definitions in `mcp_servers` and a corresponding MCPToolset for each in the `tools` array:

With many tools available, Claude selects based on tool names and descriptions. Clear, specific tool descriptions improve selection accuracy. For large tool sets (dozens of tools across several servers), consider enabling [`defer_loading`](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector#tool-configuration-options) with the [Tool search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool) so only relevant tools are surfaced per query.

## Authentication

For MCP servers that require OAuth authentication, you'll need to obtain an access token. The MCP connector beta supports passing an `authorization_token` parameter in the MCP server definition. API consumers are expected to handle the OAuth flow and obtain the access token prior to making the API call, and to refresh the token as needed.

### Obtaining an access token for testing

The MCP inspector can guide you through the process of obtaining an access token for testing purposes.

1.   Run the inspector with the following command. You need Node.js installed on your machine.

2.   In the sidebar on the left, for **Transport type**, select either **SSE** or **Streamable HTTP**.

3.   Enter the URL of the MCP server.

4.   In the right area, click **Open Auth Settings** after **Need to configure authentication?**.

5.   Click **Quick OAuth Flow** and authorize on the OAuth screen.

6.   Follow the steps in the **OAuth Flow Progress** section of the inspector and click **Continue** until you reach **Authentication complete**.

7.   Copy the `access_token` value.

8.   Paste it into the `authorization_token` field in your MCP server configuration.

### Using the access token

Once you've obtained an access token using either of the preceding OAuth flows, you can use it in your MCP server configuration:

For detailed explanations of the OAuth flow, refer to the [Authorization section](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization) in the MCP specification.

## Client-side MCP helpers

If you manage your own MCP client connection (for example, with local stdio servers, MCP prompts, or MCP resources), the SDKs provide helper functions that convert between MCP types and Claude API types. This eliminates manual conversion code when using an MCP SDK for your language (for example, the [TypeScript MCP SDK](https://github.com/modelcontextprotocol/typescript-sdk)) alongside the Anthropic SDK.

### Installation

Install both the Anthropic SDK and the MCP SDK:

### Available helpers

Import the helpers for your language:

Helper names and exact signatures follow each language's conventions; this table shows the TypeScript forms:

| Helper | Description |
| --- | --- |
| `mcpTools(tools, mcpClient)` | Converts MCP tools to Claude API tools for use with `client.beta.messages.toolRunner()` |
| `mcpMessages(messages)` | Converts MCP prompt messages to Claude API message format |
| `mcpResourceToContent(resource)` | Converts an MCP resource to a Claude API content block |
| `mcpResourceToFile(resource)` | Converts an MCP resource to a file object for upload |

### Use MCP tools

Convert MCP tools for use with the SDK's [tool runner](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner), which handles tool execution automatically:

### Use MCP prompts

Convert MCP prompt messages into Claude API message format:

### Use MCP resources

Convert MCP resources into content blocks to include in messages, or into file objects for upload:

### Error handling

The conversion functions throw `UnsupportedMCPValueError` if an MCP value isn't supported by the Claude API (in Go, the helpers return an `UnsupportedValueError`; in Java and C#, they throw `AnthropicInvalidDataException`). This can happen with unsupported content types, MIME types, or resource links (resolve resource links with your MCP client before converting).

## Batch requests

You can include `mcp_servers` in [Message Batches API](https://platform.claude.com/docs/en/build-with-claude/batch-processing) requests. MCP tool calls through the Batches API are priced the same as those in regular Messages API requests.

## Data retention

The MCP connector is not covered by ZDR arrangements. Data exchanged with MCP servers, including tool definitions and execution results, is retained according to Anthropic's standard data retention policy.

For ZDR eligibility across all features, see [API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention).

## Migration guide

If you're using the deprecated `mcp-client-2025-04-04` beta header, follow this guide to migrate to the new version.

### Key changes

1.   **New beta header:** Change from `mcp-client-2025-04-04` to `mcp-client-2025-11-20`
2.   **Tool configuration moved:** Tool configuration now lives in the `tools` array as MCPToolset objects, not in the MCP server definition
3.   **More flexible configuration:** New pattern supports allowlisting, denylisting, and per-tool configuration

### Migration steps

**Before (deprecated):**

**After (current):**

### Common migration patterns

| Old pattern | New pattern |
| --- | --- |
| No `tool_configuration` (all tools enabled) | MCPToolset with no `default_config` or `configs` |
| `tool_configuration.enabled: false` | MCPToolset with `default_config.enabled: false` |
| `tool_configuration.allowed_tools: [...]` | MCPToolset with `default_config.enabled: false` and specific tools enabled in `configs` |

## Deprecated version: mcp-client-2025-04-04

The previous version of the MCP connector included tool configuration directly in the MCP server definition:

### Deprecated field descriptions

| Property | Type | Description |
| --- | --- | --- |
| `tool_configuration` | object | **Deprecated:** Use MCPToolset in the `tools` array instead |
| `tool_configuration.enabled` | boolean | **Deprecated:** Use `default_config.enabled` in MCPToolset |
| `tool_configuration.allowed_tools` | array | **Deprecated:** Use allowlist pattern with `configs` in MCPToolset |

## Compatibility

| Supported platforms | * Claude API Beta * Claude Platform on AWS Beta * Microsoft Foundry Beta |
| --- |