# 安装 LangGraph：核心包、LangChain 与模型提供商依赖

*原文: [https://docs.langchain.com/oss/python/langgraph/install](https://docs.langchain.com/oss/python/langgraph/install) · 来源: web · 生成时间: 2026-09-18T06:31:05.701186+00:00*

## 背景

LangGraph 是 LangChain 团队推出的图编排框架，用有向图建模 LLM 应用中的节点与状态流转，解决传统 Chain/AgentExecutor 难以表达的循环、条件、持久化与多智能体协作。随着 agent 应用从单次调用走向多步骤、可中断、可恢复的工作流，需要专门的运行时来管理状态和控制流。本页面是官方入门安装页，交代最小依赖边界。

## 痛点

若不清楚安装边界，容易误以为 langgraph 自带模型调用能力，运行时才发现缺少 provider SDK。Python 版本低于 3.10 或依赖未锁定，会在安装阶段报语法或版本错误。随意用 -U 升级还可能引入破坏性变更，影响生产 agent。

## 解决办法

最小安装 langgraph 获得图执行与状态管理能力；官方推荐同时安装 langchain 作为统一 LLM/工具接入层，再按需安装 langchain-openai 等提供商包。uv add 可替代 pip，提供更快解析和锁文件。可以把 langgraph 理解为流程引擎，langchain 是设备适配层，provider 包是具体驱动。

## 关键代码示例

```python
# pip install -U langgraph langchain
from typing import TypedDict
from langgraph.graph import StateGraph, END

class State(TypedDict):
    message: str

def greet(state: State) -> State:
    return {'message': 'Hello, ' + state['message'] + '!'}

builder = StateGraph(State)
builder.add_node('greet', greet)
builder.set_entry_point('greet')
builder.add_edge('greet', END)
graph = builder.compile()

print(graph.invoke({'message': 'LangGraph'}))
```

这段代码先点出安装命令，随后定义状态和节点：State 是贯穿图的状态结构，greet 是普通 Python 函数，输入输出都遵循该状态。通过 StateGraph 注册节点、设置入口和到 END 的边，再 compile 得到可执行图。最后 invoke 验证安装和最小运行路径，对应 LangGraph 的节点、边、状态三要素。

## 关键流程

1. 用 pip install -U langgraph 或 uv add langgraph 安装核心包
2. 安装 langchain：pip install -U langchain 或 uv add langchain，需 Python 3.10+
3. 按提供商安装集成包，例如 langchain-openai 或 langchain-anthropic
4. 运行最小 StateGraph 示例验证导入和状态流转

## 关键点

- LangGraph 将 LLM 工作流抽象为节点和边组成的状态图，节点负责计算、边负责条件流转，这是其区别于链式调用的本质。
- langgraph 核心包不包含任何模型提供商的 SDK，所以需要额外安装 langchain 或具体 provider 包，否则运行时会报缺少模块。
- 官方要求 Python 3.10+，因为代码和依赖大量使用现代类型语法与 pydantic v2 特性，低版本环境无法解析或运行。
- 文档推荐 LangChain 是为了统一模型接入和工具定义，但 LangGraph 本身不强依赖 LangChain，节点可以是任意 Python 函数。
- 使用 uv add 相比 pip 能更快解析依赖并生成锁文件，更适合新项目；在旧 CI 或企业镜像中应评估兼容性。

## 对比与权衡

- 相比直接使用 OpenAI SDK 编写 agent，LangGraph 提供状态持久化、checkpoint、循环分支和人机审批等能力，但需要学习图抽象并写更多编排代码。
- 相比 LangChain 旧式 AgentExecutor，LangGraph 在复杂控制流（循环、条件边、并行、子图）上显著更灵活，但必须手动组装节点和边。
- 相比只安装 langgraph，同时安装 langchain 能获得统一模型接口和工具生态，代价是引入更多依赖和抽象层；若追求轻量可直接接 provider SDK。
- 相比 pip，uv add 依赖解析更快、锁文件更现代，但在部分企业镜像和旧版 CI 中兼容性可能不如 pip 稳定。

## 自测问题

**问: LangGraph 和 LangChain 是什么关系？为什么不只装一个？**

LangChain 是高层 LLM 应用框架，提供模型、提示、工具、链等抽象；LangGraph 是图编排运行时，管理状态、节点和边。langgraph 只带核心执行器，不含模型提供商逻辑，所以官方常配 langchain 作为统一接入层；但两者不是强绑定，完全可以直接用 OpenAI SDK 定义节点。

**问: 为什么安装要求 Python 3.10+？**

源码使用 PEP 604 的 X | Y 联合类型、内建泛型等现代语法，低版本解释器无法解析；pydantic v2 等核心依赖也抬高了最低版本。实际项目中可用 uv python pin 3.11 或 pyenv 统一解释器版本。

**问: 生产环境安装 LangGraph 依赖时要注意什么？**

不要直接无脑 pip install -U，应使用锁文件固定 langgraph、langchain-core 和 provider 包的兼容版本；在独立虚拟环境或容器中安装；CI 中加 import langgraph 的冒烟测试，避免依赖漂移或部分包未装入镜像。

**问: LangGraph 节点必须使用 LangChain 的模型或工具吗？**

不必。节点是普通 Python 函数，状态可以是 TypedDict 或 Pydantic 模型；可以直接调用 OpenAI SDK、HTTP API 或本地模型。LangChain 的价值是统一接口、减少切换 provider 时的样板代码，并方便定义工具。

**问: 如何验证安装成功？**

执行 python -c 'import langgraph; print(langgraph.__version__)' 检查版本；更可靠的是运行一个最小 StateGraph，看是否能完成 compile 和 invoke。若报 pydantic 或版本错误，优先检查 Python 版本和依赖冲突。

## 适用场景

- 构建需要循环、条件路由和工具调用的 ReAct/Reflexion 风格 agent。
- 多步骤 RAG 或数据流水线需要断点恢复、状态持久化。
- 多智能体协作，需控制消息流和共享状态。
- 需要 human-in-the-loop 审批、暂停和回溯调试的生产工作流。

## 标签

`LangGraph` `LangChain` `Python` `Agent编排` `安装依赖`
