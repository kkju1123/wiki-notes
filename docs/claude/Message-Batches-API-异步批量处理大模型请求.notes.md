# Message Batches API：异步批量处理大模型请求

*原文: [https://platform.claude.com/docs/en/build-with-claude/batch-processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing) · 来源: web · 生成时间: 2026-09-20T06:10:59.106515+00:00*

## 背景

大模型 API 的同步调用需要为低延迟预留算力，容易在波峰波谷间产生资源浪费。批处理模式将多个请求合并提交、异步执行，以一定延迟换取更高的资源利用率。Anthropic Message Batches API 就是把这种模式应用到 Claude 请求上，适合离线评估、数据批注、批量内容生产等场景。

## 痛点

同步逐个请求处理海量数据时，延迟高、API 调用频繁且成本高。如果自己实现异步队列，还要处理结果持久化、重试、并发控制等大量工程问题。没有批处理能力会让大规模评测或数据生产既慢又贵。

## 解决办法

Message Batches API 把一组 Messages 请求打包提交，系统对每个请求独立异步处理，全部结束后返回结果文件。客户端通过 custom_id 关联原始请求与结果，并轮询 processing_status 从 in_progress 到 ended。价格是标准 API 的 50%，因为异步调度能填满空闲算力并减少低延迟保障成本。单个 batch 最多 10 万请求或 256MB，一般 1 小时内完成，结果可保留 29 天供下载。

## 关键代码示例

```python
import time
from anthropic import Anthropic

client = Anthropic()

batch = client.beta.messages.batches.create(requests=[
    {'custom_id': 'req-1', 'params': {'model': 'claude-sonnet-4-5', 'max_tokens': 1024,
     'messages': [{'role': 'user', 'content': 'Summarize: ...'}]}},
])

while batch.processing_status != 'ended':
    time.sleep(5)
    batch = client.beta.messages.batches.retrieve(batch.id)

for result in client.beta.messages.batches.results(batch.id):
    if result.result.type == 'succeeded':
        print(result.custom_id, result.result.message.content)
    else:
        print(result.custom_id, result.result.error)
```

这段代码先创建一个包含单个请求的 batch，用 custom_id 标记以便后续匹配；然后轮询 retrieve 直到 processing_status 变为 ended，避免忙等；最后用 results 拉取逐条结果，并区分 succeeded 和错误，体现批量处理的异步性与部分失败处理。

## 关键流程

1. 准备批量请求列表，为每个请求设置唯一 custom_id 和标准 Messages 参数。
2. 通过 batches.create(requests=[...]) 提交，返回 processing_status 为 in_progress。
3. 定期调用 batches.retrieve(batch_id) 轮询状态，直到 processing_status 变为 ended。
4. 调用 batches.results(batch_id) 逐一获取每个请求的结果，按 custom_id 匹配原始请求。
5. 对失败请求根据返回的 error 信息进行重试或人工处理。

## 关键点

- Batch API 按标准价格 50% 计费，因为异步调度能提高资源利用率，适合成本敏感的大规模离线任务。
- 每个 batch 最多包含 10 万个请求或 256MB，大多数在 1 小时内完成，超过 24 小时未完成会过期。
- 请求在 batch 内彼此独立处理，因此可以混合视觉、工具、多轮对话、扩展思考等多种请求类型。
- stream:true、speed、max_tokens:0 不支持，因为批处理返回文件而非流，也不涉及同步低延迟与缓存预热。
- 结果保留 29 天，需要在这期间下载；batch 本身仍可查看，但结果不可下载。
- 客户端必须处理部分失败：要按 custom_id 关联每条结果，并检查 succeeded 或 errored 状态。

## 对比与权衡

- 相比同步 Messages API，Batch API 在成本上降低 50% 且吞吐更高，但响应延迟从秒级上升到分钟/小时级，不适合需要即时返回或流式输出的场景。
- 相比自建异步任务队列，官方 Batch API 开箱即用、结果持久化和模型服务集成更好，但可观测性、自定义调度和重试策略的灵活性较弱。
- 相比流式请求的实时体验，批处理不能使用 stream:true，更适合离线数据处理而非交互式产品。

## 自测问题

**问: 为什么 Batch API 能做到同步 API 50% 的成本？**

同步 API 必须预留低延迟容量，导致峰值与空闲不均衡；批处理可离线排队、在算力低峰期填充空闲资源，减少调度和连接开销，提升整体利用率。本质是用延迟换成本，类似云厂商 Spot 实例或离线计算。

**问: Batch API 中部分请求失败怎么处理？**

批处理不是事务性整体成功或失败，每个请求独立处理。结果文件逐条返回 succeeded 或 errored，需要按 custom_id 关联原始请求；对 errored 请求根据错误类型做重试、降级或人工审核，并记录幂等键避免重复计费。

**问: Batch API 为什么限制 24 小时完成、结果保留 29 天？**

24 小时是处理 SLA 与资源调度的上界，超过则过期，避免无限占用队列；29 天结果保留是为了让客户端有足够时间下载结果文件，同时控制平台存储成本。实际多数 batch 在 1 小时内完成。

**问: Batch API 不支持 stream:true、speed、max_tokens:0，分别是什么原因？**

流式结果与批量返回单个文件冲突；speed 优化的是同步低延迟，不适合异步批处理；max_tokens:0 用于缓存预热，但批处理中创建的短暂缓存很可能在后续请求运行前过期，因此官方禁用以避免误用。

**问: 什么时候不应该用 Batch API？**

需要实时响应、流式输出、强顺序依赖或低延迟交互的产品不适合；如果任务量很小、对成本不敏感，同步 API 更简单；如果单请求结果会影响后续请求的动态编排，也要用同步或工作流。

## 适用场景

- 大规模评测：一次性跑数千道题或测试集，生成评估报告。
- 内容审核：对大量 UGC 文本异步分类违规或安全风险。
- 离线数据标注与摘要：给数据集批量生成标签、摘要、情感分析等。
- 批量内容生产：生成电商商品描述、文章摘要、多语言翻译等。

## 标签

`Message Batches API` `异步批处理` `成本优化` `大规模推理` `Anthropic Claude`
