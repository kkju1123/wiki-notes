---
title: Batch Processing（批处理）
url: wikibar://summary/blog/Batch-Processing-批处理
source_type: summary
folder: blog
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T07:21:01.695287+00:00'
---

# Batch Processing（批处理）
这页 **Batch Processing（批处理）** 和刚才的 Streaming 正好是两个完全不同的使用场景。

一句话先理解：

> **Batch Processing = 我一次扔给 Claude 很多独立任务，不着急拿结果，让服务器后台慢慢处理；代价是不能实时返回，但价格只有标准 API 的 50%。** ([Claude Platform][1])

## 1. 普通 API vs Batch

假设你有 **10,000 条商品评论**，想让 Claude 判断情感：

```text
评论1 → positive / negative
评论2 → positive / negative
评论3 → positive / negative
...
评论10000
```

普通 Messages API：

```python
for review in reviews:
    response = client.messages.create(...)
```

相当于：

```text
request 1 → Claude → response 1
request 2 → Claude → response 2
request 3 → Claude → response 3
...
```

如果你的任务根本不要求用户立刻看到结果，这种方式就没必要。

Batch 则是：

```text
            ┌→ request 1
            ├→ request 2
10,000条 ───├→ request 3
            ├→ ...
            └→ request 10000

       一次提交成 Batch
              ↓
       Anthropic 后台处理
              ↓
          你先去干别的
              ↓
          过一会检查
              ↓
         下载所有结果
```

而且这些请求**独立处理**，某一个失败不会把整个 Batch 搞失败。([Claude Platform][1])

---

## 2. 为什么要 Batch？

主要三个原因：

```text
大量任务
+
不要求实时
+
想省钱
```

官方当前 Batch API 的使用量按照标准 API 价格的 **50%** 收费，而且大多数 Batch 会在 **1 小时以内完成**，但它是异步任务，不应该依赖它获得实时响应。([Claude Platform][1])

非常适合：

```text
100,000 条评论分类

50,000 篇文章摘要

10,000 个测试 Prompt 做模型评测

批量数据提取

批量生成商品描述

离线数据清洗 / 标注
```

这在做 LLM Evaluation 时尤其常见。

---

# 3. 最基本代码

假设我们一次处理两个问题：

```python
import anthropic

from anthropic.types.message_create_params import (
    MessageCreateParamsNonStreaming,
)

from anthropic.types.messages.batch_create_params import Request


client = anthropic.Anthropic()


message_batch = client.messages.batches.create(
    requests=[

        Request(
            custom_id="question-1",

            params=MessageCreateParamsNonStreaming(
                model="claude-opus-5",
                max_tokens=1024,

                messages=[
                    {
                        "role": "user",
                        "content": "Explain transformers."
                    }
                ],
            ),
        ),

        Request(
            custom_id="question-2",

            params=MessageCreateParamsNonStreaming(
                model="claude-opus-5",
                max_tokens=1024,

                messages=[
                    {
                        "role": "user",
                        "content": "Explain RAG."
                    }
                ],
            ),
        ),

    ]
)

print(message_batch)
```

核心其实就是：

```python
client.messages.batches.create(
    requests=[
        request1,
        request2,
        request3,
        ...
    ]
)
```

每个 request 内部依然是你已经很熟悉的 Messages API 参数：

```python
model=
max_tokens=
messages=
```

所以 Batch API 并没有重新发明一套 Claude 请求格式。([Claude Platform][1])

---

# 4. `custom_id` 非常重要

每个请求都需要一个唯一：

```python
custom_id="question-1"
```

为什么？

因为结果**不保证按照你提交的顺序回来**。([Claude Platform][1])

比如你提交：

```text
question-1
question-2
question-3
```

最终可能：

```text
question-3 → result
question-1 → result
question-2 → result
```

所以千万不要：

```python
results[0] == requests[0]   # ❌ 不一定
```

而应该：

```text
custom_id
    ↓
找到原来的任务
```

例如：

```text
review_00001 → 这条评论
review_00002 → 那条评论
review_00003 → 另一条评论
```

可以把 `custom_id` 理解成：

> **每一道题的学号。**

---

# 5. 提交以后不会马上给你答案

创建 Batch：

```python
message_batch = client.messages.batches.create(...)
```

返回的是类似：

```json
{
    "id": "msgbatch_xxx",
    "processing_status": "in_progress"
}
```

注意：

```text
这不是 Claude 的答案。
```

而是：

> “任务收到，正在后台处理。”

这就是**异步 asynchronous**。

([Claude Platform][1])

---

# 6. 怎么知道做完没有？

需要查询：

```python
message_batch = client.messages.batches.retrieve(
    MESSAGE_BATCH_ID
)
```

例如每分钟查一次：

```python
import time
import anthropic

client = anthropic.Anthropic()

MESSAGE_BATCH_ID = "msgbatch_xxx"

while True:

    batch = client.messages.batches.retrieve(
        MESSAGE_BATCH_ID
    )

    if batch.processing_status == "ended":
        break

    print("Still processing...")

    time.sleep(60)

print("Done!")
```

流程：

```text
create batch
     ↓
in_progress
     ↓
你的程序等待
     ↓
retrieve()
     ↓
还没好
     ↓
retrieve()
     ↓
还没好
     ↓
retrieve()
     ↓
ended
     ↓
获取结果
```

这叫：

**Polling（轮询）**。

([Claude Platform][1])

---

# 7. Batch 里面每个请求有 4 种结果

最终每一个 request 都可能是：

| 状态          | 意思            |
| ----------- | ------------- |
| `succeeded` | 成功            |
| `errored`   | 出错            |
| `canceled`  | 被取消           |
| `expired`   | 24 小时内没处理完，过期 |

其中 `errored`、`canceled`、`expired` 的请求不会收费。([Claude Platform][1])

所以可能：

```text
10,000 requests

├── 9,950 succeeded
├──    30 errored
├──    10 canceled
└──    10 expired
```

并不是：

```text
其中一个失败
↓
整个 10,000 个全部失败
```

而是每个独立处理。

---

# 8. 怎么拿结果？

```python
for result in client.messages.batches.results(
    MESSAGE_BATCH_ID
):

    if result.result.type == "succeeded":

        print(
            result.custom_id,
            result.result.message.content
        )

    elif result.result.type == "errored":

        print(
            "Error:",
            result.custom_id
        )
```

结果文件本质上是：

```text
.jsonl
```

即 **JSON Lines**。

普通 JSON：

```json
[
    {"id": 1},
    {"id": 2},
    {"id": 3}
]
```

JSONL：

```json
{"id": 1}
{"id": 2}
{"id": 3}
```

一行一个 JSON。

为什么适合 Batch？

因为可能：

```text
100,000 个结果
```

你没必要一次：

```text
全部读进 RAM
```

而可以：

```text
读一行
↓
处理
↓
丢掉

读下一行
↓
处理
↓
丢掉
```

所以官方 SDK 的 `batches.results()` 也是以节省内存的方式逐条处理结果。([Claude Platform][1])

---

# 9. Batch 和 Streaming 完全不是一回事

你刚学完 Streaming，所以这个区别一定要搞清楚：

| Streaming  | Batch      |
| ---------- | ---------- |
| 一个请求边生成边返回 | 很多请求后台处理   |
| 用户等着看答案    | 用户不需要马上看   |
| 实时         | 异步         |
| 降低感知延迟     | 提高吞吐量、降低成本 |
| Chatbot 常用 | 离线任务常用     |

所以：

```text
用户：
“帮我解释 Transformer”

→ Streaming
```

因为用户就在屏幕前等。

而：

```text
老板：
“把数据库里 100 万条评论分类。”

→ Batch
```

因为没有人需要：

```text
第 1 条结果现在立刻给我！
```

---

# 10. Batch 不能 `stream=True`

这也非常合理。

官方明确不支持：

```python
stream=True
```

因为两个设计目标冲突：

```text
Streaming
= 我要马上看到结果

Batch
= 我不着急，后台处理就行
```

Batch 也不支持用于同步低延迟优化的 `speed` 参数，以及 `max_tokens=0`。([Claude Platform][1])

---

# 11. Batch 一次最多多少？

当前限制：

```text
最多 100,000 个 Messages 请求

或者

整个 Batch 最大 256 MB
```

哪个先达到就以哪个为准。

另外：

```text
处理窗口 = 最多 24 小时
```

24 小时还没处理的 request 会变成：

```text
expired
```

Batch 结果创建后可以获取 **29 天**。([Claude Platform][1])

---

# 12. Batch 也支持我们之前学的东西

这一点很好。

Batch 里面基本仍然是普通 Messages 请求，所以可以包含：

```text
Vision
System Prompt
多轮 conversation
Thinking
Tool Use
Server Tools
大多数 beta features
```

甚至服务器工具，例如：

```text
Web Search
Web Fetch
Code Execution
MCP Connector
Advisor
Tool Search
```

也可以运行。([Claude Platform][1])

所以你可以做：

```text
10,000 个 research tasks
          ↓
Batch
          ↓
Claude
 ├── web search
 ├── web fetch
 ├── reasoning
 └── answer
          ↓
10,000 个 results
```

---

# 13. Batch + Prompt Cache

这又和你前面学的 Prompt Cache 串起来了。

假设：

```text
10000 个问题
```

全部基于同一本 500 页 PDF：

```text
共同部分：

System Prompt
+
500 页 PDF

不同部分：

Question 1
Question 2
Question 3
...
```

如果每次：

```text
500页 PDF
↓
重新处理
```

非常贵。

于是：

```text
             Shared Context
                   ↓
             Prompt Cache
                   ↓
        ┌──────────┼──────────┐
        ↓          ↓          ↓
    Question1  Question2  Question3
```

而：

```text
Batch 折扣
+
Prompt Cache 折扣
```

是可以叠加的。官方指出，由于 Batch 是异步并发处理，缓存命中只能尽力保证；实际缓存命中率通常会随流量模式变化。对于共享上下文的批次，文档建议考虑 1 小时缓存时长来提高命中率。([Claude Platform][1])

---

# 14. 一个非常现实的 AI 项目例子

假设以后你做：

```text
RAG Evaluation System
```

你有：

```text
50,000 个 QA 测试样本
```

每一个都要 Claude 判断：

```text
answer_correct: bool
grounded: bool
citation_correct: bool
score: 0-5
reason: string
```

你完全不需要：

```python
for item in dataset:
    client.messages.create(...)
```

可以：

```text
Dataset
50,000 questions
      ↓
Structured Outputs
规定评分 Schema
      ↓
Batch API
一次提交
      ↓
Claude 后台并发 evaluation
      ↓
.jsonl results
      ↓
Pandas / Database
      ↓
算指标
```

这里你前面学的东西已经开始组合起来了：

```text
Batch Processing
+
Structured Outputs
+
Prompt Cache
+
Tool Use
```

这就是很典型的 LLM 工程工作流。

---

## Cheatsheet

| 概念                 | 人话               |
| ------------------ | ---------------- |
| Batch Processing   | 一次提交大量 Claude 请求 |
| Async              | 不等答案，后台处理        |
| `batches.create()` | 创建批任务            |
| `requests`         | 一堆 Messages 请求   |
| `custom_id`        | 每个请求的唯一编号        |
| `in_progress`      | 处理中              |
| `ended`            | Batch 处理结束       |
| `retrieve()`       | 查询 Batch 状态      |
| Polling            | 隔一会查询一次          |
| `results()`        | 获取结果             |
| JSONL              | 一行一个 JSON        |
| `succeeded`        | 成功               |
| `errored`          | 失败               |
| `canceled`         | 取消               |
| `expired`          | 24 小时未处理，过期      |
| 最大请求数              | **100,000**      |
| 最大 Batch           | **256 MB**       |
| 结果保留               | **29 天**         |
| 价格                 | **标准 API 的 50%** |
| Streaming          | ❌ Batch 不支持      |
| Prompt Cache       | ✅ 可以组合使用         |

最后把两个刚学的概念压成一句：

```text
Streaming
= “我有一个任务，我现在就要边生成边看。”

Batch
= “我有十万个任务，不着急，你后台慢慢算，便宜点。”
```

这两个基本就是 **在线 LLM 服务** 和 **离线 LLM 数据处理** 两种典型模式。([Claude Platform][1])

[Claude 官方：Batch Processing](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing?utm_source=chatgpt.com)

[1]: https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing "批处理 - Claude Platform Docs"