---
title: Questions · 第 2 页
url: wikibar://questions/page-2
source_type: questions
folder: questions
count: 20
fetched_at: '2026-09-21T00:57:29.362094+00:00'
---

# Questions · 第 2 页

## Q1: Token消耗过高的核心优化方案？

输入侧：压缩或摘要历史、只传最小必要上下文、按需注入工具 schema、裁剪工具返回字段；输出侧：限制 max_tokens、要求简洁结构化输出。系统侧：使用 Prompt 缓存、将简单子任务路由到小模型、检索结果去重和截断、批量处理。还可以做动态上下文预算，优先保留目标、约束、最近关键步骤和工具结果，删除低信息量内容。注意优化后要用评测集验证效果，防止因过度裁剪导致成功率下降。


## Q2: 如何快速溯源Bad Case：Prompt、模型、工具还是流程问题？

快速溯源 Bad Case 要依赖 trace：复现后先定位失败发生在哪一步，是工具返回错误/格式异常、模型选择错误、推理遗漏还是流程分支错误。工具返回异常先查工具日志和 schema；工具结果正确但使用错误，通常是 Prompt 描述不清或模型能力不足；如果模型忽略约束或顺序，先查 Prompt 和状态管理；状态丢失或分支错误则查流程设计。再用单变量实验验证：固定其他条件，只改 Prompt、换模型、修复工具返回或调整流程，对比结果。


## Q3: 版本迭代后，如何量化验证Agent效果提升？

先建立离线回归评测集，覆盖历史 Bad Case、核心场景和边界场景，定义统一指标：任务成功率、完成质量、步数、成本、延迟。版本迭代后跑评测集，用 LLM judge 或人工标注对比旧版。线上再做小流量 A/B，观察最终业务指标和失败率；同时拆分环节指标，如检索命中率、工具准确率、任务完成率，判断提升来自哪里。要关注 Bad Case 修复是否引入回归，报告置信区间，不能只看单个案例。


## Q4: Agent-RAG与普通RAG问答的核心区别？

普通 RAG 问答是相对固定的流水线：召回、排序、拼接、生成，单轮或简单多轮，不主动决策。Agent-RAG 由模型决定是否检索、何时检索、用什么查询、检索哪些源、是否多跳、是否调用工具、是否需要验证和重试。核心区别是“自主决策和闭环控制”，能处理复杂查询，但会引入更大的不确定性、状态管理和成本。落地时 Agent-RAG 需要更严格的工作流约束、评估体系和防跑偏机制。


## Q5: 向量检索与关键词检索的适用场景？

向量检索擅长语义模糊、同义改写、跨语言、无精确词匹配的场景，但可能漏掉精确编号、专有名词和低资源术语。关键词检索适合精确匹配，如订单号、型号、法律法规编号、接口名，结果可解释、速度快。生产通常混合使用：关键词保证精确召回和可解释性，向量做语义扩展，再通过 rerank 排序。具体权重根据业务 Query 类型和文档特征调，必要时先做意图分类再分路召回。


## Q6: 检索不准，除调Prompt外有哪些优化手段？

检索不准要先从数据到生成逐层排查优化：文档清洗、去噪、结构化，优化 chunk 大小和重叠，补充元数据过滤；索引侧调整 embedding 模型或对垂直领域微调；检索侧引入混合检索、rerank、query 改写和意图分类；召回后调整 top_k 和相似度阈值，必要时用知识图谱补全。还应建立检索评测集，量化召回率和 MRR，针对 Bad Case 定向优化，而不是只调 Prompt。


## Q7: 如何保障Agent知识库输出内容可溯源、无幻觉？

保障可溯源、无幻觉要从数据、Prompt、校验三方面做。数据上，知识库要分块并带来源元数据；生成时要求模型只依据检索片段回答，并强制引用来源编号/链接。技术上在输出前做事实一致性校验或 NLI，判断生成内容是否被检索片段支持，低置信度时明确回答“不知道”或触发澄清。还要监控引用准确率和忠实度，关键场景加人工审核或 HITL，并定期治理知识库的权威性、时效和权限。

## Q8: 什么是 SDK？它通常封装了哪些能力？

SDK 是 Software Development Kit，即软件开发工具包，是平台或服务方为降低开发者接入成本而提前封装好的一套开发工具。它通常包含 API 调用封装、身份认证、参数和类型定义、返回结果解析、错误处理以及一些辅助函数。例如 Python 里的 `openai` 包就是 OpenAI 提供的 SDK。使用 SDK 后，开发者不需要手动构造底层 HTTP 请求、拼接 JSON 和处理鉴权细节。


## Q9: API 和 SDK 有什么区别？请用类比说明。

API 是 Application Programming Interface，是服务端定义的“调用规则和接口”，规定请求地址、方法、参数和返回结构；SDK 是官方或社区基于这些规则封装好的客户端工具包。类比来看，API 像餐厅的菜单和点餐规则，SDK 像餐厅提供的点餐 App。没有 SDK 也可以直接发 HTTP 请求调 API，但要自己处理 URL、Header、鉴权、JSON 构造和错误码。有 SDK 后，通常只需要调用类似 `client.xxx.create(...)` 的高级方法。简单说，API 是协议，SDK 是封装。


## Q10: Claude API 的基本调用流程是什么？请写出关键代码。

Claude API 由 Anthropic 提供，Python 中通常使用 `anthropic` SDK。流程是导入 `anthropic`，创建 `client = anthropic.Anthropic(api_key=...)`，然后调用 `client.messages.create(model=..., max_tokens=..., messages=[...])` 发送请求，其中 `messages` 数组的每个元素包含 `role` 和 `content` 两个字段。返回的 `response.content` 是内容块列表，通常遍历它并取 `block.text` 获得文本输出。核心链路是 Python -> Anthropic SDK -> Messages API -> Claude -> response。注意 Claude 使用 `messages` 参数承载多轮对话，并且需要设置 `max_tokens` 等生成限制。


## Q11: 如何理解 Claude 调用代码中的 client、messages.create、messages 参数、max_tokens 和 response 解析？

`client = anthropic.Anthropic()` 创建的是负责与 Anthropic API 通信的客户端对象，封装了鉴权和请求逻辑。`client.messages.create(...)` 是调用 Claude Messages API 的方法，表示向 Claude 发送一次请求。`messages` 参数是一个数组，每个元素包含 `role` 和 `content`，`role` 表示说话人，如 user，`content` 是具体内容。`max_tokens` 限制模型最多生成的 token 数，用于控制成本和输出长度。`response.content` 可能由多个内容块组成，因此通常用循环判断 `block.type == text` 后读取 `block.text`，或确定第一个内容块是文本时直接取 `response.content[0].text`。


## Q12: OpenAI API 的基本调用流程是什么？Responses API 如何读取输出？

OpenAI 提供官方 `openai` Python SDK，调用流程是先 `from openai import OpenAI`，再创建 `client = OpenAI(api_key=...)`。使用 Responses API 时，可以写 `response = client.responses.create(model=..., input=...)`，`input` 就是用户输入。输出可以直接通过 `response.output_text` 读取，这是 SDK 封装后的文本结果。整体链路是 Python -> OpenAI SDK -> Responses API -> GPT -> response.output_text。新项目一般优先使用 Responses API，但也要注意它和 Chat Completions 的方法名与响应结构不同。


## Q13: Claude 和 OpenAI 在 SDK 调用方式上有哪些主要区别？

两者目标链路都是 Python -> SDK -> API -> LLM -> response，但 SDK 方法和参数名不同。Claude 使用 Anthropic SDK，常用 `client.messages.create(...)`，输入参数为 `messages=[...]`，并用 `max_tokens` 控制生成长度，输出从 `response.content` 中按块读取。OpenAI 使用 `openai` SDK，新接口常用 `client.responses.create(...)`，输入可以是 `input=...`，输出用 `response.output_text`。这些方法不是 Python 内置函数，而是各自 SDK 定义的。实际开发时要查对应厂商文档，不能混用方法名和响应解析方式。


## Q14: DeepSeek API 如何使用 OpenAI SDK 调用？base_url 的作用是什么？

DeepSeek API 设计为兼容 OpenAI 接口，因此可以直接使用 `openai` SDK。创建客户端时需要设置 `api_key=DeepSeek Key`，并设置 `base_url=https://api.deepseek.com`。其中 `base_url` 的作用是告诉 SDK：请求格式仍按 OpenAI 风格组织，但请求目的地改成 DeepSeek 的服务器。如果不设置 `base_url`，默认会请求 OpenAI 服务器。配置完成后，后续调用 Chat Completions 或兼容接口即可访问 DeepSeek 模型。


## Q15: 请写出 DeepSeek 使用 OpenAI SDK 的 Chat Completions 风格调用代码，并解释关键字段。

代码示例：`from openai import OpenAI； client = OpenAI(api_key=DS_KEY, base_url=https://api.deepseek.com)； response = client.chat.completions.create(model=deepseek-chat, messages=[{role: user, content: 你好}])； print(response.choices[0].message.content)`。`base_url` 指定请求目标为 DeepSeek，`model` 指定要调用的 DeepSeek 模型。`messages` 是对话数组，每个元素包含 `role` 和 `content`。返回结果通过 `response.choices[0].message.content` 读取，这是 Chat Completions 风格的典型结构。本质是借助 OpenAI SDK 的请求封装，向 DeepSeek 的 OpenAI 兼容接口发请求。


## Q16: 请比较 Claude、OpenAI、DeepSeek 三个平台的 SDK、客户端创建、调用方式、输入和输出差异。

Claude 由 Anthropic 提供，使用 `anthropic` SDK，客户端是 `anthropic.Anthropic(api_key=...)`，常见方法 `client.messages.create(...)`，输入用 `messages`，输出从 `response.content` 读取文本块。OpenAI 原生使用 `openai` SDK，客户端是 `OpenAI(api_key=...)`，新方法 `client.responses.create(...)`，输入可以用 `input`，输出用 `response.output_text`。DeepSeek 通常也使用 `openai` SDK，但创建客户端时多设置 `base_url=https://api.deepseek.com`，调用 OpenAI 兼容的 Chat Completions 风格，输出通过 `response.choices[0].message.content` 读取。三者核心流程相同，差异集中在 SDK 名称、方法名、参数格式和响应解析结构。


## Q17: “SDK 只是包装”如何理解？一次 client.responses.create(...) 背后发生了什么？

意思是 `client.responses.create(...)` 看起来像普通函数调用，实际上 SDK 在背后把参数转换为 HTTP 请求。大致流程是：SDK 拿到模型、输入和参数后，构造请求 URL、Headers（如 Authorization）和 JSON Body，发送 POST 到 API 服务器；服务器调用 LLM 生成结果后返回 JSON；SDK 再解析 JSON，组装成 Python response 对象。因此开发者只看到“调方法 -> 拿对象”，不直接感知底层 HTTP、序列化和错误处理。理解这层包装有助于排错和优化请求，而不是把 SDK 当黑盒。


## Q18: 为什么 DeepSeek 可以使用 OpenAI SDK？什么是 OpenAI-compatible API？

因为 SDK 和真正的模型/服务器是解耦的：OpenAI SDK 本质上是一个按 OpenAI 规定的接口格式发送 HTTP 请求的 Python 工具。如果 DeepSeek 的 API 也实现同样的请求路径、参数格式和返回结构，就可以被 OpenAI SDK 调用。创建客户端时通过 `base_url` 把请求目的地改为 DeepSeek 的 API 地址即可。这就是 OpenAI-compatible API：兼容 OpenAI 接口风格，但后端模型和服务器是另一家。需要注意，使用 `openai` SDK 不代表一定调 OpenAI 模型，真正调谁由 `base_url`、API Key 和 `model` 共同决定。


## Q19: API Key 的作用是什么？项目中应该如何安全保存？

API Key 是程序访问 API 的身份凭证，服务器据此识别调用者身份、权限、额度和计费账户。它非常敏感，泄露后可能被盗刷或产生费用。项目中不要硬编码 api_key 的 sk 值，尤其不要提交到 GitHub 等公开仓库。最佳实践是使用环境变量保存，例如 OPENAI_API_KEY，然后由 SDK 自动读取或从 os.environ 获取。生产环境还应配合密钥管理服务、最小权限、定期轮换和监控告警。


## Q20: 在 LLM SDK 调用中，client 对象和 response 对象分别是什么？

`client` 是 SDK 创建出来的、负责与 API 通信的 Python 对象，例如 `OpenAI()` 或 `anthropic.Anthropic()`。它封装了 HTTP 连接、认证信息、请求序列化和接口方法，后续通过 `client.some_api.create(...)` 发起请求。`response` 是 API 返回结果经过 SDK 解析后的对象，包含模型输出和元数据，如 token 使用量等。不同 SDK 的 response 结构不同，例如 OpenAI Responses API 用 `response.output_text`，Claude 用 `response.content[...].text`，Chat Completions 用 `response.choices[0].message.content`。典型模式就是创建 client -> 发送请求 -> 拿到 response -> 解析输出。
