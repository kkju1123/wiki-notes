# Claude Effort 参数：单模型内的 token 预算与能力权衡

*原文: [https://platform.claude.com/docs/en/build-with-claude/effort](https://platform.claude.com/docs/en/build-with-claude/effort) · 来源: web · 生成时间: 2026-09-20T02:25:35.527374+00:00*

## 背景

大模型 API 默认会倾向高质量输出，但所有任务都按同一强度生成，会让简单任务也消耗过多 token、拉高延迟和成本。过去要省成本往往只能切换更小模型或靠 prompt 约束，不够精细。effort 参数的出现，是为了在同一个模型内提供统一的“推理/输出预算”旋钮，让开发者按任务价值分配 token。

## 痛点

没有 effort 时，开发者只能在默认 high 上接受高消耗，或手动切换模型、写复杂 prompt 来压成本。简单任务可能因此多付几倍 token，而复杂任务又无法稳定地提升推理深度；只用 max_tokens 还容易造成硬截断，丢失结论。

## 解决办法

在请求中设置 output_config.effort，可选 max、xhigh、high、medium、low 五档，默认 high。它作用于所有输出 token，包括正文、工具调用和 thinking，因此本质上是改变模型“愿意花多少 token 来思考和执行”。低档位会让模型更简洁、工具调用更少；高档位则增加 thinking 和工具探索。高 effort 必须配合大 max_tokens，因为思考加工具输出会占用大量 token。不同模型推荐档位不同，应通过 evals 扫描选定；部分模型支持 per-message output_config 中途切换 effort 并保留 prompt cache。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

resp = client.messages.create(
    model='claude-opus-5',
    max_tokens=4096,
    output_config={'effort': 'medium'},
    messages=[{'role': 'user', 'content': 'Summarize this report.'}],
)
print(resp.content[0].text)

# 高难度任务可调到 xhigh/max，并同步增大 max_tokens:
resp = client.messages.create(
    model='claude-opus-5',
    max_tokens=64000,
    output_config={'effort': 'xhigh'},
    messages=[{'role': 'user', 'content': 'Investigate and fix the failing test.'}],
)
print(resp.content[0].text)
```

第一段请求用 medium 覆盖默认 high，适合常规总结任务，体现通过 output_config.effort 显式控制本次输出预算。第二段请求把档位调到 xhigh，并同步将 max_tokens 设为 64000，因为高档位会产生更多 thinking 和工具调用，需要更大硬限制防止被截断。两段代码说明：effort 管“预算偏好”，max_tokens 管“硬上限”。

## 关键流程

1. 第一步：确认模型支持哪些 effort 档位，默认是 high；新项目先按官方模型建议选起点，例如 Fable 5.1/Opus 5 从 high，Opus 4.7/4.8 的 coding/agentic 场景从 xhigh。
2. 第二步：在 Messages API 请求顶层设置 output_config.effort，显式覆盖默认值。
3. 第三步：如果使用 xhigh/max 高档位，把 max_tokens 调大，例如从 64k 起步，给 thinking 和工具调用留出空间。
4. 第四步：在真实 evals 上扫描不同 effort，对比质量、延迟、token 成本，选可接受质量下的最低档。
5. 第五步：长对话需要切换档位时，使用 beta 的 per-message output_config，保留 prompt cache。

## 关键点

- effort 是“总输出预算”旋钮，作用于正文、工具调用和 thinking 所有输出 token，因此它能在不切换模型的情况下同时影响能力、成本和延迟。
- 默认是 high；没有显式设置时系统按 high 执行，需要低档或高档必须通过 output_config.effort 显式覆盖。
- 低 effort 不等于可靠地缩短可见回复；尤其在 Opus 5 上，effort 主要控制 thinking volume，要控制回复长度应通过 prompt 明确要求。
- 高 effort 档位必须有足够的 max_tokens，否则思考或工具调用会被硬截断，导致表现反而下降。
- 不同模型的 effort 建议不同，且新模型低档位可能仍比旧模型高档位更强；因此生产前要用自己的 evals 做 fresh sweep，而不是沿用旧配置。
- 部分模型在高档位禁止禁用 thinking，例如 Opus 5 的 xhigh/max 下传 thinking disabled 会返回 400，实现时需让 thinking 配置与 effort 档位保持一致。

## 对比与权衡

- 相比只用 max_tokens，effort 直接调节模型生成和推理时的 token 花费倾向，能减少冗余思考而不是直接截断；但 effort 不提供 token 总量硬保证，所以仍需配合 max_tokens 防止失控。
- 相比通过 prompt 要求“简短回答”，effort 对所有输出类型包括工具调用和 thinking 都生效，行为更稳定可量化；但 effort 不精确控制可见文本长度，需要简短输出时仍要在 prompt 中说明。
- 相比切换不同规格模型，effort 在单个模型内实现成本和能力分级，部署和路由更简单；但能力上限仍受所选模型本身限制，极端任务无法突破模型天花板。

## 自测问题

**问: effort 参数和 max_tokens 有什么区别？**

max_tokens 是输出硬上限，达到就截断；effort 是模型内部生成预算，影响它愿意在 thinking、工具调用、解释中花多少 token。两者配合使用：高 effort 需要大 max_tokens，否则可能截断思考；低 effort 不强制减少 token，只是模型倾向更省。可以概括为“一个是天花板，一个是油门/预算偏好”。

**问: 为什么 API 默认是 high 而不是 medium 或 low？**

Claude 优先保证复杂推理和 agentic 任务质量，high 等价于不设参数时的默认行为，避免用户因省 token 而意外降低质量。只有显式设置才覆盖默认，这符合 LLM API 的保守默认设计；生产环境应根据自己的 evals 主动降档。

**问: 把 effort 调低后，响应文本一定会变短吗？**

不一定。低 effort 会减少 thinking 和工具调用，整体 token 下降，但可见文本长度不一定可靠缩短。尤其在 Opus 5 上，文档指出 effort 控制的是 thinking volume 而非响应长度；要控制篇幅应通过 prompt 指定长度。

**问: 如何为生产系统选择合适的 effort 档位？**

先按任务类型设候选档位：简单抽取、子代理用 low/medium；常规智能任务用 high；复杂 coding/agentic 用 xhigh 或 max。然后在自己的 eval 集上扫描各档位，测量质量、latency、cost，选择质量达标的最高效档位。注意新模型上不要沿用旧 effort 配置，要重新做 fresh sweep。

**问: Opus 5 在 xhigh/max 下禁用 thinking 为什么会返回 400？**

因为这些档位的核心能力来自扩展 thinking，禁用 thinking 与档位语义冲突，API 会直接拒绝。实现客户端时需要根据 effort 档位校验 thinking 配置；如果业务确实需要禁用 thinking，应降到 high 或以下档位。

## 适用场景

- 简单子代理或流水线节点：如发票字段提取、意图分类、格式转换，可设置 low/medium 降低 token 和延迟。
- 复杂 coding/agentic 任务：如调试仓库、长时间自主修复测试、多步工具调用，使用 xhigh 或 max 并配大 max_tokens。
- 成本敏感的大规模生产：批量文档处理、客服回复等，在 evals 验证质量后可把高 effort 降到 medium/low。
- 长对话中动态切换：例如先用 high 探索复杂问题，再切到 low 处理后续常规步骤，通过 per-message output_config 保留 prompt cache。

## 标签

`Claude API` `effort` `token 成本` `推理预算` `agentic 调优`
