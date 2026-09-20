# 构建 Claude 对话中的编排模式（Orchestration Mode）

*原文: [https://platform.claude.com/docs/en/build-with-claude/mid-conversation-effort-example](https://platform.claude.com/docs/en/build-with-claude/mid-conversation-effort-example) · 来源: web · 生成时间: 2026-09-20T03:06:48.531457+00:00*

## 背景

Claude API 提供 effort 参数（如 low/medium/high）控制模型推理深度，同时支持在对话中途修改系统消息和工具定义。固定单一配置的 AI 应用难以应对用户输入的多样性：简单问题用高 effort 浪费成本和延迟，复杂问题用低 effort 则推理不足。编排模式（orchestration mode）正是在这种背景下出现的一种应用层设计模式，用于根据任务特征动态调整模型行为。

## 痛点

如果开发者不了解或不用编排模式，要么对所有请求统一使用高 effort，导致成本高昂、响应变慢；要么统一使用低 effort，导致复杂任务质量下降、用户不满意。此外，无法在单个会话中根据上下文切换工具或系统指令，限制了 AI 应用的灵活性和智能化程度。

## 解决办法

核心方法是利用 Claude API 支持 mid-conversation system messages 和 tool changes 的能力，在应用层实现一个“编排器”。该编排器先对用户输入或对话状态进行轻量级判断（如规则、分类器或小模型），然后动态选择 effort 等级、系统提示词以及可用工具集，再调用 Claude API。类比于微服务网关根据请求路由到不同配置的服务实例。这样可以在同一对话中无缝切换快速响应模式和深度推理模式，同时保持上下文连贯。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

def orchestrate(user_input: str) -> str:
    # 简单启发式：根据输入长度和关键词判断复杂度
    if len(user_input) > 200 or "complex" in user_input.lower():
        effort = "high"
        system = "You are an expert problem solver. Think step by step and provide detailed reasoning."
        max_tokens = 2048
    else:
        effort = "low"
        system = "You are a helpful assistant. Provide concise answers."
        max_tokens = 512

    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=max_tokens,
        effort=effort,
        system=system,
        messages=[{"role": "user", "content": user_input}]
    )
    return response.content[0].text
```

这段代码演示了编排模式的核心：根据输入特征动态选择 effort 和 system prompt。简单的输入使用低 effort 和简短指令，复杂输入使用高 effort 和详细推理指令。这样既控制了成本，又保证了复杂任务的回答质量。实际项目中判断逻辑可以替换为更复杂的分类器或规则引擎。

## 关键流程

1. 识别任务复杂度：通过输入长度、关键词、用户历史行为或轻量级分类模型判断当前任务需要何种推理深度。
2. 定义编排策略：预设多组配置（如快速模式、标准模式、深度模式），每组包含 effort、system prompt、max_tokens 和工具集。
3. 实现路由逻辑：在应用代码中根据步骤1的结果选择对应的配置组。
4. 调用 Claude API：将选定的配置应用到 messages.create 请求中，并传入完整对话历史以保持上下文。
5. 处理响应与状态更新：根据模型输出决定是否需要在下一轮切换模式，例如检测到复杂追问时提升 effort。

## 关键点

- effort 参数直接控制模型内部推理的计算量，高 effort 通常带来更深思熟虑的回答，但成本和延迟更高。
- Claude API 支持在对话中途修改 system prompt，这允许开发者在不丢失上下文的情况下改变模型的行为指令。
- 工具集也可以动态调整，例如在简单问答时只提供基础工具，在复杂任务时开放代码执行或 web search 工具。
- 编排模式的关键在于“轻量级判断 + 动态路由”，避免对所有请求都使用最高配置。
- 这种模式可以扩展到多代理系统，由主 orchestrator 将子任务分派给不同配置的 worker 调用。
- 实现时要注意上下文长度管理，频繁切换 system prompt 可能影响缓存效率，需结合 prompt caching 优化。

## 对比与权衡

- 相比固定单一 effort 的模式，编排模式在成本控制上更好，能根据任务复杂度灵活调整，但实现复杂度更高。
- 相比使用多个独立专用代理（每个代理固定配置），编排模式在对话连贯性上更好，因为共享上下文，但在模块解耦上不如多代理清晰。
- 相比完全由模型自主决定 effort（如果未来支持），应用层编排模式提供了确定性和可控性，但需要额外维护路由逻辑。

## 自测问题

**问: 什么是 orchestration mode？在 Claude API 中如何实现？**

解释它是一种动态调整模型行为的应用层模式，通过 mid-conversation system message 和 effort 参数实现，核心是根据任务复杂度路由到不同配置，并举例说明。

**问: effort 参数和 temperature 有什么区别？**

effort 控制模型内部推理的计算量（如思考步骤数），影响答案深度；temperature 控制输出的随机性，影响创造性。两者正交，可以同时使用。

**问: 在对话中途修改 system prompt 有什么注意事项？**

修改 system prompt 会改变模型后续行为，但不会影响已生成的历史消息；频繁修改可能降低 prompt caching 命中率，增加成本；需要确保新指令与历史上下文不冲突。

**问: 如何判断一个任务应该使用高 effort 还是低 effort？**

可以使用规则（如输入长度、关键词）、轻量级分类模型，或让一个低 effort 的初步调用输出复杂度标签，再决定是否用高 effort 重新处理。

**问: 编排模式与多代理架构有什么区别？**

编排模式通常在一个对话线程内动态切换配置，保持单一上下文；多代理架构将任务拆解给多个独立代理，每个代理有自己上下文，通过消息传递协作。编排模式更轻量，适合单会话内复杂度波动；多代理适合并行、分工明确的场景。

## 适用场景

- 智能客服：简单查询用低 effort 快速回复，复杂投诉或技术问题自动切换高 effort 并启用知识库工具。
- 代码助手：一般代码补全用低 effort，检测到复杂调试或架构设计请求时提升 effort 并开放代码执行工具。
- 研究助手：用户提出浅层问题时简要回答，当追问深入或要求分析时切换高 effort 和 web search 工具。
- 内容生成流水线：初稿生成用标准模式，用户要求修改或优化时提升 effort 进行深度润色。

## 标签

`Claude API` `Orchestration` `Effort` `Context Management` `Cost Optimization`
