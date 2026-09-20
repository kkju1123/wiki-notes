# Fast mode：Claude Opus 高速输出模式（研究预览）

*原文: [https://platform.claude.com/docs/en/build-with-claude/fast-mode](https://platform.claude.com/docs/en/build-with-claude/fast-mode) · 来源: web · 生成时间: 2026-09-20T02:28:51.938750+00:00*

## 背景

大模型 API 的推理速度通常受限于输出阶段，长文本生成、流式对话和 agent 循环对 output tokens per second 十分敏感。传统优化要么牺牲智能换小模型，要么购买优先级容量但不提升单请求吞吐。Fast mode 因此在保留 Opus 系列能力的前提下，用 premium 定价提供更快的推理配置。

## 痛点

使用标准速度的 Claude Opus 处理长输出或流式场景时，用户会感到生成缓慢，agent 多轮调用也会累积明显延迟。如果团队不理解 fast mode 的独立限流和降级机制，遇到 429 后容易无从处理，甚至误以为所有模型都支持该参数。

## 解决办法

Fast mode 不更换模型权重，而是用相同的 Opus 模型配合更快的推理配置，把优化重点放在输出 token 的生成速度上，而不是首 token 延迟。请求时通过 speed:'fast' 和 beta header 显式启用，服务端会分配独立于标准速度的算力与限流。响应 usage.speed 字段可观测实际使用的速度；SDK 默认对 429 自动重试，业务侧也可以捕获错误后去掉 speed 参数回退到标准速度。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

# 1. 开启 fast mode：指定 speed 和 beta header
response = client.messages.create(
    model='claude-opus-5',
    max_tokens=1024,
    messages=[{'role': 'user', 'content': '...'}],
    speed='fast',
    extra_headers={'anthropic-beta': 'fast-mode-2026-02-01'},
)
print(response.usage.speed)  # 确认实际速度：fast 或 standard

# 2. 发生 429 时回退到标准速度
try:
    response = client.messages.create(
        model='claude-opus-5',
        max_tokens=1024,
        messages=[{'role': 'user', 'content': '...'}],
        speed='fast',
        extra_headers={'anthropic-beta': 'fast-mode-2026-02-01'},
    )
except anthropic.RateLimitError:
    response = client.messages.create(
        model='claude-opus-5',
        max_tokens=1024,
        messages=[{'role': 'user', 'content': '...'}],
    )
print(response.usage.speed)  # 此时大概率是 standard
```

第一段展示了开启 fast mode 的两个必要条件：speed 参数和 beta header；同时读取 usage.speed 用于确认实际速度。第二段演示了生产环境常见做法：fast mode 独立限流触发 429 时，捕获异常并去掉 speed 参数重试，从而牺牲速度但保证可用性，而不是一直等待重试。

## 关键流程

1. 确认使用支持的模型：Claude Opus 5 或 Claude Opus 4.8。
2. 在请求中加入 speed:'fast' 并携带 fast-mode-2026-02-01 beta header。
3. 读取响应 usage.speed 字段，确认为 fast；若模型不支持或配额不足会报错或降级。
4. 根据业务策略处理速率限制：依赖 SDK 自动重试，或捕获 429 后回退到 standard speed。

## 关键点

- Fast mode 使用同一套模型权重和更快的推理配置，因此智能与能力不变，只是输出吞吐提高。
- 优化集中在中后期输出 token 的生成速度（OTPS），而不是 TTFT，所以流式场景收益最明显。
- Fast mode 有独立于标准 Opus 的专用速率限制和更高定价，并且与其他计费 modifier 叠加，需要做成本与吞吐权衡。
- 请求是否真正走了 fast mode 不能靠假设，必须检查 usage.speed 字段；例如 Opus 4.6 会静默降级为标准速度。
- 切换 fast/standard 会使 prompt cache 失效，因为不同速度的请求不共享缓存前缀，这会间接影响成本与延迟。

## 对比与权衡

- 相比标准 speed，fast mode 在输出 token 吞吐上最高可提升 2.5 倍，但单 token 成本更高，且受独立速率限制约束。
- 相比换用更小或更快的模型（如 Haiku/Sonnet），fast mode 保留了 Opus 的强推理能力，但成本通常仍高于小模型方案。
- 相比 Priority Tier 这种容量保障机制，fast mode 提供的是单请求吞吐加速而非资源预留，两者不能同时使用。

## 自测问题

**问: fast mode 会不会改变模型能力或行为？**

不会。它使用相同模型权重，只替换更快推理配置，因此智能、风格、工具使用等能力保持一致；这一点可以由 usage.speed 和文档中的 same model weights 说明。

**问: 为什么加速重点是 OTPS 而不是 TTFT？**

LLM 推理分为输入预填充和输出逐 token 解码两个阶段。长输出场景中主要耗时在 decode 阶段逐 token 生成，TTFT 主要由输入 prompt 处理决定。Fast mode 通过优化 decode 阶段或推理调度提升 OTPS，因此流式输出收益最明显。

**问: 线上服务遇到 fast mode 429 应该怎么处理？**

先看 SDK 默认自动重试是否足够，因为 fast mode 使用连续 token 补充机制，retry-after 通常较短；如果业务不能等待，可以 catch RateLimitError 并去掉 speed 参数回退到 standard speed，但要注意 prompt cache 失效。

**问: 怎样确认一次请求真的使用了 fast mode？**

查看响应 usage.speed 字段，成功时为 fast。特别地，Claude Opus 4.6 不支持 fast mode 时不会报错，而是静默降级到 standard，因此监控该字段比只看请求参数更可靠。

**问: fast mode 为什么不支持 Batch API 或 Priority Tier？**

Batch API 面向低成本后台任务，资源调度优先级不同；Priority Tier 是容量预留，而 fast mode 是 premium 吞吐加速，二者在资源池和计费模型上冲突，因此官方暂不兼容。

## 适用场景

- 需要 Claude Opus 级强推理能力，同时用户对生成延迟敏感的长文本生成或流式对话。
- Agent 工作流中模型被循环调用，输出吞吐直接决定整体任务完成时间。
- 需要在不替换小模型的前提下提升响应速度，并愿意为 premium 速度支付更高成本。
- 做容量规划或成本核算时，需要理解 fast mode 的独立限流与 pricing 叠加规则。

## 标签

`Claude` `Fast Mode` `推理加速` `API 限流` `LLM 性能优化`
