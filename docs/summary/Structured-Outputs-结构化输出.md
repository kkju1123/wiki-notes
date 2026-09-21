---
title: Structured Outputs（结构化输出）
url: wikibar://summary/summary/Structured-Outputs-结构化输出
source_type: summary
folder: summary
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T07:01:31.794219+00:00'
---

# Structured Outputs（结构化输出）
这页 **Structured Outputs（结构化输出）** 对你以后做 Agent/API 后端非常重要。它解决的问题一句话就是：

> **不要让 LLM“随便说”，而是强制它按照你规定的数据结构返回。**

比如你后端需要：

```json
{
  "name": "John Smith",
  "email": "john@example.com",
  "plan_interest": "Enterprise",
  "demo_requested": true
}
```

你最不希望 Claude 突然回答：

```text
Sure! Here's the information I found:

Name: John Smith
Email: john@example.com
...
```

因为人看没问题，**程序不好处理**。

Structured Outputs 就是告诉 Claude：

```text
我不要自由文本。

必须给我：
{
    name: string,
    email: string,
    plan_interest: string,
    demo_requested: boolean
}
```

而且 Claude Platform 当前把它分成两个互补功能：**JSON Output** 和 **Strict Tool Use**。([Claude Platform][1])

---

## 1. 为什么普通 Prompt 不够？

以前最简单的做法是：

```python
response = client.messages.create(
    model="claude-opus-5",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": """
            Extract the user's information.
            Return JSON with:
            name
            email
            age
            """
        }
    ]
)
```

你只是**请求** Claude 返回 JSON。

但模型可能返回：

```text
Here is the JSON you requested:

{
  "name": "Keke",
  "email": "xxx@example.com",
  "age": 27
}
```

甚至：

```json
{
  "name": "Keke",
  "email": "xxx@example.com"
}
```

`age` 没了。

或者：

```json
{
  "name": "Keke",
  "email": "xxx@example.com",
  "age": "twenty-seven"
}
```

你要求：

```text
age: integer
```

它却给：

```text
age: string
```

于是后端：

```text
LLM
 ↓
返回不稳定 JSON
 ↓
json.loads()
 ↓
💥 error
```

这就是 Structured Outputs 要解决的问题。官方明确列出的常见问题包括无效 JSON、缺少必填字段、类型不一致以及 schema 违规导致的重试。([Claude Platform][1])

---

# 2. Structured Outputs 的核心：Schema

`schema` 可以理解成：

> **数据结构说明书。**

例如我要：

```json
{
  "name": "Keke",
  "age": 27,
  "student": true
}
```

对应 JSON Schema：

```python
{
    "type": "object",

    "properties": {
        "name": {
            "type": "string"
        },

        "age": {
            "type": "integer"
        },

        "student": {
            "type": "boolean"
        }
    },

    "required": [
        "name",
        "age",
        "student"
    ],

    "additionalProperties": False
}
```

人话：

```text
整个东西必须是 object
        ↓
name 必须 string
age 必须 integer
student 必须 boolean
        ↓
三个字段必须全部出现
        ↓
不能偷偷增加其他字段
```

---

# 3. Claude 怎么用？

官方现在的 API 写法是：

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-5",
    max_tokens=1024,

    messages=[
        {
            "role": "user",
            "content": """
            Extract the information:
            Keke is 27 years old and is a student.
            """
        }
    ],

    output_config={
        "format": {
            "type": "json_schema",

            "schema": {
                "type": "object",

                "properties": {
                    "name": {
                        "type": "string"
                    },

                    "age": {
                        "type": "integer"
                    },

                    "student": {
                        "type": "boolean"
                    }
                },

                "required": [
                    "name",
                    "age",
                    "student"
                ],

                "additionalProperties": False
            }
        }
    }
)

print(
    next(
        block.text
        for block in response.content
        if block.type == "text"
    )
)
```

Claude 会返回类似：

```json
{
  "name": "Keke",
  "age": 27,
  "student": true
}
```

这里最值得记的就是：

```python
output_config={
    "format": {
        "type": "json_schema",
        "schema": {...}
    }
}
```

官方已经把旧 beta 版的 `output_format` API 参数迁移到 `output_config.format`，而且不再要求旧的 structured-output beta header。([Claude Platform][1])

---

# 4. `required` 是什么？

比如：

```python
"required": [
    "name",
    "email"
]
```

就是：

```text
name 不能没有
email 不能没有
```

例如：

```json
{
  "name": "Keke"
}
```

❌ 不符合 schema。

必须：

```json
{
  "name": "Keke",
  "email": "xxx@example.com"
}
```

---

# 5. `additionalProperties: False` 是什么？

这个很重要。

假设 schema 只允许：

```text
name
email
```

但 Claude 返回：

```json
{
  "name": "Keke",
  "email": "xxx@example.com",
  "favorite_food": "hotpot"
}
```

如果：

```python
"additionalProperties": False
```

那么：

```text
favorite_food
```

是不允许出现的。

也就是：

> **只能给我规定好的字段，不许自己发挥。**

Claude 的 Structured Outputs 要求对象的 `additionalProperties` 为 `false`。([Claude Platform][1])

---

# 6. 更推荐：Pydantic

如果你以后用 **Python + FastAPI + Agent**，这一块尤其值得学。

因为你其实不用天天手写巨大 JSON Schema。

可以直接：

```python
from pydantic import BaseModel
from anthropic import Anthropic


class ContactInfo(BaseModel):
    name: str
    email: str
    plan_interest: str
    demo_requested: bool


client = Anthropic()

response = client.messages.parse(
    model="claude-opus-5",
    max_tokens=1024,

    messages=[
        {
            "role": "user",
            "content": """
            John Smith (john@example.com)
            is interested in our Enterprise plan
            and wants to schedule a demo.
            """
        }
    ],

    output_format=ContactInfo
)

print(response.parsed_output)
```

这里：

```python
class ContactInfo(BaseModel):
```

相当于直接定义：

```text
我想要的数据长什么样
```

然后：

```python
response.parsed_output
```

已经是解析后的 Python 对象。官方目前把 Python 的 `client.messages.parse()` 列为推荐的 SDK 便利方法；它会把 Pydantic 模型转换成 schema、验证响应，并提供 `parsed_output`。([Claude Platform][1])

所以你可以：

```python
contact = response.parsed_output

print(contact.name)
print(contact.email)
print(contact.plan_interest)
print(contact.demo_requested)
```

而不是自己：

```python
import json

data = json.loads(response.content[0].text)
```

---

# 7. 为什么这对 FastAPI 很舒服？

因为 FastAPI 本来就大量使用 Pydantic。

例如：

```python
class UserInfo(BaseModel):
    name: str
    age: int
    interests: list[str]
```

你的系统就可以形成：

```text
用户自然语言
      ↓
Claude
      ↓
Structured Output
      ↓
Pydantic UserInfo
      ↓
Python Backend
      ↓
Database / API / Agent
```

这比：

```text
LLM
 ↓
一大段自然语言
 ↓
正则表达式提取
 ↓
JSON parse
 ↓
字段检查
 ↓
出错再 retry
```

稳定很多。

---

# 8. Structured Outputs 其实有两种

这一页最容易漏掉的知识点就是：

```text
Structured Outputs
        │
        ├── JSON Outputs
        │
        └── Strict Tool Use
```

### JSON Outputs

控制：

> **Claude 最终“说什么格式”。**

例如：

```json
{
  "answer": "...",
  "confidence": 0.95
}
```

使用：

```python
output_config={
    "format": {...}
}
```

### Strict Tool Use

控制：

> **Claude 调用工具时，参数必须是什么格式。**

例如你的工具：

```python
{
    "name": "search_flights",

    "strict": True,

    "input_schema": {
        "type": "object",

        "properties": {
            "destination": {
                "type": "string"
            },

            "date": {
                "type": "string",
                "format": "date"
            }
        },

        "required": [
            "destination",
            "date"
        ],

        "additionalProperties": False
    }
}
```

Claude 就不能乱调用：

```json
{
  "destination": 123,
  "banana": "hello"
}
```

而应该符合：

```json
{
  "destination": "Paris",
  "date": "2026-05-15"
}
```

官方对两者的区分非常清楚：JSON Output 管 **Claude 的响应格式**；Strict Tool Use 管 **Claude 的工具参数**。两者可以同时使用。([Claude Platform][1])

---

# 9. 这就是你之前看到的 Agent 项目为什么强调 Pydantic

你之前看到过这种 Agent 项目描述：

```text
Structured Output Agent

Enforce Pydantic JSON schemas
Validate tool responses
Retry on parse errors
Log validation failures
```

现在应该能串起来了。

本质就是：

```text
以前：

LLM
 ↓
自由发挥
 ↓
"我觉得应该调用天气工具，
地点大概是 Basel..."
 ↓
后端：？？？


现在：

LLM
 ↓
Schema
 ↓
{
    "location": "Basel"
}
 ↓
Pydantic validation
 ↓
Python function
```

所以 **Structured Output 是把 LLM 从“聊天机器人”变成可靠软件组件的重要一步。**

---

# 10. 它为什么能“保证”格式？

这里比 Prompt 更底层。

普通 Prompt 是：

```text
Please return valid JSON.
```

相当于：

> “Claude，拜托你遵守。”

Structured Outputs 则使用 **constrained decoding / 约束采样**。Claude Platform 会根据 schema 编译一个语法约束，限制生成过程只能产生符合结构的输出。([Claude Platform][1])

可以粗略理解成：

```text
普通生成：

下一个 token
 ↓
{ / Sure / Hello / The / I / ...
都可能


Structured Output：

Schema 规定这里必须开始 JSON object

下一个合法 token
 ↓
{
```

继续：

```text
{
  "age":
```

schema：

```text
age = integer
```

那么：

```text
27       ✅

"27"     ❌

"hello"  ❌
```

这就是它比：

```text
"Please output JSON"
```

可靠得多的原因。

---

# 11. Schema 第一次会稍微慢一点

因为 Claude Platform 要：

```text
JSON Schema
     ↓
编译
     ↓
生成约束语法
     ↓
Claude constrained decoding
```

第一次使用某个 schema 会有额外编译延迟。

编译后的语法会自动缓存 **24 小时（从最近一次使用算起）**，所以后续相同 schema 请求会快很多。改变 schema 结构，或者在同时使用工具时改变工具集合，会导致这个语法缓存失效；仅改变工具的 `name` 或 `description` 不会。([Claude Platform][1])

注意这个：

```text
Structured Output grammar cache
```

和刚才学的：

```text
Prompt Cache
```

不是完全同一个东西。

---

# 12. Structured Outputs 也不是 100% 所有情况都保证

正常完成：

```text
stop_reason = end_turn
```

通常：

```text
schema valid ✅
```

但两个我们之前刚学过的 `stop_reason` 特别重要。

### `refusal`

```text
Claude
 ↓
安全拒绝
 ↓
stop_reason = refusal
```

此时拒绝信息优先，所以输出**可能不符合你的 schema**。

### `max_tokens`

```text
JSON 正生成到一半

{
   "name": "Keke",
   "email": "...
                 ↑
             token 用完
```

于是：

```text
stop_reason = max_tokens
```

JSON 被截断，自然可能无效。

官方明确把 `refusal` 和 `max_tokens` 列为结构化输出可能不符合 schema 的例外情况。([Claude Platform][1])

所以生产代码仍然应该：

```python
if response.stop_reason == "refusal":
    # 处理 refusal

elif response.stop_reason == "max_tokens":
    # 增大 max_tokens / retry

else:
    # 使用 structured output
```

这正好把你前面学的 `stop_reason` 串起来了。

---

# 13. Schema 不是所有 JSON Schema 功能都支持

常见基础类型都支持：

```text
object
array
string
integer
number
boolean
null
```

也支持一些常见功能，比如：

```text
enum
const
required
$ref / $defs
部分 anyOf / allOf
date
email
uuid
uri
ipv4 / ipv6
```

但一些限制不直接支持，比如：

```text
minimum / maximum
minLength / maxLength
递归 schema
外部 $ref
复杂数组约束
```

如果直接给 API 一个不支持的 schema，可能得到 `400`。不过 Python 等 SDK 可以自动转换部分不支持的约束，例如把 `minimum` 之类从发送给模型的 schema 中移除、写进描述，再用原始 Pydantic 约束在客户端验证最终结果。([Claude Platform][1])

---

## 最后把你最近学的东西全部串起来

现在一个真正的 Agent Backend 可以长这样：

```text
User
 │
 ▼
Messages API
 │
 ├── Prompt Cache
 │     重复 context 更便宜/更快
 │
 ├── effort
 │     控制 Claude 投入多少工作量
 │
 ▼
Claude
 │
 ├── tool_use
 │       ↓
 │   strict: true
 │       ↓
 │   工具参数符合 Schema
 │       ↓
 │   Python Tool
 │       ↓
 │   tool_result
 │
 ▼
Claude
 │
 ├── stop_reason
 │
 │   ├─ tool_use → 执行工具
 │   ├─ refusal → 处理拒绝/fallback
 │   ├─ max_tokens → 处理截断
 │   └─ end_turn → 完成
 │
 ▼
Structured Output
 │
 │  Pydantic / JSON Schema
 ▼
可靠的 Python Object
 │
 ▼
FastAPI / DB / 前端
```

### Cheatsheet

| 概念                            | 人话                    |
| ----------------------------- | --------------------- |
| Structured Outputs            | 强制 LLM 按规定结构输出        |
| JSON Schema                   | 输出结构说明书               |
| `output_config.format`        | 规定最终回答 JSON 格式        |
| `type: "json_schema"`         | 使用 JSON Schema        |
| `required`                    | 哪些字段必须存在              |
| `additionalProperties: false` | 不允许偷偷增加字段             |
| Pydantic                      | 用 Python class 定义数据结构 |
| `messages.parse()`            | 自动 schema + 解析 + 验证   |
| `parsed_output`               | 已解析好的 Python 对象       |
| `strict: true`                | 强制工具调用参数符合 schema     |
| JSON Output                   | 管 **Claude 最后说什么格式**  |
| Strict Tool Use               | 管 **Claude 怎么调用工具**   |
| constrained decoding          | 从生成阶段限制只能产生合法结构       |
| `refusal`                     | 可能打破 schema           |
| `max_tokens`                  | JSON 可能被截断            |

最值得你记住的区别就是：

```text
Prompt：
“请你返回 JSON。”
→ 请求 Claude 遵守


Structured Outputs：
“你的输出只能符合这个 JSON Schema。”
→ 系统层面约束生成
```

以及做 Agent 时：

```text
strict tool use
= 让 Agent 的“动作”可靠

structured JSON output
= 让 Agent 的“最终结果”可靠
```

这两个组合起来，就是你之前看到的 **“LLM reliable, not random”** 背后的核心工程手段之一。([Claude Platform][1])

[Claude 官方：Structured Outputs](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs?utm_source=chatgpt.com)

[1]: https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs "结构化输出 - Claude Platform Docs"