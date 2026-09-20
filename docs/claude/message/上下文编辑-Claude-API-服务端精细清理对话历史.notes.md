# 上下文编辑：Claude API 服务端精细清理对话历史

*原文: [https://platform.claude.com/docs/en/build-with-claude/context-editing](https://platform.claude.com/docs/en/build-with-claude/context-editing) · 来源: web · 生成时间: 2026-09-20T03:01:12.428003+00:00*

## 背景

随着 agent 工作流大量使用工具和长对话增长，上下文不仅是成本问题，更会稀释模型注意力。传统做法常在客户端做截断或摘要，但容易破坏缓存、丢失推理链或引入客户端状态同步问题。Context editing 将清理能力下沉到服务端，提供结构化、可配置的精细管理。

## 痛点

没有 context editing 时，开发者往往只能手动截断消息、客户端压缩或重写历史。这类操作可能清掉有决策价值的工具调用、使 prompt cache 失效，甚至在 extended thinking 场景导致后续思维块校验失败，加大工程复杂度和出错风险。

## 解决办法

服务端策略在 prompt 进入模型前自动执行：客户端继续保存完整历史，无需同步编辑结果。工具结果清理按阈值触发，按时间顺序清除最旧工具结果，可保留最近 N 对工具交互，并替换为占位文本；思考块清理可指定保留最近若干思考轮次以维持推理连续性。两者可以组合，达到类似 GC 的分代淘汰效果——只淘汰低价值旧对象，保护新鲜、关键上下文。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model='claude-sonnet-4-6',
    max_tokens=1024,
    messages=full_history,  # 客户端仍保留完整历史
    context_management={
        'strategies': [
            {
                'type': 'clear_tool_uses_20250919',
                'trigger': {'type': 'input_tokens', 'value': 100_000},
                'keep': 3,
                'clear_at_least': 10_000,
                'clear_tool_inputs': False,
                'exclude_tools': ['read_file'],
            },
            {
                'type': 'clear_thinking_20251015',
                'keep': {'type': 'thinking_turns', 'value': 2},
            },
        ]
    },
)

print(response.context_management)
```

这段代码在 Messages API 请求中声明两条服务端清理策略。工具结果策略在输入超过 100k token 时触发，保留最近 3 组工具交互，至少清除 10k token 以抵消缓存失效成本，并排除 read_file 的结果；思考块策略只保留最近 2 个含思考块的 assistant turn。客户端传入的 messages 仍是完整历史，服务端会在模型看到前自动清理，最终可从 response.context_management 查看实际清理统计。

## 关键流程

1. 判断是否属于工具密集型或长思考链路，观察 token 增长和模型回答质量是否下降。
2. 在 Messages API 请求中增加 context_management.strategies，按需启用 clear_tool_uses 和 clear_thinking。
3. 根据场景设置 trigger、keep、clear_at_least、exclude_tools、clear_tool_inputs 等参数。
4. 结合 prompt caching 调整 keep 和 clear_at_least，在空间回收与缓存命中率之间取得平衡。
5. 从响应的 context_management 字段观察实际清除内容和 token 统计，迭代阈值。

## 关键点

- 服务端编辑对客户端透明：客户端始终维护完整历史，避免双端状态同步和前缀改动问题。
- 工具结果清理默认只清除结果、保留工具调用输入，因为调用参数反映 agent 决策意图，而结果往往是用后即弃的大文本。
- clear_at_least 是缓存经济学开关：只有一次清足够多 token 才值得承受缓存写入成本，避免频繁小规模清理反复使缓存失效。
- thinking block clearing 的 keep 参数直接影响 prompt cache：保留更多思考块可提高缓存命中，但会占用更多上下文窗口。
- 两种策略可以组合使用，处理“旧工具结果堆积 + 长思考链条”的混合场景。
- 响应中的 context_management 提供清理统计，让策略效果可观测、可调优，而不是黑盒操作。

## 对比与权衡

- 相比客户端 SDK compaction（摘要替换历史），服务端策略不改变客户端保存的完整历史，也不容易破坏后续思维块的前缀连续性；但客户端摘要更灵活，可以压缩任意语义信息，而服务端策略目前主要针对工具结果和思考块。
- 相比传统按 token 截断或丢弃最早消息，context editing 可以精确到工具 use/result 和 thinking block 边界，不会把系统指令、函数定义或最近的决策链一起截掉。
- 工具结果清理默认 clear_tool_inputs=false 比清除完整工具交互更保守：节省的 token 较少，但保留 agent 为什么调用工具的可见性；设为 true 则更省空间，但可能损失推理链。

## 自测问题

**问: Context editing 和普通上下文压缩/摘要有什么本质区别？**

普通压缩多发生在客户端，生成摘要并替换历史；context editing 是服务端在 prompt 到达模型前按策略清除特定块，客户端状态无需同步。它更结构化，针对工具结果和思考块；压缩则是语义级有损，可处理更广泛内容，但更难保留精确边界和缓存。

**问: 为什么工具结果清理默认只清 result 不清 tool_inputs？**

tool_inputs 记录了模型调用工具的意图和参数，对后续推理有价值；tool result 常包含大段文件内容、搜索结果，读完即可丢弃。设 clear_tool_inputs=true 可以进一步降本，但可能让模型看不到“我当时为什么调用这个工具”。

**问: clear_at_least 与 prompt caching 是什么关系？**

清理会使已缓存前缀失效，重新写缓存有成本。若每次只清少量 token，空间收益可能低于缓存重写成本，所以设置 clear_at_least 强制每次清够量，使失效决策值得；后续请求可复用新前缀。

**问: extended thinking 下为什么 thinking block clearing 比较特殊？**

思考块既是 token 大头，又承载推理连续性。清除太激进会丢失模型思路；保留太多会占用窗口。keep 可以按最近 N 轮保留，平衡推理质量与空间。服务端编辑通常不会破坏后续思维块有效性，而客户端手动改历史则可能导致后续思维块失效。

**问: 如何选择 trigger 阈值？**

观察应用 token 分布和缓存命中率。阈值过低会频繁清理，缓存反复失效；过高则窗口压力大。可以从默认 100k token 或工具调用次数起步，再结合响应 context_management 统计和成本分析调整。

## 适用场景

- 长时间运行的 coding/agent 应用，大量 read_file、grep、web_search 旧结果堆积。
- 启用 extended thinking 的复杂推理任务，需要清理旧思考块又保留最近推理链。
- 多轮 RAG、客服或研究 agent，检索结果只对当前步骤有用。
- 对 prompt cache 命中率和上下文成本敏感的高并发生产服务。

## 标签

`上下文工程` `Context Editing` `Claude API` `Prompt Caching` `Agent`
