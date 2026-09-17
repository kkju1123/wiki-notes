# 多轮对话：DeepSeek 无状态 API 的上下文拼接

*原文: [https://api-docs.deepseek.com/zh-cn/guides/multi_round_chat](https://api-docs.deepseek.com/zh-cn/guides/multi_round_chat) · 来源: web · 生成时间: 2026-09-17T06:42:47.432524+00:00*

## 背景

大模型单次推理本身没有跨请求记忆，服务端若保存每个用户的会话状态，会引入存储、并发一致性和隐私成本。DeepSeek 的 /chat/completions 选择无状态设计，把上下文管理责任交给客户端，使 API 更简单、可扩展。

## 痛点

如果开发者误以为 API 会自动记住上下文，只传最新问题，模型会丢失上文，导致指代不清、答非所问。另外，若没有掌握 messages 拼接方式，就无法实现真正的多轮对话；如果不理解 token 消耗会随历史线性增长，也容易遇到超长或成本失控问题。

## 解决办法

客户端维护一个 messages 列表，每轮对话前把用户新消息 append 进去，调用 API 后把 assistant 回复也 append 回同一列表，下一轮请求时全量发送这个列表。可以类比为每次打电话都要先复述之前所有聊天记录。核心是利用 role 字段区分 system/user/assistant，让模型在同一请求内看到完整上下文。对于长对话，还需要结合截断、摘要或滑动窗口等手段控制上下文长度。

## 关键代码示例

```python
from openai import OpenAI

client = OpenAI(api_key="<DeepSeek API Key>", base_url="https://api.deepseek.com")

# Round 1
messages = [{"role": "user", "content": "What's the highest mountain in the world?"}]
response = client.chat.completions.create(model="deepseek-flash", messages=messages)
messages.append(response.choices[0].message)  # 把助手回复加入历史

# Round 2
messages.append({"role": "user", "content": "What is the second?"})
response = client.chat.completions.create(model="deepseek-flash", messages=messages)
messages.append(response.choices[0].message)

```

代码第一轮初始化 messages，只包含一条 user 消息，调用 API 后把 assistant 回复追加回 messages。第二轮前再追加新的 user 消息，然后全量发送 messages，这样模型在第二轮请求中能看到第一轮的问题和回答。这个流程体现了无状态 API 下客户端负责维护上下文的原理。

## 关键流程

1. 初始化消息列表，放入首条 user 消息。
2. 调用 chat.completions.create，传入完整 messages。
3. 从 response.choices[0].message 提取助手回复并追加到 messages。
4. 下一轮继续追加新的 user 消息，重复调用和追加过程。
5. 监控 messages 长度，必要时使用截断、摘要或滑动窗口控制 token 消耗。

## 关键点

- DeepSeek /chat/completions 是无状态 API，每次请求都必须携带完整聊天历史，服务端不保存上下文。
- messages 是唯一上下文来源，每项包含 role 和 content，模型根据 role 区分用户输入、助手回复和系统指令。
- assistant 回复必须追加回 messages，否则下一轮模型看不到自己说过什么，会导致连贯性断裂。
- 上下文窗口有限，长对话需要隐私与成本管理策略，例如保留最近 N 条、摘要压缩或检索相关历史。
- DeepSeek 兼容 OpenAI SDK，只要替换 base_url 和 api_key，但模型名和部分参数仍需按文档调整。

## 对比与权衡

- 相比服务端有状态会话 API：无状态方案在水平扩展、故障恢复和简化服务端上更好，但客户端要自己维护历史，且每轮请求 token 消耗随历史线性增长。
- 相比客户端自建外部会话存储（如 Redis）：直接拼接 messages 更简单，无需额外存储依赖，但长对话传输体积和 token 成本更高，也无法跨设备恢复会话。

## 自测问题

**问: DeepSeek /chat/completions 为什么设计成无状态？**

无状态服务端不保存会话信息，请求之间互相独立，便于水平扩展、负载均衡和故障恢复；客户端负责拼接上下文，API 更简洁，但代价是每次请求都要携带完整历史，token 成本上升。

**问: 多轮对话时如果只传最新问题会发生什么？**

模型会丢失上文，无法理解指代和上下文，可能答非所问。因为模型一次请求只能看到当前 messages 里的内容，没有跨请求记忆。

**问: assistant 消息是否必须回传？**

必须。模型需要看到自己之前的回复来保持语气和逻辑一致，否则像“What is the second?”这样的追问无法正确关联第一轮答案。

**问: 历史消息太长怎么办？**

可以保留最近 N 条消息、使用滑动窗口、对早期内容做摘要压缩，或结合向量检索只保留相关历史，确保不超过模型上下文窗口并控制成本。

**问: DeepSeek 兼容 OpenAI SDK 意味着什么？**

开发者可以复用 OpenAI SDK 的调用方式，只需改 base_url 和 api_key；但 DeepSeek 模型名、参数支持和返回结构可能有差异，仍需参考官方文档。

## 适用场景

- 构建多轮聊天机器人、智能客服或教育辅导助手，需要维护用户会话历史并逐轮全量发送。
- 需要精确控制上下文长度和 token 成本的生成式应用，例如长对话摘要或对话截断策略。
- 基于 OpenAI SDK 的低成本迁移场景，将已有 OpenAI 对话代码快速切换到 DeepSeek。
- 无状态微服务架构中的对话服务，由客户端或中间层统一管理会话状态。

## 标签

`DeepSeek` `多轮对话` `无状态API` `上下文管理` `OpenAI SDK`
