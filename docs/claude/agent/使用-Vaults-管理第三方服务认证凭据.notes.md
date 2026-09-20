# 使用 Vaults 管理第三方服务认证凭据

*原文: [https://platform.claude.com/docs/en/managed-agents/vaults](https://platform.claude.com/docs/en/managed-agents/vaults) · 来源: web · 生成时间: 2026-09-20T07:41:02.298620+00:00*

## 背景

随着 agent 应用需要代表不同用户调用第三方服务，开发者面临安全存储、分发和轮换用户 token 的问题。直接放进 prompt 或代码会泄露，自建 secret store 则增加复杂度。Claude 平台内建 Vaults，将认证信息作为平台原语托管，统一解决这些问题。

## 痛点

没有 Vaults 时，开发者只能把 token 塞进 session 上下文或自建 secret store；前者易在模型日志和 prompt 中泄露，后者需要维护额外服务、处理密钥分发和网络授权。轮换 key 时通常需要手动重启会话，影响用户体验，且难以追踪 agent 代表哪个最终用户执行操作。

## 解决办法

Vault 作为 per-user 凭据集合，创建后通过 vault_ids 在 session 创建时引用。MCP 凭据按 mcp_server_url 匹配注入 token；环境变量凭据在沙箱内只存占位符，出站请求 egress 时替换为真实 secret，agent 不接触明文。平台周期性 re-resolve 凭据，使 secret 轮换、归档或删除自动传播到运行中 session，无需重启。敏感字段只写不读，结构字段不可变，变更通过 archive+create，archive 保留审计记录但清除 secret。类比：vault 是用户保险箱，agent 只拿箱号，开箱取物由平台完成。

## 关键代码示例

```python
from anthropic import Anthropic

client = Anthropic()

# 1. 为某个用户创建 vault
vault = client.beta.vaults.create(
    display_name="user-123",
    metadata={"user_id": "123"}
)

# 2. 添加环境变量凭据，secret_value 只写
client.beta.vaults.credentials.create(
    vault_id=vault.id,
    credential={
        "type": "environment_variable",
        "secret_name": "GITHUB_TOKEN",
        "secret_value": "ghp_xxx",
        "injection_location": {"type": "all_outbound_requests"}
    }
)

# 3. 创建 session 引用 vault
session = client.beta.sessions.create(
    agent_id="my-agent",
    vault_ids=[vault.id],
)
```

第一段创建 vault 对应一个最终用户，metadata 用于映射回自己的用户系统。第二段添加环境变量凭据，secret_value 只写，API 响应不会返回该值。第三段创建会话时传入 vault_ids，运行时平台会在沙箱中注入占位符并在 egress 替换真实 secret，agent 代码无法读到明文。

## 关键流程

1. 创建 vault：设置 display_name 和 metadata 映射到用户记录。
2. 添加 credential：选择 environment_variable 或 MCP 类型，配置唯一键和敏感字段。
3. 创建 session 时传入 vault_ids 引用 vault。
4. 运行时平台自动解析并注入凭据，周期性 re-resolve 以支持轮换。
5. 需要轮换时更新敏感字段或 archive+create，通过 webhook 监控 OAuth 刷新失败。

## 关键点

- Vault 以用户为单位组织凭据，session 通过 vault_ids 引用，实现用户级隔离和审计。
- 敏感字段只写不读，API 响应不会返回，减少泄露面。
- 环境变量凭据使用 egress 替换，agent 无法看到明文 secret，适合 CLI/SDK 等工具。
- key 在 vault 内唯一且不可变，重复创建返回 409，变更必须 archive+create，保证稳定引用。
- 生命周期管理自动传播到运行中 session，secret 轮换无需重启；OAuth 刷新失败会发 webhook。
- archive 和 delete 语义不同：archive 清除 secret 但保留审计记录，delete 完全删除；生产环境优先 archive。

## 对比与权衡

- 相比自建 secret store（如 HashiCorp Vault、AWS Secrets Manager），Claude Vaults 与 agent 平台深度集成，无需额外服务发现和网络授权，但功能相对受限，不支持复杂策略。
- 相比在 prompt 或 session 上下文中直接传 token，Vaults 不在模型可见文本中暴露凭据，降低日志和 prompt injection 泄露风险，但需要依赖平台 API 完成注入。
- 相比自行实现 OAuth refresh 和 secret 轮换，Vaults 提供内置 re-resolution 和 webhook 通知，开发者工作量更小，但灵活性不如完全自控的 refresh 流程。

## 自测问题

**问: Vault 和 Credential 的关系是什么？为什么 vault reference 是 per-session 参数？**

Vault 是用户级凭据容器，credential 是具体 secret；per-session 引用可以按用户会话动态选择凭据，实现不同用户不同权限的隔离，同时便于审计 agent 代表谁操作。

**问: 环境变量凭据的“opaque placeholder”原理是什么？为什么 agent 看不到 secret？**

平台在沙箱中注入占位符（如 ${SECRET_NAME}），agent 看到的是占位符；在出站请求 egress 时，平台网络层将占位符替换为真实值，因此 agent 代码和日志不会暴露明文。

**问: 为什么 credential 的结构字段不可变？如何修改 mcp_server_url？**

结构字段作为引用键，若可变会导致正在使用的会话引用不稳定；标准做法是 archive 旧 credential（清除 secret、释放键）并创建新 credential，实现有序迁移和审计。

**问: 多个 vault 都包含同一个 mcp_server_url，运行时如何选择？**

按 vault_ids 顺序，第一个含匹配 credential 的 vault 生效，需在设计时避免歧义，确保用户会话的 vault 顺序符合预期。

## 适用场景

- 多租户 SaaS agent 平台：每个用户连接自己的 GitHub/Jira 等工具。
- Claude agent 调用 MCP server：集中管理每个用户的 OAuth token 和 static bearer token。
- 需要为 CLI/SDK 工具注入环境变量凭据（如 AWS、npm 私有源）而不暴露给模型。
- 需要运行中轮换 API key 而不断续 agent 会话的生产环境。

## 标签

`vaults` `secrets management` `MCP` `OAuth` `agent authentication`
