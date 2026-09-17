# 掌握这 6 个 LangGraph 概念，你已领先 90% 的 AI 开发者

*原文: [https://pub.towardsai.net/if-you-know-these-6-langgraph-concepts-you-are-already-ahead-of-90-of-developers-69a83e701da7](https://pub.towardsai.net/if-you-know-these-6-langgraph-concepts-you-are-already-ahead-of-90-of-developers-69a83e701da7) · 来源: web · 生成时间: 2026-09-17T01:19:25.523242+00:00*

## 背景

LangChain 的 LCEL 适合线性 prompt 流水线，但真实智能体需要分支、循环、重试、持久化和人机协同。LangGraph 把智能体建模为显式有向图，使每一步状态流转和决策可观察、可恢复。2026 年报告显示超 70% 生产级 agent 采用图结构。

## 痛点

照教程跑通后，一加分支就出现死循环、跳过节点或会话丢失；状态散落在变量、内存和对话历史中，调试只能靠试错。不理解底层状态流和边路由，就无法可靠构建复杂智能体。

## 解决办法

LangGraph 将智能体定义为有向状态图：TypedDict 声明的 State 是共享白板，每个 Node 是接收状态、返回部分更新的纯函数，Edges 决定下一步；Checkpointer 在每个 super-step 保存快照，实现恢复、重试、记忆与人机协同。这个显式状态机模型让复杂控制流从隐式样板代码变成可读图结构。类比：像白板+流程图的签核系统，每步读白板、写白板，边是审批规则。

## 关键代码示例

```python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph import StateGraph, END

class AgentState(TypedDict):
    question: str
    answer: str
    messages: Annotated[list, add]  # 列表用 add 合并，而不是覆盖

def answer_node(state: AgentState) -> dict:
    # 节点只返回需要更新的字段，不返回完整 state
    return {"answer": f"Answer to: {state['question']}"}

def refine_node(state: AgentState) -> dict:
    return {"answer": state["answer"].upper()}

graph = StateGraph(AgentState)
graph.add_node("answer", answer_node)
graph.add_node("refine", refine_node)
graph.add_edge("answer", "refine")
graph.add_edge("refine", END)  # 必须连 END，否则图会挂起
graph.set_entry_point("answer")
app = graph.compile()
```

这段代码先定义类型化 State，其中 messages 用 Annotated[list, add] 声明为追加合并，避免多个节点覆盖历史。节点 answer_node 和 refine_node 都是普通函数，只返回变化的 answer 字段，符合 LangGraph 的增量更新契约。图注册节点后用 add_edge 连接 answer -> refine -> END，最后 set_entry_point 并 compile；END 是终止节点，缺失会导致图无限等待。

## 关键流程

1. 用 TypedDict 定义最少的 State 字段，列表字段用 Annotated[list, add] 声明合并策略
2. 编写节点函数：接收完整 State，只返回需要更新的字段 dict
3. 用 add_node 注册节点，用 set_entry_point 设置入口
4. 添加 direct edge 或 conditional edge，把最后节点连到 END
5. 用 checkpointer 和 thread 配置持久化与记忆（如 MemorySaver/SqliteSaver）
6. 需要人工确认时用 interrupt_before/after 在指定节点暂停并恢复

## 关键点

- State 是显式类型化的共享状态：每个节点读同一份 State，默认后写覆盖先写，Annotated[list, add] 改成列表合并，适合消息历史等累积数据。
- State 只应保存下游真正需要的字段：存储原始 LLM 响应元数据会使 checkpoint 膨胀到数百 KB，拖慢数据库写入。
- Node 只是普通 Python 函数，遵循 state in -> partial update dict out 的契约；返回完整 state 会覆盖其他节点的更新，造成隐蔽 bug。
- Edges 是图的控制层：direct edge 无条件走，conditional edge 根据 State 动态路由；忘接 END 会让图永远等待下一步。
- Checkpointer 在每个节点后保存状态快照，配合 thread ID 实现会话隔离、失败重试、时间旅行和恢复执行，是生产级能力的基础。
- Human-in-the-loop 通过 interrupt 让图在指定节点暂停，等待外部审批或修改状态后再恢复，适合安全敏感的审批流。

## 对比与权衡

- 相比 LangChain 的 LCEL 链式调用，LangGraph 在分支、循环、重试、持久化与人机协同上更强，但需要显式建图和理解状态语义，代码量与心智负担更高。
- 相比自己用 Python 手写状态机或任务队列，LangGraph 节省了 checkpoint、恢复、可视化等样板代码，但引入了框架抽象，极端定制场景可能受限。

## 面试可能会问

**问: State 中 Annotated[list, add] 的 add 是做什么的？为什么不能直接 list？**

默认两个节点更新同一字段时后写覆盖先写；add 作为 reducer 把列表追加而不是替换，适合消息历史、工具结果等累积数据。也可以自定义 reducer 函数实现更复杂合并策略。

**问: 节点返回整个 state 有什么风险？**

节点 A 和 B 并行或先后更新不同字段时，返回完整旧 state 会把其他节点刚写入的新值覆盖回旧值；LangGraph 约定节点只返回变化的字段，由框架做增量合并。

**问: LangGraph 如何支持失败重试或从某个节点恢复？**

Checkpointer 在每个 super-step 保存 State 快照，通过 thread ID 定位会话；失败后可以重新 invoke 同个 thread，或从指定 checkpoint 继续，而不是从头发起重试。

**问: human-in-the-loop 是怎么实现的？**

配置 interrupt_before/after 让图在目标节点前/后暂停，把控制权交给外部；外部可以查看并修改 state，再调用 app.invoke(Command(resume=...)) 恢复执行；checkpoint 保证暂停期间状态不丢。

**问: conditional edges 和节点内部 if-else 有什么区别？**

conditional edge 把路由逻辑提升到图结构层，使流程图可解释、可追踪、可恢复；节点内 if-else 会成为隐式控制流，复杂后难以调试和可视化。

## 适用场景

- 多工具智能体：根据用户意图在知识库检索、API 调用、回答等节点间动态路由，并支持失败重试。
- 客服机器人：保持多轮对话 memory、按会话 thread 隔离，并在敏感操作前暂停人工审批。
- 文档处理流水线：文档解析→摘要→审核→发布，每步 checkpoint 避免重复执行，审核节点 human-in-the-loop。
- 面试复习场景：把知识点拆成节点图，理解状态流转与 checkpoint 后可清晰讲解 LangGraph 生产设计。

## 标签

`LangGraph` `AI Agent` `状态机` `LLM 应用` `人机协同`
