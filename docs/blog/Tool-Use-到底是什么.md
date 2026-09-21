---
title: Tool Use 到底是什么
url: wikibar://summary/blog/Tool-Use-到底是什么
source_type: summary
folder: blog
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T08:20:35.873386+00:00'
---

# Tool Use 到底是什么
这页 **How tool use works** 是前面那些 Tool Calling 细节的“总纲”。它最重要的是把一件事讲清楚：

> **Claude 本身不会真的执行工具。Claude 负责“决定调用什么 + 生成参数”；真正执行工具的是你的程序，或者 Anthropic 的服务器。** ([Claude Platform][1])

这也是理解 Agent 最核心的一步。

## 1. Tool Use 到底是什么？

假设用户说：

```text
“帮我查一下 Basel 今天的天气。”
```

Claude 自己并不知道实时天气。

如果你给它一个工具：

```python
tools = [
    {
        "name": "get_weather",
        "description": "Get current weather for a city",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string"
                }
            },
            "required": ["city"]
        }
    }
]
```

你实际上是在告诉 Claude：

```text
你现在有一个能力：

get_weather(city)

输入必须：
{
    "city": string
}
```

Claude看到用户的问题后，可能返回：

```json
{
  "type": "tool_use",
  "id": "toolu_123",
  "name": "get_weather",
  "input": {
    "city": "Basel"
  }
}
```

注意：

**到这里 Claude 并没有查询天气。**

它只是在说：

> “你的程序，请帮我执行 `get_weather(city="Basel")`。”

官方把这个称为一个 **contract（契约）**：开发者定义有哪些操作、输入输出是什么样；Claude 决定什么时候调用、传什么参数。([Claude Platform][1])

---

# 2. 谁真正执行 Tool？

这页把 Tool 分成了 **3 类**。

### 第一类：你自己定义、你自己执行

这是最常见的：

```text
User-defined tools
```

比如你写：

```python
def get_weather(city):
    ...
```

或者：

```python
def query_database(sql):
    ...

def send_email(to, subject, body):
    ...

def search_products(query):
    ...
```

流程：

```text
Claude
   ↓
tool_use

你的 Python 程序
   ↓
真正执行函数

tool_result
   ↓
Claude
```

所以：

```text
Claude = 决策者

Python = 执行者
```

Claude甚至**看不到你的函数实现**。它只知道你给它看的 `name / description / input_schema`，以及执行之后返回的结果。([Claude Platform][1])

---

## 3. 第二类：Anthropic 定义 Schema，但还是你执行

例如：

```text
bash
text_editor
computer
browser
memory
```

Anthropic已经把这些工具的接口设计好了。

例如 Claude可能输出类似：

```text
bash
 ↓
{
    "command": "ls -la"
}
```

但是：

> **仍然是你的环境真正执行这个命令。**

区别只是：

```text
普通 client tool：

Schema 是你设计的
执行也是你


Anthropic-defined client tool：

Schema 是 Anthropic 设计好的
执行还是你
```

为什么要用 Anthropic 预定义的接口？

因为这些 tool schema 是模型训练时已经熟悉的接口，官方称 Claude 针对这些精确工具签名进行了大量优化，因此通常比你自己重新发明一个同功能接口更可靠。([Claude Platform][1])

---

# 4. 第三类：Anthropic Server Tool

这类就完全不同：

```text
web_search
web_fetch
code_execution
tool_search
```

这些是：

> **Anthropic 服务器帮你执行。**

所以：

```text
Claude
 ↓
server_tool_use
 ↓
Anthropic Server
 ↓
执行 web_search
 ↓
搜索结果
 ↓
Claude
 ↓
继续分析
 ↓
Final Answer
```

你的 Python 程序通常不需要：

```python
execute_web_search(...)
```

也不需要手动返回：

```text
tool_result
```

因为 Anthropic 在服务器内部已经完成了这个循环。([Claude Platform][1])

---

# 5. 三种 Tool 一张图就明白

```text
                 Claude
                   │
          “我要调用一个工具”
                   │
       ┌───────────┼────────────┐
       ↓           ↓            ↓

 User-defined   Anthropic     Server Tool
    Tool       defined Tool

 Schema:你      Schema:官方     Schema:官方
 执行:你        执行:你         执行:官方

       ↓           ↓            ↓

   tool_use      tool_use    server_tool_use

       ↓           ↓            ↓

  Python代码     你的环境     Anthropic服务器
```

这是这一页最值得记住的分类。([Claude Platform][1])

---

# 6. Client Tool 的 Agent Loop

这个是整页最重要的部分。

假设：

```text
User:
“Basel天气怎么样？适合出去散步吗？”
```

第一次请求：

```python
response = client.messages.create(
    model="...",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": "What's the weather in Basel? Is it good for walking?"
        }
    ],
    tools=tools
)
```

Claude：

```text
我不知道实时天气
↓
但我有 get_weather
↓
调用它
```

返回：

```json
{
  "type": "tool_use",
  "id": "toolu_123",
  "name": "get_weather",
  "input": {
    "city": "Basel"
  }
}
```

同时：

```python
response.stop_reason
```

是：

```text
tool_use
```

意思：

> **我现在停下来，因为我要等你执行工具。**

([Claude Platform][1])

---

# 7. 你的程序执行

例如：

```python
result = get_weather("Basel")
```

得到：

```text
18°C, sunny
```

然后必须告诉 Claude：

```text
刚才你让我执行的工具，结果回来了。
```

所以发送：

```python
{
    "role": "user",
    "content": [
        {
            "type": "tool_result",
            "tool_use_id": "toolu_123",
            "content": "18°C, sunny"
        }
    ]
}
```

这里：

```python
tool_use_id
```

非常重要。

它表示：

```text
toolu_123 请求
      ↕
toolu_123 结果
```

Claude就知道：

> 这个结果对应我刚才哪一个工具调用。

---

# 8. 为什么 `tool_result` 是 `role="user"`？

你之前专门问过这个。

虽然：

```python
{
    "role": "user"
}
```

但并不是说：

> 人类亲手输入了天气结果。

它只是 Messages API 协议规定：

```text
assistant
→ 发出 tool_use

user
→ 返回 tool_result

assistant
→ 根据结果继续
```

所以完整历史：

```text
user:
Basel天气怎么样？

assistant:
tool_use(get_weather)

user:
tool_result("18°C sunny")

assistant:
天气晴朗，18°C，很适合散步。
```

---

# 9. 为什么必须把 assistant 的 tool_use 也保留下来？

下一轮不能只发送：

```text
tool_result
```

完整 conversation 应该是：

```python
messages.append({
    "role": "assistant",
    "content": response.content
})

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

因为 Claude需要看到：

```text
我之前：

tool_use
id = toolu_123

然后：

tool_result
id = toolu_123
```

才能把调用和结果对应起来。

---

# 10. 然后 Claude 可能继续调用工具

这是 Agent 和“一次函数调用”的关键区别。

Claude拿到天气以后可能觉得：

```text
天气知道了
↓
但用户还问适不适合散步
↓
我还需要空气质量
```

于是：

```text
tool_use(get_air_quality)
```

你的程序：

```text
执行
↓
tool_result
```

Claude：

```text
还想找附近公园
↓
tool_use(search_parks)
```

于是：

```text
Claude
   ↓
Tool
   ↓
Result
   ↓
Claude
   ↓
Tool
   ↓
Result
   ↓
Claude
   ↓
Final Answer
```

这就是：

> **Agent Loop。**

官方的定义非常直接：只要 `stop_reason == "tool_use"`，你的应用就执行工具、返回结果，然后继续；直到 Claude 不再要求工具。([Claude Platform][1])

---

# 11. 所以代码核心就是一个 `while`

这是非常值得你真正记住的代码结构：

```python
import anthropic

client = anthropic.Anthropic()

messages = [
    {
        "role": "user",
        "content": "What's the weather in Basel?"
    }
]

while True:

    response = client.messages.create(
        model="...",
        max_tokens=1024,
        messages=messages,
        tools=tools
    )

    # Claude 已经不需要工具了
    if response.stop_reason != "tool_use":

        for block in response.content:
            if block.type == "text":
                print(block.text)

        break

    # 保存 Claude 的 tool_use
    messages.append({
        "role": "assistant",
        "content": response.content
    })

    tool_results = []

    for block in response.content:

        if block.type == "tool_use":

            if block.name == "get_weather":
                result = get_weather(
                    block.input["city"]
                )

            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": result
            })

    # 把结果返回 Claude
    messages.append({
        "role": "user",
        "content": tool_results
    })
```

核心其实就：

```text
while Claude wants tool:

    Claude决定调用什么

    ↓

    Python执行

    ↓

    返回结果

    ↓

    Claude继续
```

这就是一个最基本的 **Agent Runtime**。

---

# 12. 为什么判断 `stop_reason`？

你前面学的 stop reason 到这里终于真正派上用场：

```python
if response.stop_reason == "tool_use":
```

意味着：

```text
Claude：
“我还没回答完，我需要你的程序干活。”
```

于是：

```text
执行工具
→ tool_result
→ continue
```

而如果：

```text
end_turn
```

通常：

```text
Claude：
“我已经回答完了。”
```

所以：

```text
break
```

其他原因，例如：

```text
max_tokens
refusal
stop_sequence
```

则需要你的应用分别处理。官方也是以 `stop_reason` 作为 Agent loop 的控制键。([Claude Platform][1])

---

# 13. Server Tool 为什么不需要这个 while？

假设使用：

```text
web_search
```

流程可能是：

```text
你的 Python
     ↓
Claude API

     ↓
Claude：
“我要搜索”

     ↓
web_search
     ↓
搜索结果

     ↓
Claude：
“还不够，我再搜”

     ↓
web_search
     ↓
搜索结果

     ↓
Claude：
“够了”

     ↓
Final Answer

     ↓
你的 Python
```

中间整个：

```text
Claude → search → Claude → search → Claude
```

都发生在 Anthropic Server。

你的程序只：

```text
发一次 request
↓
收最终 response
```

这叫：

> **Server-side loop。**

([Claude Platform][1])

---

# 14. 但是 Server Loop 也可能暂停

这就是你之前学过的：

```text
pause_turn
```

例如：

```text
Claude
 ↓
web_search
 ↓
Claude
 ↓
web_search
 ↓
Claude
 ↓
web_search
 ↓
……
```

服务器内部循环达到迭代限制：

```text
stop_reason = "pause_turn"
```

意思不是：

> 做完了。

而是：

> **这轮暂时停了，但工作还没完成。**

此时把暂停的 response 放回 conversation，重新请求，让 Claude 从那里继续。([Claude Platform][1])

所以你之前那个区别现在应该特别清楚：

```text
tool_use
→ 等“你的程序”执行 Client Tool

pause_turn
→ Server Tool 内部循环暂停，需要继续请求
```

---

# 15. 什么情况下应该使用 Tool？

官方给的判断非常实用。

### 有副作用的事情

例如：

```text
发送 Email
修改数据库
创建 GitHub Issue
写文件
付款
```

Claude光“说”没用。

必须：

```text
Tool
```

真正执行。([Claude Platform][1])

### Claude自己不知道的外部/实时信息

例如：

```text
今天的天气
实时股票价格
你的数据库
公司内部 API
用户订单
最新网页
```

需要：

```text
Tool
↓
External World
```

### 调用已有系统

例如：

```text
Natural Language

“帮我查订单 123”

       ↓

Claude

       ↓

get_order(
    id=123
)

       ↓

Database
```

Tool实际上就是：

> **LLM 和传统软件系统之间的桥。**

([Claude Platform][1])

---

# 16. 一个特别好的判断标准

官方这一句话的工程味很强：

> 如果你正在写正则表达式，从 Claude 的自然语言输出里提取它的“决定”，那这个决定很可能本来就应该设计成 Tool Call。 ([Claude Platform][1])

比如不要：

```text
Claude：

"I think we should search_weather
and the city is Basel."
```

然后：

```python
re.findall(...)
```

❌。

应该直接：

```json
{
    "name": "search_weather",
    "input": {
        "city": "Basel"
    }
}
```

也就是：

```text
自由文本
❌

Structured Tool Call
✅
```

这和你前面学的 **Structured Outputs** 是完全同一个工程思想：

> **能结构化，就不要靠解析自然语言。**

---

# 17. 什么情况不需要 Tool？

比如：

```text
“把这段英文翻译成中文。”

“解释 Transformer。”

“总结这篇文章。”

“1+1 等于多少？”
```

模型自己就能完成。

没必要：

```text
Claude
 ↓
Tool
 ↓
Network
 ↓
Tool Result
 ↓
Claude
```

因为每次 Tool Call 都会增加额外 round trip 和 latency。([Claude Platform][1])

---

## 把你最近学的全部串起来

现在其实已经可以画出一个完整 Agent：

```text
                     User
                       │
                       ▼
                 Messages API
                       │
                 Prompt Cache
                       │
                       ▼
                    Claude
                       │
              Thinking / Effort
                       │
                       ▼
             “我需要外部信息吗？”
                 │            │
                NO           YES
                 │            │
                 │        tool_use
                 │            │
                 │      Client Tool?
                 │        │       │
                 │       YES      NO
                 │        │       │
                 │     Python   Server Tool
                 │        │       │
                 │   tool_result  Anthropic
                 │        │       │
                 └────────┴───────┘
                          │
                          ▼
                       Claude
                          │
                    还需要 Tool？
                     │         │
                    YES        NO
                     │         │
                     └──loop   ▼
                              Answer
                                │
                       Structured Output
                                │
                            Streaming
                                │
                                ▼
                              User
```

这就是你现在学的 Claude API 各个模块开始真正组合起来的地方。

### Cheatsheet

| 概念                            | 人话                                  |
| ----------------------------- | ----------------------------------- |
| Tool                          | Claude连接外部世界的能力                     |
| Tool Schema                   | 告诉 Claude 工具叫什么、参数是什么               |
| `tool_use`                    | Claude要求你的程序执行工具                    |
| `tool_result`                 | 你的程序把执行结果返回 Claude                  |
| `tool_use_id`                 | 把调用和结果对应起来                          |
| Client Tool                   | Claude决定，你的程序执行                     |
| User-defined Tool             | Schema你写，执行你做                       |
| Anthropic-defined Client Tool | Schema官方提供，执行你做                     |
| Server Tool                   | Anthropic服务器直接执行                    |
| `server_tool_use`             | Server Tool 调用                      |
| `stop_reason="tool_use"`      | Claude停下来等你的 Tool Result            |
| `pause_turn`                  | Server Tool loop暂时暂停                |
| Agent Loop                    | Claude → Tool → Result → Claude → … |
| `end_turn`                    | 通常代表 Claude 已经完成                    |

你现在最值得牢牢记住的其实只有这个：

```text
Claude 不执行 Client Tool。

Claude：
“调用 get_weather(city='Basel')”

        ↓

你的 Python：
真正执行 get_weather()

        ↓

tool_result

        ↓

Claude：
读取结果继续思考

        ↓

如果还需要工具 → 再调用

        ↓

不需要了 → Final Answer
```

**这个 `Claude → tool_use → 你的代码 → tool_result → Claude` 循环，就是最基础的 Agent。** ([Claude Platform][1])

[Claude 官方：工具使用的工作原理](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/how-tool-use-works?utm_source=chatgpt.com)

[1]: https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/how-tool-use-works "工具使用的工作原理 - Claude Platform Docs"