# Claude Citations：为生成式回答提供可验证的来源引用

*原文: [https://platform.claude.com/docs/en/build-with-claude/citations](https://platform.claude.com/docs/en/build-with-claude/citations) · 来源: web · 生成时间: 2026-09-20T02:31:29.973820+00:00*

## 背景

大语言模型在回答企业文档、法律、金融等场景的问题时容易产生看似合理但无法溯源的幻觉。传统 RAG 通常只能返回粗略的检索片段，无法把具体论断与原文证据精确绑定。Citations 功能为了解决这个信任和可审计性问题，将引用作为模型输出的一等公民提供给开发者。

## 痛点

没有该功能时，用户看到的答案缺少逐句来源，研发团队需要自行实现引用解析、文本对齐和位置追踪。模型一旦出错或幻觉，很难快速定位到原文验证，影响企业级应用的可信度与合规性。

## 解决办法

在请求的 document 对象上设置 citations.enabled=true 即可启用。Claude 会先把源文档按句子等粒度切分，然后在回答中返回多个 text block，每个 block 可包含 claim 及其 citations 列表。引用格式随文档类型变化：纯文本返回字符索引，PDF 返回页码，自定义内容返回块索引，均为可程序化定位的精确范围。可以类比学术论文中的自动脚注：模型每给出一个判断，就附带可回溯的原文出处。该设计还把 cited_text 作为便利字段，不计入 token，控制成本。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model='claude-sonnet-4-5',
    max_tokens=1024,
    documents=[
        {
            'type': 'text',
            'text': 'The model uses citations to reference source documents exactly.',
            'citations': {'enabled': True},
        }
    ],
    messages=[
        {'role': 'user', 'content': 'What does the text say about citations?'}
    ],
)

for block in response.content:
    if block.type == 'text':
        print(block.text)
        for citation in block.citations:
            print(citation.cited_text, citation.document_index,
                  citation.start_character_index, citation.end_character_index)
```

示例通过 documents 参数传入一个纯文本文档，并在文档对象上启用 citations。回复的每个 text content block 会携带 citations 列表，每个 citation 提供 cited_text、document_index 和 0 起始的字符索引范围（左闭右开）。这种结构与 API 文档中的 plain text citation 格式对应。

## 关键流程

1. 在请求的 documents 中提供 PDF、纯文本或自定义内容文档，并为每个文档设置 citations.enabled=true。
2. Claude 对文档内容进行分块：纯文本和 PDF 按句子分块，自定义内容按提供的 content block 原样使用。
3. 模型生成回答并返回多个 text block，每个 text block 中可包含引用列表，标明支撑该论断的原文位置。
4. 按文档类型解析 citation 的索引：纯文本用字符范围，PDF 用页码范围，自定义内容用内容块索引范围。
5. 流式场景下监听 content_block_delta 中的 citations_delta 增量，将每条 citation 追加到当前 text block 的 citations 列表。

## 关键点

- 引用必须对请求内所有文档统一开启或关闭，不能只对部分文档启用；这避免了混合行为导致引用范围不明确。
- 只有文档的 source 内容可作为引用来源，title 和 context 仅作为元数据传给模型，不参与引用；这确保引用指向真实正文而非附加信息。
- 纯文本、PDF 和自定义内容三种文档类型返回不同的引用格式，开发者必须根据类型解析索引，否则无法定位原文。
- cited_text 字段不计入输出 token，后续回传也不计入输入 token，使得引用功能在成本上非常高效。
- Citations 与 prompt caching、token counting、batch processing 兼容；引用块本身不能缓存，但源文档内容块可以加 cache_control 缓存。
- 流式响应中引用以 citations_delta 增量出现，前端需要与文本增量一起处理，以支持实时渲染引用标记。

## 对比与权衡

- 相比传统 RAG 返回粗粒度检索片段列表，Claude Citations 提供 claim 级别的精确原文引用，验证更直接；但仅限 Claude API 生态，自定义检索流程的灵活性不如开源方案。
- 相比让模型在文中以非结构化形式写“来源：[1]”，结构化 citations 更适合程序化验证、展示和下游审计；但需要适配 Anthropic 的响应结构和流式事件。

## 自测问题

**问: Citations 如何处理扫描版 PDF？**

扫描版 PDF 没有可提取的文本层，无法按句子分块，因此不可引用；需要先经过 OCR 转成带文本层的 PDF 或改用纯文本/自定义内容文档。图像引用目前也不支持。

**问: 文档里的 title 和 context 为什么不能作为引用来源？**

title 和 context 是可选元数据字段，会传给模型但不会用于生成 cited_text 或位置索引，只有 source 正文内容可被引用；这样避免引用到外部元数据造成误导。

**问: 如何在启用 prompt caching 的同时使用 citations？**

引用块本身不能直接缓存，但源文档内容块可以使用 cache_control 缓存；在后续请求复用同一文档时命中缓存，从而降低输入 token 成本。

**问: plain text 的字符索引与 PDF 页码索引有什么区别？**

plain text 返回 0 起点的字符索引范围且结束位置 exclusive；PDF 返回 1 起点的页码范围且结束页 exclusive；解析时要分别处理，尤其字符索引要按半开区间切片。

**问: 流式接口中 citations_delta 表示什么？**

流式响应中每个 citations_delta 携带一条 citation，需要追加到当前 text content block 的 citations 列表，而不是等待完整响应；这与 text_delta 增量机制类似。

## 适用场景

- 企业知识库问答：答案自动附带出处，方便人工或自动审计。
- 法律/合规文档审查：对合同、政策条款的每个解读都可回溯到原文位置。
- 金融研究报告：模型生成的结论能精确关联到底层数据或报告段落。
- RAG 产品：在检索增强生成流程中，把搜索结果以可引用文档块的形式返回给前端展示来源。

## 标签

`Claude API` `Citations` `RAG` `可解释性` `文档处理`
