# Claude 在 Microsoft Foundry 中的接入与调用

*原文: [https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry) · 来源: web · 生成时间: 2026-09-20T03:29:08.345647+00:00*

## 背景

Claude 等前沿大模型通常通过 Anthropic 官方 API 访问，但企业客户往往已经在 Azure 上管理预算、网络、身份与合规。Microsoft Foundry 将 Claude 作为 Azure Marketplace 产品接入，使团队能沿用 Azure 订阅计费、Entra ID/RBAC 和网络隔离，而不必单独与 Anthropic 建立计费关系。另一方面，Foundry 通过保持 `/anthropic/v1/*` 兼容路径，让现有 Anthropic SDK 和代码尽量少改动即可迁移。

## 痛点

如果直接使用 Anthropic API，企业要在 Azure 之外管理另一套密钥、账单与安全审计，增加合规成本。如果不知道 Foundry 的资源/部署两层模型或托管选项区别，容易在数据驻留、模型可用性和计费上踩坑，比如把需要 US Data Zone 的负载部署到 Global Standard。

## 解决办法

核心做法是先创建 Foundry 资源作为安全与计费边界，再在其中创建 Claude 部署作为实际的模型实例；部署名一旦创建不可更改，并作为 API 请求中的 model 字段。认证可采用 API Key（快速测试）或 Entra ID 令牌（生产 RBAC/托管身份）。调用时端点形如 `https://{resource}.services.ai.azure.com/anthropic/v1/*`，SDK 只需把 base_url 指向该前缀并传入密钥，之后仍使用标准 Messages API 结构。可把 Foundry 网关理解为 Azure 内部的反向代理：对客户端暴露 Anthropic 兼容 API，对后端路由到 Azure 托管或 Anthropic 托管的推理服务。

## 关键代码示例

```python
import os
from anthropic import Anthropic

resource = os.environ["ANTHROPIC_FOUNDRY_RESOURCE"]
api_key = os.environ["ANTHROPIC_FOUNDRY_API_KEY"]

client = Anthropic(
    api_key=api_key,
    base_url=f"https://{resource}.services.ai.azure.com/anthropic/",
)

response = client.messages.create(
    model="my-claude-deployment",   # Foundry 部署名，不是原始模型 ID
    max_tokens=1024,
    messages=[{"role": "user", "content": "解释 Foundry 的托管选项"}],
)
print(response.content[0].text)
```

这段代码从环境变量读取资源名和 API Key，避免硬编码；base_url 由资源名拼接为 Foundry 的 Anthropic 兼容前缀，SDK 会自动补 `/v1/messages`。`model` 对应你在 Foundry 中创建的部署名（可自定义且不可改），因为网关需要知道路由到哪个部署。请求与响应仍遵循 Anthropic Messages API，因此现有大模型调用逻辑可以复用。

## 关键流程

1. 登录 Foundry 门户，创建或选择 Foundry 资源，并配置 API Key 或 Entra ID 访问控制。
2. 在模型目录中搜索 Claude 模型，选择 Deploy → Custom settings，接受 Azure Marketplace 条款。
3. 配置部署名、区域范围（Global/Data Zone）、模型版本（对应 Hosted on Azure 或 Hosted on Anthropic）。
4. 等待部署完成，从 Build → Models → 部署详情页复制 Target URI 和 Key。
5. 在 SDK 或 HTTP 请求中使用资源名/Base URL 和认证信息，将 model 设置为部署名发起调用。

## 关键点

- Foundry 资源与部署是两层模型：资源承载安全和计费配置，部署是 API 实际调用的模型实例；理解这点才能正确映射 RBAC 权限和账单。
- 部署名创建后不可更改，并且在请求中作为 model 字段使用，因此应在创建前规划命名规范，避免后期改应用配置。
- Hosted on Azure 与 Hosted on Anthropic 的核心差异在于推理运行位置和支持范围；前者适合合规/常规负载，后者适合抢先使用新模型或新功能。
- API Key 认证适合快速验证，但生产环境应优先使用 Entra ID/RBAC 或托管身份，以便实现短时令牌、统一审计和最小权限。
- 端点保持 `/anthropic/v1/*` 兼容格式，SDK 只需替换 base_url 和认证信息，因此可以将本地或官方 API 代码低成本迁移到 Foundry。
- 通过 Azure Marketplace 用 Claude Consumption Units 计费，既享受 Azure 订阅集中结算，也要注意它不代表所有模型都自动运行在 Azure 数据中心。

## 对比与权衡

- 相比直接使用 Anthropic 官方 API，Foundry 接入在 Azure 计费、Entra ID/RBAC 和企业网络集成上更好，但模型上线和部分新功能可能晚于官方 API。
- 相比 Azure OpenAI Service，Claude in Foundry 提供了非 OpenAI 模型和 1M token 上下文等 Claude 特性，但在 Azure 原生工具链和部分 AI 服务协同上可能不如 Azure OpenAI 成熟。
- Hosted on Azure 相比 Hosted on Anthropic，在数据驻留和 US Data Zone Standard 上更好，但在模型/功能覆盖和最新可用性上不如 Hosted on Anthropic。

## 自测问题

**问: Foundry 中 resource 和 deployment 有什么区别？**

resource 是 Azure 计费、安全和网络隔离的边界，类似一级容器；deployment 是某个模型的实例，才是真正通过 API 调用并消耗 Claude Consumption Units 的对象。类比 Azure OpenAI：创建 resource 后还需要在 resource 下创建 deployment。

**问: Hosted on Azure 和 Hosted on Anthropic 应该怎么选？**

优先看数据驻留和合规：如果要求推理必须在美国境内且需要 Azure 基础设施，选 Hosted on Azure 的 US Data Zone Standard；如果团队急需最新模型或默认 Azure 托管尚未支持的功能，选 Hosted on Anthropic，但要接受只有 Global Standard 和推理在 Anthropic 基础设施。

**问: API Key 和 Entra ID 认证分别适合什么场景？**

API Key 简单，适合本地开发和一次性脚本；Entra ID 使用 Azure AD 身份，可结合 RBAC、Managed Identity 和短期 token，避免静态密钥泄露，适合生产环境、CI/CD 和多团队授权。

**问: 为什么请求路径里有 `/anthropic/v1/*`？SDK 迁移要改什么？**

Foundry 网关在 Azure 域名后暴露与 Anthropic 兼容的路径，尽量减少客户端差异。迁移时把 SDK base_url 从 `https://api.anthropic.com` 改成 `https://{resource}.services.ai.azure.com/anthropic/`，认证头换为 Foundry 的 api-key 或 Entra token；业务层 messages 结构不变。

**问: 部署名为什么不能改？它和模型 ID 是什么关系？**

部署名是 Azure 资源上的实例标识，创建后与路由和计费元数据绑定，Azure 不提供原地重命名；它默认取模型 ID，但自定义后调用时要传部署名而不是模型 ID。生产命名建议包含模型、区域和用途，如 `claude-sonnet-eastus-prod`。

## 适用场景

- 企业已在 Azure 上统一管理预算、安全和网络，想要接入 Claude 做代码生成、知识库问答等，而无需在 Azure 外单独开账单。
- 需要满足数据驻留要求（如推理必须留在美国）的合规项目，可创建 US Data Zone Standard 部署。
- 需要在官方 Anthropic API 之外用同一套 Azure 身份与审计体系，通过 SDK/CLI 构建多模型网关。
- 团队想试用 Claude 最新模型或功能，但仍希望从 Foundry 门户管理部署和访问权限。

## 标签

`Claude` `Microsoft Foundry` `Azure AI` `LLM 推理` `企业身份与计费`
