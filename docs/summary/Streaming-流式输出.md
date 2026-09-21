---
title: Streaming（流式输出）
url: wikibar://summary/summary/Streaming-流式输出
source_type: summary
folder: summary
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T07:41:28.739199+00:00'
---

# Streaming（流式输出）

这页讲的是 **Streaming（流式输出）**。它其实很好理解：

> **不等 Claude 把整个回答生成完再一次性返回，而是生成一点，就立刻传一点给你的程序。**

你现在使用 ChatGPT 时看到文字一个字一段一段“冒出来”，本质上就是类似的体验。

Claude 的 Messages API 使用 **SSE（Server-Sent Events）** 来实现流式传输。([Claude Platform][1])

## 1. 不 Streaming 是什么样？

普通请求：

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-5",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Explain transformers"}
    ]
)

print(response.content[0].text)
```

流程：

```text
你的程序
   ↓
Claude 开始生成
   ↓
生成……
生成……
生成……
   ↓
全部生成完成
   ↓
完整 response 返回
   ↓
用户终于看到答案
```

假设 Claude 需要 20 秒：

```text
0s ---------------------- 20s
用户：什么都看不到           完整答案突然出现
```

---

## 2. Streaming 呢？

改成：

```python
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-opus-5",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Explain transformers"}
    ],
) as stream:

    for text in stream.text_stream:
        print(text, end="", flush=True)
```

现在：

```text
Claude 生成 "A"
        ↓
马上发送

Claude 生成 "Transformer"
        ↓
马上发送

Claude 生成 "is..."
        ↓
马上发送
```

于是用户看到：

```text
1s: A Transformer
2s: A Transformer is a neural
3s: A Transformer is a neural network architecture...
...
```

而不是等 20 秒以后一起出现。官方 Python SDK 的最简单用法就是遍历 `stream.text_stream`。([Claude Platform][1])

---

# 3. Streaming 不代表模型生成得更快

这个区别很重要。

假设完整生成依然需要：

```text
20 秒
```

Non-streaming：

```text
0s                         20s
│---------------------------│
什么都没有                 整个答案
```

Streaming：

```text
0s  1s  2s  3s .......... 20s
    ↓   ↓   ↓              ↓
    字  字  字             完成
```

所以主要改善的是：

> **用户感知的延迟（perceived latency）。**

用户不用一直盯着空白页面。

---

# 4. `stream.text_stream` 是 SDK 帮你简化后的版本

实际上服务器发送过来的并不是单纯：

```text
Hello
world
...
```

而是一系列 **Event（事件）**。

完整生命周期大致是：

```text
message_start
      ↓
content_block_start
      ↓
content_block_delta
      ↓
content_block_delta
      ↓
content_block_delta
      ↓
content_block_stop
      ↓
message_delta
      ↓
message_stop
```

官方当前定义的基本事件流程就是这样。([Claude Platform][1])

比如 Claude 回答：

```text
Hello!
```

实际可能收到：

```text
message_start
```

然后：

```text
content_block_start
```

然后：

```json
{
  "type": "text_delta",
  "text": "Hello"
}
```

再：

```json
{
  "type": "text_delta",
  "text": "!"
}
```

最后：

```text
content_block_stop
message_delta
message_stop
```

所以：

```python
for text in stream.text_stream:
```

其实是 SDK 帮你把这些底层事件处理掉了。

---

# 5. `delta` 是什么？

这个词你以后看各种 LLM API 都会碰到。

**delta = 相比之前新增的那一点内容。**

比如完整答案：

```text
Hello Keke!
```

可能拆成：

```text
delta 1 = "Hello"
delta 2 = " Keke"
delta 3 = "!"
```

你的程序不断：

```python
full_text += delta
```

最终：

```text
Hello Keke!
```

所以：

```text
delta ≈ 增量
```

Claude 的 `content_block_delta` 就是在告诉你：

> 这个 content block 又增加了一点东西。

([Claude Platform][1])

---

# 6. 为什么还有 `content_block`？

因为 Claude 的输出不一定只有文字。

我们前面已经学过：

```text
response.content
```

可能是：

```text
[
    text,
    tool_use,
    thinking,
    ...
]
```

例如：

```text
content[0]
→ thinking

content[1]
→ text

content[2]
→ tool_use
```

因此 Streaming 不能简单设计成：

```text
一直传字符串
```

它必须告诉你：

```text
现在开始第 0 个 block
↓
这个 block 的增量……
↓
第 0 个 block 结束

现在开始第 1 个 block
↓
这个 block 的增量……
↓
第 1 个 block 结束
```

这就是：

```text
content_block_start

content_block_delta

content_block_stop
```

---

# 7. `index` 就是在说哪个 block

比如：

```json
{
    "type": "content_block_delta",
    "index": 0,
    ...
}
```

意思：

```text
这是 content[0] 的新增内容
```

如果：

```json
{
    "index": 1
}
```

就是：

```text
content[1]
```

最终可能得到：

```python
response.content[0]
response.content[1]
response.content[2]
```

官方说明每个内容块的 `index` 都对应最终 `Message.content` 数组中的位置。([Claude Platform][1])

---

# 8. Tool Calling 也能 Streaming

这就和你之前学的 Agent 串起来了。

假设 Claude 决定：

```python
get_weather(
    location="San Francisco, CA"
)
```

它不一定一次性把：

```json
{
  "location": "San Francisco, CA"
}
```

发给你。

可能是：

```text
{
```

然后：

```text
"location":
```

然后：

```text
"San
```

然后：

```text
Francisco
```

然后：

```text
CA"
}
```

底层事件叫：

```text
input_json_delta
```

例如：

```json
{
    "type": "input_json_delta",
    "partial_json": "{\"location\": \"San Fra"
}
```

这里有个非常重要的坑：

> `partial_json` 是**不完整 JSON**。

所以不能每收到一块就直接：

```python
json.loads(partial_json)
```

因为：

```text
{"location": "San Fra
```

当然不是合法 JSON。

通常：

```text
不断收集 partial_json
        ↓
content_block_stop
        ↓
拼成完整 JSON
        ↓
parse
        ↓
执行 Tool
```

SDK 也提供了处理解析后增量值的辅助能力。([Claude Platform][1])

---

# 9. Thinking 也可以 Streaming

启用相应 thinking 配置时，流里还可能出现：

```text
thinking_delta
```

以及在 thinking block 结束前出现：

```text
signature_delta
```

例如官方当前示例使用：

```python
with client.messages.stream(
    model="claude-opus-5",
    max_tokens=20000,

    thinking={
        "type": "adaptive",
        "display": "summarized"
    },

    messages=[
        {
            "role": "user",
            "content": "What is the greatest common divisor of 1071 and 462?"
        }
    ],
) as stream:

    for event in stream:

        if event.type == "content_block_delta":

            if event.delta.type == "thinking_delta":
                print(
                    event.delta.thinking,
                    end="",
                    flush=True
                )

            elif event.delta.type == "text_delta":
                print(
                    event.delta.text,
                    end="",
                    flush=True
                )
```

也就是说你可以根据：

```python
event.delta.type
```

判断现在来的到底是什么。([Claude Platform][1])

---

# 10. 不想一个 event 一个 event 处理怎么办？

这个功能非常实用。

有时候你：

> 想用 streaming 保持 HTTP 连接活跃，但其实不需要实时显示每一个字。

尤其是：

```python
max_tokens=128000
```

这种超长任务。

可以：

```python
client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-opus-5",
    max_tokens=128000,
    messages=[
        {
            "role": "user",
            "content": "Write a detailed analysis..."
        }
    ],
) as stream:

    message = stream.get_final_message()


for block in message.content:
    if block.type == "text":
        print(block.text)
```

这里：

```python
stream.get_final_message()
```

SDK 会：

```text
收到 delta
↓
收到 delta
↓
收到 delta
↓
自动帮你拼
↓
完整 Message
```

最后得到的 `Message` 对象和普通 `.create()` 返回的形式一致。官方还指出，对非常大的 `max_tokens` 请求，这种方式尤其有用，因为 SDK 会要求使用 streaming 来避免 HTTP 超时。([Claude Platform][1])

---

# 11. `message_delta` 又是什么？

前面的：

```text
content_block_delta
```

修改的是：

```text
Message.content
```

而：

```text
message_delta
```

修改的是 **整个 Message 顶层的信息**。

例如：

```json
{
  "type": "message_delta",
  "delta": {
    "stop_reason": "end_turn",
    "stop_sequence": null
  }
}
```

你之前学过：

```text
stop_reason
```

所以这里正好串起来：

```text
Claude 一直 streaming
        ↓
text_delta
text_delta
text_delta
...
        ↓
message_delta
        ↓
stop_reason = end_turn
        ↓
message_stop
```

如果 Claude 最后决定调用工具：

```text
message_delta
↓
stop_reason = tool_use
```

那么你的 Agent 就知道：

```text
执行工具
↓
返回 tool_result
↓
继续 Claude
```

官方示例中的工具流最终正是以 `stop_reason: "tool_use"` 结束。([Claude Platform][1])

---

# 12. `message_stop` = 彻底结束

很好记：

```text
message_start
= 开始

content_block_start
= 某块开始

content_block_delta
= 这块新增一点

content_block_stop
= 这块结束

message_delta
= Message 顶层信息更新

message_stop
= 整条 Message 结束
```

---

# 13. 还有 `ping`

流里可能突然出现：

```json
{
    "type": "ping"
}
```

不用紧张。

可以把它理解成：

> “连接还活着。”

官方说明一个流中可能出现任意数量的 `ping` 事件。([Claude Platform][1])

---

# 14. Streaming 中途也可能 Error

这个在后端开发里很重要。

普通 API：

```text
request
↓
HTTP 529
↓
失败
```

但 Streaming 有一种特殊情况：

```text
HTTP connection 已经成功建立
↓
开始 streaming
↓
传了一部分
↓
突然服务器过载
↓
error event
```

例如：

```text
event: error

data: {
    "type": "error",
    "error": {
        "type": "overloaded_error",
        "message": "Overloaded"
    }
}
```

所以：

> **HTTP 请求开始成功，不代表整个 stream 一定成功结束。**

生产环境必须处理流中的 error event。官方也提醒未来可能新增事件类型，因此代码应该能妥善处理未知事件。([Claude Platform][1])

---

## 15. 对你以后做 Agent，整个流程就非常清楚了

```text
用户
 │
 ▼
FastAPI Backend
 │
 │ Messages API
 ▼
Claude
 │
 ├── thinking_delta ──────→ 前端显示进度/摘要
 │
 ├── text_delta ─────────→ 前端实时显示文字
 │
 ├── tool_use
 │      │
 │      └── input_json_delta
 │             ↓
 │        拼成 Tool 参数
 │             ↓
 │        Python Tool
 │             ↓
 │        tool_result
 │             ↓
 │          Claude
 │
 ├── text_delta ─────────→ 前端继续显示
 │
 ▼
message_delta
 │
 │ stop_reason=end_turn
 ▼
message_stop
```

所以 Streaming 并不只是：

> **“打字机效果”。**

在 Agent 系统里它实际上还是一个：

> **实时事件通道。**

文字、工具参数、思考显示、停止原因、错误等都可以通过这个流逐步到达你的后端。([Claude Platform][1])

### Cheatsheet

| 概念                    | 人话                           |
| --------------------- | ---------------------------- |
| Streaming             | Claude 生成一点就返回一点             |
| SSE                   | Claude API 用来推送流事件的机制        |
| `messages.stream()`   | Python SDK 开启流               |
| `text_stream`         | 最简单，只拿文字                     |
| delta                 | 新增加的那一点内容                    |
| `message_start`       | 整条消息开始                       |
| `content_block_start` | 一个内容块开始                      |
| `content_block_delta` | 内容块增加一点                      |
| `text_delta`          | 新生成的文字                       |
| `input_json_delta`    | Tool 参数 JSON 的一部分            |
| `thinking_delta`      | thinking 内容增量                |
| `content_block_stop`  | 当前 block 完成                  |
| `message_delta`       | 更新 stop_reason 等顶层信息         |
| `message_stop`        | 整条消息完成                       |
| `ping`                | 保活事件                         |
| `error`               | Stream 中途发生错误                |
| `get_final_message()` | SDK 自动把所有 delta 拼成最终 Message |

最值得记的代码其实只有：

```python
with client.messages.stream(
    model="claude-opus-5",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Hello"}
    ],
) as stream:

    for text in stream.text_stream:
        print(text, end="", flush=True)
```

整个概念压缩成一句：

**普通 API = Claude 全说完再给你；Streaming = Claude 边说，你边收。** ([Claude Platform][1])

[Claude 官方：Streaming Messages](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)

[1]: https://platform.claude.com/docs/zh-CN/build-with-claude/streaming "流式传输消息 - Claude Platform Docs"