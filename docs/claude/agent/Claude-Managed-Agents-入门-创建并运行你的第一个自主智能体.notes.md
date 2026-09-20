# Claude Managed Agents 入门：创建并运行你的第一个自主智能体

*原文: [https://platform.claude.com/docs/en/managed-agents/quickstart](https://platform.claude.com/docs/en/managed-agents/quickstart) · 来源: web · 生成时间: 2026-09-20T06:59:55.789008+00:00*

## 背景

Claude Managed Agents 把大模型调用从单次问答扩展为可执行任务的智能体。传统模式下，开发者要自己串起模型、工具、执行环境和状态管理；该服务将它们封装成托管 API，用于构建能写代码、跑命令、查网页、产出文件的长任务应用。

## 痛点

如果没有这类托管能力，你需要自行实现工具调用循环、沙箱隔离、会话持久化和事件流，复杂度高且容易在安全与可靠性上出问题；尤其在多轮工具执行和前端实时展示时，开发成本会明显上升。

## 解决办法

核心做法是把一个智能体拆成四个概念：Agent 是静态配置（模型、system prompt、tools、MCP servers、skills），Environment 是沙箱运行配置（Anthropic 托管云沙箱或自托管），Session 是一次具体任务运行实例，Events 是应用与智能体间的实时消息协议。创建 Agent 和 Environment 后，用它们的 ID 创建 Session；向 Session 发送用户消息，服务端会按环境配置构建沙箱，进入 Agent loop 让 Claude 决定调用哪些工具，在沙箱内执行 bash、文件读写等操作，并把结果以事件流实时返回，最后发出 session.status_idle 表示空闲。可以类比成：你给一个带独立电脑和工具箱的同事下发任务，他在工位里自己写脚本、跑命令、检查产出，同时不断向你汇报进展。

## 关键代码示例

```python
# 概念性示例：具体方法名以官方 SDK/CLI 为准\n# 1. 创建 Agent：绑定模型、系统提示词和预置工具集\nagent = client.agents.create(\n    name='data-analyst',\n    model='claude-sonnet-4-5',\n    system='You are a data analyst. Write and run Python code in the sandbox.',\n    tools=[{'type': 'agent_toolset_20260401'}],\n)\n\n# 2. 创建 Environment：指定托管云沙箱（也可配置为自托管）\nenv = client.environments.create(\n    name='python-sandbox',\n    type='cloud',\n)\n\n# 3. 创建 Session：把静态配置实例化为一次运行\nsession = client.sessions.create(\n    agent_id=agent.id,\n    environment_id=env.id,\n)\n\n# 4. 发送消息并流式处理事件\nfor event in client.sessions.stream(\n    session.id,\n    '分析 data.csv，生成 report.md',\n):\n    print(event.type, event)  # user/tool_result/status 等
```

这段代码按四个核心步骤组织：先创建 Agent 定义模型、提示词和工具集；再创建 Environment 决定沙箱类型；然后通过 agent.id 和 environment.id 创建 Session，表示一次具体运行；最后向 Session 发送消息并迭代流式事件。它表达了 Managed Agents 的核心执行模型：静态配置被 Session 实例化，服务端负责沙箱、工具调用循环和事件流，客户端只通过事件感知进度。这里的代码是概念示意，真实 SDK 方法名和参数应以官方文档为准。

## 关键流程

1. 安装 CLI/SDK 并配置 API key
2. 创建 Agent：定义模型、system prompt 和可用工具
3. 创建 Environment：定义托管云沙箱或自托管沙箱
4. 保存 agent.id 和 environment.id，创建 Session
5. 发送用户消息并流式处理事件，直至收到 status_idle

## 关键点

- Agent 是静态配置而 Session 是运行实例，理解这个区别能帮你设计可复用的智能体配置，而不是每次任务都重新声明模型和工具。
- Environment 把工具执行限制在沙箱中，云沙箱和自托管沙箱的取舍直接影响安全边界、网络访问和数据驻留。
- Events 是客户端与智能体之间的核心协议，实时流式事件让长任务可观测、可干预，也能被前端框架直接渲染。
- agent_toolset_20260401 提供一组预置工具，减少逐个配置 bash、文件操作、搜索的重复工作，是快速起步的关键。
- session.status_idle 是判断智能体是否完成工作的重要信号，不同于 HTTP 响应结束，因为 Agent loop 可能多次工具调用后才空闲。

## 对比与权衡

- 相比手写 ReAct/tool-calling 循环并自己搭建沙箱，Claude Managed Agents 在环境隔离、事件流和会话管理上更省心，但可定制性与平台无关性不如自建编排方案。
- 相比普通 Chat Completions API，Managed Agents 把工具执行和会话状态托管在服务端，更适合需要长时间、多步骤运行的任务；但如果你只需要单轮文本生成，引入 Session/Environment 会偏重。

## 自测问题

**问: Agent、Environment、Session 和 Events 分别是什么？**

Agent 是模型+提示词+工具+MCP+skills 的静态配置；Environment 是沙箱运行配置；Session 是 Agent 在某个 Environment 中的一次运行实例；Events 是应用与智能体之间的消息协议。可以类比 Docker 镜像、容器与日志流来理解。

**问: 为什么工具调用要放在沙箱里执行？**

bash、文件写入等工具可能修改系统状态或访问敏感数据，沙箱提供隔离边界。云沙箱免运维但数据在平台侧；自托管沙箱满足数据驻留和自定义网络要求，但需要自己维护基础设施。

**问: 如何在一个聊天前端里实时展示工具调用过程？**

不要把会话做成一次性 HTTP 请求，而是订阅事件流。事件流中包含用户输入、工具结果和状态更新，前端可以把工具调用渲染成卡片、把 bash 命令渲染成 Allow/Deny 门禁。文章里的 assistant-ui 示例就是这么做的。

**问: 为什么使用 session.status_idle 而不是直接等待流结束？**

Agent loop 可能先调用工具、再继续推理，响应长度不可预测；status_idle 是一个语义化结束信号，表示没有更多工具要执行或消息要产出。客户端应监听该事件来关闭 UI 状态。

**问: 如何把同一个后端智能体复用到 Slack、Teams 等不同渠道？**

核心是与前端框架解耦：会话与事件流在服务端，渠道层只负责把不同聊天协议映射到同一套事件。文章提到更换 Chat SDK adapter 即可从 Vercel Chat 切换到 Slack、Teams 等。

## 适用场景

- 需要让模型在隔离环境中写代码、跑命令并生成文件的自动化数据分析任务
- 在浏览器聊天或企业 IM 中构建可实时展示工具调用过程的研究或客服助手
- 需要长期运行、多次工具调用和人工干预的智能体工作流
- 数据报告、定时任务或知识整理等可重复执行的后台任务

## 标签

`Claude Managed Agents` `AI Agent` `会话管理` `沙箱` `流式事件`
