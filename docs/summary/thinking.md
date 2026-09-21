---
title: thinking
url: wikibar://summary/summary/thinking
source_type: summary
folder: summary
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T08:17:38.903703+00:00'
---

# thinking
 这页 **Thinking（思考）** 可以理解成 Claude API 里的“**先想，再答**”。

最核心的一句话：

> **普通生成偏向直接产出答案；Thinking 允许 Claude 在最终回答前进行额外推理、尝试不同方案、检查中间结果，然后再生成最终答案。** ([Claude Platform][1])

这对你以后做 **Coding Agent、复杂 RAG、数学推理、长时间 Agent 任务** 很重要。

## 1. Thinking 到底是什么？

比如你问：

```text
1071 和 462 的最大公约数是多少？
```

不开 Thinking，可以粗略理解成：

```text
Question
   ↓
Claude
   ↓
Answer: 21
```

启用 Thinking：

```text
Question
   ↓
Claude
   ↓
Thinking
  ├─ 分析问题
  ├─ 尝试方法
  ├─ 计算
  ├─ 检查结果
  └─ 修正错误
   ↓
Final Answer
   ↓
21
```

API 的 `content` 因此可能不是只有：

```python
[
    TextBlock(...)
]
```

而是：

```python
[
    ThinkingBlock(...),
    TextBlock(...)
]
```

也就是：

```text
response.content

├── thinking
│
└── text
```

([Claude Platform][1])

---

## 2. 最基本代码

例如某些需要显式开启 adaptive thinking 的模型：

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,

    thinking={
        "type": "adaptive",
        "display": "summarized"
    },

    messages=[
        {
            "role": "user",
            "content": "What is the greatest common divisor of 1071 and 462?"
        }
    ]
)

for block in response.content:

    if block.type == "thinking":
        print("Thinking:", block.thinking)

    elif block.type == "text":
        print("Answer:", block.text)
```

关键：

```python
thinking={
    "type": "adaptive",
    "display": "summarized"
}
```

`adaptive` 的意思是：

> **Claude 自己根据问题难度决定要不要思考，以及思考多深。**

简单问题可能少想甚至跳过；困难问题会投入更多推理。([Claude Platform][1])

---

## 3. `thinking` block 长什么样？

API 可能返回：

```json
{
  "content": [
    {
      "type": "thinking",
      "thinking": "Use the Euclidean algorithm...",
      "signature": "WaUjzkypQ2mUEVM..."
    },
    {
      "type": "text",
      "text": "The greatest common divisor is 21."
    }
  ]
}
```

所以：

```text
thinking
= 推理摘要

text
= 最终给用户的回答
```

但这里有一个非常重要的地方：

> **你看到的 thinking 不是 Claude 原始、完整的内部思维链，而是推理摘要。**

Anthropic 明确说明，没有任何 `display` 设置会把原始完整 chain-of-thought 返回给开发者。([Claude Platform][1])

---

## 4. `display` 是干什么的？

现在主要有：

```text
summarized
omitted
updates (beta)
```

### `summarized`

```python
thinking={
    "type": "adaptive",
    "display": "summarized"
}
```

意思：

> 给我看可读的**思考摘要**。

例如：

```text
Thinking:
Use Euclidean algorithm.
1071 = 2×462 + 147
462 = 3×147 + 21
...
```

然后：

```text
Answer:
The GCD is 21.
```

### `omitted`

```python
thinking={
    "type": "adaptive",
    "display": "omitted"
}
```

意思：

> Claude 可以思考，但是**不要把思考摘要返回给我**。

返回可能：

```json
{
  "type": "thinking",
  "thinking": "",
  "signature": "EosnCkYICx..."
}
```

然后正常：

```json
{
  "type": "text",
  "text": "The answer is 21."
}
```

注意：

**隐藏 thinking ≠ 没有 thinking。**

Claude还是进行了推理，而且你**仍然要为完整 thinking tokens 付费**。`omitted` 主要可以减少流式传输延迟。([Claude Platform][1])

---

## 5. `signature` 是什么？

你会看到：

```json
{
  "type": "thinking",
  "thinking": "...",
  "signature": "WaUjzkypQ2mUEVM..."
}
```

这个：

```text
signature
```

可以理解成：

> **Claude 完整推理的加密凭证。**

你看不到里面真正的完整 reasoning，但服务器可以用它恢复之前的推理状态。

所以多轮对话、尤其 Tool Calling 时有一条非常重要的规则：

> **Thinking block 要原样传回，不要修改。** ([Claude Platform][1])

例如：

```text
Claude
 ↓
thinking
 ↓
tool_use
 ↓
你的程序执行工具
 ↓
tool_result
```

下一次请求时，要保留原来的 thinking block。

---

## 6. Thinking + Tool Calling 才是 Agent 真正有意思的地方

假设用户：

```text
帮我找出这个项目里的登录 Bug 并修复。
```

Claude 可以：

```text
Thinking
↓
“先检查认证相关文件”

Tool Use
↓
search_files()

Tool Result
↓
找到 auth.py

Thinking
↓
“问题可能出在 token refresh”

Tool Use
↓
read_file()

Tool Result
↓
代码返回

Thinking
↓
“确认 bug，修改这里”

Tool Use
↓
edit_file()

Tool Result
↓
修改成功

Thinking
↓
“还应该运行测试”

Tool Use
↓
run_tests()

Tool Result
↓
PASS

Final Answer
↓
“Bug 已修复……”
```

所以 Thinking 并不是：

```text
Thinking
↓
一次
↓
Answer
```

Agent 中可能是：

```text
Thinking
↓
Tool
↓
Thinking
↓
Tool
↓
Thinking
↓
Tool
↓
Answer
```

这就是为什么它特别适合 Coding Agent 和长时间 Agent 工作流。([Claude Platform][1])

---

## 7. Tool Result 回来时千万别把 Thinking 丢掉

例如 Claude 返回：

```python
response.content
```

里面是：

```text
[
    thinking,
    tool_use
]
```

你应该保留整个 assistant response：

```python
messages.append({
    "role": "assistant",
    "content": response.content
})
```

然后加入：

```python
messages.append({
    "role": "user",
    "content": [
        {
            "type": "tool_result",
            "tool_use_id": tool_id,
            "content": result
        }
    ]
})
```

不要自己变成：

```python
# ❌ 把 thinking block 删除
messages.append({
    "role": "assistant",
    "content": [
        tool_use_block
    ]
})
```

因为工具工作流中，官方要求 Thinking block **完整、未经修改地传回**。([Claude Platform][1])

---

## 8. Thinking 和 Effort 到底什么区别？

这正好和你前面学的 `effort` 串起来。

最简单记：

```text
thinking
= 要不要/如何进行思考

effort
= 整个任务愿意投入多少工作量
```

例如：

```python
thinking={
    "type": "adaptive"
}

output_config={
    "effort": "low"
}
```

意思大致：

> 可以思考，但整个任务尽量少花力气。

而：

```python
thinking={
    "type": "adaptive"
}

output_config={
    "effort": "high"
}
```

就是：

> 可以思考，而且这个任务认真处理。

在 adaptive 模式下，`effort` 也会影响 Claude **思考的频率和深度**。([Claude Platform][1])

所以不要写：

```python
# ❌ 错
output_config={
    "effort": "adaptive"
}
```

因为：

```text
adaptive
→ thinking mode

low / medium / high / xhigh / max
→ effort level
```

---

## 9. Thinking 和 `max_tokens`

这个也非常重要：

> **Thinking tokens 也算 output tokens，而且占 `max_tokens`。**

例如：

```python
max_tokens=16000
```

这个空间不是：

```text
16000 tokens 全给最终回答
```

而是：

```text
Thinking
+
Final Answer
≤ 16000 tokens
```

([Claude Platform][1])

例如：

```text
max_tokens = 16000

Thinking:
6000 tokens

Final Answer:
3000 tokens

总共:
9000
```

没问题。

但如果：

```text
Thinking:
15000

Final answer:
还需要 3000
```

就可能撞上 token 上限。

所以复杂 reasoning / Agent 请求要给足 `max_tokens`。

---

## 10. Thinking 是要钱的

即使：

```python
"display": "omitted"
```

你看不到 reasoning，也还是：

```text
Claude 实际生成 Thinking
          ↓
Thinking Tokens
          ↓
按照 output tokens 计费
```

所以：

```text
隐藏 thinking
≠
免费 thinking
```

([Claude Platform][1])

这也是为什么之前的 `effort` 很重要：

```text
effort ↓
   ↓
通常整体工作量 ↓
   ↓
thinking ↓
   ↓
tokens ↓
   ↓
成本 / latency ↓
```

如果目标只是降低启用 Thinking 后的成本和延迟，Anthropic 建议首先考虑降低 `effort`；如果需要硬性支出上限，则使用 `max_tokens`。([Claude Platform][1])

---

## 11. 有些新模型 Thinking 默认就是开的

这个是现在文档里很重要的变化。

当前文档显示，像：

```text
Claude Opus 5
Claude Sonnet 5
Claude Fable 5 / 5.1
Claude Mythos 5 / 5.1
```

Thinking 已经默认开启，不需要你显式写 `adaptive`；这些模型默认通常是：

```text
display = omitted
```

也就是说：

> **模型实际上在 thinking，但默认不给你显示摘要。**

而一些较早模型，例如 Claude Opus 4.8 / 4.7 / 4.6、Sonnet 4.6，则需要显式设置 `thinking={"type":"adaptive"}` 才开启。具体能力应按模型配置表确认。([Claude Platform][1])

---

## 12. 有些模型可以关闭 Thinking

例如文档给出的 Sonnet 5：

```python
response = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=4096,

    thinking={
        "type": "disabled"
    },

    messages=[
        {
            "role": "user",
            "content": "Summarize this article in one sentence."
        }
    ]
)
```

适合：

```text
简单摘要
简单分类
简单格式转换
高吞吐任务
```

但不是所有模型都允许关闭，而且某些高 effort 配置下 Thinking 也不能关闭。例如 Opus 5 在 `xhigh` / `max` effort 下不能搭配 `thinking: disabled`。([Claude Platform][1])

---

## 13. Thinking + Streaming

你刚学的 Streaming 也能和它组合。

```python
with client.messages.stream(
    model="claude-opus-4-8",
    max_tokens=16000,

    thinking={
        "type": "adaptive",
        "display": "summarized"
    },

    messages=[
        {
            "role": "user",
            "content": "What is the greatest common divisor of 1071 and 462?"
        }
    ]
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

于是你前面学的两个 delta 就串起来了：

```text
thinking_delta
= 思考摘要正在一点点过来

text_delta
= 最终答案正在一点点过来
```

事件大致：

```text
message_start
       ↓
thinking block start
       ↓
thinking_delta
thinking_delta
thinking_delta
       ↓
signature_delta
       ↓
thinking block stop
       ↓
text block start
       ↓
text_delta
text_delta
       ↓
message_stop
```

([Claude Platform][1])

---

## 14. 为什么最后还有 `signature_delta`？

Streaming 时完整 signature 也需要传回来。

所以：

```text
thinking_delta
thinking_delta
thinking_delta
      ↓
signature_delta
      ↓
thinking block 完成
```

SDK 最终可以帮你组装：

```python
message = stream.get_final_message()
```

官方建议尽量使用 SDK 的消息累积功能，而不是自己手动拼 thinking 和 signature。([Claude Platform][1])

---

## 15. 现在把 Thinking、Effort、Streaming、Tool Use 全串起来

一个复杂 Agent 可以是：

```text
User
 │
 ▼
Claude
 │
 │ thinking=adaptive
 │ effort=high
 ▼
Thinking
 │
 │ thinking_delta → Streaming
 ▼
“我需要先搜索代码”
 │
 ▼
tool_use
 │
 ▼
Python executes tool
 │
 ▼
tool_result
 │
 ▼
Claude
 │
 ▼
Thinking
 │
 ▼
“发现可能是 token refresh bug”
 │
 ▼
tool_use
 │
 ▼
edit_file()
 │
 ▼
tool_result
 │
 ▼
Claude
 │
 ▼
Thinking
 │
 ▼
run_tests()
 │
 ▼
PASS
 │
 ▼
Final Answer
 │
 │ text_delta → Streaming
 ▼
User
```

这基本已经是现代 Coding Agent 的核心循环了。

### Cheatsheet

| 概念                     | 人话                    |
| ---------------------- | --------------------- |
| Thinking               | Claude 先推理再回答         |
| `adaptive`             | Claude 根据任务决定思考多少     |
| `disabled`             | 关闭 Thinking（仅部分模型支持）  |
| `thinking` block       | 推理内容块                 |
| `text` block           | 最终答案                  |
| `display="summarized"` | 返回可读的推理摘要             |
| `display="omitted"`    | 思考但不显示摘要              |
| `signature`            | 完整推理的加密凭证             |
| `thinking_delta`       | Streaming 的思考增量       |
| `signature_delta`      | Streaming 的签名增量       |
| Thinking + Tools       | 可以在工具调用之间继续推理         |
| Thinking tokens        | 算 output tokens，要收费   |
| `max_tokens`           | Thinking + 最终答案共同的硬上限 |
| `effort`               | 整个任务投入多少工作量           |

最后把你最容易混的三个参数记成：

```text
thinking
= “要不要 / 怎么思考？”

effort
= “要多努力？”

max_tokens
= “最多允许花多少输出 token？”
```

所以复杂 Agent 常见的思路就是：

```python
thinking={
    "type": "adaptive",
    "display": "summarized"
}

output_config={
    "effort": "high"
}

max_tokens=16000
```

= **允许 Claude 自己判断何时需要思考 + 要求它比较认真地处理任务 + 给 Thinking 和最终回答足够的 token 空间。** ([Claude Platform][1])

[Claude 官方：Thinking](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking?utm_source=chatgpt.com)

[1]: https://platform.claude.com/docs/zh-CN/build-with-claude/thinking "思考 - Claude Platform Docs"