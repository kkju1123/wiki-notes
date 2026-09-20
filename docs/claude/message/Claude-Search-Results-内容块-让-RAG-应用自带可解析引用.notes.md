# Claude Search Results 内容块：让 RAG 应用自带可解析引用

*原文: [https://platform.claude.com/docs/en/build-with-claude/search-results](https://platform.claude.com/docs/en/build-with-claude/search-results) · 来源: web · 生成时间: 2026-09-20T02:48:00.994792+00:00*

## 背景

RAG 应用普遍需要答案可溯源，但早期做法把检索文本拼进 prompt 再让模型标注来源，引用格式不稳定且难解析。Anthropic 在 Messages API 中引入原生 search_result 内容块，让自定义检索内容像网页搜索结果一样被自动引用。这样引用成为 API 的一等公民，而不是靠提示词约束的副产物。

## 痛点

没有原生引用机制时，模型可能复述来源但不给可定位出处，甚至输出自造链接；开发者需要额外用正则或模型解析引用，既耗 token 又容易在长上下文中丢失位置。检索结果混在普通文本里，也难以区分哪些内容可作为来源。

## 解决办法

把检索到的文档封装为 search_result 块，必须带 type、source、title、content 字段，其中 content 只能由多个 text block 组成。开启 citations.enabled 后，Claude 在生成回答时会在使用到这些来源的 text block 上附加引用元数据，包含来源、标题、被引文本和块级索引。引用按文本块定位：块是最小可引用单位，因此想细粒度引用就拆小块，想整段返回就合并。可以类比为带来源标注的结构化材料：模型只负责选择引用哪些材料，API 负责返回机器可读的位置信息。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()
resp = client.messages.create(
    model='claude-sonnet-4-5',
    max_tokens=1024,
    messages=[{
        'role': 'user',
        'content': [
            {
                'type': 'search_result',
                'source': 'kb://article-1234',
                'title': 'Rate Limits',
                'content': [
                    {'type': 'text', 'text': 'Free tier: 5 requests/min.'},
                    {'type': 'text', 'text': 'Pro tier: 100 requests/min.'},
                ],
                'citations': {'enabled': True},
            },
            {'type': 'text', 'text': 'What is the free tier limit?'},
        ],
    }],
)
print(resp.content)
```

这段代码演示 Method 2：直接在 user message 顶层放入 search_result 块。search_result 的 content 被拆成两个 text block，便于引用时精确到免费或付费档；citations.enabled 必须显式开启。调用后 resp.content 中的文本块可能带 citations 字段，其中 search_result_index 为 0，start 和 end block index 会指出具体引用了哪个 block。

## 关键流程

1. 准备检索到的文档，并将每份文档内容按段落或条目拆成若干 text block。
2. 构造 search_result 块，设置 type、source、title、content，并显式设置 citations.enabled 为 true。
3. 选择传递方式：动态检索用工具返回 search_result；预取或缓存内容可直接放在 user message 顶层。
4. 正常提问，不需要额外提示词；Claude 会在使用到这些来源时自动附带引用。
5. 解析响应中的 citations 字段，通过 search_result_index、start_block_index、end_block_index 和 cited_text 定位来源。
6. 如需更细粒度引用，将 search_result 内容拆成更小的 text block；如需整段引用则合并。

## 关键点

- search_result 必须包含 type、source、title、content 四个字段，且 content 只能是 text block 数组；这是 API 校验和后续引用的基础。
- 引用默认关闭，必须显式设置 citations.enabled=true；同一请求内所有 search_result 必须使用相同设置，否则会报验证错误。
- 文本块是引用的最小单位，模型只引用整块而非子串，因此分块粒度直接决定引用精度。
- cited_text 等于 content 数组从 start_block_index 到 end_block_index（不含 end）的拼接文本，且不占输出 token，能降低生成成本。
- search_result 可以来自工具调用或用户消息顶层；工具结果中一旦包含 search_result，所有块都必须是 search_result，而顶层内容可以与其他块混合。
- search_result_index 按请求中所有 search_result 出现的顺序统一计数，跨消息和工具结果，便于混合溯源。

## 对比与权衡

- 相比在 prompt 里拼接检索文本并要求模型输出引用格式，原生 search_result 的引用是固定 schema，稳定性和可解析性更好，但要求内容按 text block 预先结构化。
- 相比自建后处理引用定位（如用相似度把答案片段映射回文档），模型直接生成块级引用省去额外模型或算法调用，但引用粒度受限于分块策略。
- 相比 Anthropic 的 custom content documents 引用，search_result 更面向搜索/RAG 结果，增加 source/title 和跨工具与顶层的统一索引，更适合动态检索链路。

## 自测问题

**问: 为什么 citations 默认关闭？**

为了兼容已有行为并避免不必要的引用元数据。开发者需要显式开启，而且同一请求内所有 search_result 必须使用相同的 citations 设置，否则 API 会返回验证错误。

**问: cited_text 字段会占用输出 token 吗？它的内容怎么确定？**

不占用输出 token。它等于 search_result 的 content 数组从 start_block_index 到 end_block_index（不含 end）拼接后的全文。模型引用的是整个文本块，不是块内子串。

**问: 如果要让引用更精确，应该怎么设计 search_result 的 content？**

把每个 search_result 的内容按段落、条目或独立事实拆成多个 text block。块是最小可引用单元，拆得越细引用边界越准；但也不能过度切割，否则语义不完整或上下文碎片化。

**问: 工具返回 search_result 和用户消息顶层提供 search_result 有什么区别？**

工具返回适合动态 RAG，运行时才获取；顶层适合预取、缓存或测试。顶层的 user content 可以混合 search_result 与其他类型块，但 tool_result 中一旦出现 search_result，所有块都必须是 search_result，不能混普通文本。

**问: 多个 search_result 混合时 search_result_index 如何计算？**

按它们在整个请求中出现的顺序统一编号，0-based，不论来自 user message 还是 tool result。这样最终引用可以准确回溯到具体来源块。

## 适用场景

- 企业知识库问答：将内部文档作为 search_result 传入，回答中自动带 kb:// 来源和标题。
- 动态 RAG 工具：工具实时查询向量库或搜索 API，把 top-k 结果构造成 search_result 返回。
- 已有搜索服务结果增强：先由外部搜索系统返回结果，再交给 Claude 综合并引用。
- 开发与测试引用策略：用固定 search_result 验证分块粒度和引用质量。

## 标签

`RAG` `Claude API` `Citations` `Search Results` `引用溯源`
