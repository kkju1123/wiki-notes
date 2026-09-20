# Claude SDK 的 Tool Runner：自动化的 Agentic 工具调用循环

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner) · 来源: web · 生成时间: 2026-09-20T03:42:12.575650+00:00*

## 背景

在使用 LLM 工具调用时，开发者通常需要手动编写 agentic loop：发送请求、解析 tool_use、执行工具、把 tool_result 拼回消息、再继续请求，直到模型不再调用工具。这个循环涉及多轮状态管理和易错的格式转换，SDK 提供 tool runner 抽象来封装这些通用逻辑。

## 痛点

手动循环容易出现消息顺序错误、工具结果格式不符合 API 要求、异常未包装导致模型误解、退出条件难以控制，并且每个项目都要重复实现这些样板逻辑。

## 解决办法

tool runner 以可迭代状态机的形式运行：每次迭代向 Messages API 发出当前状态请求并产生一个响应；如果没有外部修改，它会自动追加 assistant 消息和工具结果，并在检测到工具调用时执行对应函数，把返回值或异常封装为 tool_result 继续循环。开发者可以在循环体内读取、修改消息历史来接管自动追加，实现重试、注入或自定义错误处理。类型安全通过 SDK 提供的工具定义辅助函数和结果类型来保证。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

def get_weather(city: str) -> str:
    if city == 'Mars':
        raise ValueError('No weather data')
    return f'{city}: sunny, 22C'

tools = [{
    'name': 'get_weather',
    'description': 'Get current weather for a city',
    'input_schema': {
        'type': 'object',
        'properties': {'city': {'type': 'string'}},
        'required': ['city']
    }
}]

messages = [{'role': 'user', 'content': "What's the weather in Paris?"}]

runner = client.beta.tools.run(
    model='claude-sonnet-4-5',
    messages=messages,
    tools=tools,
    max_iterations=10,
)

for response in runner:
    if response.stop_reason == 'tool_use':
        # runner 已自动执行 get_weather 并把 tool_result 发回；
        # 这里可以检查/修改返回的工具结果
        continue
    print(response.content)
```

这段代码先定义了一个带 input_schema 的工具 get_weather，然后通过 client.beta.tools.run 初始化一个 tool runner，并传入模型、消息、工具和最大迭代次数。在 for 循环中，每次迭代得到一个 Claude 响应；如果 stop_reason 为 tool_use，说明 runner 已经自动执行了对应工具并准备继续循环，开发者可以在此处检查或修改结果；否则打印最终回复。max_iterations 和 break 条件保证了循环有界。

## 关键流程

1. 定义工具：使用 SDK helper 声明名称、描述和 input_schema，工具返回字符串或内容块，结构化数据需先序列化为字符串。
2. 初始化 runner：传入 model、messages、tools 和可选的 max_iterations。
3. 迭代 runner：每轮迭代获得一条 Claude 消息；若消息包含工具调用，runner 自动执行工具并发送结果。
4. 在循环体内读取/修改状态：可访问消息、接管消息历史、修改工具结果、设置 break 条件。
5. 循环退出：当消息不含工具调用、循环体 break 或达到 max_iterations 时结束。

## 关键点

- tool runner 自动完成 agentic loop 的核心步骤，包括工具执行、结果回传和对话状态追加，减少重复代码和格式错误。
- 工具异常默认被捕获并作为 is_error:true 的 tool_result 返回给模型，但完整堆栈只在 Python 中通过 logging 自动记录；其他语言需要开发者在工具内部自行捕获日志。
- 开发者可以在循环体内修改 runner 的消息历史来接管自动行为，用于重试回合、注入后续消息或自定义工具结果，但需要自己保证消息历史有效和循环可退出。
- 通过 Python/TypeScript 的 tool response hook 可以拦截工具结果，在发送给模型前检查错误或添加 cache_control 等元数据。
- max_iterations 是控制 agentic loop 有界的重要参数，所有 SDK 都支持；长期任务还需配合服务端 compaction 管理上下文。

## 对比与权衡

- 相比手动 loop：tool runner 在自动化程度、错误包装、类型安全和状态管理上更好，但在人类审批、自定义日志、条件执行等需要介入流程的场景上灵活性不如手动 loop。
- 相比客户端 compaction（TS/Ruby 曾支持）：服务端 compaction 对所有 SDK 的 tool runner 可用，并且更符合生产环境，因为上下文摘要由服务端统一管理，不再依赖特定语言的客户端实现。

## 自测问题

**问: tool runner 和手动处理 tool calls 的主要区别是什么？**

tool runner 把 agentic loop 封装为可迭代对象，自动运行工具、追加 assistant/tool 消息、处理错误；手动循环需要自行管理消息状态、工具结果和退出条件，但可以灵活插入人工批准等流程。

**问: 工具函数抛异常时，tool runner 会做什么？如何排查错误？**

它会捕获异常，将异常消息包装为 is_error:true 的 tool_result 发给 Claude，让模型可以感知失败并做出修正。Python 的 runner 会通过标准 logging 记录完整堆栈，其他语言需要开发者在工具内 catch 后记录日志再重抛。

**问: 什么是 taking over message history？什么场景下需要？**

在迭代循环体内直接修改 runner 的消息历史，runner 会自动跳过该轮的追加逻辑，由开发者接管控制。典型场景包括重试某一轮、丢弃不合适的响应、注入后续澄清消息或自行构建工具结果。

**问: 如何避免 agentic loop 无限循环？**

设置 max_iterations 并配合 break 条件；默认当模型返回无工具调用的消息时会退出。接管消息历史时也要确保修改后有能退出的状态路径，否则可能浪费 API 调用。

**问: 工具结果需要加缓存标记或其他元数据，该怎么做？**

在 Python/TypeScript 中使用 tool response 方法获取工具结果，在 runner 自动发送前修改它，比如添加 cache_control 以启用 prompt caching；其他 SDK 可能需要更手动的方式。

## 适用场景

- 快速构建需要多轮工具调用的 Claude agent 或聊天机器人，避免重复实现循环逻辑。
- 需要类型安全工具定义和结果验证的生产级应用。
- 需要在发送给模型前检查或修改工具结果的场景，如错误拦截、结果脱敏或缓存标记。
- 长任务代理，需要配合服务端 compaction 管理上下文，同时使用 max_iterations 控制成本。

## 标签

`Claude SDK` `Tool Runner` `Agentic Loop` `工具调用` `错误处理`
