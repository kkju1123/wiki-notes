# DeepSeek Flash 图像理解与多模态输入实践

*原文: [https://api-docs.deepseek.com/zh-cn/guides/vision](https://api-docs.deepseek.com/zh-cn/guides/vision) · 来源: web · 生成时间: 2026-09-17T01:54:21.583480+00:00*

## 背景

多模态大模型通常由视觉编码器和语言模型组成，视觉编码器先把图像转换成与文本 token 对齐的视觉 token，再与文本一起进入 Transformer，因此 API 需要一种结构化方式同时传递文本和图片。DeepSeek 选择兼容 OpenAI Chat Completions/Responses API 与 Anthropic Messages API，让现有 SDK 和工具链可以直接切换 base_url 使用。旧视觉模型 deepseek-v4-flash-vision-exp 已下线，由最新的 deepseek-flash 统一承接图片请求。

## 痛点

不使用图片输入，模型就无法处理截图、图表、票据等非文本信息；如果用错传图方式，可能遇到请求体超 48MiB、图片下载超时、重复上传无法复用、token 成本不可预测等问题。

## 解决办法

图片作为 content 块数组传入，核心是 OpenAI 兼容的 image_url 或 file 块。本地小图用 base64 data URL 内联，最简单但计入请求体大小；可公开访问的图片用外部 http(s) URL，服务端自动下载，但有 URL 长度、文件大小和下载时间限制；需要复用或突破内联限制时，先通过 Files API 上传获得 file_id，再在请求中引用。图片进入模型前会被等比缩放：小于约 544×544 的放大，大于约 1300×1300 的缩小，标准尺寸下每张图 token 上限为 1024，这样控制视觉 token 数量类似于限制图片的“字数”。detail 字段可控制推理前是否压到 512×512，low 更快更省，high/original 保留原图。

## 关键代码示例

```python
import base64
from openai import OpenAI

client = OpenAI(api_key="<DeepSeek API Key>", base_url="https://api.deepseek.com")

with open("image.jpg", "rb") as f:
    b64 = base64.b64encode(f.read()).decode("utf-8")

response = client.chat.completions.create(
    model="deepseek-flash",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "这张图片里有什么？"},
                {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{b64}"}},
            ],
        }
    ],
)

print(response.choices[0].message.content)

```

代码先用 base64 把本地图片二进制编码为文本，使它能被嵌入 JSON 请求体；content 必须是块数组而非纯字符串，text 与 image_url 子块分别表达文本和图片，这是 OpenAI 兼容多模态协议的核心；image_url.url 使用 data URL 内联携带图片，服务端根据实际文件内容判断格式。

## 关键流程

1. 判断图片大小和访问方式：本地小图、一次性使用，选 base64 data URL 内联。
2. 若图片可公开访问且 ≤32MiB、URL 长度 ≤8192 字符，选外部 http(s) URL。
3. 若图片 >32MiB、单请求可能超过 48MiB，或需要在多个请求中复用同一张图，先通过 Files API 上传获得 file_id，再用 file 块引用。

## 关键点

- Chat Completions 的 content 必须改成块数组，而不能直接传字符串；这是多模态消息协议的核心，字符串只是纯文本快捷方式，传图时必须显式声明 image_url 或 file 子块。
- 图片格式按文件实际内容判断，而不是看扩展名或 MIME 声明；这避免扩展名伪装导致服务端解析失败或安全问题。
- base64 内联最简单，但 base64 会增加约 1/3 体积并计入 48MiB 请求体；外部 URL 不占请求体但依赖公网可达和 60 秒下载超时；Files API 可复用并支持 64MiB 大图，是工程上的最优解。
- 模型推理前会将图片等比缩放至约 544×544 到 1300×1300 总像素区间，单张图 token 上限 1024；这保证成本、延迟和上下文长度可预测，超高分辨率不会额外增加 token。
- detail=low 会先把图片缩到 512×512，适合粗粒度视觉判断，更快更省 token；high/original 当前等价于保留原图，与 OpenAI 的分块 high 策略并不相同。
- Chat Completions 中图片只允许出现在 user 消息，system/assistant 携带图片会返回 400；但 Responses API 的 input_image 可出现在 user/developer 消息和特定工具输出中，这是面向 Agent 场景的扩展。

## 对比与权衡

- 相比 OpenAI Vision API 的 detail=high 分块策略，DeepSeek 采用统一等比缩放并限制每张图 1024 token，成本和延迟更可预测、实现更简单，但在密集文字或小目标细节上可能不如分块保留高分辨率信息。
- 相比外部 URL 传图，base64 内联不依赖服务端公网访问能力，但会占用请求体且增大传输体积；大图或需要复用时 Files API 更优。
- 相比 Anthropic 兼容端点的 image 块，OpenAI 兼容端点使用 image_url 块，字段结构不同；Anthropic base64 还需要显式 media_type，切换协议时容易遗漏。

## 面试可能会问

**问: 为什么多模态请求要把 content 从字符串改成数组？**

因为一条消息里既有文本又有图片，需要用子块类型区分，text 块承载自然语言，image_url/file 块承载图片；模型服务端按块解析并分别转成文本 token 和视觉 token，再拼成一个序列。

**问: base64、外部 URL、Files API 三种方式怎么选择？**

优先看图片大小、是否复用、是否公网可达。本地一次性小图用 base64；公网可访问且小于 32MiB 用 URL；大图、需要复用或单请求超过 48MiB 时用 Files API 上传一次后引用 file_id。

**问: 图片是怎么变成 token 的？为什么 2000×2000 和 5000×5000 token 一样？**

视觉编码器会把缩放后的图片切成 patch 并生成固定数量的视觉 token；DeepSeek 先等比缩放，小图放大、大图缩到约 1300×1300 总像素，所以只要大于上限，两种分辨率都落在同一缩放结果，每张图最多 1024 token。

**问: DeepSeek 的 detail=high 和 OpenAI 的 detail=high 一样吗？**

不同。DeepSeek 文档中 high 等价于 original，只是保留原图但还是会受统一缩放上限约束；OpenAI high 会把大图切成多个 512×512 tile，分辨率越高 token 越多。

**问: 为什么 Chat Completions 里 system/assistant 消息不能带图片？**

官方限制，返回 400。通常是因为对话补全协议中 system 用于指令注入、assistant 用于历史生成内容，视觉输入统一放在 user 侧能降低越权/注入风险并简化多模态状态管理；但在 Responses API 的 Agent 场景中，input_image 被允许出现在 developer 消息和工具输出中。

## 适用场景

- 图片描述、物体识别、视觉问答等需要模型理解图像内容的场景。
- 截图 OCR：识别聊天截图、票据、合同、UI 截图中的文字并结构化提取。
- 图表/数据分析：输入折线图、柱状图、流程图，让模型解读趋势、关键数据点或流程结构。
- 需要复用同一张图片的批量任务：用 Files API 上传一次，多个请求引用同一 file_id，避免重复上传和请求体膨胀。

## 标签

`多模态` `DeepSeek API` `视觉理解` `OpenAI 兼容` `Files API`
