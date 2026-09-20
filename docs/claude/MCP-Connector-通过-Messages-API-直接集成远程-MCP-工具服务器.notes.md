# MCP Connector：通过 Messages API 直接集成远程 MCP 工具服务器

*原文: [https://platform.claude.com/docs/en/agents-and-tools/mcp-connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector) · 来源: web · 生成时间: 2026-09-20T06:23:49.418747+00:00*

## 背景

MCP（模型上下文协议）是连接大模型与外部工具/数据源的开放标准，但传统上需要开发者自行实现或运行 MCP 客户端来处理协议握手、工具发现、调用执行等。对于仅想在 API 调用中利用远程工具的场景，这会引入额外组件和复杂度。MCP connector 将这部分能力集成到 Messages API 服务端，让开发者以声明式方式接入远程 MCP 服务器。

## 痛点

没有 MCP connector 时，开发者必须维护独立的 MCP 客户端进程，处理连接生命周期、OAuth 令牌刷新、工具状态同步，容易出错且增加运维负担。客户端与 API 分离还可能导致延迟和故障点，尤其在多服务器、多工具场景下配置管理繁琐。

## 解决办法

MCP connector 在 Messages API 请求中新增 mcp_servers 数组来声明远程服务器（URL、名称、认证 token），并在 tools 数组中用 mcp_toolset 配置工具启用策略。API 服务端负责连接服务器、发现工具、将工具定义注入模型上下文并执行工具调用，返回工具结果块。配置采用分层合并：工具级 configs 覆盖 default_config，再覆盖系统默认（enabled=true, defer_loading=false），因此可以轻松实现 allowlist（默认禁用，按需启用）或 denylist（默认启用，按需禁用）。认证上支持 OAuth Bearer token，由调用方在请求前获取并刷新。这样将 MCP 客户端逻辑下沉到服务端，开发者只需关注声明式配置。

## 关键代码示例

```json
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 1024,
  "mcp_servers": [
    {
      "type": "url",
      "url": "https://mcp.example.com/jira",
      "name": "jira",
      "authorization_token": "your_oauth_token"
    }
  ],
  "tools": [
    {
      "type": "mcp_toolset",
      "mcp_server_name": "jira",
      "default_config": {
        "enabled": true,
        "defer_loading": false
      },
      "configs": {
        "delete_issue": { "enabled": false }
      }
    }
  ],
  "messages": [
    { "role": "user", "content": "Find open bugs in the project" }
  ]
}
```

这段 JSON 是一个 Messages API 请求体，声明了一个名为 jira 的远程 MCP 服务器（通过 HTTPS URL 和 OAuth token 连接），并在 tools 中定义一个 mcp_toolset 引用该服务器。default_config 启用所有工具，但 configs 中单独禁用了 delete_issue，体现 denylist 模式。API 服务端会连接服务器、获取工具列表，并让 Claude 根据用户问题自动调用合适的工具。

## 关键流程

1. 准备一个公开可访问的远程 MCP 服务器，确保支持 HTTP/SSE 传输和工具调用。
2. 在 Messages API 请求中添加 mcp_servers 数组，定义服务器 URL、唯一名称和可选 OAuth token。
3. 在 tools 数组中添加 type 为 "mcp_toolset" 的工具集，通过 mcp_server_name 关联服务器。
4. 使用 default_config 和 configs 细化每个工具的启用状态与参数，例如禁用危险操作或延迟加载。
5. 调用 Messages API，Claude 根据用户请求映射到工具能力时自动执行工具调用并返回结果。

## 关键点

- MCP connector 将 MCP 客户端逻辑下沉到 Messages API 服务端，开发者无需自建客户端即可接入远程 MCP 工具。
- 通过 mcp_servers 定义服务器连接，通过 mcp_toolset 配置工具启用策略，两者通过名称关联。
- 配置合并优先级为：工具级 configs > default_config > 系统默认，这使 allowlist 和 denylist 的实现非常直观。
- 目前仅支持 MCP 规范中的工具调用，不支持资源、提示等特性；服务器必须通过 HTTPS 公开，支持 Streamable HTTP 和 SSE。
- Claude 只在用户请求显式或隐式映射到工具描述的能力时才会调用 MCP 工具，不会为一般知识问题触发工具。
- 支持同时连接多个 MCP 服务器，但每个服务器必须被恰好一个 MCPToolset 引用，且工具名在 configs 中不存在时只记录警告不报错。

## 对比与权衡

- 相比自行实现 MCP 客户端，MCP connector 在集成复杂度、部署运维上更优，无需额外进程；但在功能覆盖上不如完整客户端，仅支持工具调用，不支持资源、提示等 MCP 特性，且只支持 HTTP 传输，不支持本地 STDIO。
- 相比直接使用 Anthropic 原生工具（如 web_search、code_execution），MCP connector 更灵活，可连接任意第三方工具服务器；但需要自己管理服务器端和认证，且可能受第三方服务稳定性和安全性的影响。
- 相比在请求中手动定义 tools 的 Function Calling 方式，MCP connector 的优势是工具 schema 由 MCP 服务器动态提供，无需每次请求重复列出；缺点是增加了对远程服务器的依赖，且需要处理 MCP 协议细节和认证。

## 自测问题

**问: MCP connector 与直接使用 MCP 客户端有什么区别？**

MCP connector 将 MCP 客户端逻辑集成到 Messages API 服务端，开发者只需声明服务器和工具配置，由 API 负责连接和工具调用；传统 MCP 客户端则需要开发者自己实现或运行客户端，维护连接和协议。这样简化了集成，但也限制了控制力和支持的传输类型（仅 HTTP）。

**问: 如何实现工具的 allowlist 和 denylist？配置合并规则是什么？**

通过 default_config 设置 enabled: false 作为默认，然后在 configs 中针对具体工具设置 enabled: true 实现 allowlist；或默认 enabled: true 然后在 configs 中禁用特定工具实现 denylist。合并优先级从高到低：configs 中的工具级设置、default_config、系统默认（enabled: true, defer_loading: false）。

**问: MCP connector 目前有哪些限制？**

仅支持 MCP 规范中的工具调用，不支持资源、提示等；服务器必须公开通过 HTTPS，支持 Streamable HTTP 和 SSE，不支持本地 STDIO；认证方面只支持 OAuth Bearer token，由调用方负责获取和刷新。

**问: Claude 什么时候会调用 MCP 工具？如何引导模型不随意调用？**

当用户请求显式或隐式映射到工具描述的能力时才会调用，例如“搜索 Jira 的未解决 bug”或“什么阻塞了发布？”（附带 Jira 服务器）。对于一般知识问题不会调用。可以通过 system prompt 引导调用频率和条件，参考工具使用指南。

**问: 多个 MCP 服务器如何配置？有什么注意事项？**

在 mcp_servers 数组中定义多个服务器，每个名称唯一，并在 tools 数组中为每个服务器添加一个 MCPToolset，且每个服务器必须被恰好一个工具集引用。当工具数量很多时，建议使用 defer_loading 配合 Tool search tool 以减少上下文占用并提高选择准确性。

## 适用场景

- 需要快速接入第三方工具服务（如 Jira、Notion、Slack）而无需开发 MCP 客户端的场景。
- 构建只读助手或需要严格工具权限控制的场景，通过 allowlist/denylist 管理工具暴露。
- 在多个远程 MCP 服务器间进行工具编排，由 Claude 自动选择合适工具。
- 在现有基于 Messages API 的应用中增量添加外部工具，避免引入额外基础设施。

## 标签

`MCP` `Claude API` `工具调用` `模型上下文协议` `集成`
