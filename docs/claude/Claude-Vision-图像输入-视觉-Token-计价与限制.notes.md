# Claude Vision：图像输入、视觉 Token 计价与限制

*原文: [https://platform.claude.com/docs/en/build-with-claude/vision](https://platform.claude.com/docs/en/build-with-claude/vision) · 来源: web · 生成时间: 2026-09-20T06:09:23.925831+00:00*

## 背景

Claude 早期主要处理文本，但企业数据大量以扫描文档、截图、图表和照片存在。视觉能力让它进入多模态交互，能直接理解这些非结构化视觉信息。通过将图像切分成视觉 token，Claude 用同一个 Transformer 同时处理文本和图像，避免额外 OCR 或人工抽提。

## 痛点

没有视觉能力时，图像内容需要人工读取或额外 OCR 流水线，无法直接理解布局、图表和 UI 截图。多模态任务没有统一接口，开发和维护成本高。

## 解决办法

Claude 在 API 中用 image content block 接收图像，支持 base64、URL 和 Files API 的 file_id 三种来源。图像不会按文件大小计费，而是被切分为 28×28 像素的 patch，每个 patch 作为一个 visual token，最终按视觉 token 数计价。每个模型有最大长边和视觉 token 上限，超限会自动降采样，从而控制成本。多张图片可以在同一请求中联合分析，后续对话也能引用之前的图像。

## 关键代码示例

```python
import base64
import anthropic

client = anthropic.Anthropic()

with open("invoice.png", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

resp = client.messages.create(
    model="claude-haiku-4-5",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": image_data}},
            {"type": "text", "text": "提取发票的金额、日期和供应商。"}
        ]
    }]
)
print(resp.content[0].text)
```

这段代码先把本地图片读取并 base64 编码，再作为 image content block 的 source.data 发送；同一条消息里混用 text block 给出指令。模型收到后会把图像切分为视觉 token，与文本 token 一并处理。实际项目中，如果是公网图片可改用 source.type=url，如果是重复使用图片可先用 Files API 上传得到 file_id，避免每次请求都传 base64。

## 关键流程

1. 选择图像来源：一次性小图或本地图用 base64，公网图片用 URL，需要多次复用用 Files API 的 file_id。
2. 构造 messages 的 content 数组，将 image block 与 text block 混合，并按需给每张图加文本标签如 Image 1。
3. 发送前检查格式（JPEG/PNG/GIF/WebP）、大小（直连 API 10MB，Bedrock/Google Cloud 5MB）和数量限制。
4. 如果单请求超过 20 张图，提前把所有图片长边压到 2000px 以内，避免触发 many-image 的严格限制。
5. 用 ceil(width/28)*ceil(height/28) 估算视觉 token，必要时在客户端提前 resize 以控制成本和延迟。

## 关键点

- Claude 把图像切成 28×28 像素的 patch 作为 visual token，因此计费取决于像素分辨率，而不是文件大小。
- API 支持 base64、URL 和 Files API 三种图片来源，选择依据是请求体大小、图片可访问性和复用频率。
- 每个模型都有最大长边和视觉 token 上限，超限会自动降采样；但超过 20 张图的请求会触发更严格的 2000px 限制。
- 多图请求中 Claude 会联合分析所有图像，用 Image 1、Image 2 等文本标签可以在提示词和后续对话中准确引用。
- 高分辨率 tier 最多可能消耗约 3 倍视觉 token，只有截图、密集文档、computer use 等确实需要细节时才使用。
- 图像质量比盲目追求高分辨率更重要：清晰、文字可读、保留关键上下文，否则分辨率高也没用。

## 对比与权衡

- 相比 base64 内嵌，URL 引用能让请求体更小、避免编码开销，但要求图片公网可访问，且可能带来外链失效或隐私风险。
- 相比 URL，Files API 的 file_id 上传一次后可多次引用，适合重复使用同一批图片，但多了一步上传和管理 file_id。
- 相比标准分辨率 tier，高分辨率 tier 保留更多细节，适合密集文档和 UI 截图，但视觉 token 成本约增加 3 倍，延迟也更高。
- 相比传统“OCR + 文本 LLM”流水线，Claude Vision 端到端理解版式、图表和视觉上下文，减少组件间错误传播；但专用 OCR 在固定票据模板上可能更可预测，且调用大模型视觉接口成本通常更高。

## 自测问题

**问: Claude 的视觉 token 是怎么计算的？为什么图像不按文件大小计费？**

图像被切分成 28×28 像素的 patch，每个 patch 是一个 visual token，公式为 ceil(width/28)×ceil(height/28)。按文件大小计费不能准确反映模型实际计算量，因为压缩率、元数据会影响文件大小；按像素 patch 计费更接近编码后的视觉输入规模。

**问: API 发送图片有哪几种方式？实际该怎么选？**

有三种：base64 编码嵌入请求体、URL 引用在线图片、Files API 返回 file_id。base64 适合一次性或本地小图，URL 适合公网可访问的图片，file_id 适合需要重复引用同一图片或避免重复 base64 开销的场景。

**问: 如果一次请求要发 30 张图，会遇到什么限制？如何避免？**

API 超过 20 张图会触发 many-image 严格限制，所有 image block 都计入，包括历史轮次重发的图片和 tool_result 中的截图；每张图长边超过 2000px 可能被拒绝。解决方法是提前把图片 resize 到长边不超过 2000px，或把请求控制在 20 张以内。

**问: 高分辨率模型和标准模型在处理图像上有什么区别？**

高分辨率 tier 的最大长边为 2576px、视觉 token 上限 4784，标准 tier 是 1568px 和 1568。高分辨率自动生效，适合截图、computer use、密集文档，但费用和延迟更高；如果只做一般分类或问答，可以提前降采样到标准范围以节约成本。

**问: 在多轮对话中，前面发送过的图片还需要在新的 user turn 中重新发吗？**

在同一个会话中，模型能访问之前轮次的图片，后续提问不需要在新的 user turn 内容里重复添加图片。但从 API 层面，如果你每次都发送完整 messages 历史，之前图片块仍会在请求中并继续占用视觉 token；需要区分“会话内可引用”和“请求中重复计费”。

## 适用场景

- 扫描发票、合同、表单等文档的自动信息提取，避免单独接入 OCR 流水线。
- UI 自动化与 computer use 场景中分析回传截图，定位坐标并执行操作。
- 电商商品图、质检图等多图对比，例如找差异、识别瑕疵或判断相似性。
- 图表、架构图、白板照片等非文本信息的解读和总结。

## 标签

`Claude Vision` `多模态` `视觉 token` `图像 API` `成本优化`
