---
title: Refusal 是什么
url: wikibar://summary/summary/Refusal-是什么
source_type: summary
folder: summary
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T02:16:12.847849+00:00'
---

# Refusal 是什么
这页和我们刚才讲的 `stop_reason="refusal"` 正好接上。核心就一句：

> **某个 Claude 模型拒绝请求 ≠ 整个 Claude API 都不能回答。你可以识别 `refusal`，然后把同一个请求交给另一个允许的 Claude 模型重试，这叫 fallback（回退）。** ([Claude Platform][1])

## 1. Refusal 是什么？

例如你正常调用：

```python
response = client.messages.create(
    model="claude-fable-5",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "some request"}
    ]
)
```

模型可能返回：

```json
{
  "content": [],
  "stop_reason": "refusal",
  "stop_details": {
    "type": "refusal",
    "category": "cyber",
    "explanation": "This request was declined because it could enable cyber harm."
  }
}
```

注意：这是 **HTTP 200 正常响应，不是 API error**。

所以：

```text
HTTP 400 / 500
→ API 请求本身出问题

HTTP 200 + stop_reason="refusal"
→ API 正常工作
→ 但这个模型的安全分类器拒绝了请求
```

官方目前列出的类别包括 `cyber`、`bio`、`frontier_llm`、`reasoning_extraction` 和 `general_harms`；良性请求也可能触发某些分类器。([Claude Platform][1])

---

## 2. `stop_details` 是干嘛的？

主要看：

```python
response.stop_details.category
response.stop_details.explanation
```

例如：

```text
category = "cyber"

explanation =
"This request was declined because it could enable cyber harm."
```

也就是：

```text
stop_reason
→ 告诉你：拒绝了

stop_details.category
→ 告诉你：属于哪类

stop_details.explanation
→ 给人看的解释
```

注意官方特别说：`explanation` 文本**不保证稳定**，所以可以展示给用户，但不要写：

```python
if response.stop_details.explanation == "某个固定字符串":
    ...
```

应该根据稳定字段（例如 `stop_reason`、`category`）写程序。([Claude Platform][1])

---

# 3. Fallback 是什么？

假设：

```text
Claude Fable 5
     ↓
   refusal
```

你的程序可以：

```text
Claude Fable 5
     ↓
   refusal
     ↓
Claude Opus 4.8
     ↓
   回答成功
```

这就是：

**fallback model = 主模型拒绝后尝试的备用模型。**

这里不是为了绕过所有安全规则，而是因为不同模型、不同安全类别的处理策略可能不同，Anthropic 本身也提供了官方 fallback 机制。([Claude Platform][1])

---

# 4. 最简单：Server-side fallback

现在 Claude API 可以让 Anthropic 帮你做这个过程。

```python
from anthropic import Anthropic

client = Anthropic()

response = client.beta.messages.create(
    model="claude-fable-5",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": "Hello, Claude"
        }
    ],
    fallbacks="default",
    betas=["server-side-fallback-2026-07-01"],
)

print(response.model)
print(response.stop_reason)
```

关键：

```python
fallbacks="default"
```

意思就是：

> 如果主模型因为安全分类器拒绝，并且 Anthropic 对这个类别配置了推荐 fallback，就自动帮我换推荐模型再试。

整个过程：

```text
你的程序
   ↓
Claude API
   ↓
主模型
   ↓
refusal ?
 ├─ No → 正常返回
 │
 └─ Yes
      ↓
   推荐 fallback 存在？
      ↓
   fallback model
      ↓
   返回最终 response
```

所以你的程序只发了一次 API 请求。([Claude Platform][1])

---

# 5. 也可以自己指定 fallback

例如：

```python
from anthropic import Anthropic

client = Anthropic()

response = client.beta.messages.create(
    model="claude-fable-5",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": "Hello, Claude"
        }
    ],
    fallbacks=[
        {
            "model": "claude-opus-4-8"
        }
    ],
    betas=["server-side-fallback-2026-07-01"],
)

print(response.model)
print(response.stop_reason)
```

意思就是：

```text
先：
claude-fable-5

如果 refusal：
       ↓
claude-opus-4-8
```

目前显式 fallback 最多可以指定 **3 个模型**，按顺序尝试，而且模型必须是该主模型允许的 fallback target。([Claude Platform][1])

---

# 6. 怎么知道最后是谁回答的？

看：

```python
response.model
```

例如你本来请求：

```python
model="claude-fable-5"
```

最后：

```python
print(response.model)
```

却得到：

```text
claude-opus-4-8
```

说明 fallback 模型最终处理了这一轮。

response 里面还可能出现：

```json
{
  "type": "fallback",
  "from": {
    "model": "claude-fable-5"
  },
  "to": {
    "model": "claude-opus-4-8"
  }
}
```

人话：

```text
Fable 5
   ↓
拒绝
   ↓
[fallback 边界]
   ↓
Opus 4.8
   ↓
继续处理
```

`usage.iterations` 还会记录每一次模型尝试，所以你可以看到整个 fallback 链。([Claude Platform][1])

---

# 7. Fallback 不是什么错误都处理

这个很重要。

官方 fallback 主要针对：

```text
安全分类器 refusal
```

而不是：

```text
rate limit
server error
overloaded
```

例如：

```text
主模型
 ↓
refusal
 ↓
✅ 可以触发 fallback
```

但是：

```text
主模型
 ↓
429 rate limit
 ↓
❌ 不因为这个机制自动 fallback
```

所以：

**fallback ≠ 通用错误重试系统。** ([Claude Platform][1])

---

# 8. 也可以完全自己写 fallback

如果你想自己控制：

```python
import anthropic

client = anthropic.Anthropic()

messages = [
    {
        "role": "user",
        "content": "Hello, Claude"
    }
]

response = client.messages.create(
    model="claude-fable-5",
    max_tokens=1024,
    messages=messages
)

if response.stop_reason == "refusal":

    print("Primary model refused.")

    if response.stop_details is not None:
        print("Category:", response.stop_details.category)
        print("Explanation:", response.stop_details.explanation)

    # 使用 fallback model 重新发送原始请求
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=messages
    )

print("Model:", response.model)
print("Stop reason:", response.stop_reason)

for block in response.content:
    if block.type == "text":
        print(block.text)
```

逻辑特别简单：

```text
response = 主模型(request)

if response.stop_reason == "refusal":

    response = fallback模型(同一个 request)
```

这就是 fallback 最本质的代码。

官方还提供 **SDK middleware** 来自动完成客户端 fallback；如果自己实现重试，则还要关注官方的 fallback credit 机制以避免某些重复成本。([Claude Platform][1])

---

# 9. 流式输出有个重要坑

假设 streaming：

```text
Claude：
Linux 系统中的这个漏洞可以……
                         ↑
                     此时触发 refusal
```

这叫：

**mid-stream refusal（输出到一半才拒绝）**

官方要求：

> 如果最终发生 refusal，之前已经收到的部分输出应该视为不完整并丢弃。

因为：

```text
已经输出一部分
↓
不代表这是完整、有效的最终答案
↓
后面发生 refusal
↓
前面的 partial output 不能当最终结果
```

而 server-side fallback 在 streaming 中可以在同一个 stream 中发生模型交接；`fallback` block 会标记边界。非 streaming 情况下，如果中途拒绝，API 会省略被拒模型的部分输出，让 fallback 模型从头回答。([Claude Platform][1])

---

# 10. 计费也要知道

如果模型在**产生任何输出之前**就拒绝：

```text
input
 ↓
classifier
 ↓
refusal
```

官方说明这种拒绝尝试**不收费**，但仍计入 rate limit。

如果：

```text
已经生成一些 output
 ↓
然后 refusal
```

则已经发生的输入/输出会正常计费。

如果随后又跑 fallback：

```text
Model A → 产生输出 → refusal
Model B → 产生最终答案
```

对应尝试会分别按各自模型费率计费，`usage.iterations` 可以查看每次尝试。([Claude Platform][1])

---

# 11. Sticky routing 是什么？

还有一个挺实用的优化。

假设：

```text
第 1 轮：

Fable
 ↓
refusal
 ↓
Opus
 ↓
成功
```

第二轮如果还是：

```text
Fable
 ↓
大概率又 refusal
 ↓
Opus
```

这就浪费了。

所以 server-side fallback 有 **sticky routing（粘性路由）**：

```text
第一次：

Fable ❌
 ↓
Opus ✅


后续对话：

直接 → Opus
```

官方目前说这种路由大约保留 **1 小时**，基于对话前缀内容哈希和处理模型，且属于 best-effort，所以代码仍然要能处理主模型被再次尝试的情况。([Claude Platform][1])

---

## Cheatsheet

| 概念                         | 人话                                |
| -------------------------- | --------------------------------- |
| `stop_reason="refusal"`    | 模型拒绝这次请求                          |
| HTTP 200 + refusal         | API 没坏，是模型拒绝                      |
| `stop_details.category`    | 为什么类型的拒绝                          |
| `stop_details.explanation` | 给人看的解释，不要依赖字符串解析                  |
| `fallback`                 | 主模型拒绝 → 换另一个模型试                   |
| `fallbacks="default"`      | Anthropic 自动选择推荐 fallback         |
| `fallbacks=[...]`          | 自己指定备用模型链                         |
| `response.model`           | 最终到底哪个模型处理                        |
| `fallback` block           | 标记模型交接位置                          |
| `usage.iterations`         | 每个模型尝试的记录                         |
| sticky routing             | 一次 fallback 成功后，后续对话尽量直接去该模型      |
| mid-stream refusal         | 输出到一半拒绝，partial output 不当最终答案     |
| rate limit / overload      | **不会因为 refusal fallback 机制自动换模型** |

代码层面最值得记住：

```python
response = client.messages.create(...)

if response.stop_reason == "refusal":
    # 换 fallback model
    response = client.messages.create(
        model="fallback-model",
        max_tokens=1024,
        messages=messages
    )
```

或者让 Anthropic 自动做：

```python
response = client.beta.messages.create(
    model="primary-model",
    max_tokens=1024,
    messages=messages,
    fallbacks="default",
    betas=["server-side-fallback-2026-07-01"],
)
```

一句话：**`refusal` 是“这个模型拒绝了”；`fallback` 是“那就按允许的路由换一个模型继续处理”。** ([Claude Platform][1])

[Claude 官方：拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)

[1]: https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback "拒绝与回退 - Claude Platform Docs"