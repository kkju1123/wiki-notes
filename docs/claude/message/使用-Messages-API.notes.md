# 使用 Messages API

*原文: [https://platform.claude.com/docs/en/build-with-claude/working-with-messages](https://platform.claude.com/docs/en/build-with-claude/working-with-messages) · 来源: web · 生成时间: 2026-09-20T02:16:56.127539+00:00*

## 背景

早期 LLM 接口以单次文本补全为主，开发者需要自行拼接对话历史、处理多模态输入和工具调用。Claude Messages API 为统一这些场景而生，提供结构化的消息数组和内容块，使多轮对话、图像输入、工具调用能够在同一接口中完成。它取代了旧版 Text Completions API，成为构建 Claude 应用的标准入口。

## 痛点

如果仍使用旧 Text Completions API 或自行拼接对话，会遇到上下文管理混乱、无法原生支持图像和工具调用、停止原因处理不明确等问题。不理解 Messages API 的内容块结构，还可能在处理流式响应和工具循环时写出脆弱代码。

## 解决办法

Messages API 将一次请求建模为 system 提示加 messages 数组；messages 中每条消息有 role（user 或 assistant）和 content（列表），content 块可以是 text、image、tool_use、tool_result 等类型。模型按顺序读取历史消息生成下一条 assistant 消息，并返回 stop_reason 指示结束原因。流式模式通过 SSE 增量推送内容，工具调用则要求客户端执行工具后把结果作为 tool_result 块回填到 messages 中继续请求。可以类比为剧本：system 是舞台规则，messages 是角色台词，模型负责续写 assistant 的下一句。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model='claude-sonnet-4-5',
    max_tokens=1024,
    system='You are a helpful assistant.',
    messages=[
        {'role': 'user', 'content': 'Hello, Claude'},
        {'role': 'assistant', 'content': 'Hello! How can I help?'},
        {'role': 'user', 'content': 'What is the weather?'}
    ]
)
print(response.content[0].text)   # 输出助手回复文本
print(response.stop_reason)        # end_turn / max_tokens / tool_use 等
```

这段代码展示了一次多轮对话调用：system 提供全局指令，messages 按时间顺序包含 user 和 assistant 历史，模型基于此生成下一条 assistant 回复。content[0].text 是第一个内容块的文本，stop_reason 用来判断是否正常结束或需要后续处理（如工具调用）。注意 content 是列表，所以用 [0] 访问第一个内容块。

## 关键流程

1. 获取 API key 并安装 anthropic SDK（或直接使用 HTTP API）。
2. 构造请求：设置 model、max_tokens、可选 system，以及包含多轮历史的 messages 数组。
3. 调用 client.messages.create 发送请求。
4. 解析响应：读取 content 中文本块，检查 stop_reason。
5. 若 stop_reason 为 tool_use，执行对应工具并将结果以 tool_result 块回填到 messages，再次调用。
6. 多轮场景中，把新 assistant 消息和用户消息追加到 messages 数组以维护历史。

## 关键点

- messages 数组是服务端唯一看到的对话历史，客户端必须自行维护多轮消息顺序，服务端不保存状态。
- content 是数组而非字符串，每个块有 type 字段，支持文本、图像、工具调用结果等混合内容，这是 Messages API 灵活性的核心。
- stop_reason 指示模型停止原因，常见有 end_turn、max_tokens、tool_use、stop_sequence，必须正确处理，否则会出现循环或截断。
- 流式模式通过 stream=True 和 SSE 事件推送增量内容，能显著降低首字延迟，适合交互式产品。
- 工具调用流程是一个多步循环：模型返回 tool_use → 客户端执行 → 将 tool_result 以 user 消息回填 → 重新请求直到 end_turn。
- system 提示不放在 messages 中，有助于系统级指令稳定，也便于 prompt caching 优化。

## 对比与权衡

- 相比旧版 Text Completions API，Messages API 原生支持多轮对话、图像输入和工具调用，结构更清晰，但需要开发者理解内容块模型，迁移有一定成本。
- 相比 OpenAI Chat Completions API，Messages API 的 content 数组和块类型设计更显式，适合多模态和工具调用，但生态工具和社区示例相对少一些。

## 自测问题

**问: Messages API 中 system 和 messages 有什么区别？为什么 system 单独一个字段？**

system 是全局指令，定义角色、边界和输出格式；messages 是实际对话历史。分开设计方便缓存 system prompt（prompt caching），也避免被历史消息中用户内容干扰。面试时可提 system 可以是一个字符串或数组，支持多个 system 块以容纳不同阶段指令。

**问: 如果模型返回 stop_reason 为 max_tokens 该怎么处理？**

说明输出被截断，需要增大 max_tokens 或优化 prompt 减少生成长度，也可以继续把已生成的 assistant 消息追加到 messages，并发送“继续”请求让模型接着写，但要注意保持内容连续性。

**问: 工具调用时，为什么需要把 tool_result 放在 user 消息里？**

因为对话历史中 assistant 发起了 tool_use 请求，而工具执行结果必须作为新输入进入上下文。API 规定 tool_result 必须紧跟对应的 tool_use 放在后续的 user 消息中，这样才能保持“谁做了什么”的因果链，模型才能基于结果继续推理。

**问: 如何实现流式输出？客户端如何解析？**

在 create 调用中设置 stream=True，服务端通过 SSE 返回 message_start、content_block_start、content_block_delta、message_stop 等事件。客户端监听 content_block_delta 累积文本增量，直到 message_stop。注意流式也要处理工具调用块。

**问: Messages API 的 content 数组为什么设计成块而不是一个字符串？**

为了在同一消息中混合不同模态和语义单元，例如文本说明加图片加工具调用请求。块有 type 和各自的字段，解析更清晰，也方便未来扩展新类型，比如 document、thinking 等。

## 适用场景

- 构建多轮聊天机器人或客服系统，需要维护上下文。
- 多模态应用，如图像理解、文档问答。
- 工具调用型 Agent，自动调用外部 API 完成复杂任务。
- 流式文本生成，提升用户交互体验。

## 标签

`Claude API` `Messages API` `对话式 AI` `Anthropic` `工具调用`
