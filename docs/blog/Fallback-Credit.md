---
title: Fallback Credit
url: wikibar://summary/blog/Fallback-Credit
source_type: summary
folder: blog
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T02:23:51.480036+00:00'
---

# Fallback Credit
这页 **Fallback Credit（回退抵扣）** 是上一页 `refusal + fallback` 的“**怎么避免重复付缓存钱**”部分。

最核心一句：

> **模型 A 拒绝了 → 你改用模型 B 重试 → 因为 Prompt Cache 是按模型分开的，B 原本需要重新付一次 cache write；Fallback Credit 让这部分重复成本按 cache read 来计费。** ([Claude Platform][1])

## 1. 为什么需要 Fallback Credit？

假设你有很长的上下文：

```text
System Prompt
+ 100轮聊天记录
+ Tools 定义
+ 当前问题
```

你已经在：

```text
Claude Fable 5
```

建立了 Prompt Cache。

正常情况：

```text
第一次：

很长的 Prompt
      ↓
Fable 5
      ↓
Cache Write 💰
```

以后继续聊天：

```text
同样的历史
   ↓
Cache Read 💰（便宜）
```

但突然：

```text
Fable 5
   ↓
refusal ❌
```

于是你 fallback：

```text
Claude Opus 4.8
```

问题是：

> **Prompt Cache 是按模型隔离的。**

所以 Opus 原本看不到 Fable 的 cache：

```text
Fable Cache
    ❌
    │ 不能直接共享
    ↓
Opus Cache
```

于是如果没有 Fallback Credit：

```text
Fable 5
  ↓
已经 Cache Write 过

  ↓ refusal

Opus 4.8
  ↓
又 Cache Write 一遍 💰💰
```

这就属于因为 fallback 导致的重复成本。([Claude Platform][1])

---

# 2. Fallback Credit 干什么？

Anthropic 在 Fable 拒绝时给你一个：

```text
fallback_credit_token
```

你可以把它理解成一张：

> 🎫 “这是因为上一个模型拒绝，我才换模型的，请不要让我重新付那部分 cache write。”

然后：

```text
Fable
  ↓
refusal
  ↓
给你 fallback_credit_token 🎫
  ↓
你把 token 给 Opus
  ↓
Opus 重试
  ↓
原本应该算 Cache Write 的部分
  ↓
按 Cache Read 方式计费
```

所以它不是：

> 免费调用 Opus ❌

而是：

> **避免因为模型拒绝而重复承担 Prompt Cache 写入成本。** ([Claude Platform][1])

---

# 3. 哪些人需要自己管？

如果你用：

```python
fallbacks="default"
```

这种 **Server-side Fallback**，不用管。

如果你使用 Anthropic 提供的 SDK fallback middleware，也不用管。

它们会自动处理 Fallback Credit。

只有你自己写：

```python
if response.stop_reason == "refusal":
    # 自己换模型
```

这种**手动 fallback**，才需要理解这一页。([Claude Platform][1])

---

# 4. 整个流程只有 4 步

### 第一步：启用 Beta

```python
response = client.beta.messages.create(
    model="claude-fable-5",
    betas=["fallback-credit-2026-07-01"],
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Hello, Claude"}
    ]
)
```

关键：

```python
betas=["fallback-credit-2026-07-01"]
```

---

### 第二步：发生 refusal 后拿 token

检查：

```python
response.stop_reason
```

如果：

```text
refusal
```

再看：

```python
response.stop_details.fallback_credit_token
```

以及：

```python
response.stop_details.fallback_has_prefill_claim
```

官方规定，没有可用 credit 时，这两个字段会是 `null`。([Claude Platform][1])

---

# 5. `fallback_has_prefill_claim` 是什么？

这是这页第二个重要概念。

它告诉你：

> fallback 模型能不能**接着上一个模型已经生成的内容继续写**？

有两种情况。

### `true`

例如 Fable 已经生成：

```text
LLM is a type of neural network that...
```

然后拒绝。

这时候可以：

```text
Fable:
LLM is a type of neural network that...
                                  ↑
                                停止

Opus:
                                  ↑
                              从这里继续
```

所以你把 Fable 已经生成的 `content` 作为：

```python
{
    "role": "assistant",
    "content": response.content
}
```

追加回 messages。

---

### `false`

说明不能接着生成。

那么：

```text
Fable
 ↓
refusal

Opus
 ↓
从原始 request 重新开始
```

不用追加 assistant response。([Claude Platform][1])

所以直接记：

```text
fallback_has_prefill_claim = true
→ 接着写

fallback_has_prefill_claim = false
→ 从头重新回答
```

---

# 6. 简化版完整代码

官方代码考虑了很多边缘情况。先看最容易理解的版本：

```python
from anthropic import Anthropic

client = Anthropic()

request = {
    "max_tokens": 1024,
    "messages": [
        {
            "role": "user",
            "content": "Hello, Claude"
        }
    ]
}

# 第一次请求
response = client.beta.messages.create(
    model="claude-fable-5",
    betas=["fallback-credit-2026-07-01"],
    **request
)

# 如果 Fable 拒绝
if (
    response.stop_reason == "refusal"
    and response.stop_details
    and response.stop_details.fallback_credit_token
):

    details = response.stop_details

    token = details.fallback_credit_token

    # 基础 fallback request
    fallback_request = {
        **request,
        "fallback_credit_token": token
    }

    # 如果允许从之前的输出继续
    if details.fallback_has_prefill_claim:

        fallback_request["messages"] = [
            *request["messages"],
            {
                "role": "assistant",
                "content": response.content
            }
        ]

    # 使用 fallback model
    response = client.beta.messages.create(
        model="claude-opus-4-8",
        betas=["fallback-credit-2026-07-01"],
        **fallback_request
    )

print("Final model:", response.model)
print("Stop reason:", response.stop_reason)

for block in response.content:
    if block.type == "text":
        print(block.text)
```

核心实际上就是：

```python
if response.stop_reason == "refusal":

    token = response.stop_details.fallback_credit_token

    response = client.beta.messages.create(
        model="claude-opus-4-8",
        fallback_credit_token=token,
        ...
    )
```

---

# 7. 为什么 token 不能随便拿来用？

Anthropic 会检查：

> 你是不是**真的在重试刚才那个被拒绝的请求**？

所以一些影响 Prompt 的字段必须完全一致，例如：

```text
system
messages
tools
tool_choice
thinking
cache_control
output_config
mcp_servers
context_management
container
```

但一些不改变 prompt 本身的东西可以改，比如：

```text
model            ← 肯定要换
max_tokens
stop_sequences
stream
metadata
service_tier
```

所以不能：

```text
Fable：

"帮我解释 Transformer"
       ↓
refusal
       ↓

拿 credit token

       ↓

Opus：

"帮我写另外一个完全不同的问题"
```

这样当然不能拿这张“抵扣券”。([Claude Platform][1])

---

# 8. Token 只有 5 分钟

这个也要记：

```text
fallback_credit_token
        ↓
有效期 5 分钟
```

超过五分钟：

```text
token expired
```

就正常 fallback，不再带 token。

而且 token 绑定组织/工作区等调用身份，不能拿别人的 token 来抵扣。([Claude Platform][1])

---

# 9. 怎么知道 Credit 真的生效了？

看：

```python
response.usage
```

特别是：

```text
cache_creation_input_tokens
cache_read_input_tokens
```

成功使用 credit 后，相比不带 token：

```text
cache_creation_input_tokens ↓

cache_read_input_tokens     ↑
```

而且减少和增加的 token 数对应。

也就是说：

```text
原本：

重新 Cache Write
        ↓
       贵


Fallback Credit：

那部分改算 Cache Read
        ↓
       便宜
```

如果变化是 `0`，也不一定说明失败，有可能 fallback 模型本身已经有热缓存，因此没有需要重新定价的内容。([Claude Platform][1])

---

# 10. 如果 Credit 兑换失败怎么办？

官方设计了一个降级顺序：

```text
① 尝试 continuation
   把旧模型 response.content
   作为 assistant message 接上
          ↓
        400？

② 用原始 request
   + fallback_credit_token
          ↓
        400？

③ 放弃 token
   正常 fallback
```

但有一个重要例外：

如果前一个模型已经执行过 **Server Tool**：

```text
web_search
code_execution
...
```

你直接放弃 token、从头重试可能导致：

```text
Server Tool 再执行一次
        ↓
重复调用
        ↓
重复计费
```

所以这种情况官方建议不要静默地重新跑，而应该把费用/错误情况暴露给调用方。([Claude Platform][1])

---

## Cheatsheet

| 概念                                 | 人话                                |
| ---------------------------------- | --------------------------------- |
| Fallback Credit                    | fallback 时避免重复付 Prompt Cache 写入成本 |
| `fallback_credit_token`            | 🎫 抵扣凭证                           |
| `fallback_has_prefill_claim=true`  | fallback 模型可以接着之前输出继续             |
| `fallback_has_prefill_claim=false` | fallback 模型从原始请求重新回答              |
| Server-side fallback               | 自动处理 credit                       |
| SDK middleware                     | 自动处理 credit                       |
| 手动 fallback                        | 你需要自己处理 credit                    |
| Token 有效期                          | **5 分钟**                          |
| Prompt 字段                          | 基本必须与被拒绝请求一致                      |
| `model`                            | 可以换成允许的 fallback model            |
| `usage`                            | 用来检查 credit 是否生效                  |
| Server Tool                        | 小心从头重试导致工具重复执行、重复计费               |

最后把我们最近学的三个概念串起来：

```text
Claude 请求
    ↓
stop_reason = refusal
    ↓
这个模型拒绝
    ↓
fallback
    ↓
换另一个允许的模型
    ↓
fallback_credit_token
    ↓
避免因为换模型
重复承担 Prompt Cache Write 成本
    ↓
fallback model 继续/重新回答
```

所以：

**`refusal` = 为什么要换；`fallback` = 换模型；`fallback credit` = 换模型时别让我重复付缓存写入的钱。** ([Claude Platform][1])

[Claude 官方：Fallback Credit](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)

[1]: https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit "回退抵扣 - Claude Platform Docs"