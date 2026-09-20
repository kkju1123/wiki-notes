# Claude PDF 支持：文本抽取与视觉理解

*原文: [https://platform.claude.com/docs/en/build-with-claude/pdf-support](https://platform.claude.com/docs/en/build-with-claude/pdf-support) · 来源: web · 生成时间: 2026-09-20T03:14:41.584442+00:00*

## 背景

PDF 是版式驱动格式，传统解析工具往往只能抽出文本流，难以还原表格、图表和版面语义。企业大量财报、合同、扫描件需要自动问答或结构化录入，纯规则解析成本高且脆弱。Claude 将每页 PDF 同时渲染为图像并提取文本，用多模态模型统一理解文档，使非结构化 PDF 变成可对话、可分析的数据源。

## 痛点

如果只使用普通 PDF 文本提取，模型看不到图表趋势、版面关系和图片信息，财务分析或合同审阅会丢失关键信号。请求超限或 Amazon Bedrock 上未启用 citations 时，可能只拿到纯文本结果，误以为模型没有视觉能力。不进行 token 估算还容易导致大文件处理失败或成本失控。

## 解决办法

核心机制不是把 PDF 当纯文本喂给模型，而是对每一页做两件事：渲染成图像、提取文本层，然后将同一页的文本和图像一并送入 Claude 的多模态上下文。这样模型既像人一样“看”图表、布局和扫描图片，又能精确引用文字；可类比为给既会读图又会读文字的助手同时提供打印稿和文字稿。传入方式有 URL、base64 和 Files API file_id 三种，生产环境推荐用 Files API 减小请求体积。优化上通过 prompt caching 缓存重复 PDF、Message Batches 批量处理、拆分大文件来控制成本和延迟。

## 关键代码示例

```python
import base64
from anthropic import Anthropic

client = Anthropic()

with open('report.pdf', 'rb') as f:
    pdf_data = base64.standard_b64encode(f.read()).decode()

response = client.messages.create(
    model='claude-sonnet-4-5',
    max_tokens=1024,
    messages=[{
        'role': 'user',
        'content': [
            {
                'type': 'document',
                'source': {
                    'type': 'base64',
                    'media_type': 'application/pdf',
                    'data': pdf_data,
                },
            },
            {
                'type': 'text',
                'text': '总结这份财务报告，并解释第2页的营收趋势图。',
            },
        ],
    }],
)

print(response.content[0].text)
```

这段代码用本地 PDF 的 base64 数据构造 document 内容块，并同时发出一条文字指令。document source 指定 media_type 为 application/pdf，Claude 会按页渲染图像并抽取文本，因此后续问题既能引用文字，也能解释图表趋势。生产中可把同一个 PDF 先传到 Files API，再用 file_id 替换 source，避免大 base64 增大请求体。

## 关键流程

1. 检查 PDF 合规：标准 PDF、无密码/加密，请求总负载不超过 32MB，页数不超过 600 页（低上下文时 100 页）。
2. 选择传输方式：在线文件用 URL，本地文件用 base64，需复用或减小请求时用 Files API 的 file_id。
3. 构造请求：把 PDF 内容放在文本提示之前，并明确要求分析图表、表格或提取结构化字段。
4. 模型处理：Claude 将每页渲染为图像并抽取文本，结合视觉和文字进行理解。
5. 优化与扩展：重复分析启用 prompt caching，大批量走 Message Batches，大文件拆分后再提交。

## 关键点

- PDF 支持的本质是视觉多模态：每页被渲染成图像，同时提取文本层，因此能同时理解文字和图表。这决定了它比纯文本抽取更适合财报、研报等图形密集文档。
- 请求限制是指整个 payload 的 32MB 和页数限制，不是单个 PDF 文件大小；base64 编码和附带文本都会计入，因此大文件应优先通过 Files API 引用。
- Amazon Bedrock Converse API 有模式陷阱：不启用 citations 会回退到仅文本提取的 Document Chat，必须开启引用才能进入视觉 PDF Chat。
- 成本由文本 token 和图像 token 两部分组成：每页文本约 1500–3000 token，图像按视觉计算；没有额外 PDF 费用，但视觉模式通常比纯文本贵数倍。
- 输入顺序和文档质量会显著影响效果：PDF 放前面、使用标准字体、页面方向正确、提示里使用 PDF 阅读器页码，能降低误解。
- 三种传入方式的取舍：URL 简单、base64 适合本地敏感文件、Files API 适合复用和避免编码开销，实际项目中应按数据来源和大小选择。

## 对比与权衡

- 相比 PyPDF2/pdfplumber 等传统文本抽取库，Claude PDF 在图表理解、版面语义和自然语言问答上更强，但在确定性结构化输出、离线部署和单位页成本上不如传统库。
- 相比纯文本的 Converse Document Chat 模式，Claude PDF Chat 视觉模式能分析图表和布局，但 token 消耗大约从每 3 页 1000 上升到 7000，成本与延迟更高。
- 相比 Amazon Bedrock InvokeModel API，Converse API 集成更统一、更容易上手，但强制要求开启 citations 才能视觉分析，灵活性不如 InvokeModel。
- 相比 URL 或 base64 直接内联 PDF，Files API 显著减小单次请求体积且可复用文件，但需要额外的上传步骤和文件生命周期管理。

## 自测问题

**问: Claude 为什么能看懂 PDF 里的图表？**

它并不是解析图表矢量数据，而是把每页渲染成图像，同时抽取该页文本层，将两者一起送入多模态模型。模型在视觉和文本之间建立对齐，所以能看到图表趋势、图例和坐标轴，并关联正文文字。扫描件没有文本层时主要依赖视觉，但手写或低清图仍受视觉限制。

**问: 为什么我用 Amazon Bedrock 的 Converse API 看不到 PDF 里的图片/图表？**

先检查是否启用了 citations。Converse API 有两种模式：未启用引用时是 Document Chat，仅文本提取；启用引用后才进入 Claude PDF Chat，每页按文本加图像理解。若需要无引用的视觉分析，可改用 InvokeModel API，它对 PDF 处理模式有完全控制。

**问: 如何估算处理一个 PDF 的成本？**

成本来自文本和图像两部分。文本每页约 1500–3000 token，取决于页密度；图像按视觉模型计价，取决于页面尺寸和细节。可以先用 token counting 跑样例估计，再乘以页数；重复分析可以用 prompt caching 节省输入 token 成本。

**问: 大 PDF 比如 200MB 或 800 页怎么处理？**

先看平台 request size 和页数限制，超过就必须拆分。可以按章节或页区间切分，分别传入；若需要跨文档问答，先对每段抽取摘要或结构化字段，再二次汇总。高吞吐场景用 Message Batches 异步批处理，重复部分用 Files API 加 prompt caching。

**问: URL、base64、Files API 三种方式在工程上怎么选？**

URL 最简单，适合可公开访问且无需保密的文档，但依赖对方可达性和稳定性；base64 适合本地或内网文件，但会增加约 33% 体积并计入 request payload；Files API 适合大文件、复用文件和长期缓存，但多一步上传与 file_id 管理。

## 适用场景

- 财务报告与研报分析：对多页 PDF 提问，解读营收趋势图、利润率变化和表格数据。
- 合同/法律文档审阅：从长合同中抽取关键条款、日期、金额、违约责任并汇总成结构化表格。
- 扫描件/发票信息录入：对扫描 PDF 做视觉识别和字段提取，输出 JSON 或表格给下游系统。
- 多语言文档翻译与摘要：将 PDF 内容翻译成目标语言，或按章节生成摘要与要点。

## 标签

`Claude API` `PDF处理` `多模态` `文档理解` `Amazon Bedrock`
