# Amazon Bedrock 上的 Claude 集成（Opus 4.6 及更早版本）

*原文: [https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock-legacy](https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock-legacy) · 来源: web · 生成时间: 2026-09-20T03:22:50.121386+00:00*

## 背景

Claude 官方 API 适合直接使用，但企业往往要求模型流量留在 AWS VPC 内、通过 IAM 统一鉴权、账单合并到 AWS 账号并满足审计合规要求。Amazon Bedrock 作为托管模型网关把这些能力开放给企业，用户无需自行部署模型或管理公网 API key。本文覆盖的是 Bedrock 上较旧的 InvokeModel/Converse 集成方式，适用于 Opus 4.6 及更早版本。

## 痛点

只熟悉官方 Claude API 的人直接迁移到 Bedrock，会遇到鉴权机制不同、模型 ID 不是普通字符串而是 ARN 或 inference profile、新模型必须加路由前缀等问题，常见表现是 400 或权限拒绝。不了解 Bedrock 功能差异还可能误用 Files API、server-side tools 等不支持能力，导致方案返工。

## 解决办法

把 Bedrock 当作 AWS 内部的模型目录和网关：先在 Model Access 订阅 Anthropic 模型，再用 AWS SigV4 凭证或 bearer token 鉴权。调用时走 bedrock-runtime 的 InvokeModel 或 Converse API，模型 ID 使用 Bedrock 的 ARN 版本化标识；Opus 4.6、Sonnet 4.6 等新模型通常必须把 base ID 换成带 global/us/eu/jp/apac 前缀的 inference profile ID 或完整 ARN，否则会返回 400。Anthropic 官方 SDK 的 Bedrock backend 封装了这些差异，自动处理凭证和 Messages 格式翻译。可以类比为企业采购：不直接联系厂商，而是通过内部采购系统下单，获得统一审批、审计和成本控制。

## 关键代码示例

```python
import json
import os

import boto3

client = boto3.client('bedrock-runtime', region_name=os.environ['AWS_REGION'])

payload = {
    'anthropic_version': 'bedrock-2023-05-31',
    'max_tokens': 512,
    'messages': [{'role': 'user', 'content': 'What is S3?'}],
}

resp = client.invoke_model(
    # 新模型需要 inference profile，例如 global/us/eu 前缀 + base ID
    modelId=os.environ['BEDROCK_MODEL_ID'],
    contentType='application/json',
    accept='application/json',
    body=json.dumps(payload),
)

result = json.loads(resp['body'].read())
print(result['content'][0]['text'])
```

示例创建 bedrock-runtime 客户端，因为模型推理走该端点，而 Model Access/ListModels 走 control plane。payload 使用 Anthropic Messages 格式，并带 Bedrock 要求的 anthropic_version 字段。modelId 从环境变量读取，避免硬编码；实际部署时替换为 Bedrock 控制台或文档中的模型 ID 或 inference profile。返回体是 JSON，读取 body 流后按 content[0].text 提取文本。

## 关键流程

1. 安装或升级 AWS CLI 到 2.13.23 及以上，运行 aws configure 配置访问密钥并验证凭证。
2. 安装 Anthropic 客户端 SDK（推荐）或 boto3，用于访问 Bedrock。
3. 在 AWS Console 的 Bedrock Model Access 页面请求访问 Anthropic 模型，注意区域可用性。
4. 从 AWS 文档确认模型 ID 或推理配置文件；新版模型使用带 global/us/eu/jp/apac 前缀的 profile ID 或完整 ARN。
5. 在代码中创建 bedrock-runtime 或 AnthropicBedrock 客户端，构造 Messages 请求并调用 InvokeModel/Converse。
6. 在 Bedrock 中开启 invocation logging，至少保留 30 天以便审计和排查。

## 关键点

- Bedrock 使用 AWS IAM/SigV4 或专用 bearer token 认证，而不是 Anthropic 的 x-api-key；理解这一点才能正确设计跨账号访问、权限边界和审计。
- 新版 Claude 模型依赖 cross-region inference profiles，直接传 base model ID 会返回 HTTP 400；必须按区域支持表选择 global/us/eu/jp/apac 前缀或完整 ARN。
- Bedrock 上的模型生命周期由 AWS 控制，退役时间可能与 Anthropic 官方 deprecation 不一致；生产上线前应查询 Amazon Bedrock model lifecycle。
- Bedrock 支持 Messages API、prompt caching、thinking、tool use、citations、structured outputs 等核心能力，能覆盖大多数生成场景。
- Bedrock 不支持 Files API、URL input sources、server-side tools、Agent Skills、Message Batches、Admin/Compliance API 等；需要这些能力时应保留官方 Claude API 通道。
- 建议开启 invocation logging 至少 30 天，用于安全审计、成本归因和滥用调查。

## 对比与权衡

- 相比直接使用 Anthropic Claude API，Bedrock 集成在 AWS IAM 权限模型、VPC 私有访问、统一账单和 CloudTrail 审计上更好，但在新模型上线速度和 server-side 功能覆盖上不如官方 API 及时。
- 相比直接使用 boto3 调用 InvokeModel，使用 Anthropic 官方 SDK 的 Bedrock backend 在类型安全、参数校验和 Messages API 兼容性上更好，但在需要完全控制 AWS 元数据或复用已有 boto3 代码时不如 boto3 透明。
- 相比 Bedrock Converse API 的统一跨模型接口，InvokeModel 对 Claude 特定参数更灵活且兼容历史代码，但 Converse 在多供应商模型切换和避免厂商锁定上更有优势。

## 自测问题

**问: Bedrock 调用 Claude 与官方 Claude API 鉴权有什么不同？为什么企业更倾向 Bedrock？**

官方用 x-api-key，Bedrock 用 AWS SigV4 或 bearer token。企业倾向 Bedrock 是因为模型流量可以留在 VPC，IAM 角色做最小权限控制，CloudTrail 记录 API 调用，账单合并到 AWS，审计和合规更简单。

**问: 调用 Opus 4.6 时直接传 base ID 报 400，怎么排查？**

典型原因是新版模型不提供 on-demand base ID，必须传 cross-region inference profile。先确认已订阅模型且区域可用，再查 AWS inference profiles support 页面，选择合适的 global/us/eu/jp/apac 前缀，传 profile ID 或完整 ARN。

**问: Bedrock 集成不支持哪些常用功能？如果需求用到 Files API 怎么办？**

不支持 Files API/URL sources、server-side tools、Agent Skills/MCP connector、Message Batches、Admin/Compliance/Usage Cost API、managed agents、server-side fallback、automatic prompt caching。Files API 可改成把文件读入内存或对象存储，以 base64 或文本块交给 Messages；server-side tools 改成客户端工具执行。

**问: 在 Bedrock 上怎么做 prompt caching？和官方 API 的 automatic caching 有什么区别？**

Bedrock 只支持显式 cache breakpoints，不支持顶层 cache_control 自动缓存。请求中对内容块加 cache_control: {type: ephemeral} 标记缓存边界，按 token 计费并有 TTL。显式控制更可预测，但需要开发者识别可缓存前缀；官方 API 可自动处理。

**问: 如果团队要统一接入多家模型，应该优先用 Bedrock Converse 还是 InvokeModel？**

多模型统一接入优先选 Converse API，因为它提供统一消息结构、系统提示和工具调用协议，切换模型只需改 modelId，降低供应商锁定。如果重度依赖 Claude 特有字段或历史代码，可继续用 InvokeModel，并在适配层做隔离。

## 适用场景

- 已运行在 AWS 上、需要通过 VPC 私有链路和 IAM 角色访问 Claude 的生产应用。
- 需要把 Claude 用量合并到 AWS 账单，并用 tags/CloudTrail 做成本分摊与审计的企业场景。
- 已通过 Bedrock 接入其他基础模型，希望用统一 Converse API 切换或加入 Claude 的多模型平台。
- 需要长期保存提示词和补全日志，用于滥用调查、安全合规或效果分析的系统。

## 标签

`AWS Bedrock` `Claude` `模型集成` `IAM认证` `inference profiles`
