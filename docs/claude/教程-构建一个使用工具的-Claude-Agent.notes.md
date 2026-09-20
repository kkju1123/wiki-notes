# 教程：构建一个使用工具的 Claude Agent

*原文: [https://platform.claude.com/docs/en/agents-and-tools/tool-use/build-a-tool-using-agent](https://platform.claude.com/docs/en/agents-and-tools/tool-use/build-a-tool-using-agent) · 来源: web · 生成时间: 2026-09-20T03:34:34.346478+00:00*

## 背景

大语言模型本质是文本生成器，无法直接查询数据库、调用接口或执行代码。为了让模型在对话中完成实际业务动作，Claude 等平台引入工具调用协议：模型输出结构化的“调用意图”，由开发者在外部执行后再把结果回传。这解决了实时数据获取、副作用操作和复杂多步任务问题。

## 痛点

没有工具使用能力，模型只能生成文本，无法查询实时数据、调用企业内部 API 或执行操作，容易编造答案。开发者若自己解析模型文本来调用工具，格式不稳定、容易出错。不懂工具调用的循环协议，往往会只调一次工具就结束，无法完成多步推理。

## 解决办法

核心做法是通过 JSON Schema 定义工具的名称、描述和输入参数，随请求一起传给 Claude。模型在需要时返回 stop_reason=tool_use 和结构化的 tool_use 块，其中包含唯一 id 和参数 input。开发者执行真实函数后，把结果以 tool_result 块回传，并再次发起请求，让模型基于结果继续推理。循环这个过程，直到 stop_reason=end_turn，模型生成最终回答。可以把工具理解为一个“可调用的函数目录”，模型只负责决策调用哪个函数和传什么参数，真正执行始终由开发者控制。

## 关键代码示例

```python
import anthropic

client = anthropic.Anthropic()

def get_weather(city: str) -> str:
    return f"{city}: 25°C, sunny"

tools = [{
    "name": "get_weather",
    "description": "Get current weather for a city",
    "input_schema": {
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"]
    }
}]

messages = [{"role": "user", "content": "What's the weather in Paris?"}]

while True:
    resp = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=1024,
        tools=tools,
        messages=messages
    )
    if resp.stop_reason == "tool_use":
        messages.append({"role": "assistant", "content": resp.content})
        for block in resp.content:
            if block.type == "tool_use":
                result = get_weather(**block.input)
                messages.append({
                    "role": "user",
                    "content": [{
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    }]
                })
    else:
        print(resp.content[0].text)
        break
```

代码先定义 get_weather 工具及其 JSON Schema，让模型知道何时调用、需要哪些参数。循环中每次调用 client.messages.create 时都带上 tools 和完整 messages 历史，保证模型不丢失上下文。当 stop_reason 为 tool_use 时，遍历返回的 tool_use 块，根据 block.input 调用真实函数，并把结果以 tool_result 形式追加为 user 消息。这样模型在下一轮能看到工具结果并生成最终回答。

## 关键流程

1. 定义工具列表：为每个工具提供 name、description 和 input_schema。
2. 把 tools 参数随 messages 一起传给 Claude Messages API。
3. 检查响应的 stop_reason，判断是否等于 tool_use。
4. 从 content 中提取 tool_use 块，按 block.input 执行真实的业务函数。
5. 将执行结果封装为 tool_result 块，并关联原 tool_use_id，追加到消息历史。
6. 再次发起请求，循环直到 stop_reason 为 end_turn，然后输出最终文本。

## 关键点

- 工具 schema 的 name、description 和 input_schema 是模型决策的核心依据，描述模糊会直接导致误调用或参数错误。
- 模型只返回工具调用意图，不执行任何函数，真正的执行和副作用必须由开发者在外部完成。
- tool_use_id 是关联工具调用与结果的唯一标识，回传 tool_result 时必须准确对应，否则模型无法正确理解结果。
- 完整的消息历史必须保留 assistant 的 tool_use 内容和 user 的 tool_result 内容，这是多轮工具调用能继续推理的前提。
- 要显式处理 stop_reason，区分 tool_use 和 end_turn，并在循环中设置最大次数或退出条件，避免无限调用。
- 工具执行属于高风险操作，开发者需要对模型传入的参数做校验和权限控制，不能直接信任模型输入。

## 对比与权衡

- 相比 OpenAI 的 function calling，Claude 的工具调用使用 content 中的 tool_use/tool_result 内容块而非独立的 tool_calls 字段，且用 stop_reason=tool_use 明确标识；两者核心思想一致，但迁移时要注意消息结构差异。
- 相比手写 ReAct 提示让模型输出 JSON 再自行解析，Claude 原生工具调用在结构可靠性上更好，能保证参数尽量符合 schema，但需要严格按平台协议回传 tool_result，灵活性略低。

## 自测问题

**问: 请描述 Claude 工具调用的完整流程**

先定义工具 schema，再把 tools 传入 Messages API；模型返回 stop_reason=tool_use 和 tool_use 块；开发者执行函数后以 tool_result 回传；循环直到 end_turn。要强调模型不执行工具，只发出调用意图。

**问: 如果模型一次返回多个 tool_use 块，应该怎么处理？**

这是并行工具调用。可以按顺序或并发执行每个工具，然后为每个 tool_use_id 分别构造对应的 tool_result 块，一次性作为 user 消息回传，让模型同时获取所有结果。

**问: 如何避免工具调用陷入无限循环？**

设置最大迭代次数、在每轮检查 stop_reason、对连续重复调用做熔断；还可以在系统提示中要求模型在获得足够信息后直接回答，并在最终没有 tool_use 时退出。

**问: 工具 schema 设计有哪些最佳实践？**

name 要清晰动词化，description 要说明用途和使用时机，input_schema 尽量精确并限制 required 字段；每个工具只做一件事，避免参数过于复杂，因为模型对工具的准确选择高度依赖这些描述。

**问: Claude 工具调用和 MCP 有什么关系？**

MCP 是工具接入的标准化协议，可以把外部工具封装成 MCP server 供模型调用；而 tool use 是 Claude 模型侧的具体调用机制。MCP 解决的是“工具怎么连接”，tool use 解决的是“模型怎么决定调用”和“结果怎么回传”。

## 适用场景

- 实时信息查询：天气、股价、汇率等模型训练数据中不存在或会过期的信息。
- 企业数据检索：根据用户问题查询订单、CRM、知识库等内部系统数据。
- 触发业务操作：创建工单、发送邮件、写入数据库等有副作用的流程。
- 复杂多步 Agent：先查数据，再根据结果执行计算或调用另一个工具，最后生成综合回答。

## 标签

`Claude` `Tool Use` `Function Calling` `Agent` `Anthropic API`
