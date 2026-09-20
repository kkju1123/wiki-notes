# 远程 MCP 服务器：通过 Claude API Connector 接入第三方工具

*原文: [https://platform.claude.com/docs/en/agents-and-tools/remote-mcp-servers](https://platform.claude.com/docs/en/agents-and-tools/remote-mcp-servers) · 来源: web · 生成时间: 2026-09-20T06:22:23.772759+00:00*

## 背景

MCP（Model Context Protocol）出现是为了解决 LLM 应用与海量外部工具/数据源之间的集成爆炸问题。传统方式下每个模型、每个应用都要单独对接每个 SaaS，产生 M×N 集成成本。MCP 提供一个统一开放协议，把工具封装成可复用服务器；远程 MCP 服务器由第三方托管，开发者只需配置 URL 和凭证即可接入 Claude API。

## 痛点

若没有远程 MCP 或不懂这套机制，接入每个 SaaS 工具都需要自己写 REST 调用、维护 schema、处理鉴权和重试，工具增多后代码和密钥管理会失控。团队无法复用工具，Claude 的能力也被限制在模型内置知识中。

## 解决办法

远程 MCP 服务器本质是一个暴露在公网/内网上、实现 MCP 协议的工具网关。它至少提供 initialize、list_tools、call_tool 三类能力：客户端先握手，再发现可用工具及其 JSON schema，最后按 schema 调用工具并返回结果。Claude API 的 MCP Connector 让开发者把这些远程服务器作为 mcp_servers 参数随请求传入，由 Claude 端自动完成发现、注入工具定义、执行 tool_use 和回填结果，免去手写 agent 工具循环。可以类比成 USB-C 扩展坞：Claude 是笔记本，远程 MCP 服务器是各种外设，Connector 负责协商协议和传输。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "List my open issues in Linear"}],
    mcp_servers=[
        {
            "type": "http",
            "url": "https://mcp.linear.app/mcp",
            "headers": {"Authorization": "Bearer <LINEAR_TOKEN>"}
        }
    ],
)

for block in response.content:
    if block.type == "tool_use":
        print(block.name, block.input)

```

这段代码演示通过 Claude API 的 MCP Connector 声明一个远程 MCP 服务器。mcp_servers 中传入 type、url 和鉴权 headers，Claude 会自动连接 Linear 服务器并发现其工具。用户消息要求列出未关闭 issues，模型生成 tool_use 后，Connector 执行远程 Linear MCP 工具并把结果回填，最终生成可读回答。开发者只需替换 <LINEAR_TOKEN>，无需手写 Linear REST 调用或工具循环。

## 关键流程

1. 选择受信任的远程 MCP 服务器，从服务方获取 MCP URL 和鉴权 token
2. 在 Claude API 请求中通过 mcp_servers 参数声明服务器：type 设为 http，填写 url 和 headers
3. Claude Connector 连接远程服务器并自动执行 list_tools 发现可用工具
4. 当模型判断需要外部数据或操作时返回 tool_use，Connector 执行远程 call_tool 并把 tool_result 回填
5. 生产环境中将 token 放入密钥管理系统，定期轮换并限制最小权限

## 关键点

- MCP 把模型与工具的 M×N 集成问题变成 N+M，每个工具只需实现一次 MCP server 即可被多个客户端复用，这是生态降本的核心。
- 远程 MCP 服务器由第三方托管，通过 URL 和凭证连接，开发者不必自己部署和维护进程，适合快速接入 SaaS 工具。
- Anthropic MCP Connector 把远程服务器作为 API 请求的一部分传入，Claude 自动完成工具发现、调用和结果回填，开发者无需实现 agent 工具循环。
- 安全边界必须前置：远程服务器可能看到工具调用参数和相关业务数据，因此只能连接信任的服务器，并审查隐私条款、使用最小权限 token。
- MCP 工具不是硬编码在客户端，而是在运行时发现，因此同一客户端可以动态接入新工具而不需要改代码。

## 对比与权衡

- 相比本地 MCP server（通过 stdio 子进程运行），远程 MCP server 免去自托管和运维，适合团队共享和 SaaS 场景，但网络延迟更高、数据经过第三方，信任边界更复杂。
- 相比原生 Function Calling 或 REST 集成，MCP 是跨厂商的开放标准，工具实现可复用，但增加了一层协议抽象，简单一次性集成可能过重。
- 相比在应用里直接写 REST API，MCP server 把鉴权、重试、工具 schema、错误处理封装统一，但调试链路更长，依赖服务器实现质量。

## 自测问题

**问: MCP 是什么，为什么它重要？**

MCP 是 Model Context Protocol，由 Anthropic 提出的开放协议，让 LLM 应用通过统一接口连接工具和数据源。重要性在于降低集成成本：每个工具实现一次 MCP server，所有兼容客户端都能复用，而不是每个模型或应用重复对接。

**问: 远程 MCP 服务器和本地 MCP 服务器的区别？**

本地通常以 stdio 子进程形式与本机客户端通信，适合本地文件、个人开发；远程通过 HTTP/SSE 或 streamable HTTP 暴露 URL，适合 SaaS 服务，需要鉴权和网络信任。远程免部署，但数据会经过第三方。

**问: Claude API 的 MCP Connector 如何工作？**

请求中传入 mcp_servers 配置，Claude 端连接远程服务器、调用 initialize 和 list_tools，把工具 schema 注入上下文；模型返回 tool_use 后，Connector 调用远程 call_tool，再把结果作为 tool_result 回填并继续生成。

**问: 连接远程 MCP 服务器时有哪些安全注意事项？**

只连接受信任服务器；使用最小权限 token；通过 headers 传凭据并加密存储；审查第三方隐私政策；生产环境加审计/网关；关键数据可考虑自托管开源 MCP server 替代远程 SaaS。

**问: MCP 与 OpenAI function calling 或 GPT Actions 相比如何？**

两者都让模型调用工具，但 function calling 是厂商绑定的接口，MCP 是跨平台开放标准。MCP 工具可以在 Claude、IDE、其他 agent 中复用；代价是引入协议层，简单场景可能不如原生 function calling 直接。

## 适用场景

- 需要把 Claude API 快速接入 Linear、Notion、CRM、支付等 SaaS 工具，但不想逐个自建集成。
- 需要为团队统一治理远程工具，通过同一 MCP Connector 配置多个数据源。
- 需要在不修改客户端代码的情况下动态增加工具或数据源。
- 需要先快速验证某个 MCP 工具生态，再决定是否自托管为本地 MCP server。

## 标签

`MCP` `Model Context Protocol` `Claude API` `Remote MCP` `Tool Integration`
