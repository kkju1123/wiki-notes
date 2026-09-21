---
title: effort
url: wikibar://summary/summary/effort
source_type: summary
folder: summary
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T03:14:42.447985+00:00'
---

# effort
这页的 `effort` 可以直接理解成：

> **你告诉 Claude：“这个问题你要花多大力气处理？”**

它是在 **能力 / 思考深度 / token 消耗 / 速度 / 成本** 之间做权衡的参数。当前默认是 `high`。([Claude Platform][1])

## 1. 最基本怎么用

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-5",
    max_tokens=4096,
    messages=[
        {
            "role": "user",
            "content": "Analyze the trade-offs between microservices and monolithic architectures"
        }
    ],
    output_config={
        "effort": "medium"
    }
)

for block in response.content:
    if block.type == "text":
        print(block.text)
```

关键就是：

```python
output_config={
    "effort": "medium"
}
```

意思是：

> Claude，这个问题不用使出最大力气，用中等 effort 处理。

([Claude Platform][1])

---

## 2. 有哪些等级？

目前有：

```text
low
medium
high      ← 默认
xhigh
max
```

大概可以理解成：

| Effort   | 人话         | 适合                     |
| -------- | ---------- | ---------------------- |
| `low`    | 快点做，别想太多   | 简单任务、高吞吐               |
| `medium` | 认真一点，但控制成本 | 普通任务                   |
| `high`   | 好好想        | 复杂推理、Agent、Coding      |
| `xhigh`  | 深入想        | 困难 Coding、复杂 Agent、长任务 |
| `max`    | 能想多少想多少    | 极难问题、不在乎 token         |

而：

```python
# 不写
```

和：

```python
output_config={"effort": "high"}
```

行为相同，因为默认就是 `high`。并不是所有支持 effort 的模型都支持所有五档，尤其 `xhigh` 的支持范围更窄。([Claude Platform][1])

---

## 3. Effort 不是严格的 token 数量

这一点非常重要。

假设：

```python
output_config={"effort": "low"}
```

**不是：**

```text
Claude 最多只能思考 1000 tokens
```

而更像：

> “尽量少花计算和 token，但如果问题真的很难，你还是可以思考。”

所以官方把 effort 称为：

> **behavioral signal（行为信号）**

而不是严格 token budget。([Claude Platform][1])

可以理解：

```text
low
↓
Claude：这个问题我尽量快速解决


high
↓
Claude：我要认真分析


xhigh
↓
Claude：我要深入探索、可能多次调用工具


max
↓
Claude：token 不用省，尽可能做到最好
```

---

## 4. Effort 控制的不只是“思考”

这个特别容易误解。

`effort` 影响**整个输出过程的 token 使用**，包括：

```text
① thinking

② 最终回答

③ tool calls

④ tool 参数

⑤ 工具前后的解释
```

所以它不是单纯：

```text
effort = thinking level
```

而更接近：

```text
effort
   ↓
Claude 整个任务愿意投入多少工作量
```

([Claude Platform][1])

---

## 5. 对 Tool Calling / Agent 影响很明显

比如你让 Agent：

> 帮我分析整个 GitHub 项目，找 bug，然后修改。

### low

可能：

```text
看几个文件
↓
调用少量工具
↓
直接修改
↓
简单说一句完成
```

### xhigh

可能：

```text
分析项目结构
↓
搜索相关代码
↓
读取多个文件
↓
调用工具
↓
发现依赖关系
↓
再搜索
↓
修改
↓
运行测试
↓
发现错误
↓
继续修改
↓
重新测试
↓
详细总结
```

官方明确指出，低 effort 往往会减少并合并工具调用、少铺垫、完成后简短确认；高 effort 则可能调用更多工具、行动前解释计划、给出更完整的总结和代码注释。([Claude Platform][1])

所以对于 Agent：

```text
effort ↑

通常意味着

更多探索
更多工具调用
更深入分析
更多 token
更慢
更贵
```

---

## 6. `effort` 和 `thinking` 不是一个东西

这个很重要。

### thinking

控制：

> **Claude 是否/如何进行思考。**

### effort

控制：

> **Claude 整个任务投入多少工作量。**

所以：

```text
thinking
    ↓
要不要进入思考过程

effort
    ↓
整个回答要多努力
```

在 adaptive thinking 下，effort 还会影响 Claude **多经常思考、思考多深**。官方也特别提醒：

```python
output_config={"effort": "adaptive"}
```

❌ 是错的。

因为：

```text
adaptive = thinking 模式

low / medium / high / xhigh / max = effort
```

([Claude Platform][1])

---

## 7. `effort` 和 `max_tokens` 也不是一回事

例如：

```python
response = client.messages.create(
    model="claude-opus-5",
    max_tokens=4096,
    output_config={"effort": "xhigh"},
    messages=[...]
)
```

这里：

```text
effort = xhigh
```

是：

> 你可以很努力。

而：

```text
max_tokens = 4096
```

是：

> 但是整个输出最多就给你这么多 token 空间。

所以：

```text
effort
= 行为指导 / 想花多少力气

max_tokens
= 硬上限
```

这就可能出现：

```text
effort = xhigh
↓
Claude：我要深入思考！

max_tokens = 1000
↓
但是只有这么点空间

→ 很快撞上 max_tokens
```

因此官方对 `xhigh/max` 的复杂 Agent/Coding 场景建议给较大的 `max_tokens`；对部分 Opus 模型，文档甚至建议可以从 64k 开始再调整。([Claude Platform][1])

---

## 8. Effort 不等于回答长度

这个也特别重要。

你可能以为：

```text
low → 回答短
max → 回答特别长
```

不一定。

尤其官方明确指出，对 Opus 5：

> effort 控制的是**工作/思考量**，不能可靠地用来控制最终可见回答长度。

如果你想要：

> “只回答三句话。”

应该写 Prompt：

```python
messages=[
    {
        "role": "user",
        "content": "Explain transformers in no more than three sentences."
    }
]
```

而不是：

```python
output_config={"effort": "low"}
```

([Claude Platform][1])

所以：

```text
effort → 控制工作量

prompt → 控制回答风格、长度、格式
```

---

## 9. 可以动态调整 effort

例如你的 Agent 收到不同任务：

```python
def choose_effort(task_type):
    if task_type == "simple":
        return "low"

    elif task_type == "normal":
        return "medium"

    elif task_type == "coding":
        return "high"

    elif task_type == "hard_coding":
        return "xhigh"

    else:
        return "high"


effort = choose_effort("hard_coding")

response = client.messages.create(
    model="claude-opus-5",
    max_tokens=64000,
    messages=[
        {
            "role": "user",
            "content": "Analyze this large codebase and find the concurrency bug."
        }
    ],
    output_config={
        "effort": effort
    }
)
```

这其实非常适合 Agent 系统：

```text
简单问题
→ low

普通问题
→ medium

复杂 Coding
→ high / xhigh

极难任务
→ max
```

官方也建议根据自己的真实任务做评估，而不是无脑全部使用最高档。([Claude Platform][1])

---

## 10. 为什么不全部 `max`？

因为：

```text
max
↓
更多 token
↓
更慢
↓
更贵
```

而且不一定明显更好。

比如：

```text
用户：1+1 等于多少？
```

用：

```text
max
```

没有什么意义。

甚至官方提醒，`max` 在一些结构化输出或对智能不敏感的任务上可能出现 **overthinking（过度思考）**。([Claude Platform][1])

所以更合理的是：

```text
简单任务 → low

普通任务 → medium

复杂任务 → high

困难 Agent / Coding → xhigh

真正的前沿难题 → max
```

---

## 11. 和 Prompt Cache 有关系

这正好接你刚才问的 Prompt Cache。

如果一个长对话一直：

```python
output_config={"effort": "high"}
```

缓存前缀可以正常复用。

但下一轮突然把**顶层**设置改成：

```python
output_config={"effort": "low"}
```

它会影响渲染后的 prompt，因此早期的 Prompt Cache 前缀不能继续匹配。

所以对于依赖 Prompt Cache 的长对话，官方建议：

> 顶层 effort 最好保持不变。

部分新模型支持 **per-turn effort**，可以通过特殊的 system message 在对话中途改变 effort，同时保留缓存前缀；该能力目前是 beta。([Claude Platform][1])

例如官方这种形式：

```python
import anthropic

client = anthropic.Anthropic()

response = client.beta.messages.create(
    model="claude-fable-5-1",
    max_tokens=4096,
    output_config={"effort": "high"},
    messages=[
        {
            "role": "user",
            "content": "Plan a migration from SQLite to PostgreSQL in three short steps."
        },
        {
            "role": "assistant",
            "content": "1. Export the SQLite data. 2. Create the PostgreSQL schema. 3. Import the data and verify row counts."
        },

        # 从下一个 user turn 开始改成 low
        {
            "role": "system",
            "content": [],
            "output_config": {
                "effort": "low"
            }
        },

        {
            "role": "user",
            "content": "Summarize the plan in one sentence."
        }
    ],
    betas=[
        "mid-conversation-output-config-2026-07-01"
    ],
)

for block in response.content:
    if block.type == "text":
        print(block.text)
```

([Claude Platform][1])

## Cheatsheet

| 参数           | 人话                              |
| ------------ | ------------------------------- |
| `low`        | 少花力气，快、省钱                       |
| `medium`     | 平衡质量/速度/成本                      |
| `high`       | 认真处理，**默认值**                    |
| `xhigh`      | 深入处理困难 Agent/Coding             |
| `max`        | 尽可能发挥能力，不限制 effort 层面的 token 花费 |
| `effort`     | Claude 整个任务投入多少工作量              |
| `thinking`   | Claude 的思考模式                    |
| `max_tokens` | 输出 token **硬上限**                |
| Prompt       | 控制回答内容、格式、长度等                   |

最重要的关系：

```text
effort
= “你有多努力？”

thinking
= “你怎么/是否思考？”

max_tokens
= “最多允许你输出多少？”

prompt
= “我要你具体怎么做？”
```

以及代码最值得记：

```python
response = client.messages.create(
    model="claude-opus-5",
    max_tokens=4096,
    messages=[
        {
            "role": "user",
            "content": "Analyze this problem."
        }
    ],
    output_config={
        "effort": "medium"
    }
)
```

所以你可以把 `effort` 看成现在 Claude API 给开发者的一个**高级“算力/工作量旋钮”**：简单任务调低省钱提速，困难 Coding/Agent 调高换取更充分的处理。([Claude Platform][1])

[Claude 官方：Effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort?utm_source=chatgpt.com)

[1]: https://platform.claude.com/docs/zh-CN/build-with-claude/effort "Effort - Claude Platform Docs"