# Task Budgets：为 Claude 的 Agentic Loop 设置 Token 预算

*原文: [https://platform.claude.com/docs/en/build-with-claude/task-budgets](https://platform.claude.com/docs/en/build-with-claude/task-budgets) · 来源: web · 生成时间: 2026-09-20T02:27:35.207574+00:00*

## 背景

Agentic 任务中，Claude 会在一个用户请求下进行多轮思考、工具调用和结果处理，成本与延迟难以事先估计。传统 max_tokens 只限制单次生成，无法给整个 loop 一个总预算。Task budgets 因此被设计成服务端注入的倒计时信号，让模型像盯时间盒一样主动分配剩余 token。

## 痛点

没有 task budget 时，开发者只能用 max_tokens 硬截断，模型可能在工具执行中途被切断，或者无法在预算快耗尽时总结收尾。客户端自己统计 loop 总 token 也很难实时反馈给模型，导致长任务成本不可控。

## 解决办法

在 output_config 中配置 task_budget，type 固定为 tokens，total 设置整个 agentic turn 的总预算。服务端会在后续请求中注入剩余 token 标记，模型看到倒计时后主动减少低价值工具调用并优先完成结论。预算只对模型新看到的内容计数，因此客户端重复发送历史不会重复扣费。若客户端自行压缩历史，可传 remaining 延续倒计时。生产上通常与 max_tokens 搭配：task_budget 是软导航，max_tokens 是硬顶。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

client.messages.create(
    model='claude-sonnet-4-5',
    max_tokens=4096,
    messages=[{'role': 'user', 'content': '审计这个仓库中不安全的 eval 使用并给出报告'}],
    output_config={
        'task_budget': {
            'type': 'tokens',
            'total': 100_000,
            # 'remaining': 81_000,  # 仅在自行压缩历史后续期时传入
        }
    },
    # 请同时按文档在请求头中携带对应的 beta 标识
)
```

示例配置了一个 100000 token 的 agentic loop 预算，type 固定为 tokens。首次请求省略 remaining，服务端默认从 total 开始。只有当客户端自行压缩历史时，才需要手动传 remaining；否则让服务端自动跟踪倒计时。max_tokens 仍保留作为单次响应的硬上限。

## 关键流程

1. 在 output_config 中配置 task_budget 的 type 和 total，并按文档携带 beta 请求头。
2. 保持同一个 agentic turn：把所有 tool_result 随消息回传，服务端会继续当前预算倒计时。
3. 如果客户端自行压缩或重写历史，计算被移除历史消耗的 tokens，并通过 remaining 延续预算。
4. 设置合理的 max_tokens 作为硬上限，避免软预算被突破时失控。
5. 如需中途调整预算，直接在新请求设置新的 task_budget，但注意 prompt cache 会因预算值变化而失效。

## 关键点

- task_budget 是软提示而非强制限制：模型通常会优雅收尾，但可能因完成当前动作而略微超出预算。
- 预算覆盖一个 agentic turn 中所有新 token，包括模型生成、工具调用和工具结果，但不包括上下文中已存在的历史 token。
- 客户端每轮重复发送完整历史会让 payload 变大，但预算只扣新内容，避免重复计数。
- 只有不带 tool_result 的用户消息会开启新 turn 并获得新预算；携带 tool_result 的消息继续当前 turn。
- remaining 是给自定义压缩场景续期用的：服务端不记得压缩前预算，客户端需要计算并传递。
- task_budget 与 max_tokens 搭配：前者给模型节奏目标，后者防失控生成。

## 对比与权衡

- 相比 max_tokens，task_budget 不会在动作中途生硬截断，而是引导模型在接近预算时总结收尾；但它是软提示，不能保证绝对不超。
- 相比 effort 参数，task_budget 控制整个 agentic loop 的 token 总量，effort 控制每步推理深度；两者是互补关系。
- 相比客户端自己统计 token 花费，task_budget 由服务端注入倒计时，模型能实时看到并自我调节，开发者不必把预算转化成复杂提示。
- 相比携带完整历史让服务端自动计数，remaining 适合客户端压缩历史时续期，但需要开发者准确计算被移除历史的 usage。

## 自测问题

**问: task_budget 和 max_tokens 有什么区别？**

task_budget 是咨询性总预算，跨多个请求，模型看到倒计时自我调节；max_tokens 是单次生成硬上限，达到即 stop_reason=max_tokens 截断。实际生产建议两者一起用。

**问: 什么情况下应该传 remaining？**

如果客户端每个请求都重发完整未压缩历史，由服务端跟踪，无需 remaining。如果客户端做了压缩或重写历史，服务端无法知道压缩前的消耗，需要计算被移除历史的 usage 并传 remaining，且每请求都传该值，不要逐请求递减。

**问: task_budget 到底计哪些 token？**

计模型新看到的内容：本轮模型生成的 thinking/text/tool_use、新出现的 tool_result。历史中已存在的消息不重复计；请求 payload 大小和缓存输入大小不等于预算消耗。

**问: 为什么说 budget 是 advisory 而不是 enforced？**

因为模型可能在执行工具动作中途，立即停止会破坏状态或产生更差结果，所以允许略微超额。真正硬顶由 max_tokens 执行。这样设计兼顾成本控制和任务完整性。

**问: 预算倒计时如何影响模型行为？**

服务端会在上下文中注入剩余 token 标记，模型看到标记后调整节奏，例如减少低价值工具调用、优先完成核心结论，在接近 0 前给出总结或进度报告。

## 适用场景

- 多工具调用的代码审计或安全扫描 agent，需要在一定 token 内给出结论。
- 有成本上限或延迟 SLA 的批量文档处理或研究任务。
- 希望模型在预算快耗尽时生成阶段性总结的交互式任务。
- 客户端会自定义压缩历史的长会话，需要 remaining 续期。

## 标签

`Claude API` `Agentic Loop` `Token Budget` `任务预算` `工具调用`
