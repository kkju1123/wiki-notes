# 掌握这 6 个 LangGraph 核心概念，你已领先 90% 的 AI 开发者

*原文: [https://pub.towardsai.net/if-you-know-these-6-langgraph-concepts-you-are-already-ahead-of-90-of-developers-69a83e701da7](https://pub.towardsai.net/if-you-know-these-6-langgraph-concepts-you-are-already-ahead-of-90-of-developers-69a83e701da7) · 来源: web · 生成时间: 2026-09-17T08:40:41.020951+00:00*

## 背景

LangChain 的 LCEL（prompt | llm | parser）擅长线性任务，但真实 Agent 需要分支、循环、重试、持久化和人工审批。LangGraph 引入有向图显式建模控制流，让每一步状态变化可观测、可恢复；行业报告显示多数生产 Agent 已从线性链转向图结构。

## 痛点

不懂这些核心概念，你会在修改分支时遇到图不停止、节点被跳过、会话记忆丢失等问题；只能靠逐行试错，无法理解为什么图的行为不可预测。

## 解决办法

LangGraph 把 Agent 抽象为有向图：State 是带类型的共享状态，Nodes 是接收 state 并返回局部更新的普通函数，Edges 决定下一步走固定边还是基于 state 的条件边。状态默认覆盖，用 Annotated reducer 可合并列表；Checkpointer 在每个 super-step 保存状态快照，实现跨会话记忆和时间旅行；interrupt 在关键节点暂停等待人工输入。掌握这些就理解了图如何编译、执行和恢复。

## 关键代码示例

```python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver

class AgentState(TypedDict):
    question: str
    answer: str
    messages: Annotated[list, add]

def answer_node(state: AgentState) -> dict:
    ans = f"Answer to: {state['question']}"
    return {"answer": ans, "messages": [ans]}

def refine_node(state: AgentState) -> dict:
    return {"answer": state["answer"].upper()}

def route_after_answer(state: AgentState) -> str:
    return "refine" if len(state["answer"]) < 30 else END

g = StateGraph(AgentState)
g.add_node("answer", answer_node)
g.add_node("refine", refine_node)
g.add_conditional_edges("answer", route_after_answer)
g.add_edge("refine", END)
g.set_entry_point("answer")
app = g.compile(checkpointer=MemorySaver())
```

这段代码展示了 State、Nodes、Edges 和 Checkpointer 的核心用法：AgentState 中用 Annotated[list, add] 让 messages 字段合并而不是覆盖；节点函数只返回变化字段；条件边根据 answer 长度决定进入 refine 还是直接 END；compile 时传入 MemorySaver 后，图会按 thread_id 持久化状态快照。

## 关键流程

1. 定义带类型的 AgentState，明确哪些字段需要共享、哪些字段用 add reducer 合并
2. 把每个处理步骤写成节点函数：接收 state，返回需要更新的字段
3. 用 add_node 注册节点，用 add_edge 或 add_conditional_edges 连接它们
4. 设置 set_entry_point，并把所有终止路径连到 END
5. 通过 compile(checkpointer=...) 编译图，并用 thread_id 区分会话

## 关键点

- State 是显式、带类型的共享记忆；Annotated[list, add] 控制合并行为，默认覆盖会丢消息。
- 节点是普通 Python 函数，只应返回变化字段；返回整个 state 会覆盖其他节点的更新，产生隐蔽 bug。
- 边分 direct 和 conditional；conditional edge 让同一节点根据状态走向不同后续节点，这是分支/循环的基础。
- 必须把所有终止路径连到 END，否则图会一直等待下一步，表现为运行不结束。
- Checkpointer 在每个 super-step 保存 state 快照，是 memory、human-in-the-loop、time-travel 的底层机制；没有它跨会话不会记住任何东西。
- State 设计要克制，只存下游真正需要的字段；把原始 LLM 响应或 usage metadata 全塞进 state 会让 checkpoint 膨胀并拖慢持久化。

## 对比与权衡

- 相比 LangChain 的 LCEL/Chain，LangGraph 在分支、循环、重试、持久化上更好，但代码结构更复杂、学习曲线更高。
- 相比 LlamaIndex Workflows 等图式编排框架，LangGraph 的 checkpointer/时间旅行生态更成熟，但在 RAG/数据索引集成上不如 LlamaIndex 自然。
- 相比 OpenAI Assistants 等托管 Agent 方案，LangGraph 的可控性、可观测性和可移植性更好，但需要自己部署与管理运行时。

## 自测问题

**问: LangGraph 的 State 里 Annotated[list, add] 和普通 list 有什么区别？**

普通字段默认覆盖，后写的节点会覆盖前一个节点的更新；add 是 reducer，会把两次更新合并成一个 list。适合 messages 这种需要持续追加的字段。还可以自定义 reducer 实现更复杂的合并策略。

**问: 节点函数可以返回完整 state 吗？为什么？**

语法上可以，但不推荐。LangGraph 会把返回的 dict 作为该节点的状态更新，若返回整个 state，会把所有字段都覆盖成你返回的值，可能覆盖其他节点刚更新的字段，产生难以调试的竞态和丢失更新问题。正确做法是只返回变化的字段。

**问: 条件边和普通边有什么区别？怎么实现循环？**

普通边固定从 A 到 B；条件边执行一个函数，根据 state 返回下一个节点名或 END。循环就是把条件边指向之前执行过的节点，比如 refine 后再次进入 answer。关键是必须有条件能回到 END，否则无限循环。

**问: LangGraph 的 memory 是怎么实现的？为什么需要 thread_id？**

memory 依赖 checkpointer，每个 super-step 把 state 快照持久化。thread_id 是会话标识，同一个 thread_id 的多次 invoke 会从上一个 checkpoint 恢复状态，实现跨会话记忆；不同 thread_id 状态隔离。没有 checkpointer 就不会持久化。

**问: Human-in-the-loop 在 LangGraph 中如何实现？**

在需要审批的节点前使用 interrupt，图会暂停并把控制权返回给调用方；调用方检查或修改 state 后通过 update_state 和 resume 继续执行。它本质是 checkpoint 的暂停与恢复，所以状态不会丢失。

## 适用场景

- 客服 Agent：根据用户意图分支到知识库、工具或人工，失败重试，并通过 thread_id 记住完整会话。
- 多步文档处理流水线：抽取、审核、改写等步骤，在关键节点暂停人工确认，利用 checkpoint 避免重复调用 LLM。
- 需要可调试、可回放的多智能体协作：出现错误时从历史 checkpoint 恢复，分叉重试，观察每一步状态变化。
- 审批类业务流：状态根据规则流动，不同分支执行不同节点，并能持久化与审计。

## 标签

`LangGraph` `AI Agents` `LLM` `State Management` `面试准备`
