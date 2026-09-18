# LangGraph 概览：低层有状态 Agent 编排框架与运行时

*原文: [https://docs.langchain.com/oss/python/langgraph/overview](https://docs.langchain.com/oss/python/langgraph/overview) · 来源: web · 生成时间: 2026-09-18T06:09:53.385500+00:00*

## 背景

早期 LLM 应用多为单次请求-响应，而 Agent 需要多步推理、工具调用和循环决策。手写这些逻辑常把状态管理、错误恢复和业务控制流搅在一起，难以测试、恢复和观测。LangGraph 借鉴 Pregel、Apache Beam 的图计算/流水线思想，把 Agent 建模为显式状态机/图，提供底层编排基础设施。

## 痛点

没有<mark>统一编排层</mark>时，复杂 Agent 的状态难以持久化，进程中断或异常后需要从头重跑；人工审批、暂停/继续、流式输出等需求要自己实现；确定性与 LLM 步骤混杂在一起，出问题后很难定位和恢复。

## 解决办法

LangGraph 采用 StateGraph 图结构：开发者定义共享状态和若干节点，节点可以是普通函数（确定性）或 LLM 调用（agentic），节点间通过边或条件边连接。执行时状态沿图传播，框架负责 <mark>checkpoint 持久化</mark>、状态恢复、流式输出和中断点。类比工作流引擎/状态机，但针对 LLM 长任务做了持久化与可恢复设计：失败重启后从最近 checkpoint 继续，人工可在关键节点暂停并修改状态后恢复。

## 关键代码示例

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class State(TypedDict):
    messages: list[str]

def deterministic_node(state: State) -> State:
    return {'messages': state['messages'] + ['deterministic']}

def llm_node(state: State) -> State:
    # 实际可替换为模型调用/工具调用
    reply = 'llm_decision'
    return {'messages': state['messages'] + [reply]}

graph = StateGraph(State)
graph.add_node('rules', deterministic_node)
graph.add_node('model', llm_node)
graph.add_edge(START, 'rules')
graph.add_edge('rules', 'model')
graph.add_edge('model', END)

app = graph.compile()
print(app.invoke({'messages': []}))
```

这段代码先定义共享状态 State，其中 messages 列表会在节点间传递和累积。deterministic_node 是普通函数，llm_node 可以替换为真实模型调用，二者都只返回部分状态更新。通过 add_node 注册节点、add_edge 定义执行顺序，最后 <mark>compile 生成可执行 app，invoke 触发一次完整图执行</mark>，体现 LangGraph 用图结构混合确定性与 agentic 步骤的核心原理。

## 关键流程

1. 定义共享状态结构（如 TypedDict），明确节点间传递哪些字段。
2. 实现节点函数：确定性步骤写普通逻辑，LLM 步骤封装模型调用；均返回部分状态更新。
3. 用 add_node 注册节点，用 add_edge/add_conditional_edges 定义执行流程和条件分支。
4. 调用 compile 编译为可执行图，再用 invoke/stream 运行，并可通过 checkpoint 配置持久化。

## 关键点

- LangGraph 定位为低层编排运行时，不封装 prompt 和固定架构，因此比高层 Agent 框架更灵活，但需要自行设计流程。
- 核心价值是把确定性步骤与 LLM 驱动步骤放在同一张图中，让可审计规则和模型决策各司其职。
- 持久化/checkpoint 机制使长任务支持断点续跑和错误恢复，这是生产级 Agent 的关键能力。
- human-in-the-loop 通过 interrupt 在任意状态处暂停、检查或修改状态，再恢复执行，适用于审批等场景。
- LangGraph 可脱离 LangChain 使用，但与 LangChain/LangSmith 配合可覆盖模型集成、可观测性和部署。
- 图执行模型受 Pregel/Apache Beam 启发，接口风格来自 NetworkX，理解这些有助于掌握其状态传播与容错设计。

## 对比与权衡

- 相比 LangChain 高层的 AgentExecutor，LangGraph 在控制流、状态持久化和可恢复执行上更强，但需要开发者自行设计节点和边，代码量更大。
- 相比 AutoGen/CrewAI 等偏多智能体框架，LangGraph 更偏通用底层编排，不强制预设角色/消息流，因此更灵活，但缺少约定好的多智能体开箱即用模板。
- 相比直接手写 Python 循环调用 LLM，LangGraph 提供 checkpoint、中断、流式和可视化调试，但引入了图结构和编译过程的学习/抽象成本。

## 自测问题

**问: LangGraph 与 LangChain 的 agent 高层封装有什么不同？**

LangChain 提供预置的 AgentExecutor 和 tool-calling loop，上手快但控制力有限；LangGraph 下沉到图和状态机层面，可自定义节点、边、条件路由、持久化和中断，适合复杂长期运行流程。可以指出 LangGraph 常被高层 agent 用作底层 runtime。

**问: LangGraph 如何实现长时间运行和失败恢复？**

基于 checkpoint 持久化每次图状态迁移，异常或进程重启后从最近 checkpoint 恢复；这区别于无状态的普通链式调用。可以补充数据库后端、线程/进程级别恢复。

**问: 什么是条件边（conditional edges）？**

根据节点输出或状态值决定下一个节点，实现循环、分支和工具调用路由，例如 LLM 返回 'continue'/'finish' 或选择工具。实现上使用 add_conditional_edges 注册路由函数，使图具备动态决策能力。

**问: 为什么将确定性步骤和 agentic 步骤混合很重要？**

企业场景需要可预测、可审计的硬规则（如权限、合规、数据校验），也需要 LLM 灵活理解意图。全 LLM 不可控且成本高，全规则难以处理自然语言。混合图可精确控制 AI 应用边界。

**问: LangGraph 借鉴 Pregel/Apache Beam 具体体现在哪？**

Pregel 是 Google 的图计算模型，节点围绕状态迭代处理消息，适合大规模图容错；Apache Beam 提供统一流批处理与状态/窗口语义。LangGraph 吸收“图节点 + 状态传递 + 容错恢复”的思想，而不是简单的顺序管道。

## 适用场景

- 需要多步工具调用、条件路由和重试的 AI 助手/客服系统。
- 包含人工审批、状态检查或敏感操作确认的自动化流程。
- 运行数分钟到数小时、要求失败后不从头开始的研究或数据处理 Agent。
- 将规则引擎与 LLM 判断结合的审核、风控或内容安全流程。

## 标签

`LangGraph` `Agent编排` `有状态Agent` `Human-in-the-loop` `LLM`

## 追问记录

**划线：** LangGraph is a low-level orchestration framework and runtime for building, managing, and deploying long-running, stateful agents

**解释：** 这句话的意思是：LangGraph 不是那种帮你预设好智能体行为的高层封装，而是一个偏底层的“编排框架 + 运行时”，负责把 Agent 的多个步骤组织成可执行、可管理、可部署的流程。它特别强调“long-running, stateful agents”，也就是那些不是一问一答、而是会连续运行很久、并且需要记住中间状态的智能体。这正好对应笔记里的核心设计：用 StateGraph 把节点和边显式建模，再通过 checkpoint 保存每一步状态，从而支持失败恢复、暂停继续和人工审批。因此，可以把 LangGraph 理解成智能体流程的“骨架和运行环境”，而不是直接替你完成任务的“成品大脑”。

*2026-09-18 06:45 UTC*

**划线：** focused entirely on agent orchestration

**解释：** 你划线的这句话是在说明 LangGraph 的定位边界：它只负责“Agent 编排”，也就是把 Agent 执行过程中的状态、节点、边、条件路由、检查点恢复这些流程层面的东西管好，而不去预设模型怎么调用、提示词怎么写或 Agent 必须采用哪种固定架构。这个定位很重要，因为复杂 Agent 的核心难点往往不是“调用一次大模型”，而是多步骤之间如何流转、状态如何保存、出错后如何恢复；专注编排正好切中了这些底层控制问题。这也解释了笔记里为什么说它比 LangChain 高层 AgentExecutor 更灵活、但需要自己设计流程——它提供的是运行骨架和规则，而不是一个替你完成任务的成品大脑。所以后面与 AutoGen、CrewAI 对比时强调的“低层、通用、不强制预设角色”，都是这个聚焦带来的自然结果。

*2026-09-18 07:14 UTC*

**划线：** LangGraph gives you fine-grained control to mix deterministic

**解释：** 你划出的这句话，核心是在说 LangGraph 提供了一种“细粒度控制”能力，让你能把**确定性的普通代码步骤**（比如规则校验、格式转换、权限判断）和**由 LLM 驱动的智能步骤**，灵活地编排在同一张执行图里。这意味着你不需要把整个流程都交给模型黑盒去跑，而是可以精确决定哪个节点走固定逻辑、哪个节点调用模型、节点之间怎么连接、什么条件下走哪条分支。这在复杂 Agent 中非常重要，因为很多关键环节必须可预测、可审计，不能完全依赖模型的不稳定输出。结合笔记里的 StateGraph 设计，就是让 `deterministic_node` 和 `llm_node` 各司其职，由你显式控制它们的执行顺序和条件边，从而在“可控的规则”和“智能的决策”之间取得平衡。

*2026-09-18 07:14 UTC*

**划线：** One of LangGraph’s core strengths is the ability to mix deterministic steps with LLM-driven agentic steps in a single graph

**解释：** 这句话是说 LangGraph 的核心优势在于能把“固定逻辑的普通代码步骤”和“需要模型自主判断的 LLM 步骤”放在同一张流程图里编排。它重要是因为这样既保留了可预测、可审计的规则控制，又能借助 LLM 处理灵活任务，避免全部交给模型导致不可控。

*2026-09-18 07:32 UTC*

**划线：** planning, subagents, filesystem tools, and context management on top of LangGraph

**解释：** 这句话是在说：任务规划、多个子代理、文件系统工具、上下文管理这些高级能力都不是 LangGraph 自带的功能，而是需要你在它提供的图状态机基础上自己搭。所以它才叫“低层”框架——灵活，但不省事。

*2026-09-18 07:32 UTC*

**划线：** 统一编排层

**解释：** “统一编排层”指的是用一套框架统一管理 Agent 的所有节点、状态流转和控制逻辑，而不是把状态保存、错误恢复、暂停继续这些事散落在业务代码里；它重要在能让复杂 Agent 具备持久化、断点续跑和人工介入等生产级能力。

*2026-09-18 07:39 UTC*

**划线：** checkpoint 持久化

**解释：** checkpoint 持久化就是把 Agent 每一步的状态快照存下来；这样任务中断、报错或进程重启后，能从最近一次成功状态继续跑，不用从头再来。它是长任务断点续跑、失败恢复和人工暂停后继续执行的基础。

*2026-09-18 07:40 UTC*

**划线：** compile 生成可执行 app，invoke 触发一次完整图执行

**解释：** compile 把你定义好的节点和边打包成一个能运行的程序对象，invoke 就是给它一个初始状态并从头到尾执行完整张图。它重要在把“流程定义”和“实际运行”分开，这样框架才能在运行过程中统一负责状态保存、恢复和中断等底层能力。

*2026-09-18 07:40 UTC*
