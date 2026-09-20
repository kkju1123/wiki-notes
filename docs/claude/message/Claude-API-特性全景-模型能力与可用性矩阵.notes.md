# Claude API 特性全景：模型能力与可用性矩阵

*原文: [https://platform.claude.com/docs/en/build-with-claude/overview](https://platform.claude.com/docs/en/build-with-claude/overview) · 来源: web · 生成时间: 2026-09-20T02:15:14.694857+00:00*

## 背景

早期大模型 API 主要提供单次文本补全，企业落地还需要长文档处理、可控推理、严格输出格式、来源引用、成本优化与多云合规。Anthropic 把这些能力抽象为五大领域，并用统一特性矩阵说明每个能力在 Claude API、Bedrock、Vertex AI、Foundry 等平台上的阶段与可用性，方便开发者按场景选型。

## 痛点

如果只关注模型效果而忽视特性阶段，很容易在生产中误用 Beta 或已弃用能力；分不清 ZDR 会让数据合规评审失败；不理解 thinking/effort 会浪费 token 或导致复杂任务推理不足。多平台部署时，不同云平台特性上线节奏不同，选错平台可能造成架构返工。

## 解决办法

Claude 将 API 能力组织为五个区域：模型能力、工具、工具基础设施、上下文管理和文件资产。模型能力内部的核心思路是两类控制：一是“想多少”，通过 adaptive thinking + effort 让模型动态决定思考深度，生成思维链 token 提升复杂推理；二是“输出成什么样”，通过 structured outputs 做 schema 约束，citations/search results 锚定来源，batch 用异步换成本，fallback 在模型拒绝时按链重试。选型时还要看 Availability 阶段和 ZDR 标记：无标签表示稳定生产可用，Beta 只适合探索，Deprecated/Retired 要迁移或禁用。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

resp = client.messages.create(
    model='claude-4-7-sonnet',
    max_tokens=4000,
    # adaptive thinking：模型动态决定何时思考、思考多少
    thinking={'type': 'adaptive', 'effort': 'high'},
    messages=[{
        'role': 'user',
        'content': '审查这份合同，找出三个主要风险点并给出理由。',
    }],
)

for block in resp.content:
    if block.type == 'thinking':
        print('[思考]', block.thinking[:200])
    elif block.type == 'text':
        print('[答案]', block.text)
```

这段代码开启 adaptive thinking，让模型在复杂任务上投入更多推理，而不是简单请求也固定消耗大量思考 token；effort=high 是宏观深度控制。响应中的 thinking block 是透明的逐步推理过程，text block 是最终答案，这与表中 Thinking/Adaptive thinking 的能力对应。若下游要求可解析结构，可在同一请求中叠加 structured outputs。

## 关键点

- 五大能力域把 Claude API 从“模型怎么想”到“模型怎么用数据/工具”拆成完整栈，复习时要能举例说明每个域解决什么问题。
- Availability 阶段决定生产可用性：平台名后无标签才是 stable，Beta 不保证持续生产，Deprecated 要迁移，Retired 已下线，选型前必须查表。
- ZDR 零数据保留不是默认适用所有能力，Batch 和 fallback 因异步存储/跨模型重试等机制不具备 ZDR 资格或受限，合规场景要单独评估。
- Adaptive thinking 与 effort 是控制推理成本和深度的关键旋钮：模型自己决定何时思考，effort 控制总体 token 投入，比固定 budget_tokens 更易用。
- Structured outputs 通过受控生成保证 schema 一致，避免下游解析失败；Citations/Search results 则解决 RAG 可验证性。
- 多平台可用性矩阵是 Claude 企业落地的重要优势，但同一特性在不同云平台阶段不同，不能只按一个平台假设所有平台可用。

## 对比与权衡

- 相比 OpenAI 的 reasoning_effort 手动三档控制，Claude adaptive thinking 可让模型动态决定思考长度，简单问题更省 token，但思考上界和可预测性不如显式 budget_tokens 直观。
- 相比 OpenAI 的 Structured Outputs 主要聚焦 JSON schema，Claude 提供 JSON outputs 与 strict tool use 两条路径，工具输入校验更自然，但跨平台 ZDR 资格更复杂。
- 相比只绑定单一云厂商的闭源模型服务，Claude 的多平台支持便于在 AWS Bedrock、Vertex AI、Foundry 之间做合规与成本权衡，但特性上线节奏不一致，需要持续核对 availability。

## 自测问题

**问: Claude API 的五大能力域分别是什么？**

模型能力控制推理和输出格式；工具让模型在环境或 Web 上执行动作；工具基础设施负责大规模发现和编排；上下文管理优化长时间会话；文件资产管理文档和数据输入。可以举例：模型能力选 thinking/effort，工具选 function calling，上下文管理选 caching。

**问: Beta、Deprecated、Retired 有什么区别？**

Beta 用于收集反馈，可能变化或下架，不保证生产；Deprecated 仍可用但官方不再推荐，应尽快迁移；Retired 已经不可用。面试时可强调生产选型只选无标签的 stable 能力。

**问: ZDR 是什么？为什么 Batch 和 fallback 不是 ZDR eligible？**

ZDR 指零数据保留，模型请求/响应不会被保留用于训练或改进模型等目的。Batch 异步处理需要在队列中短暂保存数据和结果，fallback 需要在一次调用内把同一请求重试到不同模型，存在跨模型处理保留，因此默认不符合或需要额外条件。

**问: Adaptive thinking 和传统 thinking/budget_tokens 有什么区别？**

传统 extended thinking 通常要求开发者为每次请求设定思考 token 预算，预算太高浪费、太低复杂任务不够；adaptive thinking 让模型根据问题动态决定何时思考、思考多少，effort 提供宏观控制，减少微调成本。

**问: Structured outputs 的 JSON outputs 和 strict tool use 该怎么选？**

如果目标是让模型输出可直接解析的 JSON 数据，用 JSON outputs；如果目标是让模型调用工具并且工具参数必须符合 schema，用 strict tool use，它既约束输入格式又能减少无效工具调用。

## 适用场景

- 长文档/大型代码库分析：利用最高 1M token 上下文窗口和 PDF support，适合合同审查、代码审计等场景。
- 复杂推理任务：在数学、策略分析、风险判断等场景开启 adaptive thinking 并提高 effort，换得更深推理。
- 需要可验证来源的 RAG 问答：使用 Citations 或 Search results 让回答引用精确句子，适合金融、医疗、法律等强合规场景。
- 大规模离线数据处理：Batch processing 将大量标注、分类、摘要请求异步执行，成本降低 50%，适合非实时任务。

## 标签

`Claude API` `模型能力` `思考模式` `结构化输出` `零数据保留`
