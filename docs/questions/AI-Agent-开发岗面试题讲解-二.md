---
title: AI Agent 开发岗面试题讲解（二）
url: wikibar://summary/questions/AI-Agent-开发岗面试题讲解-二
source_type: summary
folder: questions
author: null
tags: []
summary: ''
fetched_at: '2026-09-21T12:00:51.905733+00:00'
---

# AI Agent 开发岗面试题讲解（二）

---

# 一、自我介绍与项目经历

## 1. 做一下自我介绍

### 面试官在考什么？

自我介绍不是让你重新念一遍简历。

面试官主要想快速知道三件事：

1. 你以前做过什么
2. 你现在主要会什么
3. 为什么你的经历和 Agent 岗位匹配

因此最好按照：

```text
背景
↓
技术能力
↓
Agent / LLM 项目
↓
为什么申请这个岗位
```

来组织。

### 面试回答思路

例如：

> 我目前是计算机相关专业硕士，之前有两年前端开发经验。读研之后我的学习和项目方向逐渐转向 AI 和后端工程，主要接触了机器学习、深度学习、强化学习以及分布式系统。
>
> 最近我重点在做 LLM 和 Agent 相关项目，比如 Agentic RAG。我使用 FastAPI 构建后端服务，通过向量数据库进行知识检索，同时结合 Agent 的 Tool Calling 完成不同任务，并使用 Docker、Kubernetes 做服务部署。
>
> 我现在比较感兴趣的是 Agent 和后端工程结合的方向，因为我觉得真正把 Agent 做成产品，不只是调用模型 API，还涉及检索、工具调用、状态管理、评测、服务部署和系统可靠性，这也是我希望继续深入的方向。

重点：

**不要花两分钟介绍学校课程。**

Agent 岗应该尽快把话题引向：

```text
LLM
Agent
RAG
Backend
Deployment
```

---

# 二、个人助理 Agent 方案设计

## 2. 如果实现一个可以查天气、规划路线的个人助理 Agent，怎么设计？

这是非常经典的 Agent System Design。

假设用户说：

> 帮我看看今天杭州天气怎么样，然后规划一下从杭州东站到西湖的路线。

系统需要完成两个任务：

```text
天气查询
+
路线规划
```

最简单架构：

```text
User
 ↓
Agent / LLM
 ↓
判断用户意图
 ↓
选择 Tool
 ├── Weather Tool
 ├── Route Tool
 └── Web Search Tool
 ↓
执行 Tool
 ↓
Observation
 ↓
LLM
 ↓
Final Answer
```

### 为什么需要 Agent？

因为传统程序通常需要我们自己写：

```python
if "天气" in query:
    weather()

elif "路线" in query:
    route()
```

但 Agent 的思想是：

> 把有哪些 Tool、每个 Tool 是干什么的告诉 LLM，让模型根据自然语言自己决定调用哪个 Tool。

例如定义两个工具：

```python
@tool
def get_weather(city: str):
    """查询指定城市今天的天气"""
    ...

@tool
def get_route(origin: str, destination: str):
    """规划两个地点之间的路线"""
    ...
```

模型看到：

```text
用户：
杭州今天天气怎么样？
```

结合 Tool Description：

```text
get_weather:
查询指定城市今天的天气
```

模型判断：

```text
这个问题需要 get_weather
```

于是产生 Tool Call。

---

# 三、从 LangChain / LangGraph 代码层面怎么实现？

如果使用 LangGraph，可以把系统理解成一个状态机。

最简单：

```text
START
  ↓
Agent Node
  ↓
需要 Tool？
 /       \
Yes       No
 ↓         ↓
Tool Node  END
 ↓
Agent Node
 ↓
END
```

Agent Node 负责：

```text
调用 LLM
+
决定下一步
```

Tool Node 负责：

```text
真正执行 Tool
```

---

## 3.1 State 是什么？

Agent 每运行一步，都需要保存当前状态。

例如：

```python
class State(TypedDict):
    messages: list
```

里面可能是：

```text
SystemMessage
HumanMessage
AIMessage
ToolMessage
```

比如：

```text
Human:
杭州今天天气怎么样？

AI:
调用 get_weather(city="杭州")

Tool:
杭州 25°C，小雨

AI:
杭州今天大约 25°C，有小雨……
```

这些 Messages 就组成了 Agent 当前的状态。

---

# 四、「杭州今日天气」不需要 RAG，怎么办？

这是很重要的一道题。

首先要理解：

**不是所有问题都应该走 RAG。**

RAG 解决的是：

```text
从自己的知识库中寻找信息
```

例如：

```text
公司年假是多少？
产品退款政策是什么？
内部 API 怎么使用？
```

这些适合：

```text
Query
 ↓
Vector DB
 ↓
Retrieve Documents
 ↓
LLM
```

但：

> 杭州今天多少度？

这是**实时信息**。

知识库里的天气信息很快就过期了。

所以应该：

```text
杭州今日天气
 ↓
Agent 判断
 ↓
Weather API / Web Search
```

而不是：

```text
杭州今日天气
 ↓
RAG
```

因此 Agent 系统实际上可能同时拥有：

```text
Retriever Tool
Weather Tool
Search Tool
Route Tool
Calculator Tool
...
```

LLM 根据问题选择。

---

# 五、模型怎么触发 Web Search Tool？

这里一定要理解一个非常关键的概念：

> LLM 本身通常不会真的去执行 Python 函数。

模型做的是：

**决定应该调用哪个 Tool，并生成结构化 Tool Call。**

例如用户：

```text
杭州今天天气怎么样？
```

系统告诉模型有：

```json
{
  "name": "get_weather",
  "description": "查询指定城市天气",
  "parameters": {
    "city": "string"
  }
}
```

模型可能输出逻辑上类似：

```json
{
  "name": "get_weather",
  "arguments": {
    "city": "杭州"
  }
}
```

注意：

**到这里 Tool 还没有执行。**

模型只是说：

> 我要调用 get_weather，参数是杭州。

---

# 六、模型不能执行 Tool，到底是谁执行的？

这是这一组题最重要的知识点。

记住一句：

> LLM 决定调用什么，Agent Runtime 真正执行 Tool。

完整流程：

```text
① User

杭州今天天气怎么样？

        ↓

② LLM

分析问题

        ↓

③ LLM 输出 Tool Call

get_weather(city="杭州")

        ↓

④ Agent Runtime

解析 Tool Call

        ↓

⑤ 本地执行函数 / 请求 API

get_weather("杭州")

        ↓

⑥ Tool 返回

25°C，小雨

        ↓

⑦ Agent Runtime

把结果作为 ToolMessage
重新放回模型上下文

        ↓

⑧ LLM

根据 Tool Result 生成自然语言

        ↓

⑨ User

杭州今天约 25°C，有小雨……
```

所以：

```text
LLM ≠ Tool Executor
```

而是：

```text
LLM
负责 Decision Making

Agent Runtime
负责 Orchestration

Tool
负责 Execution
```

这是 Agent 最核心的架构思想之一。

---

# 七、模型输出什么，让本地 Agent 执行 Tool？

模型输出的是一个：

```text
Structured Tool Call
```

例如：

```json
{
  "name": "get_weather",
  "arguments": {
    "city": "Hangzhou"
  }
}
```

真实 API 中一般还会包含：

```json
{
  "id": "call_123",
  "type": "function",
  "function": {
    "name": "get_weather",
    "arguments": "{\"city\":\"Hangzhou\"}"
  }
}
```

Agent Runtime 看到：

```text
name = get_weather
```

就在自己的 Tool Registry 中寻找：

```python
tools = {
    "get_weather": get_weather,
    "get_route": get_route
}
```

然后：

```python
tool = tools["get_weather"]

result = tool(city="Hangzhou")
```

这样才是真正执行。

---

# 八、这种能力叫什么？

叫：

# Function Calling / Tool Calling

核心思想：

> 让模型按照规定 Schema 生成结构化的函数调用请求，而不是直接生成自然语言答案。

例如定义：

```json
{
  "name": "get_weather",
  "description": "Get current weather",
  "parameters": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string"
      }
    },
    "required": ["city"]
  }
}
```

LLM 根据 Schema 生成：

```text
function = get_weather
city = Hangzhou
```

然后：

```text
Application / Agent Runtime
```

负责执行函数。

### 一句话面试答案

> Function Calling 本质上是让 LLM 根据预定义的 Tool Schema，决定是否调用某个函数，并生成符合 Schema 的结构化函数名和参数；真正的函数执行仍然由外部 Agent Runtime 或应用程序完成，然后执行结果再作为上下文返回给模型。

这句话非常值得记住。

---

# 九、MCP 是什么？

MCP：

```text
Model Context Protocol
```

先理解普通 Tool。

以前每个 Agent 都可能自己写：

```text
Agent A
├── GitHub Tool
├── Database Tool
└── Files Tool

Agent B
├── GitHub Tool
├── Database Tool
└── Files Tool
```

问题是：

**每个 Agent 都要重新实现不同系统的接入。**

于是 MCP 想解决：

> Agent / AI Application 与外部工具、资源之间的标准化连接问题。

可以简单理解为：

```text
以前：

Agent
 ↓
各种自定义 Integration
 ↓
GitHub / DB / Files / API


MCP：

Agent
 ↓
MCP Client
 ↓
统一协议
 ↓
MCP Server
 ↓
GitHub / DB / Files / API
```

---

# 十、MCP 和普通本地 Tools 很像，它到底解决了什么？

这题面试官是在看你是不是真的理解 MCP。

确实：

```text
Tool Calling
```

和：

```text
MCP Tool
```

最终都可能让模型调用：

```text
search_file()
query_database()
```

区别主要不在：

> 模型怎么调用。

而在：

> Tool 怎么被发现、描述、连接和复用。

普通 Tool：

```python
@tool
def search_database():
    ...
```

通常和你的 Agent Application 强绑定。

MCP 则提供统一协议。

例如：

```text
Claude
Codex
IDE
自己的 Agent
```

理论上都可以连接同一个：

```text
Database MCP Server
```

所以可以记：

```text
Function Calling
解决：
LLM 如何表达“我要调用工具”

MCP
解决：
AI Application 如何标准化连接和使用外部工具/资源
```

这个区别非常重要。

---

# 十一、Skills 是什么？

Skill 可以理解成：

> 给 Agent 封装好的“某类任务应该怎么完成”的能力模块。

例如：

```text
PDF Skill

Excel Skill

Code Review Skill

Research Skill
```

一个 Skill 里面可能包含：

```text
Instructions
Scripts
Templates
Examples
Resources
```

例如：

```text
pdf_skill/
├── SKILL.md
├── scripts/
├── templates/
└── examples/
```

`SKILL.md` 告诉 Agent：

```text
什么时候使用这个 Skill
应该按照什么流程做
可以使用什么工具
最终应该输出什么
```

---

# 十二、Skill 直接写 System Prompt 不就行了吗？

理论上可以。

例如把：

```text
如何分析 PDF
如何做 Excel
如何写报告
如何分析代码
...
```

全部塞进 System Prompt。

但是会出现一个问题：

```text
System Prompt
越来越大
越来越复杂
越来越贵
```

假设有 100 个 Skills：

```text
PDF
Excel
PPT
Research
Coding
Finance
...
```

用户只是问：

> 帮我分析 Excel。

模型根本没必要看到其他 99 个 Skill 的完整说明。

所以 Skill 的思想是：

```text
Progressive Disclosure
```

也就是：

```text
第一阶段

只让模型知道：

Skill Name
Skill Description


第二阶段

发现任务需要 Excel Skill

        ↓

加载 Excel Skill 的完整 Instructions
```

优势：

```text
减少 Context
降低 Token Cost
模块化
方便维护
方便复用
```

---

# 十三、模型怎么知道什么时候加载哪个 Skill？

依赖 Skill Metadata。

例如：

```yaml
name: pdf-analysis
description: Analyze PDF files, extract tables and summarize documents.
```

系统先把：

```text
Skill Name + Description
```

告诉模型。

模型看到用户：

> 帮我分析这个 PDF。

就可以判断：

```text
pdf-analysis
```

相关。

然后 Agent Runtime 加载：

```text
SKILL.md
```

进入 Context。

所以：

```text
Skill Metadata
      ↓
Skill Selection
      ↓
Load Skill Instructions
      ↓
Execute
```

---

# 十四、Skill 是文件夹，Agent 怎么加载？

假设：

```text
skills/
├── pdf/
│   └── SKILL.md
├── excel/
│   └── SKILL.md
└── research/
    └── SKILL.md
```

启动时首先扫描：

```text
skills/
```

但不一定加载所有完整内容。

只读取：

```text
name
description
```

形成 Skill Registry：

```python
skills = {
    "pdf": "Analyze PDF documents",
    "excel": "Analyze spreadsheets",
    "research": "Perform web research"
}
```

模型选择：

```text
excel
```

Agent Runtime 再读取：

```text
skills/excel/SKILL.md
```

把具体 Instructions 加入 Context。

完整流程：

```text
Scan Skill Directory
        ↓
Build Skill Registry
        ↓
LLM Select Skill
        ↓
Load SKILL.md
        ↓
Add Instructions to Context
        ↓
Execute Task
```

---

# 十五、Web 多用户、多 Session Agent 怎么实现？

现在把单机 Agent：

```text
User
 ↓
Agent
```

升级成：

```text
很多用户
+
每个用户很多 Conversation
```

核心数据模型：

```text
User
 ↓
Session
 ↓
Messages
```

例如：

```text
user_id = 1001

session_1
├── message_1
├── message_2
└── message_3

session_2
├── message_1
└── message_2
```

Backend 可以使用：

```text
FastAPI
```

例如：

```text
POST /chat
GET /sessions
GET /sessions/{session_id}
DELETE /sessions/{session_id}
```

数据库：

```text
users

sessions

messages
```

关系：

```text
User
1:N
Session

Session
1:N
Message
```

Agent 请求：

```text
Frontend
   ↓
FastAPI
   ↓
Authentication
   ↓
user_id + session_id
   ↓
读取 Conversation State
   ↓
Agent
   ↓
LLM / Tools
   ↓
保存 Message
   ↓
Response
```

---

# 十六、长会话 Context 怎么处理？

这是 Agent 面试高频题。

假设用户聊了：

```text
1000轮
```

不能每次都把：

```text
1000轮 × 所有 Token
```

全部发给模型。

因为：

```text
Context Window 有限
Token Cost 很高
Latency 增加
Noise 增加
```

最简单：

```text
Sliding Window
```

只保留最近：

```text
最近 N 条 Messages
```

更好的方案：

```text
Recent Messages
+
Conversation Summary
+
Long-term Memory
```

例如：

```text
System Prompt

Conversation Summary
"用户正在开发一个旅游 Agent……"

Relevant Long-term Memory

Recent 10 Messages

Current User Query
```

因此长对话可以：

```text
旧消息
 ↓
Summarization
 ↓
Conversation Summary

重要长期信息
 ↓
Memory Store

最近消息
 ↓
直接保留
```

---

# 十七、模型 Context 里面到底有什么？

这是非常容易被问的一道题。

可以简单理解：

```text
Context =
System Instructions
+
Developer Instructions
+
Conversation History
+
Relevant Memory
+
Retrieved Documents
+
Tool Definitions
+
Tool Results
+
Current User Message
```

例如：

```text
System Prompt

"You are a personal assistant."

        +

Tool Schema

get_weather(...)
get_route(...)

        +

Conversation History

User: ...
Assistant: ...

        +

Retrieved Context

Document Chunk 1
Document Chunk 2

        +

Tool Result

Weather = 25°C

        +

Current Query
```

最终这些信息一起进入模型的 Context Window。

注意：

> 不是所有信息每一次都必须放进去。

Agent Context Engineering 的重要工作就是：

**决定什么时候把什么信息放进 Context。**

---

# 十八、通过哪些渠道了解 AI 信息？

不要只说：

> 小红书、知乎。

最好分层回答。

### 第一层：官方

```text
OpenAI
Anthropic
Google DeepMind
Meta AI
Hugging Face
```

看：

```text
Blog
Documentation
Research
Release Notes
GitHub
```

### 第二层：论文

```text
arXiv
Papers with Code
Google Scholar
```

### 第三层：开发者社区

```text
GitHub
Hugging Face
Reddit
Hacker News
技术社区
```

### 第四层：实际使用

自己持续使用：

```text
ChatGPT
Claude
Codex
Cursor
各种 Agent 产品
```

然后思考：

```text
它为什么这么设计？
Tool Calling 怎么做？
Memory 怎么做？
Context 怎么做？
```

这比只看新闻更重要。

---

# 十九、生活和工作中有哪些 AI 使用场景？

可以从：

```text
Coding
Learning
Research
Productivity
```

讲。

例如：

### Coding

```text
代码生成
Debug
Code Review
Test Generation
Documentation
```

### Learning

```text
论文解释
概念学习
知识总结
Quiz
```

### Research

```text
Web Search
Information Extraction
Document Analysis
```

### Productivity

```text
邮件
日程
文件整理
数据分析
```

重点不是说：

> 我天天用 ChatGPT。

而是体现：

> 我知道什么时候 AI 有价值，以及什么时候应该通过 Tool / RAG / Agent 扩展模型能力。

---

# 二十、如果作为 Codex 开发者，怎么实现 Codex 核心能力？

这其实是一道综合 Agent System Design。

先问：

Codex 本质需要什么能力？

用户说：

```text
帮我修复这个 Bug。
```

系统需要：

```text
理解任务
↓
读取 Repository
↓
搜索相关代码
↓
理解 Dependency
↓
制定修改计划
↓
修改代码
↓
运行 Tests
↓
观察 Error
↓
继续修改
↓
Tests Pass
↓
返回结果
```

所以它本质是：

```text
Coding Agent
```

---

## 20.1 Tools

至少需要：

```text
read_file

write_file

edit_file

search_code

list_directory

run_command

run_tests

git_diff
```

模型本身不会：

```text
真的修改文件
真的运行 pytest
真的运行 git
```

还是前面那个核心思想：

```text
LLM
↓
Tool Call
↓
Agent Runtime
↓
执行 Tool
```

---

## 20.2 Agent Loop

核心：

```text
User Task
    ↓
Understand Repository
    ↓
Plan
    ↓
Tool Call
    ↓
Execute
    ↓
Observation
    ↓
Reason
    ↓
Tool Call
    ↓
Execute
    ↓
...
    ↓
Tests Pass
    ↓
Final Answer
```

伪代码：

```python
while step < max_steps:

    response = llm(context, tools)

    if response.has_tool_call:

        result = execute_tool(
            response.tool_call
        )

        context.append(result)

    else:

        return response
```

---

## 20.3 Context Management

代码库可能：

```text
100 MB
```

显然不能：

```text
整个 Repository → Context
```

所以需要：

```text
Repository Index
+
Code Search
+
Selective File Reading
```

模型根据任务逐步读取：

```text
README
↓
相关目录
↓
相关文件
↓
相关函数
```

而不是一次把所有代码塞进去。

---

## 20.4 Execution Environment

Coding Agent 最大的问题之一：

> 模型生成代码以后，怎么知道代码到底对不对？

答案：

**执行。**

例如：

```text
Generate Patch
 ↓
Apply Patch
 ↓
Run Tests
 ↓
Fail
 ↓
Read Error
 ↓
Modify Code
 ↓
Run Tests
 ↓
Pass
```

这就是一个典型：

```text
Agent Feedback Loop
```

---

## 20.5 Safety

Coding Agent 能执行：

```bash
rm
git
python
npm
curl
```

所以风险非常高。

需要：

```text
Sandbox
Permission Control
Command Allowlist / Risk Classification
Timeout
Resource Limit
User Confirmation
```

例如：

```text
read file
→ 自动允许

run unit test
→ 自动允许

修改项目代码
→ 根据权限策略

删除大量文件
→ 用户确认

访问敏感 Credential
→ 拒绝 / 权限检查
```

---

# 二十一、把这一整套知识串起来

这些面试题表面上在问：

```text
RAG
Tool
Function Calling
MCP
Skills
Memory
Web Backend
Codex
```

实际上都可以放进同一个 Agent 架构：

```text
                    User
                      ↓
                 Agent Runtime
                      ↓
              Context Construction
                      ↓
                     LLM
                ↙           ↘
           Final Answer     Tool Call
                               ↓
                         Tool Executor
                         ↙     ↓      ↘
                       RAG    MCP     API
                              Tools
                               ↓
                          Observation
                               ↓
                              LLM
                               ↓
                         Final Answer
```

一定要理解三个角色：

```text
LLM
= Reasoning / Decision Making

Agent Runtime
= Orchestration / State / Loop

Tool
= Execution / External Capability
```

然后再理解三个扩展：

```text
RAG
= 给模型提供外部知识

MCP
= 标准化连接外部工具和资源

Skills
= 模块化提供完成某类任务的方法和流程
```

最后是：

```text
Memory
= 保存跨轮 / 跨 Session 信息

Evaluation
= 判断 Agent 做得对不对

Safety
= 控制 Agent 什么能做、什么不能做
```

如果把这套关系真正理解了，很多 Agent 面试题即使以前没见过，也可以现场推出来。