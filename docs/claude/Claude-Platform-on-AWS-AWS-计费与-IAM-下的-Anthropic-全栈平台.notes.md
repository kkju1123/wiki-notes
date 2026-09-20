# Claude Platform on AWS：AWS 计费与 IAM 下的 Anthropic 全栈平台

*原文: [https://platform.claude.com/docs/en/build-with-claude/claude-platform-on-aws](https://platform.claude.com/docs/en/build-with-claude/claude-platform-on-aws) · 来源: web · 生成时间: 2026-09-20T03:24:58.339893+00:00*

## 背景

企业客户普遍已在 AWS 上运行工作负载，采购、审计、网络和权限体系都围绕 AWS 构建。直接使用 Anthropic 第一方 API 会引入独立账单和认证体系，增加管理成本；而 Amazon Bedrock 虽然原生集成 AWS，但由 AWS 运营，功能发布滞后且不支持 Claude 的某些最新能力。Claude Platform on AWS 因此出现，让客户用 AWS 的计费和访问控制，同时使用 Anthropic 运营的完整 Claude API 平台。

## 痛点

如果团队不了解该集成，可能在 Bedrock 和 Claude Platform on AWS 之间选错：Bedrock 无法透传 anthropic-beta header、缺少 Agent Skills，功能更新慢；直接 Anthropic API 则无法使用 AWS Marketplace 统一采购、IAM 权限和 PrivateLink。此外，数据处理者不同会影响合规判断，选错可能导致监管风险。

## 解决办法

该方案本质是 AWS 作为商业与访问层，Anthropic 保留推理栈运营。客户端通过 AWS SigV4 或 API key 认证后，请求发送到 aws-external-anthropic.{region}.api.aws 端点，由平台专用 SDK（如 Python 的 AnthropicAWS）负责签名和调用。API 表面与第一方 Claude API 相同（/v1/{endpoint}），因此可以直接使用 Messages API、Agent Skills 和 beta headers。AWS Marketplace 处理订阅与账单，IAM 策略控制谁能调用，PrivateLink 可将 VPC 私密连接到端点。数据默认在 AWS 上处理，inference_geo 可固定推理地理，ZDR 可申请。

## 关键代码示例

```python
import os
from anthropic import AnthropicAWS

client = AnthropicAWS(
    aws_region=os.environ['AWS_REGION'],
)

response = client.messages.create(
    model='claude-sonnet-4-5',
    max_tokens=1024,
    messages=[{'role': 'user', 'content': '解释 Claude Platform on AWS 和 Bedrock 的区别'}],
    # inference_geo='us'  # 可选：固定推理地理区域
)

print(response.content[0].text)
```

这段代码创建平台专用客户端 AnthropicAWS，它会使用 AWS SigV4 签名向 aws-external-anthropic.{region}.api.aws 发送请求。凭据从 boto3 会话继承，因此可以复用 EC2 实例角色或本地 AWS 配置，避免硬编码密钥。messages.create 与第一方 Claude API 调用方式一致，inference_geo 参数用于对单次请求固定推理地理区域，体现数据驻留控制。

## 关键流程

1. 在 AWS Console 打开 Claude Platform on AWS 服务页并选择 Sign up，接受条款后等待 AWS 完成 Marketplace 订阅
2. 完成 Anthropic 组织设置：输入 owner 邮箱，通过邮件链接登录并填写组织信息，接受 Anthropic 条款
3. 创建 workspace 并记录 workspace ID，确认区域绑定和 IAM 资源范围
4. 登录 Claude Console（platform.claude.com）进行平台管理和使用 API

## 关键点

- Claude Platform on AWS 由 Anthropic 运营推理栈，AWS 只负责认证、IAM 和 Marketplace 计费；这决定了数据处理者是 Anthropic，合规团队必须按 Anthropic 的数据使用条款评估。
- API 表面是 Claude API 的 /v1/{endpoint}，并支持 anthropic-beta headers 和 Agent Skills，功能通常与第一方同日发布；因此它是 Bedrock 之外获取最新 Claude 能力的首选 AWS 集成。
- 认证支持 AWS IAM/SigV4 或 API key，且支持 AWS PrivateLink；这让企业可以把平台纳入现有 IAM 策略和 VPC 网络边界，而不是另建一套凭据体系。
- 它使用独立的容量池，与第一方 API 和 Bedrock 分开；虽然提供多平台故障转移可能，但配额和速率限制由 Anthropic 管理，容量规划时需注意。
- 创建账户涉及 AWS Console 注册、Anthropic 组织设置和工作区创建三步，AWS 自动处理 Marketplace 订阅；理解该流程有助于快速排错，尤其是重定向和邮箱验证环节。

## 对比与权衡

- 相比 Amazon Bedrock，Claude Platform on AWS 在 API 兼容性和功能速度上更好（支持 /v1、Agent Skills、anthropic-beta header，功能同日发布），但在强合规认证（FedRAMP High、IL4/IL5、HIPAA-ready）和 AWS 作为唯一数据处理者方面不如 Bedrock。
- 相比直接使用 Anthropic 第一方 API，本方案在 AWS 计费整合、IAM 权限管理和 PrivateLink 私密连接上更好，但增加了 AWS Marketplace 订阅和区域约束，且部分功能仍有平台限制。
- 相比 Bedrock legacy（Opus 4.6 及更早）的 bearer token 和 Bedrock SDK，本方案认证更灵活（API key 或 SigV4），SDK 更贴近第一方 Claude API，但需要引入新的平台专用客户端。

## 自测问题

**问: Claude Platform on AWS 和 Amazon Bedrock 的核心区别是什么？**

从操作方入手：前者 Anthropic 运营推理栈，AWS 只做认证计费；后者 AWS 运营。API 表面也不同：前者是第一方 /v1，支持 beta headers 和 Agent Skills；后者是 Bedrock Converse/InvokeModel，功能发布滞后。还要提到数据处理者不同，合规要求不同。

**问: 什么时候应该选 Claude Platform on AWS，而不是 Bedrock？**

当团队需要最新 Claude API 能力（如 Agent Skills、beta 参数），同时希望继续使用 AWS Marketplace 采购、IAM 权限和 PrivateLink，就可以选它。如果监管要求 AWS 必须是唯一数据处理者或需要 FedRAMP High/IL4/IL5/HIPAA，则必须选 Bedrock。

**问: 这个平台的认证是怎么工作的？API key 和 SigV4 如何选择？**

API key 是 Anthropic 颁发的静态密钥，适合非 AWS 的客户端或快速测试；SigV4 使用 AWS IAM 凭据签名请求，适合部署在 AWS 内部的服务，能继承角色和权限策略。平台专用 SDK 会自动处理 SigV4 签名。

**问: 数据驻留如何控制？ZDR 是什么？**

请求级可用 inference_geo 参数固定推理地理；工作区内容默认存储在 AWS 上，但子服务可能变更。ZDR 是零数据保留，默认关闭，需联系 Anthropic 代表为组织启用。

**问: Claude Platform on AWS 的容量和配额由谁管理？对容量规划有什么影响？**

速率限制和配额由 Anthropic 管理，与第一方 API 和 Bedrock 分开。这意味着不能依赖 Bedrock 的配额，需要单独向 Anthropic 申请或查看 Claude Console；同时可以利用多平台部署进行故障转移。

## 适用场景

- 需要在 AWS Marketplace 统一采购和计费的企业，且希望使用 Claude 最新 API 能力。
- 已经用 IAM 和 VPC PrivateLink 管理内部服务访问，想将 Claude API 纳入现有安全边界的团队。
- 需要 Agent Skills 或 beta 参数，但 Amazon Bedrock 尚未支持这些功能的场景。
- 对 Bedrock 和第一方 API 做多平台容灾，希望在不同容量池之间故障转移的架构。

## 标签

`Claude` `AWS` `Anthropic API` `Bedrock` `IAM`
