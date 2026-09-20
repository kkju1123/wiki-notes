# LangGraph 思维：用节点、状态与图构建生产级 Agent

*原文: [https://docs.langchain.com/oss/python/langgraph/thinking-in-langgraph](https://docs.langchain.com/oss/python/langgraph/thinking-in-langgraph) · 来源: web · 生成时间: 2026-09-20T00:54:48.182651+00:00*

## 背景

传统 LLM 应用常用链式调用或自主循环，但一遇到多步分支、人工审批、失败恢复就难以工程化。LangGraph 把智能体建模为有向图：节点做具体工作，边表达流转，共享 state 保存上下文。它因此适合生产环境中的复杂 Agent 工作流。

## 痛点

不懂这种方法，开发者容易把业务逻辑写成线性脚本或嵌套 if/else：失败后不知道从哪恢复，重试会重复已成功的昂贵步骤，人工介入要自行拼接状态，LLM 决策路径不透明，导致生产事故难排查。

## 解决办法

核心做法是五步：先按业务流程拆出离散节点；识别节点属于 LLM、数据、动作还是用户输入步骤；设计只存原始数据的 state；实现节点函数并分类处理错误；最后用 Command 声明路由并编译图。节点边界形成 checkpoint，重试和恢复都从最近节点开始；interrupt() 挂起等待人工输入，checkpointer 跨会话保存状态。类比工单流水线：每个工位只做一件事、共享工单，不在工单上写满格式化草稿。

## 关键代码示例

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt
from langgraph.checkpoint.memory import MemorySaver

class EmailState(TypedDict):
    email: str
    sender: str
    intent: dict | None
    draft: str | None

def classify(state: EmailState) -> Command[Literal["draft", "human_review"]]:
    # 实际中调用 LLM，返回分类结果
    intent = {"category": "billing", "urgency": "high"}
    if intent["urgency"] == "high":
        return Command(update={"intent": intent}, goto="human_review")
    return Command(update={"intent": intent}, goto="draft")

def human_review(state: EmailState) -> Command[Literal["send"]]:
    # 需要人工批准时暂停，等外部输入后继续
    decision = interrupt({"draft": state["draft"]})
    return Command(update={"approved": decision}, goto="send")

def send(state: EmailState) -> dict:
    return {"status": "sent"}

graph = StateGraph(EmailState)
graph.add_node("classify", classify)
graph.add_node("human_review", human_review)
graph.add_node("send", send)
graph.add_edge(START, "classify")
graph.add_edge("send", END)
app = graph.compile(checkpointer=MemorySaver())
```

这个示例体现了 LangGraph 的核心原理：EmailState 只保存原始数据；classify 节点用 Command(update=..., goto=...) 同时完成状态更新和路由声明；human_review 用 interrupt() 暂停等待人工输入；编译时传入 checkpointer 才能持久化状态并支持稍后恢复。节点边界就是 checkpoint 边界，因此恢复时不会重复分类等已完成步骤。

## 关键流程

1. 第一步：把业务流程拆成离散节点，并画出它们之间的可能路径
2. 第二步：识别每个节点的类型（LLM 步骤、数据步骤、动作步骤、用户输入步骤）及其所需上下文
3. 第三步：设计共享 state，只放原始数据，不放入提示词模板或格式化字符串
4. 第四步：实现节点函数，按错误类型分别用重试、LLM 回环、interrupt、error_handler 或冒泡处理
5. 第五步：用 StateGraph 连接节点，通过 Command 声明路由，编译时挂载 checkpointer 以支持持久化和人工介入

## 关键点

- 状态只存原始数据，提示词在节点内部按需生成：这样不同节点可对同一数据做不同格式化，提示词变更不影响状态结构，调试也更清晰。
- 节点边界就是 checkpoint 边界：节点越小，失败后重做的工作越少，但节点过细会增加图复杂度；这是可靠性与可观测性的权衡。
- 路由决策放在节点内部并用 Command 显式声明，可让 LLM 基于完整上下文决定下一步，同时保持图结构简单、可追踪。
- 错误处理要分类：瞬时错误自动重试，LLM 可恢复错误把错误写回 state 回环，用户可修复错误用 interrupt 暂停，重试耗尽走 error_handler 补偿，未知错误直接冒泡。
- human-in-the-loop 不是事后补丁，而是通过 interrupt() + checkpointer 原生实现，能保存快照并随时恢复。
- 外部服务调用应隔离到独立节点，便于独立配置 timeout、retry 和 error_handler，避免拖垮 LLM 调用或污染主流程。

## 对比与权衡

- 相比 LangChain 的 AgentExecutor，LangGraph 用显式图结构表达循环、分支和人工介入更可控，但需要开发者自己设计节点和状态，学习曲线更陡。
- 相比传统有限状态机，LangGraph 的节点可以包含 LLM 动态决策和非确定性输出，状态也是结构化字典，但调试这种概率性流程比确定性 FSM 更难。
- 相比 CrewAI/AutoGen 这类预设协作角色的框架，LangGraph 更底层、更贴近生产级编排，适合定制错误处理与持久化，但缺少开箱即用的多智能体协作抽象。

## 自测问题

**问: 为什么 LangGraph 建议把 state 存成 raw data，而不是直接存格式化后的 prompt？**

解耦数据与展示；不同节点可基于同一原始数据构造不同 prompt；改提示词模板不需要改 state schema；调试时能清楚看到每个节点真正读到的数据；避免在 state 中提前引入节点特定格式。

**问: 节点粒度怎么决定？**

核心是 checkpoint 在节点边界。节点越细，失败恢复时重放的工作越少，可观测性越好；但节点过多会让图结构复杂、编排成本上升。一般把外部 API 调用、LLM 调用、人工交互等不同失败域拆成独立节点，同一失败域内部可合并。

**问: LangGraph 里怎么实现失败重试和恢复？**

可用 retry_policy 对瞬时错误自动重试，timeout 限制单次调用；LLM 可恢复错误把错误写入 state 并回到 LLM 节点重新决策；需要人工输入用 interrupt() 暂停；重试耗尽后通过 error_handler 走补偿分支；未知错误冒泡交给开发者。

**问: 为什么示例里用 Command 在节点内路由，而不是大量条件边？**

节点内部有完整上下文，LLM 可以在同一处完成分类和决策；Command[Literal["a","b"]] 类型提示把可跳转目标显式化，图结构简单且可追踪。条件边更集中，但复杂流程中可能让图配置和节点逻辑分离、不好维护。

**问: human-in-the-loop 是怎么被 checkpointer 支持的？**

interrupt() 会暂停图执行并保存当前 state；编译时传入 checkpointer 后，每次运行按 thread_id 持久化。恢复时从暂停节点的开头继续，不会重复已成功的上游节点；即使暂停几天也能继续。

## 适用场景

- 客户支持邮件 Agent：分类、知识库检索、起草、人工审批、发送。
- 需要多步 LLM 决策并可能失败的 RAG 研究助手，比如先检索、再判断是否充分、再生成或重新检索。
- 审批/表单处理工作流：需要系统处理、人工介入、记录断点和恢复能力。
- 需要严格错误隔离和重试策略的外部 API 编排，如 CRM、工单系统、支付网关等多服务协同。

## 标签

`LangGraph` `智能体` `工作流编排` `状态管理` `人机协同`
