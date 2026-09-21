---
title: Questions · 第 3 页
url: wikibar://questions/page-3
source_type: questions
folder: questions
count: 3
fetched_at: '2026-09-21T00:57:29.368571+00:00'
---

# Questions · 第 3 页

## Q1: 请总结 LLM API 调用的最基本代码模式，不同厂商的差异主要在哪些方面？

最基本模式有四步：导入 SDK、创建客户端、调用接口发送请求、从响应中读取模型输出。伪代码为：`import sdk； client = sdk.Client(api_key=...)； response = client.xxx(model=..., input=...)； print(response.output)`。不同厂商的差异主要是 SDK 包名、客户端类名、API 地址、方法名、请求参数格式和 response 结构不同。例如 Claude 用 `messages.create` 和 `messages`，OpenAI 用 `responses.create` 和 `input`，DeepSeek 用 OpenAI 兼容接口加 `base_url`。底层思想一致：通过 SDK 封装 HTTP 请求，调用指定 LLM 并解析返回结果。


## Q2: 基础 LLM API 调用与 Agent / Tool Calling 有什么区别？Tool Calling 的基本流程是什么？

基础调用是“用户输入 -> LLM -> 文本输出”，模型只能生成文字，不能真正执行搜索、查数据库或操作系统。Tool Calling 允许模型输出结构化的工具调用指令，由程序去执行实际工具，再把结果返回给模型继续推理。基本流程是：用户输入 -> 模型判断需要调用工具 -> 输出工具名和参数 -> 程序执行工具 -> 把结果作为消息回传 -> 模型综合结果生成最终回答。当模型能自主循环调用工具、规划步骤时，就逐步进入 Agent 范畴。学习路线一般从基础 SDK 调用开始，再到 Structured Output、Tool Calling，再到 Agent 和 RAG 等系统。


## Q3: 请总结 API、SDK、client、API Key、response、model、base_url 这些概念之间的关系和核心数据流。

API 是服务端提供的程序接口；SDK 是用来方便调用 API 的开发工具包；client 是 SDK 创建的通信对象；API Key 是调用 API 的身份凭证；model 是真正要调用的 LLM；base_url 指定 API 请求发往哪台服务器；response 是 API 返回给程序的结果对象。核心数据流是：Python 代码 -> SDK -> client -> API -> LLM 模型 -> API -> response -> 程序解析输出。理解这组关系后，Claude、OpenAI、DeepSeek 等调用本质上都是“用某个 SDK 构造 API 请求，调用指定模型，拿到 response 后取出模型输出”。
