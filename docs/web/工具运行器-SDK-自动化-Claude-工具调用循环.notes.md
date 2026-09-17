# 工具运行器（SDK）：自动化 Claude 工具调用循环

*原文: [https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner) · 来源: web · 生成时间: 2026-09-17T03:02:25.863888+00:00*

## 背景

Claude 的 tool use 是基于多轮对话实现的：模型返回 tool_use 块，应用执行对应工具后把结果作为 tool_result 发回，循环直到模型输出最终文本。手动实现这个循环需要维护消息数组、判断 stop_reason、处理并行工具调用和异常重试，重复且容易出错。为了降低集成门槛，Anthropic 在官方 SDK 中提供了工具运行器来封装这些通用逻辑。

## 痛点

如果没有工具运行器，开发者需要自己编写 agentic loop：解析每条响应中的 tool_use、执行工具、追加 assistant/tool 消息、再次请求模型。这个过程涉及大量模板代码，尤其在并行工具调用、工具执行异常、输入输出校验等场景下非常容易遗漏边界条件，导致循环卡死或状态不一致。

## 解决办法

工具运行器将工具调用循环抽象为声明式配置：开发者只需传入模型、工具定义和工具实现映射，运行器内部自动完成“模型请求工具 -> 执行工具 -> 回传结果”的循环。它监听模型的 stop_reason/tool_use 内容块，自动调用对应函数并将返回值包装为 tool_result 消息；同时维护完整对话历史，并利用 JSON Schema 对工具输入输出进行类型校验。当工具抛出异常时，运行器会捕获异常并转换为模型可理解的错误结果，避免整个会话中断。可以把它类比为轻量级的官方 agent executor：保留核心循环能力，但不引入复杂的框架抽象。

## 关键代码示例

```python
def run_with_tool_runner(client, model, messages, tools, implementations):
    while True:
        response = client.messages.create(
            model=model,
            messages=messages,
            tools=tools,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return response

        for block in response.content:
            if block.type == "tool_use":
                fn = implementations[block.name]
                result = fn(**block.input)
                messages.append({
                    "role": "user",
                    "content": [{
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    }]
                })
```

这段代码展示了工具运行器内部自动循环的核心原理：每次收到响应后检查 stop_reason 是否为 tool_use；如果是则遍历所有 tool_use 块，调用对应的实现函数，并把返回值包装成 tool_result 消息追加到对话中，然后继续下一轮请求，直到模型返回非 tool_use 的最终结果。真实 SDK 还会处理并行执行、异常封装、输入 schema 校验和输出类型安全，但这个简化版本已经说明了它替开发者省掉的那部分重复逻辑。

## 关键流程

1. 定义工具的 JSON Schema，包括名称、描述和 input_schema。
2. 准备与 schema 对应的工具实现函数，确保输入输出类型匹配。
3. 创建 SDK client 并初始化工具运行器，传入模型、工具定义和实现映射。
4. 调用运行器的 run 方法并传入用户消息，运行器自动执行工具调用循环。
5. 获取最终自然语言响应或最终结果对象。

## 关键点

- 工具运行器自动执行“模型要求调用工具 -> 运行工具 -> 返回结果”的循环，避免手动管理多轮对话状态。
- 它提供错误包装能力：工具抛出的异常会被捕获并转成模型可以理解的 tool_result，让循环继续而不是崩溃。
- 它通过 JSON Schema 对工具输入输出进行验证，提供类型安全，减少无效调用和参数错误。
- 工具运行器目前处于 beta 阶段，API 可能变动，生产环境使用时应关注版本更新和迁移指南。
- 当需要人工审批、自定义日志记录或条件执行时，应改用手动循环，因为封装后的运行器不易插入这类定制逻辑。
- 该能力已在 Python、TypeScript 和 C# SDK 中提供，适合不同技术栈的团队采用。

## 对比与权衡

- 相比手动实现 agentic loop，工具运行器大幅减少样板代码和状态管理错误，但在灵活性上不如手动循环：人工审批、条件分支和细粒度日志等定制行为难以插入。
- 相比 LangChain/LlamaIndex 等框架的 agent executor，工具运行器更轻量、由官方 SDK 维护、无额外抽象依赖，但缺少跨模型生态支持和复杂工作流编排能力。

## 面试可能会问

**问: 工具运行器解决了什么问题？**

解释 tool use 需要多轮 tool_use/tool_result 循环，手动实现需要管理消息状态、stop_reason 判断、并行工具调用和错误处理；工具运行器自动完成这些，并用类型校验提升可靠性。

**问: 什么场景下不应该使用工具运行器？**

需要人工审批、自定义日志记录、条件执行或需要精确控制每一步时，应该使用手动循环；同时 beta 阶段也要考虑 API 稳定性。

**问: 工具运行器如何保证类型安全？**

通过工具的 JSON Schema 定义输入结构，SDK 在调用工具前后进行参数和返回值校验，避免把无效参数传给模型或把错误类型的结果返回给模型。

**问: 并行工具调用在工具运行器中如何处理？**

模型可能在一个响应中返回多个 tool_use 块，工具运行器会识别并执行所有块，聚合所有 tool_result 后再回传；手动循环容易漏掉部分块或顺序错乱。

**问: 如果工具执行过程中抛出异常怎么办？**

工具运行器会捕获异常并包装成结构化的错误 tool_result，让模型看到错误信息后可以调整参数或改变策略，而不是直接中断整个会话。

## 适用场景

- 快速原型开发：需要让 Claude 调用几个工具完成简单任务，不想手写循环样板。
- 内部工具集成：将已有的 Python/TypeScript/C# 函数暴露给 Claude，由 SDK 自动托管交互。
- 多轮工具调用任务：如查数据库、调外部 API、聚合结果等需要多次往返的场景。
- 教学与演示：展示 Claude tool use 的基本流程，减少干扰代码，聚焦业务工具本身。

## 标签

`Claude API` `工具调用` `SDK` `智能体循环` `函数调用`
