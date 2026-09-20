# 在 Amazon Bedrock 上使用 Claude（Opus 4.7 及之后版本）

*原文: [https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock) · 来源: web · 生成时间: 2026-09-20T03:20:06.848472+00:00*

## 背景

企业应用大模型时，常要求推理流量不出自有云、访问控制纳入 IAM、账单统一走云账号。Amazon Bedrock 是 AWS 托管的模型网关，早期通过 InvokeModel 等接口暴露模型，但其请求封装与 Anthropic 一方 API 不一致，迁移成本高。为此 Bedrock 推出新端点（bedrock-mantle）直接复用 Anthropic Messages API 请求形状，并提供 Anthropic 零操作员访问，满足高敏场景。

## 痛点

直接使用 Anthropic 一方 API 时，数据会离开 AWS 边界，企业需要自建 PrivateLink、管理额外 API 密钥，且难以用 AWS CloudTrail 审计模型调用。不懂 Bedrock 集成会误用旧 InvokeModel 端点或依赖 Bedrock 上不支持的参数，导致生产事故。

## 解决办法

通过 Bedrock 的 bedrock-mantle 端点暴露 /anthropic/v1/messages，复用标准 SSE 和一方 API 请求体，但接入 AWS 凭证链。认证首选 Bedrock service role：管理员建角色并授予 iam:PassRole，请求时由 Bedrock 代入角色，避免长期密钥。模型 ID 使用 anthropic. 前缀，SDK 按标准 AWS 优先级解析凭证和区域。全球端点动态路由提供高可用，区域端点满足数据驻留但收 10% 溢价。

## 关键代码示例

```python
from anthropic import AnthropicBedrock

client = AnthropicBedrock(
    aws_region="us-east-1",
    # 凭证自动从 AWS 标准链获取：
    # 环境变量 -> 共享配置文件 -> SSO/角色/ECS/IMDS
)

response = client.messages.create(
    model="anthropic.claude-sonnet-5",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "解释零信任安全模型"}
    ],
)
print(response.content[0].text)
```

这段代码使用 Anthropic 官方 SDK 的 Bedrock 专用客户端，避免手写 SigV4 签名。AnthropicBedrock 会从 AWS 标准凭证链读取 region 和临时密钥，因此不需要在代码中硬编码密钥。模型 ID 带 anthropic. 前缀，请求体与一方 Messages API 完全一致，体现新端点兼容性。

## 关键流程

1. 管理员在 AWS 控制台启用目标 Claude 模型的 Bedrock 访问权限
2. 根据安全需求选择认证路径：推荐 Bedrock service role，或 IAM assumed roles、短期 bearer token
3. 管理员创建服务角色并授予开发者 iam:PassRole 权限（或配置 IAM 信任策略和模型 ARN 权限）
4. 开发者安装 Anthropic SDK 的 Bedrock 模块，使用 AWS 标准凭证链解析区域和凭证
5. 调用 https://bedrock-mantle.{region}.api.aws/anthropic/v1/messages 端点，模型 ID 带 anthropic. 前缀

## 关键点

- Bedrock 版 Claude 的模型 ID 必须带 anthropic. 前缀，这与 Bedrock 多供应商命名规范一致，权限策略和 ARN 都依赖正确前缀。
- 新端点使用标准 SSE 和一方 API 相同的 Messages 请求体，因此现有 Anthropic SDK 代码只需更换客户端和模型 ID 即可迁移，不需要重写消息结构。
- 首选认证是 Bedrock service role 而非长期访问密钥，因为它把权限限制在模型 ARN 上，并通过 iam:PassRole 实现最小授权。
- 并非所有一方 API 功能都在 Bedrock 可用：structured outputs、server-side tools、batch API 等不支持，架构设计前必须核对功能列表，否则需要客户端兜底。
- 全球端点和区域端点的选择是可用性与合规/成本的权衡：全球端点免费提供跨区域路由，区域端点有 10% 溢价但数据不出指定区域。
- Anthropic 人员对 Bedrock 推理基础设施零操作员访问，这意味着支持排障和审计都完全在 AWS 边界内，适合高敏场景。

## 对比与权衡

- 相比直接调用 Anthropic first-party API，Bedrock 方案在 AWS 安全边界、IAM 认证和统一计费上更好，但在功能覆盖上不如一方 API（如缺少 structured outputs 和 server-side tools），且模型上线可能滞后。
- 相比旧版 Bedrock InvokeModel 集成，新的 bedrock-mantle 端点在请求体兼容性和 SSE 流式体验上更好，开发迁移成本更低；但在某些区域或功能上可能仍有限制。

## 自测问题

**问: 在 Bedrock 上调用 Claude 时，三种认证路径分别适用于什么场景？**

Bedrock service role 适合长期服务端应用，管理员预置角色并授予 pass role，Bedrock 自动代入；IAM assumed roles 适合企业联邦身份（SAML/OIDC/Identity Center），会话最长 12 小时，临时凭证由 STS 签发；bearer token 适合短期无 IAM 的脚本或测试，但最不安全，需用 policy 禁止长期 token 类型。

**问: 为什么 Bedrock 上某些 Claude 功能不可用，比如 structured outputs？**

Bedrock 作为托管平台需要为多租户和合规增加控制层，且新功能通常先在 Anthropic 一方 API 发布，再逐步适配到云平台。使用前必须查阅 feature support 矩阵；对于不支持的功能，可以改用客户端实现，如 client-side fallback 或提示工程约束 JSON。

**问: 如何理解 'zero operator access' 对安全性的实际意义？**

它意味着 Anthropic 作为模型提供商无法登录、查看或修改 AWS 托管推理服务的底层实例、日志或数据，即使有内部权限也不行。结合 AWS IAM、VPC endpoint 和 CloudTrail 审计，客户可以满足金融、医疗等行业的合规要求，数据和控制面完全留在 AWS 内。

**问: 全球端点与区域端点如何选择？10% 溢价值不值得？**

如果应用没有硬性数据驻留要求且追求最高可用性，选全球端点，由 Bedrock 动态路由跨区域，无溢价。如果法律或合同要求数据不出某国，必须选区域端点，溢价相当于为合规和可预测延迟付费。跨多个区域同地理可用 inference profile 配置路由。

**问: 从 Anthropic 一方 API 迁移到 Bedrock，代码需要改哪些地方？**

主要改三处：客户端从 Anthropic 换成 AnthropicBedrock（或对应 SDK 模块）；模型名加上 anthropic. 前缀；认证从 API key 换成 AWS 凭证链。请求体 Messages 结构保持不变，但需确认所用功能在 Bedrock 上是否受支持。

## 适用场景

- 金融、医疗等强合规行业，推理数据必须保留在 AWS 账户的 VPC 和安全边界内
- 企业已使用 AWS IAM 身份中心和 CloudTrail，需要把 Claude 调用纳入统一审计和账单
- 希望利用 AWS 全球端点实现跨区域高可用，避免单区域故障影响推理服务
- 已有 Anthropic SDK 代码，想以最小改动迁移到云平台托管环境

## 标签

`Claude` `Amazon Bedrock` `AWS IAM` `Messages API` `企业安全合规`
