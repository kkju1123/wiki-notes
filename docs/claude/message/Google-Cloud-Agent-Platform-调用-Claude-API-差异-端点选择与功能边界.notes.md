# Google Cloud Agent Platform 调用 Claude：API 差异、端点选择与功能边界

*原文: [https://platform.claude.com/docs/en/build-with-claude/claude-on-vertex-ai](https://platform.claude.com/docs/en/build-with-claude/claude-on-vertex-ai) · 来源: web · 生成时间: 2026-09-20T03:27:15.220473+00:00*

## 背景

Claude 原生 API 由 Anthropic 提供，而许多企业需要在 Google Cloud 上统一治理模型访问。Google Cloud 通过 Agent Platform/Model Garden 托管 Claude，允许复用 GCP 的 IAM、审计、账单和数据驻留能力。为了让多模型、多区域路由可行，Google Cloud 对 Anthropic Messages API 做了少量适配。

## 痛点

直接把 Anthropic 原生 HTTP 请求搬到 Google Cloud 会因 model 位置和 version 字段错误而失败。开发者还容易忽视端点类型和功能裁剪，导致数据驻留不满足合规，或上线后才发现 Files API、批处理等能力缺失。

## 解决办法

复用 Messages API 的消息语义，但把 model 放到 Google Cloud endpoint URL 路径中，并在 body 固定传 anthropic_version: vertex-2023-10-16。官方 SDK 会将模型 ID 映射为 URL 路径并自动注入版本，而底层仍遵循该请求格式。端点选择上，优先用 global endpoint 获得最高可用性和无溢价；有数据驻留要求时选择 multi-region 或 regional，并接受 10% 溢价。

## 关键代码示例

```python
import os
import httpx

project = os.environ['GOOGLE_CLOUD_PROJECT']
location = 'us-east5'
model_id = 'claude-sonnet-4-6'

url = (
    f'https://{location}-aiplatform.googleapis.com/v1/projects/{project}/'
    f'locations/{location}/publishers/anthropic/models/{model_id}:rawPredict'
)

access_token = os.environ['GOOGLE_ACCESS_TOKEN']
headers = {
    'Authorization': 'Bearer ' + access_token,
    'Content-Type': 'application/json',
}

payload = {
    'anthropic_version': 'vertex-2023-10-16',
    'max_tokens': 1024,
    'messages': [
        {'role': 'user', 'content': '解释全球端点、多区域端点和区域端点的区别'}
    ],
}

resp = httpx.post(url, headers=headers, json=payload)
print(resp.json())

```

这段代码模拟底层 HTTP 调用，重点展示 Google Cloud 与原生 Messages API 的两个差异：model_id 只出现在 URL 路径中，不在 body；anthropic_version 放在 JSON body 且固定为 vertex-2023-10-16。认证使用 gcloud 获取的 access token；真实项目中可直接使用 AnthropicVertex SDK，其内部做同样的 URL 构造和版本注入。

## 关键流程

1. 安装 Anthropic 官方客户端 SDK
2. 运行 gcloud auth application-default login 获取 Google Cloud 凭据
3. 确认目标区域可用 Claude 模型，并从 Model Garden 获取准确模型 ID
4. 创建客户端或构造 endpoint URL，将 model 放在 URL 路径、anthropic_version 放在 body
5. 根据数据驻留和可用性需求选择 global / multi-region / regional endpoint
6. 发送请求并处理 Messages API 响应，开启 request-response logging 便于审计

## 关键点

- 请求格式的两个关键差异：model 从 body 移到 endpoint URL，anthropic_version 从 header 移到 body 并固定为 vertex-2023-10-16。这个细节是能否在 Google Cloud 上成功调用的根本。
- 官方 SDK 已支持 Agent Platform，能自动处理模型 ID 到 URL 的映射并注入版本；熟悉底层差异有助于排错和面试讲解。
- global endpoint 无溢价且可用性最高，multi-region/regional 有 10% 溢价但提供数据驻留保证；provisioned throughput 仅 regional 支持。
- 功能支持并非与 Anthropic API 完全一致：不支持 Files API、Message Batches、服务端工具等，迁移前必须做差距分析。
- 最新 Claude 模型在 Agent Platform 上提供 1M token 上下文，但 30MB payload 限制可能先于 token 限制触发。
- 数据保留和活动日志由 Google Cloud 治理，Anthropic 建议至少 30 天滚动日志，便于异常排查和合规审计。

## 对比与权衡

- 相比 Anthropic 原生 Messages API，Google Cloud Agent Platform 复用了消息结构和 SDK，但请求格式有差异，且模型生命周期与弃用时间由 Google Cloud 管理，可能和官方 Claude API 不同步。
- 相比 Amazon Bedrock 上的 Claude，两者都提供区域部署和云平台治理能力；Google Cloud 的 global endpoint 在跨区域可用性上更灵活，而 Bedrock 与 AWS 原生服务（IAM、CloudWatch、VPC）集成更深。
- 相比直接调用 Anthropic API，托管在 Google Cloud 更适合需要 GCP 统一账单、IAM 权限与审计的企业，但必须接受功能裁剪、30MB payload 限制和区域端点的 10% 溢价。

## 自测问题

**问: 在 Google Cloud Agent Platform 调 Claude 时，请求格式与标准 Messages API 有哪些区别？**

标准 Messages API 的 model 在 body，anthropic_version 在 header；Agent Platform 要求 model 放到 endpoint URL 路径，anthropic_version 放到 body 并设为 vertex-2023-10-16。本质是为了让云平台网关能先根据 URL 中的模型 ID 做路由，再透传兼容的消息体。

**问: global、multi-region、regional 端点有什么区别，如何选择？**

global 动态路由到任意可用区域，可用性最高且无溢价，适合数据驻留不敏感场景；multi-region 在指定地理范围（如 us、eu）内路由，兼顾数据驻留和可用性；regional 固定区域，提供严格驻留保证，有 10% 溢价。需要注意 provisioned throughput 只能在 regional 使用。

**问: 在 Google Cloud 上使用 Claude 有哪些功能不支持？如果业务需要文件输入或批处理怎么处理？**

不支持 Files API、Message Batches、服务端工具、Admin/Usage/Cost 等。文件输入可在客户端读取文件内容后放入 messages 的 content 块；批处理可用应用层并发和重试替代。

**问: 如何做认证和日志审计？**

本地用 gcloud auth application-default login 获取 ADC，生产环境用 Workload Identity；Agent Platform 提供 request-response logging，建议至少保留 30 天，并结合 Cloud Logging 做异常检测和合规审计。

**问: 1M token 上下文是否意味着可以无限制发大文档？**

不是，Agent Platform 有 30MB payload 限制，发送大量图片或文档时可能在 token 达到上限前触发；需要评估文件大小，必要时压缩或分块。

## 适用场景

- 已经在 Google Cloud 上运行，希望统一通过 GCP 治理接入 Claude 的企业生成式 AI 应用。
- 要求数据驻留在欧盟或美国境内，并需要该地理范围内高可用的场景。
- 希望利用 Google Cloud 的 IAM、审计日志、Model Garden 和账单能力管理 Claude 模型访问权限的团队。
- 无需 Files API、服务端代码执行或原生 Anthropic Batch/Admin API 的标准对话、工具调用和结构化输出项目。

## 标签

`Claude` `Google Cloud` `Agent Platform` `Messages API` `LLM 部署`
