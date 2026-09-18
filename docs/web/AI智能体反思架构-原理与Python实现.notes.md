# AI智能体反思架构：原理与Python实现

*原文: [https://cloud.tencent.com/developer/article/2588358](https://cloud.tencent.com/developer/article/2588358) · 来源: web · 生成时间: 2026-09-18T02:38:50.895649+00:00*

## 背景

大型语言模型单次生成的结果常存在逻辑漏洞、边界遗漏和效率低下等问题，尤其在代码生成和技术文档场景中难以直接使用。反思架构借鉴软件工程中的代码审查流程，让模型先生成初稿，再以批评者视角审视并改进，从而提升输出可靠性。这种模式是智能体工作流中最基础也最实用的质量保障手段之一。

## 痛点

没有反思机制时，LLM通常一次性输出答案，缺乏自我纠错能力，容易把明显错误交付给用户。复杂任务如生成算法实现时，简单递归等低效方案屡见不鲜，无法满足生产级质量要求。

## 解决办法

反思架构将任务拆为三个串行节点：生成节点按需求产出初稿；批判节点强制模型从错误、效率和最佳实践等具体维度进行结构化评审，生成布尔判断和改进建议列表；精炼节点基于批评意见重写出最终版本。LangGraph负责编排状态流转，Pydantic模型约束各节点输出格式，避免非结构化文本导致信息丢失。类比初级工程师写完代码后由资深工程师 review 再修改，确保质量闭环。

## 关键代码示例

```python
from langgraph.graph import StateGraph, END

def generator(state):
    # 结构化输出 DraftCode
    return {"draft": llm(f"Write code for: {state['user_request']}")}

def critic(state):
    # 结构化输出 Critique
    return {"critique": llm(f"Review: {state['draft']}")}

def refiner(state):
    return {"refined": llm(f"Refine: {state['critique']}")}

graph = StateGraph(dict)
graph.add_node("generator", generator)
graph.add_node("critic", critic)
graph.add_node("refiner", refiner)
graph.set_entry_point("generator")
graph.add_edge("generator", "critic")
graph.add_edge("critic", "refiner")
graph.add_edge("refiner", END)
```

这段代码展示了反思架构的核心骨架：三个节点分别对应生成、批判、精炼；critic 节点通过 with_structured_output 强制输出结构化 Critique 对象，避免模型只给模糊评价；LangGraph 的 StateGraph 将节点线性串联，状态在节点间自动传递。

## 关键流程

1. 生成节点根据用户请求产出初始草稿
2. 批判节点以评审视角检查错误和效率，输出结构化批评意见
3. 精炼节点根据批评意见重写输出，形成改进版本

## 关键点

- 反思架构的核心是引入一个独立的批判节点，强制模型从错误、效率和最佳实践等具体维度审视自身输出，避免泛泛而谈。
- 使用 Pydantic 结构化输出（如 Critique 中的 has_errors 和 suggested_improvements）能保证节点间数据传递稳定，并迫使模型给出可执行的改进建议。
- LangGraph 的 StateGraph 将生成、批判、精炼串成线性工作流，状态通过 TypedDict 定义，便于调试和监控。
- 该模式特别适合对输出质量敏感的任务，如复杂代码生成和技术文档撰写，因为单次生成往往存在明显缺陷。
- 批判节点的提示词设计至关重要，应明确指定分析维度和输出格式，否则模型容易给出'看起来还行'的无效反馈。

## 对比与权衡

- 相比 ReAct 循环，反思架构更专注于单次输出的质量迭代，不涉及工具调用和外部环境交互，因此实现更简单、成本更低，但在需要实时信息或执行动作的场景中不如 ReAct 灵活。
- 相比多智能体集成决策（如投票），反思架构是串行改进而非并行采样，质量提升更明显，但会增加推理延迟和 token 成本。

## 自测问题

**问: 反思架构和普通的一次生成相比，主要增加了哪些成本？**

额外增加两轮 LLM 调用（批判和精炼），如果是简单任务，边际收益较低；建议根据任务复杂度动态决定是否启用，或采用轻量级规则检查替代部分批判。

**问: Critique 结构化输出为什么重要？如果不用会怎样？**

没有结构化约束，模型可能输出自由文本，难以程序化提取错误标志和改进建议，下游节点无法可靠消费；用 Pydantic 还能在解析失败时快速暴露问题。

**问: 反思架构可以嵌套多次循环吗？如何实现？**

可以，将精炼后的输出再次送入批判节点，形成循环，直到满足退出条件（如 has_errors 为 false 或达到最大迭代次数）；LangGraph 中可通过条件边实现。

**问: 反思架构适合所有 LLM 任务吗？**

不适合，简单分类或抽取任务只需一次生成，反思反而增加延迟和成本；它更适合生成质量要求高、有明确评估维度的任务。

**问: 如何评估反思架构的效果？**

可通过对比初始草稿和精炼版本的自动化指标（如代码测试通过率、BLEU/ROUGE、人工评分）以及 LangSmith 等追踪工具分析每次迭代的变化。

## 适用场景

- 复杂代码生成场景，如算法实现、系统设计原型，需要修正递归效率问题或边界情况。
- 技术文档或方案报告撰写，要求逻辑严谨、表述准确，通过自我批判改进质量。
- 需要可审计质量保障的自动化流水线，如 CI 中自动修复代码 review 意见。
- 教学演示或面试准备，展示智能体如何应用设计模式提升 LLM 可靠性。

## 标签

`AI智能体` `反思架构` `LangGraph` `LLM应用` `代码生成`
