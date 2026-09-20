# 上下文窗口：Token 计数、思考块与工具调用的长对话管理

*原文: [https://platform.claude.com/docs/en/build-with-claude/context-windows](https://platform.claude.com/docs/en/build-with-claude/context-windows) · 来源: web · 生成时间: 2026-09-20T02:57:11.170163+00:00*

## 背景

LLM 本身是无状态的，多轮对话必须把历史显式放进请求，上下文窗口就是模型每次生成时的“工作记忆”。随着 Agent、工具调用和长文档任务普及，上下文窗口大小、计费与检索质量成为工程核心。

## 痛点

如果开发者不清楚窗口计数规则，会频繁遇到 token 超限、工具结果回传断裂，或把历史思考块反复计费造成成本浪费。更隐蔽的问题是长上下文里无关内容太多，导致模型对关键信息 recall 下降。

## 解决办法

上下文窗口的输入侧包括 system prompt、messages 中的所有文本/图片/文档/工具结果以及工具定义；输出侧包括本次生成的文本和 thinking tokens。Thinking tokens 是 max_tokens 的子集，按输出计费，多轮保留策略因模型而异：旧模型默认剥离，较新模型默认保留。工具调用时，要原样回传 assistant 内容，并把 tool_result 紧跟其后，这样才能保持推理链条。实际管理上下文可以把它类比成工作台：训练语料是图书馆，上下文窗口是桌面，token 是桌面面积；不是桌子越大越好，桌面堆放无关材料会降低效率，因此要用 usage 监控、prompt caching 和 compaction 来主动策展。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()
SYSTEM = 'You are a precise assistant.'
TOOLS = [{'name': 'search', 'description': 'Search the web', 'input_schema': {'type': 'object', 'properties': {}}}]
messages = []

def do_search(args): return f'results for {args}'

def compact(msgs): return msgs[-20:]

for text in user_turns():
    est = client.messages.count_tokens(model='claude-sonnet-4-5', system=SYSTEM, messages=messages, tools=TOOLS).input_tokens
    if est > 180_000:
        messages = compact(messages)
    messages.append({'role': 'user', 'content': text})
    resp = client.messages.create(model='claude-sonnet-4-5', max_tokens=1024, system=SYSTEM, messages=messages, tools=TOOLS)
    print(resp.usage)
    messages.append({'role': 'assistant', 'content': resp.content})
    for block in resp.content:
        if block.type == 'tool_use':
            result = do_search(block.input)
            messages.append({'role': 'user', 'content': [{'type': 'tool_result', 'tool_use_id': block.id, 'content': result}]})
```

这段代码先估算当前历史 token，超过阈值触发裁剪；再发起请求并解析 usage。关键是 assistant 消息完整追加回 messages，若模型请求工具，则下一条 user 消息携带与 tool_use_id 对应的 tool_result，这样 tool use 的上下文链不会断。

## 关键流程

1. 先确认所用模型的窗口大小和单次 max_tokens 上限，例如部分模型 1M、部分 200k。
2. 发送前用 token counting API 估算当前请求的 input_tokens，而不是凭字符数猜。
3. 收到响应后解析 usage 字段，区分 input_tokens、output_tokens、cache_read_input_tokens 和 cache_creation_input_tokens。
4. 如果 assistant 响应包含 tool_use，要把完整的 assistant 消息原样保留，并在下一条 user 消息中返回对应的 tool_result。
5. 当估算值接近窗口阈值时，优先使用 server-side compaction 或裁剪旧轮次，避免直接截断关键上下文。

## 关键点

- 上下文窗口是模型当前的工作记忆，不等于训练语料；更大窗口不自动带来更好效果，token 增长还会带来 context rot。
- 所有请求组成部分都会计入窗口：system prompt、messages 中的文本/工具结果/图片/文档、工具定义，以及本轮输出和 thinking tokens。
- Thinking tokens 既是 max_tokens 的一部分，也按输出 token 计费；多轮中是否保留由模型默认行为决定，并可通过 thinking block clearing 覆盖。
- 工具调用要求把 assistant 回复原样回传，tool_result 与对应的 tool_use 保持关联；在需要时，thinking block 必须随 tool_result 一起返回。
- prompt caching 会把 input tokens 拆分为 cache_read 和 cache_creation，三者都计入窗口，但缓存命中可以显著降低成本。
- 长对话和 Agent 场景的首选上下文管理策略是 server-side compaction，而不是无限放大窗口或简单首条丢弃。

## 对比与权衡

- 相比训练语料，上下文窗口容量小但可精确控制，它决定模型本次生成能直接看到什么。
- 相比简单 FIFO 滚动丢弃（常见于聊天界面），server-side compaction 会压缩/总结旧内容以保留关键决策，但可能牺牲细节和引入摘要偏差。
- 相比旧模型默认剥离 thinking blocks，较新模型默认保留思考块可让多轮推理更连贯，但后续轮次会把这些思考块作为输入 token 再次计费。
- 相比 200k 窗口模型，1M 窗口能容纳更多长文档，但延迟、成本和 context rot 风险也更高，仍需内容策展。

## 自测问题

**问: 上下文窗口里哪些内容会被计数？**

按输入侧和输出侧拆开讲：输入侧包括 system prompt、messages 中所有文本、图片、文档、工具结果和工具定义；输出侧包括文本和 thinking tokens。prompt caching 的 cache_read/cache_creation 也属于 input tokens，全部占用窗口。

**问: extended thinking 在多轮中会保留吗？**

要分模型：Opus 4.5+、Sonnet 4.6+ 等新模型默认保留，之前思考块进入后续输入并按输入 token 计费；旧模型和 Haiku 默认自动剥离。面试官想听到你理解默认行为差异以及用 thinking block clearing 覆盖。

**问: 为什么 tool use 时要特别注意 thinking block 和 tool_result 的顺序？**

因为工具调用依赖模型生成 tool_use 前的推理，丢失思考块会破坏因果链；API 要求把 assistant 消息原样保留，tool_result 紧跟其后，否则可能报错或模型行为异常。

**问: 上下文窗口越大，效果一定越好吗？**

不一定。要提到 context rot：token 增长会导致 recall 和准确性下降、延迟和成本上升，因此关键是让窗口里的内容更相关，而不是塞得更多。

**问: 多轮 agent 如何防止撑爆上下文？**

先 token counting 估算，再用 usage 监控；对重复前缀使用 prompt caching 降低成本；快触顶时用 server-side compaction 压缩，必要时按优先级裁剪工具结果或旧轮次，而不是无限追加。

## 适用场景

- 多轮对话/客服机器人：需要在保留上下文和接近窗口上限之间平衡。
- Agent 工具调用流水线：工具定义、工具结果和 thinking blocks 会快速占满上下文，需要显式管理。
- 长文档分析：处理大量 PDF/图片时需理解 1M 窗口与请求大小限制。
- 成本优化：通过 usage 字段和 prompt caching 监控长期运行的会话成本。

## 标签

`上下文窗口` `Claude API` `Token 管理` `Prompt Caching` `Agent`
