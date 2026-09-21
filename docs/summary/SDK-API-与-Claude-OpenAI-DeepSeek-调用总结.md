---
title: SDK、API 与 Claude / OpenAI / DeepSeek 调用总结
url: wikibar://summary/summary/SDK-API-与-Claude-OpenAI-DeepSeek-调用总结
source_type: summary
folder: summary
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T00:56:07.985485+00:00'
---

# SDK、API 与 Claude / OpenAI / DeepSeek 调用总结

## 1. SDK 是什么？

SDK = Software Development Kit，即「软件开发工具包」。

人话理解：

> SDK 就是某个平台为了方便程序员使用自己的服务，提前封装好的一套开发工具。

例如：

```python
from openai import OpenAI
```

这里使用的 `openai` Python 包就是 OpenAI 提供的 Python SDK。

SDK 往往会包含：

* API 调用封装
* 身份认证
* 参数和类型定义
* 返回结果解析
* 错误处理
* 一些辅助函数

因此开发者不需要自己处理大量底层 HTTP 请求。

---

## 2. API 和 SDK 的区别

API = Application Programming Interface，即「应用程序编程接口」。

可以简单理解为：

```text
API = 服务端规定“你应该怎么调用我”
SDK = 官方给你的工具，让你更方便地调用 API
```

类比：

```text
API ≈ 餐厅的点餐规则和菜单
SDK ≈ 餐厅提供的点餐 App
```

没有 SDK 时，调用一个 LLM API 可能需要自己：

```text
构造 HTTP 请求
    ↓
填写 API URL
    ↓
添加 Authorization / API Key
    ↓
构造 JSON
    ↓
发送 POST 请求
    ↓
获取 JSON
    ↓
解析返回结果
    ↓
处理错误
```

有 SDK 后，可以简化成：

```python
client.responses.create(...)
```

所以：

```text
Python 程序
    ↓
SDK
    ↓
API
    ↓
服务器
    ↓
LLM
```

---

# 3. Claude API

Anthropic 为 Claude 提供了自己的 Python SDK：

```python
import anthropic
```

创建客户端：

```python
client = anthropic.Anthropic(
    api_key="你的 Claude API Key"
)
```

然后调用 Claude：

```python
response = client.messages.create(
    model="你的 Claude 模型",
    max_tokens=1000,
    messages=[
        {
            "role": "user",
            "content": "你好"
        }
    ]
)
```

读取返回内容：

```python
for block in response.content:
    if block.type == "text":
        print(block.text)
```

或者在确定第一个内容块就是文本时：

```python
print(response.content[0].text)
```

整个流程：

```text
Python
 ↓
Anthropic SDK
 ↓
client.messages.create(...)
 ↓
Anthropic API
 ↓
Claude
 ↓
response
```

---

## 4. Claude 代码逐行理解

例如：

```python
import anthropic
```

意思是：

> 导入 Anthropic 官方提供的 Python SDK。

然后：

```python
client = anthropic.Anthropic()
```

意思是：

> 创建一个负责和 Anthropic API 通信的客户端对象。

接下来：

```python
message = client.messages.create(...)
```

意思是：

> 通过 Anthropic SDK 调用 Messages API，向 Claude 发送一次请求。

例如：

```python
messages=[
    {
        "role": "user",
        "content": "What should I search for?"
    }
]
```

这里：

```text
role = "user"
```

表示这句话来自用户。

```text
content = "..."
```

表示用户实际发送的内容。

而：

```python
max_tokens=1000
```

表示限制模型最多生成一定数量的 token。

最后：

```python
for block in message.content:
    if block.type == "text":
        print(block.text)
```

因为 Claude 返回的 `content` 可能由多个内容块组成，所以逐个遍历：

```text
如果 block 是 text
↓
读取 block.text
↓
打印
```

---

# 5. OpenAI API

OpenAI 有自己的 Python SDK：

```python
from openai import OpenAI
```

创建客户端：

```python
client = OpenAI(
    api_key="你的 OpenAI API Key"
)
```

然后可以通过 Responses API 调用模型：

```python
response = client.responses.create(
    model="你的 OpenAI 模型",
    input="你好"
)
```

读取结果：

```python
print(response.output_text)
```

整体：

```text
Python
 ↓
OpenAI Python SDK
 ↓
client.responses.create(...)
 ↓
OpenAI API
 ↓
GPT
 ↓
response.output_text
```

---

# 6. Claude 和 OpenAI 的区别

虽然两个平台的目标基本一样：

> Python → API → LLM → 返回结果

但 SDK 的具体语法由不同公司自己设计。

Claude：

```python
client.messages.create(...)
```

OpenAI：

```python
client.responses.create(...)
```

所以要特别注意：

```python
client.messages.create()
```

和：

```python
client.responses.create()
```

都不是 Python 自带语法。

它们分别是 Anthropic SDK 和 OpenAI SDK 定义的方法。

---

# 7. DeepSeek API

DeepSeek 的一个特点是：

> API 设计兼容 OpenAI 风格。

因此 Python 中可以直接使用 `openai` SDK：

```python
from openai import OpenAI
```

但是创建客户端的时候，需要告诉 SDK：

> 不要请求 OpenAI 的服务器，而是请求 DeepSeek 的服务器。

例如：

```python
client = OpenAI(
    api_key="你的 DeepSeek API Key",
    base_url="https://api.deepseek.com"
)
```

关键就是：

```python
base_url="https://api.deepseek.com"
```

它改变了请求的目标服务器。

概念上：

```text
OpenAI SDK
     ↓
base_url
     ↓
DeepSeek API
     ↓
DeepSeek 模型
```

所以这里虽然写：

```python
from openai import OpenAI
```

并不意味着调用的一定是 OpenAI 模型。

真正请求谁，还取决于：

```text
base_url
API Key
model
```

---

# 8. DeepSeek 的 OpenAI 兼容写法

例如 Chat Completions 风格：

```python
from openai import OpenAI

client = OpenAI(
    api_key="你的 DeepSeek API Key",
    base_url="https://api.deepseek.com"
)

response = client.chat.completions.create(
    model="你的 DeepSeek 模型",
    messages=[
        {
            "role": "user",
            "content": "你好"
        }
    ]
)

print(response.choices[0].message.content)
```

这里：

```python
client.chat.completions.create(...)
```

也是 SDK 提供的方法。

可以理解为：

```text
client
 ↓
chat
 ↓
completions
 ↓
create()
```

最终发送一个 API 请求。

---

# 9. 三个平台放在一起比较

| 项目              | Claude                     | OpenAI                      | DeepSeek                  |
| --------------- | -------------------------- | --------------------------- | ------------------------- |
| 公司              | Anthropic                  | OpenAI                      | DeepSeek                  |
| Python SDK      | `anthropic`                | `openai`                    | 可使用 `openai`              |
| 创建客户端           | `anthropic.Anthropic()`    | `OpenAI()`                  | `OpenAI(base_url=...)`    |
| 常见调用方式          | `client.messages.create()` | `client.responses.create()` | OpenAI 兼容接口               |
| 输入              | `messages=`                | `input=` 等                  | `messages=` / 兼容接口        |
| 文本输出            | `response.content[...]`    | `response.output_text`      | `response.choices[...]` 等 |
| 是否兼容 OpenAI SDK | 不直接使用                      | 原生                          | 是                         |

---

# 10. 最重要的理解：SDK 只是“包装”

比如：

```python
response = client.responses.create(
    model="...",
    input="你好"
)
```

看起来像普通 Python 函数调用。

实际上背后发生的是：

```text
Python
 ↓
OpenAI SDK
 ↓
把参数转换成 HTTP 请求
 ↓
发送到 API Server
 ↓
服务器调用 LLM
 ↓
LLM 生成结果
 ↓
服务器返回 JSON
 ↓
SDK 解析 JSON
 ↓
Python response 对象
```

所以 SDK 帮你隐藏了大量底层细节。

---

# 11. 为什么 DeepSeek 可以使用 OpenAI SDK？

因为 SDK 和真正的模型是两回事。

可以理解成：

```text
OpenAI SDK
= 一个会按照某种格式发送 HTTP 请求的 Python 工具
```

如果另外一个服务商告诉你：

> “我的 API 也支持这种格式。”

那么 OpenAI SDK 就可以向它发送请求。

于是：

```python
client = OpenAI(
    api_key="DS_KEY",
    base_url="DeepSeek API 地址"
)
```

相当于告诉 SDK：

```text
请求格式：
继续按照 OpenAI 风格组织

但是

请求目的地：
改成 DeepSeek
```

这就是所谓的：

> OpenAI-compatible API

---

# 12. API Key 是干什么的？

例如：

```python
client = OpenAI(
    api_key="sk-..."
)
```

API Key 可以理解成：

> 程序访问 API 时使用的“身份凭证”。

服务器收到请求后需要判断：

```text
你是谁？
↓
有没有 API 使用权限？
↓
账户有没有额度？
↓
应该把费用算到哪个账户？
```

所以 API Key 非常重要。

真实项目中通常不要直接写：

```python
api_key="sk-xxxx"
```

尤其不要上传到 GitHub。

更常见的是使用环境变量。

例如：

```text
OPENAI_API_KEY=...
```

然后 SDK 从环境变量中读取。

---

# 13. `client` 到底是什么？

这一点很重要。

例如：

```python
client = OpenAI()
```

这里 `client` 可以理解成：

> 一个负责和 OpenAI API 通信的 Python 对象。

所以之后：

```python
client.responses.create(...)
```

本质上是在告诉这个客户端：

> 帮我向 Responses API 发送一个请求。

Claude 同理：

```python
client = anthropic.Anthropic()
```

然后：

```python
client.messages.create(...)
```

就是：

> 让 Anthropic 客户端帮我向 Messages API 发送请求。

---

# 14. `response` 又是什么？

例如：

```python
response = client.responses.create(...)
```

这里：

```text
client = 请求发送者
response = API 返回结果
```

所以典型模式永远类似：

```python
client = ...
response = client.xxx.create(...)
print(response.xxx)
```

可以记成：

```text
创建客户端
↓
发送请求
↓
得到 response
↓
读取 response
```

---

# 15. LLM API 最基本的代码模式

以后看到 Claude、GPT、DeepSeek、Gemini 或其他模型 SDK，都可以先找这四部分：

```text
① import SDK

② 创建 client

③ client.xxx(...) 发送请求

④ 从 response 中读取模型输出
```

例如抽象成：

```python
import 某个SDK

client = 某个SDK.Client(
    api_key="..."
)

response = client.某个接口(
    model="某个模型",
    input="你好"
)

print(response.输出)
```

不同厂商主要只是：

```text
SDK 名字不同
API 地址不同
方法名不同
参数格式不同
response 结构不同
```

底层思想基本一样。

---

# 16. 和 Agent / Tool Calling 的关系

最基础的 LLM API 是：

```text
User
 ↓
LLM
 ↓
Text
```

例如：

```python
input="What should I search for?"
```

Claude/GPT 可以告诉你：

> 你应该搜索 `renewable energy latest developments`

但这不代表它真的执行了搜索。

如果希望模型：

```text
自己判断需要搜索
↓
调用 Search Tool
↓
获取搜索结果
↓
阅读结果
↓
继续推理
↓
生成答案
```

就开始进入：

> Tool Calling / Agent

因此可以把学习路线理解为：

```text
HTTP
 ↓
API
 ↓
SDK
 ↓
LLM API 调用
 ↓
Structured Output
 ↓
Tool Calling
 ↓
MCP / Tools
 ↓
Agent
 ↓
RAG / Multi-Agent / Agentic System
```

---

# 17. 一句话总结

```text
API = 服务端提供的程序接口
SDK = 帮你方便调用 API 的开发工具包
client = SDK 创建出来、负责和 API 通信的对象
API Key = 调用 API 的身份凭证
response = API 返回给程序的结果
model = 指定真正要调用的 LLM
base_url = 指定 API 请求发送到哪台服务器
```

最核心的数据流：

```text
你的 Python 代码
        ↓
       SDK
        ↓
      Client
        ↓
       API
        ↓
      LLM 模型
        ↓
       API
        ↓
     Response
        ↓
你的 Python 程序读取结果
```

理解这一层之后，再看 Claude、OpenAI、DeepSeek 的代码，本质上都是同一件事情：

> **用某个 SDK 构造 API 请求 → 调用指定 LLM → 得到 response → 从 response 中取出模型输出。**