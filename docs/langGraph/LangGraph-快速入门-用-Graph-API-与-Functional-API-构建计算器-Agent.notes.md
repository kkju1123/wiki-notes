# LangGraph 快速入门：用 Graph API 与 Functional API 构建计算器 Agent

*原文: [https://docs.langchain.com/oss/python/langgraph/quickstart](https://docs.langchain.com/oss/python/langgraph/quickstart) · 来源: web · 生成时间: 2026-09-20T00:49:57.467180+00:00*

## 背景

LangGraph 是 LangChain 生态中的有状态图编排框架，主要解决 LLM 应用从单次问答走向多步工作流时的控制流、状态持久化与可观测性问题。相比早期 AgentExecutor 的线性循环，LangGraph 用图结构显式定义节点、边和条件分支，让复杂 Agent 可以像流程图一样设计和调试。它出现后逐渐成为构建可中断、可恢复、可回溯 Agent 的主流底层运行时。

## 痛点

手写 Agent 循环时，开发者需要自己管理消息历史、工具调用循环、异常重试、中间状态保存等，容易写出难以扩展的意大利面代码。不理解这份 Quickstart，就无法正确使用 LangGraph 的状态 reducer、条件边和工具节点，后续遇到多步工具调用或人工介入需求时会很吃力。

## 解决办法

LangGraph 把 Agent 抽象为状态图和任务函数两种形态。Graph API 中，StateGraph 维护一个可累积的状态，例如 messages 用 operator.add 追加；节点执行模型推理或工具调用，条件边根据 AIMessage 中是否含 tool_calls 决定继续调用工具还是结束。Functional API 则用 @entrypoint 把整个 Agent 写成一个普通函数，运行时自动处理状态流转和断点。两者共享同一 Runtime，区别只是表达方式。可以类比：Graph API 像画电路图，Functional API 像写一个支持暂停和恢复的普通函数。

## 关键代码示例

```python
from typing import TypedDict, Annotated
import operator
from langchain.tools import tool
from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode

class State(TypedDict):
    messages: Annotated[list, operator.add]

@tool
def add(a: int, b: int) -> int:
    '''Add a and b.'''
    return a + b

model = init_chat_model('claude-sonnet-4-6', temperature=0).bind_tools([add])

def call_model(state):
    return {'messages': [model.invoke(state['messages'])]}

def route(state):
    return 'tools' if state['messages'][-1].tool_calls else END

builder = StateGraph(State)
builder.add_node('model', call_model)
builder.add_node('tools', ToolNode([add]))
builder.add_edge(START, 'model')
builder.add_conditional_edges('model', route, {'tools': 'tools', END: END})
builder.add_edge('tools', 'model')
graph = builder.compile()
```

State 中的 messages 使用 Annotated[list, operator.add]，表示每次节点返回的新列表会追加到旧列表末尾，而不是覆盖历史消息。@tool 装饰器把 add 函数签名和 docstring 转成模型可识别的工具 schema，bind_tools 则让模型能输出 tool_calls。call_model 节点只返回一个 dict，LangGraph 会自动按 reducer 更新状态；route 条件边检查最后一条消息是否包含 tool_calls，从而在 tools 节点和 END 之间形成循环，直到模型不再请求工具。

## 关键流程

1. 准备 Anthropic API Key 并设置 ANTHROPIC_API_KEY 环境变量
2. 定义工具 add/multiply/divide，并用 @tool 暴露 schema，通过 bind_tools 绑定到 chat model
3. 定义 State schema，为 messages 列表配置 operator.add reducer，以支持追加而非替换
4. Graph API：用 StateGraph 添加 model 和 tools 节点，并用条件边实现工具循环后 compile
5. Functional API：用 @entrypoint 定义单个函数，在函数内完成模型调用和工具执行循环
6. 调用编译后的 graph 或函数并传入初始消息，观察多步工具调用结果

## 关键点

- LangGraph 中 State 默认是覆盖更新，必须给 messages 列表配置 Annotated[... , operator.add] 或 add_messages，否则新消息会覆盖历史消息，模型将失去上下文。
- bind_tools 只是向模型声明工具 schema，模型返回的是 tool_calls，真正执行工具由 ToolNode 完成，两者分工是 Agent 可靠性的关键。
- 条件边通过检查最后一条 AIMessage 是否有 tool_calls 来决定循环或结束，这是实现 multi-step tool calling agent 的核心控制流。
- Graph API 与 Functional API 共享 Runtime，前者适合显式流程图和持久化控制，后者适合用更少样板表达单入口 Agent。
- 编译后的 graph 是惰性可执行对象，同一 graph 可多次 invoke、stream、批处理，便于测试、部署和接入 LangSmith 追踪。

## 对比与权衡

- 相比 LangChain 早期的 AgentExecutor，LangGraph 在控制流表达力、状态持久化和可观测性上明显更好，但需要编写更多样板代码，学习成本更高。
- 相比直接手写 while + function calling 循环，LangGraph 提供了 checkpoint、中断、时间旅行、流式事件等现成能力，但引入了框架依赖和抽象概念。
- 相比 Functional API，Graph API 在可视化、细粒度节点治理和复杂分支编排上更好，但表达同等简单逻辑时代码更啰嗦。

## 自测问题

**问: 为什么 messages 状态要用 Annotated[list, operator.add]？**

LangGraph 每次节点返回 dict 更新状态时，默认同名键会覆盖；这里用 reducer 告诉运行时把新列表追加到旧列表末尾。可以提到 add_messages 是更语义化的工厂函数，除了追加还能按消息 ID 做去重和合并。

**问: 模型 bind_tools 后会自动执行工具吗？**

不会。bind_tools 只注入 JSON schema，模型只能输出 tool_calls；执行由 ToolNode 负责，ToolNode 会真正调用函数并把结果封装为 ToolMessage 放回状态。这样可以控制执行权限、插入人工审批或重试。

**问: Graph API 中条件边如何形成 Agent 循环？**

model 节点执行后，条件边函数读取最新 AIMessage 的 tool_calls；若存在则路由到 tools 节点，tools 执行完通过静态边回到 model；否则路由到 END。该循环会持续到模型不再请求工具。

**问: Graph API 和 Functional API 怎么选？**

需要复杂分支、并行节点、子图、人工中断或可视化调试选 Graph API；简单单入口任务、团队更习惯普通函数写法选 Functional API。两者共享 runtime，后期可以在 Function 内部使用 graph 或迁移。

**问: LangGraph 和 LangChain 是什么关系？**

LangChain 侧重模型调用、工具、消息、检索等组件；LangGraph 侧重工作流编排和状态管理。LangChain 的组件可以在 LangGraph 节点中使用，LangGraph 解决 LangChain AgentExecutor 难以表达的复杂控制流、持久化、可恢复性问题。

## 适用场景

- 构建需要多轮工具调用，如计算器、SQL、搜索等场景的智能助手或 Agent。
- 需要人工审批、暂停、恢复或时间旅行调试的生产级工作流。
- 需要把复杂控制流画成图、便于团队理解和维护的多智能体协作系统。

## 标签

`LangGraph` `Agent` `State Management` `Function Calling` `Python`
