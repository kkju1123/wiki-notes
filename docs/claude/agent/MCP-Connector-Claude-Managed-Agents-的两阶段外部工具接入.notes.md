# MCP Connector：Claude Managed Agents 的两阶段外部工具接入

*原文: [https://platform.claude.com/docs/en/managed-agents/mcp-connector](https://platform.claude.com/docs/en/managed-agents/mcp-connector) · 来源: web · 生成时间: 2026-09-20T07:14:42.670278+00:00*

## 背景

MCP（Model Context Protocol）由 Anthropic 提出，目标是标准化模型与外部工具、数据源的连接，避免每个服务都写定制胶水代码。Claude Managed Agents 需要复用同一套 Agent 定义，但不同租户/会话可能有不同凭据，因此需要一种安全且可审计的接入方式。本文对应的 `MCP connector` 就是该平台落地 MCP 的入口：把连接声明与运行时认证拆开。

## 痛点

如果把 API token 等凭据直接写在 Agent 定义里，会污染可复用配置并增大泄露风险；如果缺少工具级开关，MCP 服务端新增工具可能被 Agent 意外使用；若没有明确的连接失败语义，排障时只能盲猜是网络、认证还是配置问题。

## 解决办法

核心做法是两阶段分离。第一阶段创建 Agent：在 `mcp_servers` 里用 `type/name/url` 声明远端 MCP 服务器，并在 `tools` 数组中写对应的 `mcp_toolset`，用 `mcp_server_name` 做绑定，不需要任何 token。第二阶段创建 Session：传入 `vault_ids` 引用预注册的凭据，平台按规范化 URL 把 vault 凭据匹配到已声明的服务器。工具暴露由 `default_config` 和 `configs` 控制：可以默认全开再逐个禁用，也可以默认全关再白名单放行。连接失败不阻断会话启动，而是通过 `session.error` 异步上报，并携带可重试状态。

## 关键代码示例

```python
# Step 1: 创建 Agent，只声明连接与工具白名单，不包含凭据
agent_payload = {
    'mcp_servers': [
        {'type': 'url', 'name': 'github', 'url': 'https://mcp.github.com'}
    ],
    'tools': [
        {
            'mcp_toolset': {
                'mcp_server_name': 'github',
                'default_config': {'enabled': False},
                'configs': [
                    {'name': 'search_repositories', 'enabled': True, 'permission_policy': 'ask'}
                ]
            }
        }
    ]
}

# Step 2: 创建 Session，只挂载 vault；凭据按 URL 匹配
session_payload = {'vault_ids': ['vault_123']}
```

第一段创建 Agent 时只声明 MCP 服务器地址和工具白名单，连 token 都不出现；第二段创建 Session 时仅传 vault_ids，由平台按 URL 匹配凭据。这正好对应两阶段分离的核心思想：Agent 定义可复用、可入库，凭据按运行期注入。

## 关键流程

1. 创建 Agent：在 `mcp_servers` 中声明每个 MCP server 的 `type`、唯一 `name`、`url`。
2. 在 `tools` 数组中为每个 server 添加 `mcp_toolset`，并用 `mcp_server_name` 绑定；确保无未引用服务器或悬空工具集。
3. 通过 `default_config` 与 `configs` 配置工具默认开关、白名单/黑名单与 `permission_policy`。
4. 预先把凭据注册到 Vault，并确保凭据的 `mcp_server_url` 与声明 URL 在规范化后一致。
5. 创建 Session 时传入 `vault_ids`；平台按 URL 匹配凭据，不校验连通性。
6. 监听 `session.error` 事件，根据 `mcp_connection_failed_error` 或 `mcp_authentication_failed_error` 决定阻断、轮换凭据或继续会话。

## 关键点

- 两阶段配置把 MCP 连接声明和认证解耦，Agent 定义不携带 secret，可安全复用，不同会话也可用不同凭据。
- `mcp_servers` 与 `mcp_toolset` 必须一一对应，API 会拒绝未引用服务器或悬空工具集，这能避免配置漂移。
- 工具级 `default_config.enabled=false` 加 `configs` 白名单是生产环境的推荐模式，因为 MCP 服务端新增工具不会自动暴露给 Agent。
- 凭据匹配依赖 URL 规范化，理解 scheme/host 小写、默认端口和尾斜杠会被忽略，不同 path 或非默认端口不会匹配，有利于排障。
- Session 创建不会主动验证 MCP 连接或凭据，失败以 `session.error` 异步事件暴露，调用方必须实现容错策略，而不是假设工具一定可用。
- MCP 工具输出超过约 25k tokens 会落盘并给模型截断预览，这可以防止超长工具结果撑爆上下文窗口，同时保留完整数据。

## 对比与权衡

- 相比把外部工具直接实现为内置 tool（例如 `web_search` 这类平台自带工具），MCP connector 的好处是服务端可独立扩展工具集且跨客户端复用，但代价是需要额外管理 MCP server 的连通性与认证。
- 相比朴素地在 Agent 定义中嵌入 API token，Vault + 两阶段方案在安全性和多租户能力上更好，但引入额外的凭据注册与 URL 匹配步骤，复杂度更高。
- 相比 OpenAI 早期的 function calling 插件/自定义 API 集成，MCP 是开放协议且工具发现与调用格式统一，但生态成熟度与实时调试体验仍在演进。
- 相比本地 stdio MCP server，这里使用的远端 URL 型 MCP server 更适合托管 Agent 场景，但要求服务端具备网络可达性和可靠认证。

## 自测问题

**问: 为什么 MCP 配置要拆成 agent creation 和 session creation 两个阶段？**

核心是安全与复用。Agent 定义是可版本化、可复制、可共享的模板，如果掺入凭据，容易泄露且无法按会话隔离；运行期 session 代表一次具体执行，携带该次运行的身份。多数平台都会把静态配置与运行时 secret 分离，类似 K8s 的 Deployment 与 Secret。还可补充：这套机制也允许同一个 Agent 被不同用户/租户用不同 vault 凭据调用同一 MCP server。

**问: `mcp_servers` 和 `mcp_toolset` 为什么要互相引用？如果 API 不强制会有什么问题？**

互相引用能在创建时静态校验配置完整性，防止声明了 server 却没有任何工具入口，或 tools 引用了不存在的 server。若不做强制，运行时可能遇到莫名其妙的“工具不存在”或“服务器声明未使用”，排障成本高。可以类比类型系统：声明与使用必须匹配。

**问: 怎么控制 MCP server 暴露的工具范围？**

使用 `default_config.enabled` 做全局开关，再在 `configs` 里按工具名覆盖。推荐 production 默认全关、白名单启用，这样服务端新增工具不会自动获得权限。还可结合 `permission_policy` 处理敏感工具确认；注意 configs 条目只接受 `name/enabled/permission_policy`，不像内置工具有 type 和 web 域名限制。

**问: Session 创建不校验 MCP 连接，这样设计合理吗？你会怎么处理失败？**

合理，因为同步校验会拖慢 session 启动，而且网络/认证失败可能是瞬时的。系统选择异步发 `session.error`，给调用方灵活决策。工程上应监听事件流，根据 `retry_status` 和错误类型决定：`mcp_authentication_failed_error` 触发凭据轮换或告警，`mcp_connection_failed_error` 可用指数退避重试；非关键工具可降级继续会话。

**问: 凭据按 URL 匹配，有哪些容易踩的坑？**

URL 会先规范化：scheme/host 小写，去默认端口和尾斜杠；但 path、子域名、非默认端口不同就不匹配。因此要确保 vault 中的 `mcp_server_url` 与 agent 中 `url` 在业务上完全一致，不要期望它忽略 path 或换端口。还有安全上注意：如果多个 MCP server 共用一个 host 但不同 path，可能需要不同凭据，要分别创建 credential。

## 适用场景

- 需要让多个租户或客户复用同一个 Agent 模板，但各自使用自己的第三方服务凭据时。
- 通过 MCP 接入 GitHub、数据库、CRM 等大量外部工具，且希望只暴露少数几个给特定 Agent 时。
- 内部平台需要审计工具调用事件，按 `mcp_server_name` 追踪每个 MCP server 的调用和失败情况时。
- MCP 服务端不断新增工具，团队希望新增工具默认关闭并逐个评审后启用时。

## 标签

`MCP` `Claude Managed Agents` `Agent 工具编排` `Vault 认证` `安全配置`
