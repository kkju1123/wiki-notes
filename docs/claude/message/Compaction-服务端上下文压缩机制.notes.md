# Compaction：服务端上下文压缩机制

*原文: [https://platform.claude.com/docs/en/build-with-claude/compaction](https://platform.claude.com/docs/en/build-with-claude/compaction) · 来源: web · 生成时间: 2026-09-20T02:59:19.705750+00:00*

## 背景

大模型上下文窗口有限，长对话与多轮工具调用会迅速耗尽 token；即使未超限，过长的上下文也会稀释模型注意力，导致响应质量下降、延迟与成本上升。为兼顾长期任务连续性和可控上下文规模，需要一种自动压缩历史上下文而不依赖客户端反复实现摘要逻辑的机制。Compaction 正是为此在服务端增加的自适应摘要能力。

## 痛点

没有自动压缩时，开发者要么在接近上限时手动截断或手动摘要，容易丢失关键状态、破坏多轮一致性；要么放任上下文增大，导致模型响应变差、成本与首 token 延迟上升。上下文超限还可能直接触发错误，使长期任务被迫中断。

## 解决办法

启用 threshold compaction 后，服务端在每个采样迭代开始时检查输入 token 是否达到 trigger 阈值；一旦触发，先额外调用模型对当前对话生成一段摘要，写为 assistant 响应开头的 compaction block，再基于压缩后的上下文继续生成。后续请求只需把包含该 block 的响应内容传回，API 会忽略 block 之前的所有旧内容，自动完成“历史折叠”。它类似于日志压缩或内存快照：不是简单丢弃，而是用模型把旧上下文压缩成紧凑且有信息量的表示，从而在不丢失关键任务状态的前提下缩小 active context。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

# 启用 threshold compaction
context_management = {
    'edits': [
        {
            'type': 'compact_20260112',
            'trigger': {'type': 'input_tokens', 'value': 150000},
            'pause_after_compaction': False,
        }
    ]
}

response = client.messages.create(
    model='claude-fable-5.1',
    max_tokens=1024,
    context_management=context_management,
    messages=[{'role': 'user', 'content': '继续处理这批日志...'}],
)

# 后续请求：追加整个 assistant content，API 会自动忽略 compaction 之前的历史
messages = [{'role': 'user', 'content': '继续处理这批日志...'}]
messages.append({'role': 'assistant', 'content': response.content})
client.messages.create(
    model='claude-fable-5.1',
    max_tokens=1024,
    context_management=context_management,
    messages=messages,
)

```

这段代码展示了启用 compaction 的核心用法：在 context_management.edits 中加入 compact_20260112 策略，并设置 input_tokens 触发阈值。触发后 response.content 中会包含 compaction block，后续请求将整个 assistant content 追加回消息列表。这样 API 在收到 compaction block 时会自动忽略其之前的旧内容，无需开发者手动裁剪，体现了“用摘要替代旧上下文”的原理。

## 关键流程

1. 在 Messages API 请求的 context_management.edits 中加入 type=compact_20260112 策略，启用压缩。
2. 通过 trigger.input_tokens 配置触发阈值，默认 150000、最低 50000；可选 pause_after_compaction 和 instructions。
3. 当输入 token 达到阈值时，API 在某个采样迭代开始前自动触发一次摘要采样，生成对话摘要。
4. API 在 assistant 响应开头返回 compaction block，并在需要时可能按 stop_reason=compaction 暂停。
5. 后续请求把整个 assistant content 追加回 messages，API 自动忽略 compaction block 之前的旧内容。
6. 若同一会话多次压缩，最后一个 compaction block 代表当前继续所需的状态。

## 关键点

- Compaction 是对旧上下文的语义化有损压缩，而不是机械截断；它用模型摘要保留继续任务所需信息，因此更适合需要强状态连续性的长对话。
- 压缩由 input_tokens 阈值触发且可能在一个请求内多次发生，因为每次采样迭代都会重新检查阈值。
- 后续请求必须把 compaction block 传回；传回后 API 忽略其之前的所有 content blocks，开发者可以手动删除旧消息，也可保留让 API 处理。
- 压缩会在正常生成之外增加一次采样迭代，因此会产生额外 input/output token 并计入限流和计费；可通过 usage.iterations 区分压缩迭代与消息迭代。
- 自定义 instructions 会完全替换默认摘要提示，不是追加；需要明确要求模型保留什么，尤其是此前 thinking blocks 通常不会跨越压缩边界。

## 对比与权衡

- 相比手动按 token 截断历史：Compaction 的摘要能保留更多语义和任务状态，但会额外消耗 token 并引入一次模型采样延迟。
- 相比客户端/框架层摘要（如自建 memory summary）：Compaction 集成在服务端，自动感知实际 token 数并生成标准 compaction block，减少客户端一致性问题；但可观测性和定制灵活性略弱于完全自管方案。
- 相比 on-demand compaction：threshold compaction 在发生阈值时自动压缩并继续生成，适合在线任务；on-demand 将摘要步骤独立出来、可后台运行，适合需要精确控制压缩时机或手动替换历史的场景。

## 自测问题

**问: compaction block 传回后，API 如何处理之前的内容？**

API 收到 compaction block 时会忽略其之前的所有 content blocks，只保留 compaction block 之后的内容作为有效上下文。可以选择手动删除旧消息或保留原消息由 API 处理；但要注意 thinking blocks 通常不会被带过压缩边界，摘要成为模型对早期工作的唯一记忆。

**问: compaction 和普通截断（truncate）有什么区别？**

截断直接丢弃最早 token，可能切断关键任务状态或多轮依赖；compaction 用模型额外采样生成摘要，是有选择的语义压缩，损失更多 token 和成本，但信息保留率和任务连贯性更高。可以扩展：压缩是有损的，摘要质量取决于默认或自定义 instruction。

**问: compaction 的 token 成本如何计算？为什么 usage 里出现 iterations？**

压缩本质是一次额外采样迭代，因此会贡献 input/output token 并计入限流和计费。usage.iterations 会分别列出 compaction iteration 和 message iteration；最终 message iteration 的 token 数反映压缩后有效上下文大小，可以和原始上下文对比看出压缩收益。

**问: 如何避免 compaction 导致 system prompt 缓存失效？**

在 system prompt 末尾添加 cache_control breakpoint，使 system prompt 与对话内容分开缓存。这样压缩产生新摘要写入新缓存条目时，system prompt 缓存仍有效，只需为摘要写新缓存，从而提高缓存命中率并降低成本/延迟。

**问: 自定义 instructions 与默认摘要提示是什么关系？**

自定义 instructions 会完全替换默认摘要提示，而不是在默认提示上追加。因此必须在自定义指令中明确要求模型保留哪些关键信息。某些模型上摘要只基于可见对话，不包含之前的 thinking blocks，所以更要在指令中说明保留策略。

## 适用场景

- 面向长期对话的客服/私人助理，需要在同一会话中持续数小时甚至数天并保持任务背景。
- 带大量工具调用的 Agent 任务，例如长链路代码调试、数据分析或自动化运维，容易超过默认上下文窗口。
- 有总 token 预算的长任务，结合 pause_after_compaction 与计数器，在达到预算前压缩并优雅收尾。
- 希望在长对话中保持响应质量、避免上下文过大导致延迟上升和注意力稀释的生产服务。

## 标签

`上下文管理` `Compaction` `Claude API` `长对话` `摘要压缩`
