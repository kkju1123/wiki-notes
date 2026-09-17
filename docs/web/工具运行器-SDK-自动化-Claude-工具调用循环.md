---
title: 工具运行器（SDK）：自动化 Claude 工具调用循环
url: https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner
source_type: web
author: null
tags:
- Claude API
- 工具调用
- SDK
- 智能体循环
- 函数调用
summary: 工具运行器是 Anthropic SDK 提供的辅助层，自动执行工具调用、管理对话状态与错误处理，替代手动编写 agentic loop 样板代码。
fetched_at: '2026-09-17T03:02:25.863888+00:00'
---

[Claude Platform Docs](https://platform.claude.com/docs/zh-CN/home)
*   [Messages](https://platform.claude.com/docs/zh-CN/intro)
*   [Managed Agents](https://platform.claude.com/docs/zh-CN/managed-agents/overview)
*   [管理](https://platform.claude.com/docs/zh-CN/manage-claude/admin-api)
*   
资源
    *   [最佳实践](https://platform.claude.com/docs/zh-CN/about-claude/use-case-guides/overview)
    *   [模型与定价](https://platform.claude.com/docs/zh-CN/models/overview)
    *   [CLI、SDK 和库](https://platform.claude.com/docs/zh-CN/cli-sdks-libraries/overview)
    *   [Claude API 技能](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/claude-api-skill)
    *   [发布说明](https://platform.claude.com/docs/zh-CN/release-notes/overview)

[API reference](https://platform.claude.com/docs/zh-CN/api/overview)

简体中文

[Console](https://platform.claude.com/)[Log in](https://platform.claude.com/login?returnTo=%2Fdocs%2Fzh-CN%2Fagents-and-tools%2Ftool-use%2Ftool-runner)



Search Ctrl K

入门

[Claude 简介](https://platform.claude.com/docs/zh-CN/intro)[获取 API 密钥](https://platform.claude.com/docs/zh-CN/get-api-key)[快速开始](https://platform.claude.com/docs/zh-CN/get-started)[身份验证](https://platform.claude.com/docs/zh-CN/manage-claude/authentication)

使用 Claude 构建

[功能概览](https://platform.claude.com/docs/zh-CN/build-with-claude/overview)[使用 Messages API](https://platform.claude.com/docs/zh-CN/build-with-claude/working-with-messages)[停止原因与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/handling-stop-reasons)[拒绝与回退](https://platform.claude.com/docs/zh-CN/build-with-claude/refusals-and-fallback)[回退额度](https://platform.claude.com/docs/zh-CN/build-with-claude/fallback-credit)

模型能力

[Effort](https://platform.claude.com/docs/zh-CN/build-with-claude/effort)[任务预算（测试版）](https://platform.claude.com/docs/zh-CN/build-with-claude/task-budgets)[快速模式（研究预览版）](https://platform.claude.com/docs/zh-CN/build-with-claude/fast-mode)[结构化输出](https://platform.claude.com/docs/zh-CN/build-with-claude/structured-outputs)[引用](https://platform.claude.com/docs/zh-CN/build-with-claude/citations)[流式传输消息](https://platform.claude.com/docs/zh-CN/build-with-claude/streaming)[批处理](https://platform.claude.com/docs/zh-CN/build-with-claude/batch-processing)[搜索结果](https://platform.claude.com/docs/zh-CN/build-with-claude/search-results)[流式传输拒绝](https://platform.claude.com/docs/zh-CN/test-and-evaluate/strengthen-guardrails/handle-streaming-refusals)[多语言支持](https://platform.claude.com/docs/zh-CN/build-with-claude/multilingual-support)[嵌入](https://platform.claude.com/docs/zh-CN/build-with-claude/embeddings)

[思考](https://platform.claude.com/docs/zh-CN/build-with-claude/thinking)

工具

[概览](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/overview)[工具使用的工作原理](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/how-tool-use-works)[教程：构建使用工具的智能体](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/build-a-tool-using-agent)[定义工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/define-tools)[处理工具调用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/handle-tool-calls)[并行工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/parallel-tool-use)[Tool Runner（SDK）](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner)[严格工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/strict-tool-use)[服务器工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/server-tools)[网页搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-search-tool)[网页抓取工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/web-fetch-tool)[代码执行工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/code-execution-tool)[顾问工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/advisor-tool)[工具搜索工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-search-tool)[记忆工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/memory-tool)[Bash 工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/bash-tool)[文本编辑器工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/text-editor-tool)[计算机使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/computer-use-tool)[浏览器使用工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/browser-use-tool)[故障排除](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/troubleshooting-tool-use)

工具基础设施

[工具参考](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-reference)[管理工具上下文](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/manage-tool-context)[工具组合](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-combinations)[工具使用与提示缓存](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-use-with-prompt-caching)[编程式工具调用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/programmatic-tool-calling)[细粒度工具流式传输](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/fine-grained-tool-streaming)

上下文管理

[上下文窗口](https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows)[压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)[上下文编辑](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing)[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)[对话中途的系统消息与工具变更](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-system-messages)[构建编排模式](https://platform.claude.com/docs/zh-CN/build-with-claude/mid-conversation-effort-example)[缓存诊断（测试版）](https://platform.claude.com/docs/zh-CN/build-with-claude/cache-diagnostics)[Token 计数](https://platform.claude.com/docs/zh-CN/build-with-claude/token-counting)

处理文件

[Files API](https://platform.claude.com/docs/zh-CN/build-with-claude/files)[PDF 支持](https://platform.claude.com/docs/zh-CN/build-with-claude/pdf-support)

[图像与视觉](https://platform.claude.com/docs/zh-CN/build-with-claude/vision)

技能

[概览](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/overview)[快速开始](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/quickstart)[最佳实践](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/best-practices)[企业版技能](https://platform.claude.com/docs/zh-CN/agents-and-tools/agent-skills/enterprise)[在 API 中使用技能](https://platform.claude.com/docs/zh-CN/build-with-claude/skills-guide)

MCP

[远程 MCP 服务器](https://platform.claude.com/docs/zh-CN/agents-and-tools/remote-mcp-servers)[MCP 连接器](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-connector)

[MCP 隧道](https://platform.claude.com/docs/zh-CN/agents-and-tools/mcp-tunnels/overview)

云平台上的 Claude

[Amazon Bedrock（Opus 4.7 及更高版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-amazon-bedrock)[Amazon Bedrock（Opus 4.6 及更早版本）](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-amazon-bedrock-legacy)[AWS 上的 Claude Platform](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-platform-on-aws)[Google Cloud](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-on-vertex-ai)[Microsoft Foundry](https://platform.claude.com/docs/zh-CN/build-with-claude/claude-in-microsoft-foundry)

[Console](https://platform.claude.com/)

[Messages](https://platform.claude.com/docs/zh-CN/intro)工具

# 工具运行器（SDK）

Copy page



使用 SDK 的工具运行器自动处理智能体循环、错误包装和类型安全。

Copy page



"Tool runner"（工具运行器）会为您处理 "agentic loop"（智能体循环）、错误包装和类型安全，让您无需亲自处理。当您需要人工参与审批、自定义日志记录或条件执行时，请改用[手动循环](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/handle-tool-calls)。

工具运行器无需您手动处理工具调用、工具结果和对话管理，而是自动：

*   在 Claude 调用工具时运行工具
*   处理请求/响应周期
*   管理对话状态
*   提供类型安全和验证



工具运行器目前处于 beta 阶段，可在 [Python SDK](https://github.com/anthropics/anthropic-sdk-python/blob/main/tools.md)、[TypeScript SDK](https://github.com/anthropics/anthropic-sdk-typescript/blob/main/helpers.md#tool-helpers)、[C# SDK](https://github.com/anthropics/anthropic-sdk-csharp/blob/main/examples/ToolRunnerExample/Program.cs)、[Go SDK](https://github.com/anthropics/anthropic-sdk-go/blob/main/tools.md)、[Java SDK](https://github.com/anthropics/anthropic-sdk-java/blob/main/anthropic-java-example/src/main/java/com/anthropic/example/BetaToolRunnerExample.java)、[PHP SDK](https://github.com/anthropics/anthropic-sdk-php/blob/main/examples/beta/beta_tool_runner.php) 和 [Ruby SDK](https://github.com/anthropics/anthropic-sdk-ruby/blob/main/helpers.md#3-auto-looping-tool-runner-beta) 中使用。

## 基本用法

使用 SDK 辅助工具定义工具，然后使用工具运行器运行它们。

根据 SDK 的工具签名，工具以字符串或内容块（文本、图像或文档块）的形式返回结果，因此工具可以返回多模态结果。返回的字符串会成为单个文本内容块。要返回结构化数据（例如 JSON 对象或数字），请先将其编码为字符串。

Python TypeScript C#Go Java PHP Ruby

使用 `@beta_tool` 装饰器通过类型提示和文档字符串定义工具。



如果您使用的是异步客户端，请将 `@beta_tool` 替换为 `@beta_async_tool`，并使用 `async def` 定义函数。

```
import json
from anthropic import Anthropic, beta_tool

client = Anthropic()

@beta_tool
def get_weather(location: str, unit: str = "fahrenheit") -> str:
    """Get the current weather in a given location.

    Args:
        location: The city and state, e.g. San Francisco, CA
        unit: Temperature unit, either 'celsius' or 'fahrenheit'
    """
    return json.dumps({"temperature": "20°C", "condition": "Sunny"})

@beta_tool
def calculate_sum(a: int, b: int) -> str:
    """Add two numbers together.

    Args:
        a: First number
        b: Second number
    """
    return str(a + b)

runner = client.beta.messages.tool_runner(
    model="claude-opus-5",
    max_tokens=1024,
    tools=[get_weather, calculate_sum],
    messages=[
        {
            "role": "user",
            "content": "What's the weather like in Paris? Also, what's 15 + 27?",
        }
    ],
)
for message in runner:
    print(message)
```



`@beta_tool` 装饰器会检查函数参数和文档字符串，为您推导出 JSON schema。

## 迭代工具运行器

工具运行器是一个可迭代对象，会产出来自 Claude 的消息。在每次迭代中，运行器会检查 Claude 是否请求了工具使用。如果是，它会运行该工具并自动将结果发送回 Claude，然后产出来自 Claude 的下一条消息以继续您的循环。

您可以在任意一次迭代中使用 `break` 语句结束循环。运行器会一直循环，直到 Claude 返回一条不含工具使用的消息，或者（如果您设置了 `max_iterations`）直到达到 `max_iterations`。

如果您不需要中间消息，可以直接获取最终消息：

Python TypeScript C#Go Java PHP Ruby

使用 `runner.until_done()` 获取最终消息。

```
client = anthropic.Anthropic()
# ...
runner = client.beta.messages.tool_runner(
    model="claude-opus-5",
    max_tokens=1024,
    tools=[get_weather, calculate_sum],
    messages=[
        {
            "role": "user",
            "content": "What's the weather like in Paris? Also, what's 15 + 27?",
        }
    ],
)
final_message = runner.until_done()
for block in final_message.content:
    if block.type == "text":
        print(block.text)
```



## 高级用法

在循环内部，您可以读取每条响应消息，并在下一次 API 调用之前修改运行器的状态。每次迭代遵循以下生命周期：

1.   运行器使用其当前状态向 Messages API 发送请求。
2.   运行器将响应消息产出给您的循环体。
3.   您的循环体运行。您可以读取消息，并可选择修改运行器的状态。
4.   当您的循环体返回时，运行器会检查您是否修改了其消息历史。
    *   **如果您未修改消息历史：** 如果消息包含工具调用，运行器会追加助手消息和工具结果，然后继续。如果没有工具调用，循环退出。
    *   **如果您修改了消息历史：** 运行器会跳过自动追加，并原样使用您的状态。请参阅[接管消息历史](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner#taking-over-message-history)。

### 接管消息历史

默认情况下，运行器会为您管理对话状态：在每个工具调用轮次之后，它会将助手消息和所有工具结果追加到自己的消息历史中。当您想要重试某个轮次（丢弃响应并重新发送）、注入后续消息或自行构建工具结果时，您可以接管消息历史。

您可以通过在循环体内部修改运行器的消息来接管。具体方法取决于 SDK。请参阅下面各语言的标签页。

当您在某次迭代中接管时，运行器不会追加该轮次的助手消息或工具结果。您需要负责保持对话有效：自行追加助手消息和工具结果（如果您希望该轮次计入），有条件地修改状态以便在没有工具调用时循环仍能退出，并传入 `max_iterations` 以限制循环次数。全部七个 SDK 都支持 `max_iterations`。

Python TypeScript C#Go Java PHP Ruby

使用 `generate_tool_call_response()` 检查或计算工具结果。在循环内部调用 `append_messages()` 会告知运行器您正在自行管理历史，因此请在您追加的内容中包含助手消息和工具结果。

```
runner = client.beta.messages.tool_runner(
    model="claude-opus-5",
    max_tokens=1024,
    max_iterations=10,
    tools=[get_weather],
    messages=[{"role": "user", "content": "What's the weather in San Francisco?"}],
)

for message in runner:
    tool_response = runner.generate_tool_call_response()
    if tool_response is not None:
        # append_messages() 会将状态标记为已修改，因此 runner 会跳过
        # 本次迭代的自动追加。您需要自行追加助手消息和
        # 工具结果，以及任何后续内容。
        runner.append_messages(
            message,
            tool_response,
            {"role": "user", "content": "Please be concise."},
        )
    # 当没有工具调用时，保持状态不变，以便循环退出。
```



要在不接管消息历史的情况下更改 `max_tokens` 等请求参数，请使用 `set_messages_params()`。运行器仍会自动追加助手消息和工具结果。

```
for message in runner:
    runner.set_messages_params(lambda params: {**params, "max_tokens": 2048})
```



### 自动上下文管理

对于长时间运行的智能体任务，TypeScript 和 Ruby 工具运行器支持自动[压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/context-editing#client-side-compaction-sdk)，当令牌使用量超过阈值时会生成摘要，使对话能够超越 "context window"（上下文窗口）限制继续进行。这两个 SDK 都已弃用此客户端选项，转而推荐[服务端压缩](https://platform.claude.com/docs/zh-CN/build-with-claude/compaction)，后者通过 `context_management` 请求参数适用于每个 SDK 的工具运行器。Python SDK（v1.0 及更高版本）以及 Go、Java、C# 和 PHP 工具运行器不包含客户端压缩。

### 调试工具执行

当工具抛出异常时，工具运行器会捕获它，并将错误作为带有 `is_error: true` 的工具结果返回给 Claude。工具结果携带异常的消息（在 Python 中为其类型和消息），而非完整的堆栈跟踪。

SDK 记录的日志内容因语言而异。每当工具引发未处理的异常时，Python SDK 会通过标准 `logging` 模块记录完整的异常，包括其堆栈跟踪。Python、TypeScript 和 Java SDK 会读取 `ANTHROPIC_LOG` 环境变量以开启 SDK 的日志记录，其中包括请求和响应详情：

```
# 以 info 级别记录日志
export ANTHROPIC_LOG=info

# 以 debug 级别记录日志以获得更详细的输出
export ANTHROPIC_LOG=debug
```



Go、Ruby、C# 和 PHP SDK 不读取 `ANTHROPIC_LOG`。除 Python 外，没有 SDK 会记录失败的工具：要查看工具失败的原因，请在工具函数内部捕获并记录异常，然后再返回或重新抛出。

### 拦截工具错误

默认情况下，工具错误会传回给 Claude，Claude 随后可以做出适当响应。但是，您可能希望检测错误并以不同方式处理，例如提前停止执行或实现自定义错误处理。

在 Python 和 TypeScript SDK 中，使用工具响应方法（Python 中为 `generate_tool_call_response()`，TypeScript 中为 `generateToolResponse()`）拦截工具结果，并在发送给 Claude 之前检查错误。其他 SDK 不公开该钩子。它们的标签页描述了最接近的替代方案：

Python TypeScript C#Go Java PHP Ruby

```
client = anthropic.Anthropic()
# ...
runner = client.beta.messages.tool_runner(
    model="claude-opus-5",
    max_tokens=1024,
    tools=[my_tool],
    messages=[{"role": "user", "content": "Run my_tool with the query 'hello'."}],
)

for message in runner:
    tool_response = runner.generate_tool_call_response()

    if tool_response is not None:
        # tool_response 是一个 dict：{"role": "user", "content": [...]}
        # 检查是否有任何工具结果包含错误
        for block in tool_response["content"]:
            if block.get("is_error"):
                # 选项 1：抛出异常以停止循环
                raise RuntimeError(f"Tool failed: {json.dumps(block['content'])}")

                # 选项 2：记录日志并继续（交由 Claude 处理）
                # logger.error(f"Tool error: {json.dumps(block['content'])}")

    # 正常处理消息
    print(message.content)
```



### 修改工具结果

您可以在工具结果发送回 Claude 之前修改它们。这对于添加 `cache_control` 等元数据以在工具结果上启用 "prompt caching"（提示缓存），或转换工具输出非常有用。请参阅[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)。

在 Python 和 TypeScript SDK 中，使用工具响应方法获取工具结果，然后在运行器继续之前修改它。是显式追加修改后的结果还是就地变更，取决于 SDK。请参阅每个标签页中的代码注释。

Python TypeScript C#Go Java PHP Ruby

```
client = anthropic.Anthropic()
# ...
runner = client.beta.messages.tool_runner(
    model="claude-opus-5",
    max_tokens=1024,
    tools=[search_documents],
    messages=[
        {
            "role": "user",
            "content": "Search for information about the climate of San Francisco",
        }
    ],
)

for message in runner:
    tool_response = runner.generate_tool_call_response()

    if tool_response is not None:
        # tool_response 是一个 dict：{"role": "user", "content": [...]}
        # 修改工具结果以添加缓存控制
        for block in tool_response["content"]:
            if block["type"] == "tool_result":
                # 添加 cache_control 以缓存此工具结果
                block["cache_control"] = {"type": "ephemeral"}

        # 追加修改后的响应（这会阻止自动追加原始响应）
        runner.append_messages(message, tool_response)

    print(message.content)
```





当工具返回大量数据（例如文档搜索结果）且您希望为后续 API 调用缓存这些数据时，向工具结果添加 `cache_control` 尤其有用。有关缓存策略的更多详情，请参阅[提示缓存](https://platform.claude.com/docs/zh-CN/build-with-claude/prompt-caching)。

## 流式传输

启用 "streaming"（流式传输）以增量处理每个轮次的响应。每次迭代会产出一个流对象，您可以迭代它以获取事件。

Python TypeScript C#Go Java PHP Ruby

设置 `stream=True` 并使用 `get_final_message()` 获取累积的消息。

```
client = anthropic.Anthropic()
# ...
runner = client.beta.messages.tool_runner(
    model="claude-opus-5",
    max_tokens=1024,
    tools=[calculate_sum],
    messages=[{"role": "user", "content": "What is 15 + 27?"}],
    stream=True,
)

# 流式传输时，runner 返回 BetaMessageStream
for message_stream in runner:
    for event in message_stream:
        print("event:", event)
    print("message:", message_stream.get_final_message())

print(runner.until_done())
```



## 后续步骤



[严格工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/strict-tool-use)

通过语法约束采样强制 Claude 的工具输入符合 JSON Schema。

[处理工具调用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/handle-tool-calls)

解析 `tool_use` 块、格式化 `tool_result` 响应，并使用 `is_error` 处理错误。



[并行工具使用](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/parallel-tool-use)

启用、格式化和禁用并行工具调用，并提供消息历史指导和故障排除。



[定义工具](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/define-tools)

指定工具 schema、编写有效的描述，并控制 Claude 何时调用您的工具。

Was this page helpful?



*   [基本用法](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner#basic-usage)
*   [迭代工具运行器](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner#iterating-over-the-tool-runner)
*   [高级用法](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner#advanced-usage)
*   [接管消息历史](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner#taking-over-message-history)
*   [自动上下文管理](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner#automatic-context-management)
*   [调试工具执行](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner#debugging-tool-execution)
*   [拦截工具错误](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner#intercepting-tool-errors)
*   [修改工具结果](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner#modifying-tool-results)
*   [流式传输](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner#streaming)
*   [后续步骤](https://platform.claude.com/docs/zh-CN/agents-and-tools/tool-use/tool-runner#next-steps)

工具运行器（SDK）

[Claude Platform Docs](https://platform.claude.com/docs/zh-CN/home)

[](https://x.com/claudeai)[](https://www.threads.com/@claudeai)[](https://www.linkedin.com/showcase/claude)[](https://www.youtube.com/@anthropic-ai)[](https://instagram.com/claudeai)



### Solutions

*   [AI agents](https://claude.com/solutions/agents)
*   [Code modernization](https://claude.com/solutions/code-modernization)
*   [Coding](https://claude.com/solutions/coding)
*   [Customer support](https://claude.com/solutions/customer-support)
*   [Financial services](https://claude.com/solutions/financial-services)
*   [Government](https://claude.com/solutions/government)
*   [Higher education](https://claude.com/solutions/education)
*   [K-12 teachers](https://claude.com/solutions/teachers)
*   [Life sciences](https://claude.com/solutions/life-sciences)

### Partners

*   [Claude on AWS](https://claude.com/partners/amazon-bedrock)
*   [Claude on Google Cloud](https://claude.com/partners/google-cloud-vertex-ai)

### Learn

*   [Blog](https://claude.com/blog)
*   [Courses](https://claude.com/resources/courses)
*   [Use cases](https://claude.com/resources/use-cases)
*   [Connectors](https://claude.com/partners/mcp)
*   [Customer stories](https://claude.com/customers)
*   [Engineering at Anthropic](https://www.anthropic.com/engineering)
*   [Events](https://www.anthropic.com/events)
*   [Powered by Claude](https://claude.com/partners/powered-by-claude)
*   [Service partners](https://claude.com/partners/services)
*   [Startups program](https://claude.com/programs/startups)

### Company

*   [Anthropic](https://www.anthropic.com/company)
*   [Careers](https://www.anthropic.com/careers)
*   [Economic Futures](https://www.anthropic.com/economic-futures)
*   [Research](https://www.anthropic.com/research)
*   [News](https://www.anthropic.com/news)
*   [Responsible Scaling Policy](https://www.anthropic.com/news/announcing-our-updated-responsible-scaling-policy)
*   [Security and compliance](https://trust.anthropic.com/)
*   [Transparency](https://www.anthropic.com/transparency)

### Learn

*   [Blog](https://claude.com/blog)
*   [Courses](https://claude.com/resources/courses)
*   [Use cases](https://claude.com/resources/use-cases)
*   [Connectors](https://claude.com/partners/mcp)
*   [Customer stories](https://claude.com/customers)
*   [Engineering at Anthropic](https://www.anthropic.com/engineering)
*   [Events](https://www.anthropic.com/events)
*   [Powered by Claude](https://claude.com/partners/powered-by-claude)
*   [Service partners](https://claude.com/partners/services)
*   [Startups program](https://claude.com/programs/startups)

### Help and security

*   [Availability](https://www.anthropic.com/supported-countries)
*   [Status](https://status.claude.com/)
*   [Support](https://support.claude.com/)
*   [Discord](https://www.anthropic.com/discord)

### Terms and policies

*   [Privacy policy](https://www.anthropic.com/legal/privacy)
*   [Responsible disclosure policy](https://www.anthropic.com/responsible-disclosure-policy)
*   [Terms of service: Commercial](https://www.anthropic.com/legal/commercial-terms)
*   [Terms of service: Consumer](https://www.anthropic.com/legal/consumer-terms)
*   [Usage policy](https://www.anthropic.com/legal/aup)

Ask Docs